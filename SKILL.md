---
name: dokploy-cloudflared-cgnat
description: "Expose self-hosted web apps that run on a home/office machine behind CGNAT (e.g. Jio/Airtel residential broadband) to the public internet, with no port-forwarding and no public IP, by pairing a Dokploy PaaS install with a Cloudflare Tunnel. Use when the user wants to host apps on a local Ubuntu server/laptop behind CGNAT, map custom domains/subdomains to those apps, route everything through one tunnel, and front it with Cloudflare TLS. Covers the full install, per-app and multi-domain routing loop, the memory/RAM constraints of small boxes, and the common 1033/502/redirect-loop failure modes."
license: MIT
metadata:
  version: "1.0"
  author: yash
---

# Dokploy + Cloudflare Tunnel behind CGNAT

Stand up a self-hosted PaaS (Dokploy) on a machine that has **no public IP** because it sits
behind CGNAT, and make its apps reachable on real domains using a **Cloudflare Tunnel**. The
tunnel dials **outbound** from the box to Cloudflare's edge, so CGNAT (which only blocks inbound)
never matters and no router port-forwarding is required.

Use this skill when the user wants to:

- host web apps on a local Ubuntu server / laptop that is behind CGNAT (Jio, Airtel, most Indian
  residential broadband, campus/office NAT);
- reach those apps on `app.theirdomain.com` or on entirely separate domains;
- front everything with Cloudflare (free TLS, DDoS, stable hostname);
- avoid a VPS being in the data path (an Oracle/other VM is optional, not required).

## Mental model (say this first, it resolves the usual confusion)

CGNAT blocks **inbound** connections — nobody can open a socket *to* the box. Cloudflare Tunnel
never needs one: `cloudflared` on the box holds an **outbound** connection open to Cloudflare,
and public traffic flows back down that pipe. This is the canonical CGNAT-bypass; it is NOT
defeated by CGNAT. Only port-forwarding, a public static IP, and "outside world opens a fresh
connection to the box" are impossible behind CGNAT — none of which this setup uses.

Traffic flow:

```
browser → https://app.domain.com
  → Cloudflare edge (TLS terminates here)
  → Cloudflare Tunnel (OUTBOUND from the box — beats CGNAT)
  → cloudflared → http://localhost:80 (Dokploy's Traefik)
  → Traefik matches the Host header → the app's container
```

Only the Dokploy dashboard points at `localhost:3000`. **Every app points at `localhost:80`**
(Traefik), which routes by Host header. That is why new apps need no tunnel change beyond one
ingress line + one DNS record.

## Where an Oracle/other VM fits

Not required. Options:
- **Default:** no VM in the path. Cloudflare Tunnel connects the box straight to Cloudflare.
- **Redundancy:** run a second `cloudflared` replica of the same tunnel on the VM for failover.
- **Public gateway variant (only if raw TCP/UDP or a self-owned public IP is required):** WireGuard
  from the box out to the VM, VM reverse-proxies in. More setup, a public VM to secure. Steer the
  user to the default Cloudflare path unless they need non-HTTP or a dedicated IP.

## Prerequisites

- A domain on Cloudflare's nameservers (free plan). The zone must show **Active**.
- Ubuntu server/laptop reachable over SSH on the LAN.
- Small boxes (≈8GB RAM): add swap and expect to run only a couple of heavy stacks. See Constraints.

## Procedure

### 1. Prepare the box (once)

```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname dokploy-node

# swap — critical on 8GB; Docker/Next.js builds spike memory and OOM-kill otherwise
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf && sudo sysctl --system

# laptop: stop suspend on lid close. NOTE the path is /etc/systemd/logind.conf
sudo sed -i 's/^#\?HandleLidSwitch=.*/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
sudo sed -i 's/^#\?HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf
sudo sed -i 's/^#\?HandleLidSwitchDocked=.*/HandleLidSwitchDocked=ignore/' /etc/systemd/logind.conf
sudo systemctl restart systemd-logind
```

### 2. Install Dokploy

Dokploy's installer needs a REAL root shell — `| sh` under a sudo user fails with
"This script must be run as root". Pipe through `sudo sh`:

```bash
curl -sSL https://dokploy.com/install.sh | sudo sh
```

It installs Docker and runs Traefik + Postgres + Redis + the UI. Confirm the three ports are up
(all via `docker-proxy`):

```bash
sudo ss -tlnp | grep -E ':(80|443|3000)\b'   # 3000=dashboard, 80/443=Traefik
curl -I http://localhost:3000                  # expect 200/302
```

Open `http://<LAN-IP>:3000` and create the admin account **before** exposing anything.

### 3. Install + authenticate cloudflared

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared

cloudflared tunnel login   # open the printed URL in any browser, pick the domain
```

### 4. Create the tunnel + stage credentials for root

```bash
cloudflared tunnel create dokploy
cloudflared tunnel list      # note the UUID

