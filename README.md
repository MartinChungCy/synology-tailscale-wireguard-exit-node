> **Disclaimer:** This is a personal, unofficial guide, not affiliated with Synology, Tailscale, Surfshark or gluetun. It worked on one setup (a Synology NAS running DSM with Container Manager). Use it at your own risk and adapt it to your own network. Never commit real keys, passwords or auth keys.

# How to set up WireGuard + a Tailscale exit node on a Synology NAS with a dedicated IP

This guide records what worked on a Synology DSM NAS (Container Manager / Docker) to make a **Tailscale exit node** whose internet traffic leaves through a **Surfshark dedicated IP** over **WireGuard**.

Result: any device on your tailnet can pick `nas-exit` as its exit node and appear on the internet from the dedicated IP.

> **Secrets:** this guide uses placeholders. Never paste real keys, passwords or auth keys into chats, tickets or screenshots. Rotate anything you have already exposed.

---

## 1. How it fits together

```
Your Mac / phone
      │  Tailscale (WireGuard, 100.x addresses)
      ▼
 ts-exit container ──shares network with──▶ gluetun container
 (Tailscale exit node)                       (WireGuard client to Surfshark)
                                                   │ tun0
                                                   ▼
                                    Surfshark dedicated IP (e.g. Singapore)
                                                   ▼
                                                Internet
```

- **gluetun** connects the NAS to Surfshark with WireGuard.
- **ts-exit** (Tailscale) runs *inside gluetun's network* (`network_mode: service:gluetun`), so all exit-node traffic leaves through the VPN.
- Three small fixes make this work on a Synology (sections 4–6).

## 2. Requirements

- Synology NAS with **Container Manager** (Docker + Compose) and SSH access.
- A Surfshark **dedicated IP** that offers a **WireGuard** config (check your account's manual setup page; if only OpenVPN is offered, this guide's WireGuard part will not apply).
- A Tailscale account. In the admin console you will need to **approve the exit node** after it first advertises itself.

## 3. Get the WireGuard values from Surfshark

From your dedicated-IP WireGuard config note:

| Config line | Used as |
|---|---|
| `Endpoint = <host>:<port>` | `VPN_ENDPOINT_IP` (resolve the hostname to an **IP**), `VPN_ENDPOINT_PORT` |
| `[Peer] PublicKey` | `WIREGUARD_PUBLIC_KEY` |
| `[Interface] PrivateKey` | `WIREGUARD_PRIVATE_KEY` |
| `[Interface] Address` | `WIREGUARD_ADDRESSES` |

- gluetun needs an **IP address** for the endpoint, not a hostname. Resolve it with `nslookup <hostname>`.
- The port is normally **`51820`**. A swapped-digit typo (`51280`) leaves the tunnel up but failing its health check and restarting in a loop.

## 4. Create the helper files on the NAS

SSH into the NAS and create the project folder and the gluetun rules file. gluetun's firewall drops forwarded traffic by default, so this file allows Tailscale ⇄ VPN forwarding and masquerades outgoing traffic.

```bash
sudo mkdir -p /volume1/docker/vpn-gateway/iptables
cd /volume1/docker/vpn-gateway
sudo tee iptables/post-rules.txt >/dev/null <<'EOF'
iptables -I FORWARD -i tailscale0 -o tun0 -j ACCEPT
iptables -I FORWARD -i tun0 -o tailscale0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
EOF
```

## 5. The compose file

Create a project in Container Manager (path `/volume1/docker/vpn-gateway`) with this YAML. Put real values in your own copy only.

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    sysctls:
      - net.ipv4.ip_forward=1
    environment:
      - VPN_SERVICE_PROVIDER=custom
      - VPN_TYPE=wireguard
      - VPN_ENDPOINT_IP=<dedicated-server-IP>
      - VPN_ENDPOINT_PORT=51820
      - WIREGUARD_PUBLIC_KEY=<server public key>
      - WIREGUARD_PRIVATE_KEY=<your private key>
      - WIREGUARD_ADDRESSES=<your Address line, e.g. 10.14.0.2/16>
    volumes:
      - ./gluetun:/gluetun
      - ./iptables/post-rules.txt:/iptables/post-rules.txt
    restart: unless-stopped

  tailscale:
    image: tailscale/tailscale:latest
    container_name: ts-exit
    network_mode: service:gluetun
    depends_on:
      - gluetun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - TS_AUTHKEY=<first run only; remove after the node is registered>
      - TS_HOSTNAME=nas-exit
      - TS_EXTRA_ARGS=--advertise-exit-node --netfilter-mode=off
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
    volumes:
      - ./tailscale:/var/lib/tailscale
    restart: unless-stopped
