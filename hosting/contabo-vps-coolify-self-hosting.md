# Running Coolify on a Budget Contabo VPS

## The Context

I wanted a personal PaaS for side projects without the metered burn of Render, Railway, or Fly.io. Contabo's VPS pricing (4 vCPU, 8 GB RAM, 200 GB SSD for ~€6/month) is hard to beat, and Coolify gives a Heroku-like deployment experience entirely self-hosted. Today I wired the two together.

## The Approach

### 1. VPS Provisioning

Ordered a **Cloud VPS S** from Contabo with Ubuntu 22.04. Key first steps after SSHing in:

```bash
# Update and install basics
apt update && apt upgrade -y
apt install -y curl git ufw

# Harden SSH: disable root password login, move to key-only
sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd

# Firewall: only 22, 80, 443, and Coolify's default port
ufw default deny incoming
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 8000/tcp   # Coolify web UI
ufw enable
```

> **Note:** Contabo instances live in Germany, the US, or Singapore. Pick the region closest to your users; latency to South America from Germany is ~200 ms.

### 2. Coolify Installation

Coolify provides a one-line installer that sets up Docker and pulls the orchestration stack:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

The script:
1. Installs Docker Engine + Compose plugin
2. Creates `/data/coolify` as the persistent volume root
3. Spins up Coolify's core containers (UI, scheduler, workers)

After install, the web UI is available at `http://<vps-ip>:8000`. I completed onboarding and immediately:
- Changed the default admin password.
- Added a **Server** (the local instance) and verified Docker connectivity.
- Enabled **Auto Update** so patch releases apply automatically.

### 3. Domain & TLS with Let's Encrypt

Coolify can request and renew Let's Encrypt certificates automatically, but it needs a domain pointed at the VPS first.

1. Created an `A` record (`coolify.mydomain.com` → VPS IP).
2. In Coolify → **Settings** → **General**, set the instance domain to `coolify.mydomain.com`.
3. Under **Servers** → **localhost** → **Proxy**, confirmed Traefik is the default reverse proxy.
4. Added a test project + service; Coolify requested the cert on first deploy and redirected HTTP → HTTPS.

### 4. First Application Deploy

Pushed a simple Node.js API to GitHub and added it as a **Private GitHub App** source in Coolify. Created a resource with:

- **Build Pack:** Nixpacks (zero-config Node builder)
- **Port:** `3000`
- **Domain:** `api.mydomain.com`
- **Environment Variables:** injected via the UI, encrypted at rest

Hit **Deploy**. Coolify cloned the repo, built the image, and Traefik routed `api.mydomain.com` to the container. Total time from git push to live URL: ~45 seconds.

## Lessons Learned

- **Contabo's "unmetered" bandwidth is real, but CPU is burstable.** Sustained builds peg the vCPUs. For heavier CI-like workloads, bump to the next tier or add a GitHub Actions runner for builds and only ship artifacts to the VPS.
- **Coolify's Nixpacks support is a killer feature.** I didn't write a single line of Dockerfile boilerplate for Node, Python, or static sites. When I needed custom system dependencies, switching to a raw Dockerfile inside the repo was a one-click toggle.
- **Backup the `/data/coolify` directory.** Everything—DBs, service definitions, uploaded env files—lives there. A nightly `rsync` to S3 or another VPS is cheap insurance.
- **Don't expose port 8000 to the public long-term.** After onboarding, I locked the Coolify UI to `localhost:8000` and tunneled in via SSH (`ssh -L 8000:localhost:8000 user@vps`) for day-to-day management. Keeps the attack surface tiny.

## Key Learning

Self-hosting a PaaS on a budget VPS trades **managed convenience** for **control and predictability**. The real win isn't just the lower monthly bill—it's that every deployed service runs on the same Docker network, so inter-service communication (e.g., an API talking to a self-hosted Postgres or Redis) is latency-free and bandwidth-free. Coolify abstracts the ops; Contabo provides the metal. Together they're a powerful combo for indie hackers and small teams.