sudo mkdir -p /root/.cloudflared
sudo cp ~/.cloudflared/<UUID>.json /root/.cloudflared/
sudo cp ~/.cloudflared/cert.pem   /root/.cloudflared/
```

### 5. Write the ingress config

`/etc/cloudflared/config.yml` — one `hostname` block per thing exposed. Apps → `localhost:80`,
dashboard → `localhost:3000`, catch-all last:

```yaml
tunnel: <UUID>
credentials-file: /root/.cloudflared/<UUID>.json

ingress:
  - hostname: dokploy.domain.com
    service: http://localhost:3000
  - hostname: app1.domain.com
    service: http://localhost:80
  - service: http_status:404      # MUST be last
```

```bash
cloudflared tunnel ingress validate
```

### 6. DNS records (one per hostname)

```bash
cloudflared tunnel route dns dokploy dokploy.domain.com
cloudflared tunnel route dns dokploy app1.domain.com
dig +short app1.domain.com        # expect Cloudflare IPs (104.x / 172.x) = proxied
```

### 7. Run as a service

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
cloudflared tunnel info dokploy   # expect 2-4 active edge connections
```

### 8. Cloudflare SSL mode → Full

Dashboard → SSL/TLS → Overview → **Full** (NOT Flexible → redirect loops; NOT Full-Strict →
needs a valid origin cert the tunnel doesn't present).

### 9. Attach the domain inside Dokploy (per app)

App → **Domains** → Add Domain:
- Host: the exact subdomain (must match an ingress hostname).
- Container Port: the port the app listens on **inside its container** (distinct from the
  `localhost:80` the tunnel talks to Traefik on).
- **HTTPS: OFF, Certificate: None.** Cloudflare terminates TLS at the edge; a per-app Let's Encrypt
  cert would fail (ACME HTTP-01 can't reach port 80 through the tunnel) and is redundant.

## The repeatable "add an app" loop

1. Add an ingress block in `/etc/cloudflared/config.yml` → `sudo systemctl restart cloudflared`
2. `cloudflared tunnel route dns dokploy newapp.domain.com`
3. Dokploy → app → Domains → add `newapp.domain.com`, set container port, HTTPS off.

## Multiple domains

One tunnel serves any number of hostnames across any number of domains — the `ingress:` list just
grows. For a **different** domain (not the first one): add it to the same Cloudflare account, set
its nameservers at the registrar, wait for **Active**, then re-run `cloudflared tunnel login` and
also select the new zone (updates `cert.pem`) so `route dns` is authorized. Apex/root domains route
fine via `route dns`; remove any pre-existing conflicting `A`/`AAAA`/`CNAME` on the apex first.

## Constraints (small boxes)

- Dokploy's own stack idles at ~1–1.5GB. On 8GB that leaves ~6GB for apps. Run one or two heavy
  stacks, not everything at once. Swap is an OOM safety net, not runtime RAM.
- Heavy multi-container apps (NestJS + Next.js + Postgres + Redis) deploy as a Dokploy **Compose**
  app to preserve service wiring. Build memory can OOM a small box mid-build — build elsewhere and
  deploy the prebuilt image, or build with swap present.
- WhatsApp-automation / headless-browser / long-lived-session workers (Baileys/openwa, Electron
  desktop apps) are NOT a fit — not a tunnel limit, they need a persistent logged-in session; keep
  those local. Only the stateless request/response web half belongs here.

## Security

- Behind the tunnel you do NOT open 80/443 to the LAN/world. Firewall: allow SSH from LAN only;
  the tunnel reaches Traefik over loopback.
- Behind Cloudflare, apps serve plain HTTP internally — do NOT run a second internal nginx TLS
  terminator; point the tunnel at the plain-HTTP service that also proxies `/api`. Pointing at a
  static-only service that lacks the `/api` proxy causes `POST /api/... → 405` (a known footgun).
- Set any admin password / session secret as a Dokploy **env var**, never in the repo. Apps whose
  dev-mode auth falls back to an admin role when a password var is unset become publicly exposed if
  that var is missing.
- First-boot migrate/seed: compose apps that migrate on boot need a fresh DB volume to come up
  usable; verify a fresh-volume boot yields an authorizing app, not a locked one.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Cloudflare **1033** | hostname not in tunnel ingress, or cloudflared not running | add ingress block; `systemctl status cloudflared` |
| **502 Bad Gateway** | reached Traefik but no app claims that Host | fix the Dokploy Domains tab; confirm container is running |
| **Too many redirects** | SSL mode is Flexible | set Cloudflare SSL → Full |
| `dig` returns non-Cloudflare IP / nothing | record is grey-cloud (not proxied) or missing | re-run `route dns`; turn the orange cloud on |
| Installer "must be run as root" | piped under a sudo user | `curl ... | sudo sh` |
| `sed: can't read /etc/logind.conf` | wrong path | it's `/etc/systemd/logind.conf` |
| login `405` on `/api/...` | tunnel points at a static-only service with no `/api` proxy | point ingress at the service that proxies `/api` to the backend |

## Verify end-to-end

```bash
cloudflared tunnel info dokploy      # active edge connections
dig +short app1.domain.com           # Cloudflare IPs
curl -I https://app1.domain.com      # 200/302 through the whole chain
```
