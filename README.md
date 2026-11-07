# Dokploy + Cloudflare Tunnel behind CGNAT — Setup Guide

Host your own web apps on a machine that sits **behind CGNAT** (Jio / Airtel / most Indian
residential broadband, campus or office NAT) — **no public IP, no port-forwarding** — and reach
them on real domains with free TLS.

> **This setup fully supports CGNAT connections like Jio in India — no public IP required.**
> A Cloudflare Tunnel dials outbound from your box, so a carrier-grade NAT with no inbound
> reachability and no static IP is not a blocker. If you can browse the web from the box, this works.

- **Dokploy** — a self-hosted PaaS (Heroku/Vercel-style) that runs Docker apps with a web UI and
  a built-in Traefik reverse proxy.
- **Cloudflare Tunnel (`cloudflared`)** — an agent on your box that dials **outbound** to
  Cloudflare and holds the connection open. Public traffic rides back down that pipe. Because the
  connection is outbound, **CGNAT does not block it** — this is the whole point.

> **CGNAT myth, busted.** CGNAT only blocks *inbound* connections. This setup never needs one.
> It is the standard way to self-host from behind CGNAT. The only things CGNAT truly prevents —
> router port-forwarding, a public static IP, the outside world opening a fresh socket to your box
> — are not used here.

---

## Architecture

```
browser → https://app.yourdomain.com
   → Cloudflare edge            (TLS terminates here; DDoS protection)
   → Cloudflare Tunnel          (OUTBOUND from your box — beats CGNAT)
   → cloudflared → localhost:80 (Dokploy's Traefik reverse proxy)
   → Traefik routes by Host header → your app's container
```

- The **dashboard** is the only thing pointed at `localhost:3000`.
- **Every app** is pointed at `localhost:80` (Traefik). Traefik tells apps apart by the `Host`
  header, which you set per-app in Dokploy. That is why adding an app needs just one ingress line
  and one DNS record — never a new tunnel.

An Oracle/other VM is **optional**: default has no VM in the path. Use one only for tunnel
redundancy, or as a public gateway if you need raw TCP/UDP or a self-owned IP (WireGuard variant).

---

## Prerequisites

1. A domain on **Cloudflare** (free plan). Set the two Cloudflare nameservers at your registrar and
   wait until the zone shows **Active**.
2. An Ubuntu server/laptop reachable over SSH on your LAN.
3. On an **8GB** box: add swap (below) and plan to run only one or two heavy stacks at a time.

---

## Step-by-step

### 1. Prepare the box

```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname dokploy-node

# Swap — important on 8GB. Docker/Next.js builds spike memory; without swap they get OOM-killed.
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf && sudo sysctl --system
```

Laptop only — stop it suspending when the lid closes. The file is `/etc/systemd/logind.conf`
(not `/etc/logind.conf`):

```bash
sudo sed -i 's/^#\?HandleLidSwitch=.*/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
sudo sed -i 's/^#\?HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf
sudo sed -i 's/^#\?HandleLidSwitchDocked=.*/HandleLidSwitchDocked=ignore/' /etc/systemd/logind.conf
sudo systemctl restart systemd-logind
grep -E 'HandleLidSwitch' /etc/systemd/logind.conf   # verify all three = ignore, uncommented
```

### 2. Install Dokploy

The installer needs a **real root shell**. Running `| sh` as a normal sudo user fails with
*"This script must be run as root"*. Pipe through `sudo sh`:

```bash
curl -sSL https://dokploy.com/install.sh | sudo sh
```

Verify the ports came up (all via `docker-proxy`), then that the dashboard answers:

```bash
sudo ss -tlnp | grep -E ':(80|443|3000)\b'   # 3000 = dashboard, 80/443 = Traefik
curl -I http://localhost:3000                  # expect HTTP 200 or 302
```

Open `http://<LAN-IP>:3000` and create the admin account **before** exposing anything publicly.

### 3. Install and authenticate cloudflared

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared
cloudflared --version

