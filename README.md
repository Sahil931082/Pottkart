# Pottkart  — E-commerce Platform

An e-commerce web app for buying/selling pottery items, built for real-world use
(Gurgaon, India), self-hosted on a home server.

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | Next.js (React) + Tailwind CSS | SSR/SSG for SEO, scales to CDN/edge later with no rewrite |
| Backend | Node.js + Express (or NestJS as it grows) | Matches existing MERN experience, stateless = easy to scale horizontally |
| Database | PostgreSQL | Orders/inventory/payments are relational; avoids join/consistency pain of NoSQL at checkout time |
| Cache/Sessions | Redis | Cart state, rate limiting, query caching |
| Search | Meilisearch | Fast, typo-tolerant product search, self-hostable |
| Payments | Razorpay | India-first, UPI support, PCI compliance handled for you |
| Auth | NextAuth.js / Lucia | Session + credential/social auth |
| Infra | Docker Compose → Kubernetes/Traefik later | Same containers scale out without app rewrite |
| Reverse proxy | Nginx | TLS termination, routing, basic rate limiting |
| Tunneling | Cloudflare Tunnel | Expose home server without opening router ports or leaking home IP |

## Architecture

```
                    ┌────────────────┐
   Internet ──────► │  Cloudflare    │  (CDN, DDoS protection, Tunnel)
                    └───────┬────────┘
                            │
                    ┌───────▼────────┐
                    │     Nginx      │  (reverse proxy, rate limiting)
                    └───┬───────┬────┘
                        │       │
              ┌─────────▼─┐   ┌─▼──────────┐
              │  Next.js  │   │  Express    │
              │ (frontend)│   │  (backend)  │
              └───────────┘   └──┬───┬──────┘
                                 │   │
                       ┌─────────▼┐ ┌▼──────────┐
                       │ Postgres │ │   Redis    │
                       └──────────┘ └────────────┘
                            │
                    ┌───────▼────────┐
                    │  Meilisearch   │
                    └────────────────┘
```

## Repo Structure

```
pottery-store/
├── docker-compose.yml
├── .env.example
├── nginx/
│   └── nginx.conf
├── backend/
│   ├── Dockerfile
│   └── src/            # Express app, routes, models, controllers
├── frontend/
│   ├── Dockerfile
│   └── src/            # Next.js app
└── scripts/             # backup scripts, DB migrations, etc.
```

## Local / Home Server Setup

### Prerequisites
- Docker + Docker Compose installed
- Node.js 20+ (for local dev outside containers)
- A domain name (for Cloudflare Tunnel + SSL)
- A Cloudflare account (free tier is fine)
- Razorpay account (test mode keys to start)

### Steps

1. **Clone and configure**
   ```bash
   git clone <your-repo-url>
   cd pottery-store
   cp .env.example .env
   # Edit .env — fill in real secrets, generate strong passwords
   ```

2. **Generate strong secrets** (don't use placeholder values in production)
   ```bash
   openssl rand -base64 32   # use for NEXTAUTH_SECRET, MEILI_MASTER_KEY, etc.
   ```

3. **Bring up the stack**
   ```bash
   docker compose up -d --build
   docker compose ps        # confirm all services are healthy
   ```

4. **Expose to the internet safely (recommended: Cloudflare Tunnel)**

   Instead of port-forwarding 80/443 on your router (which exposes your home IP),
   install `cloudflared` on the host and create a tunnel:
   ```bash
   cloudflared tunnel login
   cloudflared tunnel create pottery-store
   cloudflared tunnel route dns pottery-store yourdomain.com
   cloudflared tunnel run pottery-store
   ```
   Point the tunnel's local service at `http://localhost:80` (your Nginx container).
   This gets you free SSL, DDoS protection, and no exposed router ports.

5. **Backups (critical for a home-hosted production DB)**
   Add a cron job (see `scripts/backup.sh`) that dumps Postgres and syncs to
   offsite storage (e.g. Backblaze B2) daily.

6. **Monitoring**
   Run [Uptime Kuma](https://github.com/louislam/uptime-kuma) as an additional
   container or on a separate cheap always-on device, to get alerted if your
   home server or internet goes down.

## Security Checklist (before going "live" with real customers)
- [ ] All `.env` secrets are strong, unique, not committed to git
- [ ] Postgres/Redis ports bound to `127.0.0.1` only (already set in compose file)
- [ ] Razorpay webhook signature verification implemented on backend
- [ ] Rate limiting active on Nginx and/or backend
- [ ] Automated offsite backups running and tested (restore, not just backup!)
- [ ] UPS in place for the home server
- [ ] HTTPS enforced everywhere (handled by Cloudflare Tunnel)
- [ ] No raw card data ever touches your own server (Razorpay handles this)

## Roadmap
- [ ] Phase 1: Product catalog, cart, checkout, Razorpay integration
- [ ] Phase 2: Order management + basic admin panel
- [ ] Phase 3: Search (Meilisearch), reviews, wishlist
- [ ] Phase 4: Analytics, recommendations (future: Python/scikit-learn service)

## License
Private / proprietary — internal use.