```

Notes:
- **`--netfilter-mode=off`** is the key change for Synology (see problem 1 below). Without it Tailscale fails to configure its router and the exit node carries no traffic.
- The Tailscale login is saved in `./tailscale`, so `TS_AUTHKEY` is only needed the first time.
- Keep secrets in a `.env` file or remove them from files you share.

Start the project, then in the Tailscale admin console **approve `nas-exit` as an exit node**.

## 6. The routing fix (without it, replies are dropped)

gluetun's WireGuard policy rule (priority 101) sends everything into the VPN's table with a default route out `tun0`. That beats Tailscale's own rule (priority 5270), so replies addressed to your device's `100.x` address were sent back into the VPN instead of to Tailscale, and were dropped. Symptom: traffic leaves, nothing returns, "no internet" on the exit node.

Add a higher-priority rule that sends `100.64.0.0/10` to Tailscale's table (52):

```bash
sudo docker exec gluetun ip rule add to 100.64.0.0/10 lookup 52 priority 90
```

This vanishes whenever gluetun is recreated or restarted, so make it permanent.

### Make it permanent

Create a small script that re-adds the rule every 20 seconds:

```bash
sudo tee /volume1/docker/vpn-gateway/fix-rule.sh >/dev/null <<'EOF'
#!/bin/sh
while true; do
  docker exec gluetun sh -c 'ip rule | grep -q "to 100.64.0.0/10" || ip rule add to 100.64.0.0/10 lookup 52 priority 90' >/dev/null 2>&1
  sleep 20
done
EOF
sudo chmod +x /volume1/docker/vpn-gateway/fix-rule.sh
(sudo nohup /volume1/docker/vpn-gateway/fix-rule.sh >/dev/null 2>&1 &)
```

Start it at every boot: **Control Panel → Task Scheduler → Create → Triggered Task → User-defined script**, user `root`, event **Boot-up**, command:

```bash
nohup /volume1/docker/vpn-gateway/fix-rule.sh >/dev/null 2>&1 &
```

## 7. Verify

On the NAS:

```bash
# WireGuard is up and the endpoint is right
sudo docker logs gluetun 2>&1 | grep -i wireguard | tail -5

# The routing rule exists
sudo docker exec gluetun ip rule | grep 100.64

# Forwarding and masquerade rules loaded (use iptables-legacy on Synology)
sudo docker exec gluetun sh -c 'iptables-legacy -S FORWARD; iptables-legacy -t nat -S POSTROUTING'
```

On the client (turn off any other VPN app first):

1. Select `nas-exit` as the exit node in the Tailscale app.
2. `curl https://ifconfig.me` should print the dedicated IP.
3. Websites load, and DNS shows `100.100.100.100`.

## 8. Problems we hit and how they were solved

| Symptom | Cause | Fix |
|---|---|---|
| gluetun: `endpoint IP is not set` | WireGuard settings missing, endpoint is a hostname | Add `VPN_ENDPOINT_IP` (an IP), port, public key, private key, addresses; remove the OpenVPN lines |
| gluetun restarts, `failed to pass the healthcheck` | Wrong port (`51280` instead of `51820`) | Use the port from the config's `Endpoint` line |
| Exit node gives **no internet**; `ts-exit` log shows `Extension CONNMARK revision 0 not supported` and `router config failed` | Synology kernel has no CONNMARK module (`/lib/modules` has `xt_conntrack`/`xt_mark` but no way to load CONNMARK), so Tailscale cannot set up its firewall | Add `--netfilter-mode=off` and provide forwarding/masquerade rules through gluetun's `post-rules.txt` |
| Traffic leaves but replies are dropped (FORWARD policy dropped thousands of packets, return rule counted 0) | VPN routing rule beats Tailscale's | The `ip rule add to 100.64.0.0/10 lookup 52 priority 90` fix |
| `iptables` inside gluetun errors (`nf_tables ... Invalid argument`) | Kernel only supports legacy iptables | Use `iptables-legacy` for inspection |
| Container Manager "Stop" hangs; YAML is read-only while the project runs | Project state out of sync after several rebuilds | `cd /volume1/docker/vpn-gateway && sudo docker compose down`, edit the YAML, save and rebuild |
| Pinging the exit's `100.x` address fails | With netfilter off, the container's INPUT policy drops it | Expected; it does not affect exit-node traffic |

Tip for editing the YAML in the DSM web editor: it auto-closes quotes, and the horizontal scrollbar can swallow a click on the last visible line. Check each edited line after typing.

## 9. Performance (what to expect)

Measured from a Mac in Hong Kong with a dedicated IP in Singapore:

| Path | Download | Upload | Latency to Cloudflare |
|---|---|---|---|
| Surfshark app on the Mac, direct | ~23–25 MB/s | ~11 MB/s | ~0.05 s |
| Tailscale exit via the NAS (WireGuard) | ~6 MB/s | ~4 MB/s | ~0.12 s |
| Tailscale exit via the NAS (earlier OpenVPN) | ~4–7 MB/s | ~2–3 MB/s | ~0.12–0.2 s |

- The NAS did the VPN and Tailscale encryption in software ("userspace" WireGuard, because the Synology kernel has no WireGuard support), and its CPU peaked near 88% during tests. That is the likely limit.
- At home, using the Surfshark app on your computer directly is about four times faster. Use the NAS exit when you are away from home.
- Do not run another VPN (such as the Surfshark app) at the same time as the exit node when testing; they stack and distort results.

## 10. Housekeeping

- **Rotate secrets** that have been shared anywhere: the Surfshark credentials / WireGuard private key, and the Tailscale auth key (revoke it under Settings → Keys).
- Remove `TS_AUTHKEY` from the compose file once the node is registered.
- Keep `fix-rule.sh` running (boot task from section 6), or the exit node stops working after a NAS restart.
- This setup is a workaround for a missing kernel feature. A future Synology/DSM or Tailscale update could change the behaviour; re-check the `ts-exit` log (`router config failed`) if the exit stops working after an update.