cloudflared tunnel login       # copy the printed URL into any browser, pick your domain
ls -l ~/.cloudflared/cert.pem  # confirm the cert was written
```

### 4. Create the tunnel and stage credentials for root

```bash
cloudflared tunnel create dokploy
cloudflared tunnel list        # note the UUID

sudo mkdir -p /root/.cloudflared
sudo cp ~/.cloudflared/<UUID>.json /root/.cloudflared/
sudo cp ~/.cloudflared/cert.pem   /root/.cloudflared/
```

### 5. Write the tunnel config

`/etc/cloudflared/config.yml` — one block per hostname. Apps → `localhost:80`, dashboard →
`localhost:3000`, catch-all last:

```yaml
tunnel: <UUID>
credentials-file: /root/.cloudflared/<UUID>.json

ingress:
  - hostname: dokploy.yourdomain.com
    service: http://localhost:3000
  - hostname: app1.yourdomain.com
    service: http://localhost:80
  - service: http_status:404      # MUST be last
```

```bash
cloudflared tunnel ingress validate
```

### 6. Create DNS records (one per hostname)

```bash
cloudflared tunnel route dns dokploy dokploy.yourdomain.com
cloudflared tunnel route dns dokploy app1.yourdomain.com
dig +short app1.yourdomain.com    # expect Cloudflare IPs (104.x / 172.x) → proxied
```

### 7. Run cloudflared as a service

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
cloudflared tunnel info dokploy   # expect 2–4 active edge connections
```

`https://dokploy.yourdomain.com` should now load the dashboard over HTTPS.

### 8. Set Cloudflare SSL mode → Full

Cloudflare dashboard → **SSL/TLS → Overview → Full**. Not *Flexible* (redirect loops), not
*Full (strict)* (needs a valid origin cert the tunnel doesn't present).

### 9. Point a domain at an app in Dokploy

App → **Domains** → Add Domain:

- **Host:** the exact subdomain (must match an ingress hostname).
- **Container Port:** the port the app listens on **inside its container** — this is a different
  field from the `localhost:80` the tunnel uses to reach Traefik.
- **HTTPS: OFF, Certificate: None.** Cloudflare already handles TLS; a per-app Let's Encrypt cert
  would fail (ACME can't reach port 80 through the tunnel) and is redundant.

---

## Quick test before real apps

Deploy a tiny throwaway to prove the whole chain:

1. Dokploy → Create Service → Application → **Docker image** `nginxdemos/hello`.
2. Domains tab → host `test.yourdomain.com`, container port `80`, HTTPS off.
3. Add ingress + DNS:
   ```bash
   # add under ingress, above the catch-all:
   #   - hostname: test.yourdomain.com
   #     service: http://localhost:80
   sudo systemctl restart cloudflared
   cloudflared tunnel route dns dokploy test.yourdomain.com
   ```
4. Open `https://test.yourdomain.com` — the nginx "Hello" page confirms the full path works.

---

## Adding more apps

Three small edits each time:

1. Add an ingress block → `sudo systemctl restart cloudflared`
2. `cloudflared tunnel route dns dokploy newapp.yourdomain.com`
3. Dokploy → app → Domains → add the hostname, set the container port, HTTPS off.

## Adding a different domain

One tunnel serves any number of domains. For a domain other than your first:

1. Add it to the **same Cloudflare account**, set its nameservers at the registrar, wait for Active.
2. `cloudflared tunnel login` again and also pick the new zone (updates `cert.pem`), so `route dns`
   is authorized for it.
3. Then the normal add-app loop. Apex/root domains route fine via `route dns`; remove any existing
   conflicting `A`/`AAAA`/`CNAME` on the apex first.

---

## Access control — not everything should be public

**The tunnel solves *reachability*, not *authorization*.** By default, every hostname you add to the
ingress is reachable by the entire internet — including your Dokploy dashboard and any internal
admin apps. Removing the network boundary (ports, public IP) means **authentication is now your only
boundary**. A public tunnel hostname in front of a weak or default-admin app is as exposed as port
forwarding would have been — arguably more so, because it has a clean Cloudflare TLS front and looks
legitimate.

**Principle: reachable ≠ should-be-public. Classify every hostname, and gate the ones that aren't
meant for everyone.**

| Hostname | Example | Exposure |
|---|---|---|
| Public | landing page, guest RSVP form, marketing site | open |
| Protected | `dokploy.*` dashboard, admin panels, internal tools, staging | behind auth |

### Put protected hostnames behind Cloudflare Access (Zero Trust)

[Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/policies/access/) is free up to
50 users and enforces identity **before** a request ever reaches your box, so an attacker can't even
probe the app:

1. In the Cloudflare dashboard, enable **Zero Trust** (one-time setup, free plan).
2. **Zero Trust → Access → Applications → Add an application → Self-hosted.**
3. Set the **Application domain** to the hostname to protect, e.g. `dokploy.yourdomain.com`.
4. Add a **policy**: Action **Allow**, with a rule such as *Emails* = your address, or *Emails ending
   in* = your domain. Everyone else is blocked at Cloudflare's edge.
5. Pick a login method (one-time PIN to email works with zero IdP setup; Google/GitHub/etc. also
   supported). Save.

Now `dokploy.yourdomain.com` shows a Cloudflare login screen first; only approved identities reach
Dokploy. **Do this for the dashboard at minimum** — it's the highest-value target on the box.

### Machine-to-machine (webhooks, APIs)

For hostnames that are hit by software, not people (incoming webhooks, API callers), don't use an
interactive login — use an Access **service token** (a client ID/secret pair the caller sends as
headers), or restrict by `cloudflared` + mTLS. Keep interactive Access for human-facing apps.

### Defense in depth

Access is the clean front gate, but keep the app's own auth hardened too — in particular the repo's
warning below about dev-mode auth falling back to an admin role when a password env var is unset.
Behind a public tunnel that's a takeover risk; behind Access it becomes a second layer, not the only
one.

---

## Resource notes (8GB box)

- Dokploy idles at ~1–1.5GB → ~6GB left for your apps. Run one or two heavy stacks, not all at once.
- Multi-container apps (NestJS + Next.js + Postgres + Redis) deploy as a Dokploy **Compose** app.
  Build memory can OOM a small box — build the image elsewhere and deploy prebuilt, or build with
  swap present.
- WhatsApp-automation / headless-browser / Electron-desktop workers are **not** a fit — they need a
  persistent logged-in session. Keep those local; host only the stateless request/response web half.

## Security notes

- Don't open 80/443 to the LAN/world — the tunnel reaches Traefik over loopback. Firewall: allow
  SSH from LAN only.
- Don't run a second internal nginx TLS terminator behind Cloudflare. Point the tunnel at the
  plain-HTTP service that also proxies `/api`. Pointing at a static-only service with no `/api`
  proxy causes `POST /api/... → 405`.
- Put admin passwords / session secrets in Dokploy **env vars**, never in the repo. If an app's
  auth falls back to an admin role when its password var is unset, a missing var = public exposure.
- **Gate non-public hostnames with Cloudflare Access** (see *Access control* above). The dashboard
  and any admin/internal app should never be openly reachable just because the tunnel can route to
  them.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Cloudflare **1033** | hostname missing from tunnel ingress, or cloudflared not running | add the ingress block; check `systemctl status cloudflared` |
| **502 Bad Gateway** | reached Traefik but no app claims that Host | fix the Dokploy Domains tab; confirm the container is running |
| **Too many redirects** | Cloudflare SSL = Flexible | set SSL → Full |
| `dig` → non-Cloudflare IP / nothing | record is grey-cloud (not proxied) / missing | re-run `route dns`; enable the orange cloud |
| Installer: *"must be run as root"* | piped under a sudo user | `curl ... | sudo sh` |
| `sed: can't read /etc/logind.conf` | wrong path | use `/etc/systemd/logind.conf` |
| login `405` on `/api/...` | tunnel points at a static-only service | point ingress at the service that proxies `/api` |

## Verify end-to-end

```bash
cloudflared tunnel info dokploy      # active edge connections
dig +short app1.yourdomain.com       # Cloudflare IPs
curl -I https://app1.yourdomain.com  # 200/302 through the whole chain
```
