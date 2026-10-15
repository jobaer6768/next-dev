# Module 44: VPS Deployment Using Docker, NGINX and CI/CD Setup Using GitHub Actions

> Mission 5 — Ship Software to Production
> "From `localhost:3000` to `https://yourdomain.com`."

---

## 1. Why this module matters

Modules 38–43 got you to a fully working multi-container app on your laptop. Now we put it on a server, behind a domain name, with TLS, and we set up the automation that rebuilds and redeploys it every time you push to `main`.

This is the moment the SaaS becomes real. By the end of this lesson you will have:

- A running Ubuntu VPS (Hetzner, DigitalOcean, Vultr — pick one).
- Docker installed and configured for non-root use.
- An Nginx reverse proxy that terminates TLS and routes by hostname.
- A `docker-compose.prod.yml` that orchestrates the production stack.
- A GitHub Actions workflow that builds, scans, pushes, and deploys.
- Two conceptual sessions that anchor everything in real-world thinking.

You will also know how to read the logs when it breaks at 3am, because it will.

---

## 2. Learning objectives

- Explain why a reverse proxy is needed (TLS termination, host-based routing, single entrypoint).
- Write an Nginx config for HTTP→HTTPS redirect, TLS termination, and reverse-proxy to upstream services.
- Configure load balancing (round-robin and least-conn).
- Set up an Ubuntu 22.04 VPS with UFW, fail2ban, and a non-root deploy user.
- Author a cache-friendly production Dockerfile (you did this in Module 42; here we make it bulletproof).
- Author a production `docker-compose.prod.yml` and supporting `.env.production`.
- Build a GitHub Actions workflow with `on: push`, `on: pull_request`, and `on: tags`.
- Use GitHub Environments and secrets to manage deployment credentials.
- Trace a request from browser to container.
- Diagnose a failed deploy from logs.

---

## 3. The reverse proxy: why you need one

Your three services — Postgres, API, frontend — listen on ports. Only one of them (the frontend, ultimately) should be reachable from the public internet. But you have multiple problems to solve:

1. **TLS termination.** Browsers expect HTTPS. Terminating TLS in every container is a nightmare. Terminate it once, in front.
2. **Host-based routing.** `yourdomain.com` → frontend. `api.yourdomain.com` → API. Same IP, same port (443), different upstreams.
3. **Single entrypoint.** One firewall rule (allow 443). One place to enforce TLS, rate-limit, and add security headers.
4. **WebSockets / long-lived connections.** Nginx handles them better than most apps.
5. **Hot reload of routing config** without restarting your application containers.

The answer is **Nginx**. Alternatives: HAProxy, Caddy, Traefik, Cloudflare Tunnel. For this mission we use Nginx because it is ubiquitous, well-documented, and you will encounter it in every real production environment.

```
   Browser  ──HTTPS──►  Nginx (TLS + routing)  ──HTTP──►  containers
                            │
                            ├── /          → web:3000
                            ├── /api/      → api:3000
                            └── /assets/   → web:3000 (static)
```

---

## 4. NGINX config basics

Nginx config is a tree of blocks. The high-level structure:

```nginx
http {
  # global settings
  upstream api_servers { ... }   # load balancer
  
  server {                       # virtual host (by hostname/port)
    listen 80;
    server_name api.example.com;
    
    location / {                 # URL prefix matcher
      proxy_pass http://api_servers;
      # ... proxy settings
    }
  }
}
```

### The minimum useful config

`/etc/nginx/nginx.conf`:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log warn;

events {
  worker_connections 1024;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;
  sendfile on;
  keepalive_timeout 65;
  gzip on;
  gzip_types text/plain text/css application/json application/javascript;

  # App-specific config
  include /etc/nginx/conf.d/*.conf;
}
```

We put our app config in `/etc/nginx/conf.d/saas.conf` (loaded by the `include` above).

`/etc/nginx/conf.d/saas.conf` — TLS termination + reverse proxy:

```nginx
# Redirect HTTP -> HTTPS
server {
  listen 80;
  server_name example.com api.example.com;
  return 301 https://$host$request_uri;
}

# Web (Next.js)
server {
  listen 443 ssl http2;
  server_name example.com;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers on;
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-Frame-Options "DENY" always;

  client_max_body_size 25m;

  location / {
    proxy_pass http://web:3000;
    proxy_http_version 1.1;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        "upgrade";
    proxy_redirect off;
  }
}

# API (Node)
server {
  listen 443 ssl http2;
  server_name api.example.com;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  ssl_protocols TLSv1.2 TLSv1.3;
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

  location / {
    proxy_pass http://api:3000;
    proxy_http_version 1.1;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_redirect off;
  }
}
```

### What each directive does

- `listen 443 ssl http2` — TLS on port 443, HTTP/2 enabled.
- `server_name example.com` — virtual host matching by `Host` header.
- `ssl_certificate` — PEM-encoded cert + chain (Let's Encrypt gives you this).
- `proxy_pass http://web:3000` — forward to upstream; **note the hostname** (`web`) resolves via Docker DNS because Nginx runs as a container on the same Docker network.
- `proxy_set_header Host $host` — pass the original `Host` header so the upstream knows which virtual host was requested.
- `proxy_set_header X-Real-IP $remote_addr` — pass the client's real IP (Nginx overwrites `remote_addr`).
- `add_header Strict-Transport-Security ...` — tell browsers to use HTTPS for a year.

### Test and reload

```bash
$ docker compose exec nginx nginx -t
nginx: configuration file /etc/nginx/nginx.conf test is successful
$ docker compose exec nginx nginx -s reload
```

---

## 5. Load balancing with Nginx

When you scale the API to multiple replicas, Nginx load-balances across them.

`/etc/nginx/conf.d/saas.conf` (excerpt):

```nginx
upstream api_servers {
  least_conn;                      # or `ip_hash` for sticky, or default round-robin
  server api:3000 max_fails=3 fail_timeout=10s;
  keepalive 32;
}

server {
  listen 443 ssl http2;
  server_name api.example.com;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  location / {
    proxy_pass http://api_servers;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 60s;
  }
}
```

Three load-balancing algorithms:

- **Round-robin** (default) — request N goes to replica N mod count. Good general-purpose choice.
- **Least connections** (`least_conn`) — send to the replica with the fewest in-flight requests. Better for long-running requests.
- **IP hash** (`ip_hash`) — sticky by client IP. Useful for sessions.

In Docker Compose, scaling is `docker compose up -d --scale api=3`, but for production you'll move to Swarm or Kubernetes.

---

## 6. Setting up the VPS

We assume **Ubuntu 22.04 LTS** on a small Hetzner / DigitalOcean / Vultr instance (2 vCPU, 4GB RAM is a fine starting point).

### First-boot hardening

```bash
# Update everything
$ sudo apt update && sudo apt upgrade -y

# Create a non-root deploy user
$ sudo adduser deploy
$ sudo usermod -aG sudo deploy
$ sudo -iu deploy

# Set up SSH key auth (copy your pubkey into ~/.ssh/authorized_keys)
$ mkdir -p ~/.ssh && chmod 700 ~/.ssh
$ nano ~/.ssh/authorized_keys  # paste your pubkey
$ chmod 600 ~/.ssh/authorized_keys

# Disable password auth
$ sudo sed -i 's/^#\?PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
$ sudo sed -i 's/^#\?PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
$ sudo systemctl restart ssh

# Firewall: allow SSH, HTTP, HTTPS only
$ sudo ufw allow OpenSSH
$ sudo ufw allow 80/tcp
$ sudo ufw allow 443/tcp
$ sudo ufw enable
$ sudo ufw status

# fail2ban: blocks brute-force SSH
$ sudo apt install -y fail2ban
$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
$ sudo systemctl enable --now fail2ban

# Automatic security updates
$ sudo apt install -y unattended-upgrades
$ sudo dpkg-reconfigure -plow unattended-upgrades
```

### Install Docker

```bash
$ sudo apt-get remove docker docker-engine docker.io containerd runc 2>/dev/null
$ sudo apt-get install -y ca-certificates curl gnupg
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
    https://download.docker.com/linux/ubuntu jammy stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin
$ sudo usermod -aG docker deploy
$ newgrp docker
$ docker run hello-world
```

### Project directory layout on the server

```
/home/deploy/
└── saas/
    ├── .env.production       # production secrets, mode 600
    ├── docker-compose.prod.yml
    ├── nginx/
    │   ├── nginx.conf
    │   └── conf.d/
    │       └── saas.conf
    └── deploy.sh             # the script that GitHub Actions calls
```

We will set this up on first deploy.

---

## 7. Cache-friendly Dockerfile (smart ordering)

You wrote this in Module 42. The single most important rule for production Dockerfiles: **maximize cache hits.**

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22.11.0-bookworm-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --include=dev

FROM node:22.11.0-bookworm-slim AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22.11.0-bookworm-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist

RUN groupadd --system --gid 1001 nodejs \
    && useradd --system --uid 1001 --gid nodejs appuser \
    && apt-get update && apt-get install -y --no-install-recommends tini \
    && rm -rf /var/lib/apt/lists/*

COPY --chown=appuser:nodejs --from=build /app/dist ./dist
USER appuser
EXPOSE 3000

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "dist/server.js"]
```

Order: manifests → install → source → build → runtime manifest → install prod deps → copy dist. A change to a single source file invalidates only the bottom layers; the `npm ci` layer is reused.

Always pair with a strong `.dockerignore` (Module 40):

```gitignore
node_modules
dist
.next
out
.git
.github
.gitignore
.env
.env.*
!.env.example
coverage
*.log
*.test.ts
__tests__
.vscode
.idea
.DS_Store
README.md
docs
```

---

## 8. Production docker-compose

`docker-compose.prod.yml`:

```yaml
name: saas-prod

services:
  nginx:
    image: nginx:1.27-alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro
    depends_on:
      - web
      - api
    networks:
      - frontend
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 5s
      retries: 3

  certbot:
    image: certbot/certbot
    restart: unless-stopped
    volumes:
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:rw
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h; done'"

  db:
    image: postgres:16.4-bookworm
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - backend
    # No `ports:` — Postgres is not exposed to the host.

  api:
    image: ghcr.io/${GITHUB_REPOSITORY}/saas-api:${RELEASE_TAG:-latest}
    restart: always
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      JWT_SECRET: ${JWT_SECRET}
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend
      - frontend
    healthcheck:
      test: ["CMD-SHELL", "node -e \"require('http').get('http://127.0.0.1:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))\""]
      interval: 15s
      timeout: 3s
      retries: 5
      start_period: 15s

  web:
    image: ghcr.io/${GITHUB_REPOSITORY}/saas-web:${RELEASE_TAG:-latest}
    restart: always
    environment:
      NODE_ENV: production
      PORT: 3000
      HOSTNAME: 0.0.0.0
      NEXT_PUBLIC_API_URL: https://api.${DOMAIN}
    depends_on:
      - api
    networks:
      - frontend
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:3000/ >/dev/null || exit 1"]
      interval: 15s
      timeout: 3s
      retries: 5
      start_period: 20s

networks:
  backend:
  frontend:

volumes:
  pgdata:
```

### `.env.production`

```bash
DOMAIN=example.com
POSTGRES_USER=app
POSTGRES_PASSWORD=<generated-32-char-random>
POSTGRES_DB=saas
JWT_SECRET=<generated-64-char-random>
STRIPE_SECRET_KEY=sk_live_...
RELEASE_TAG=1.0.0
```

`chmod 600 .env.production`. Never commit it.

### NGINX config — production

`nginx/nginx.conf`:

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
  worker_connections 2048;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;
  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" "$http_user_agent" '
                  'rt=$request_time';
  access_log /var/log/nginx/access.log main;

  sendfile on;
  tcp_nopush on;
  keepalive_timeout 65;
  types_hash_max_size 2048;
  server_tokens off;
  client_max_body_size 25m;

  gzip on;
  gzip_vary on;
  gzip_min_length 1024;
  gzip_types text/plain text/css application/json application/javascript
             application/xml+rss application/atom+xml image/svg+xml;

  include /etc/nginx/conf.d/*.conf;
}
```

`nginx/conf.d/saas.conf`:

```nginx
# HTTP -> HTTPS
server {
  listen 80;
  server_name example.com api.example.com;
  location /.well-known/acme-challenge/ {
    root /var/www/certbot;
  }
  location / {
    return 301 https://$host$request_uri;
  }
}

# Web
server {
  listen 443 ssl http2;
  server_name example.com;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
  ssl_protocols       TLSv1.2 TLSv1.3;
  ssl_ciphers         HIGH:!aNULL:!MD5;
  ssl_session_cache   shared:SSL:10m;
  ssl_session_timeout 1d;

  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-Frame-Options "DENY" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;

  location / {
    proxy_pass http://web:3000;
    proxy_http_version 1.1;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        "upgrade";
  }
}

# API
upstream api_servers {
  least_conn;
  server api:3000 max_fails=3 fail_timeout=10s;
  keepalive 32;
}

server {
  listen 443 ssl http2;
  server_name api.example.com;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
  ssl_protocols       TLSv1.2 TLSv1.3;

  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

  location / {
    proxy_pass http://api_servers;
    proxy_http_version 1.1;
    proxy_set_header Connection        "";
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

### TLS with Let's Encrypt + certbot in a container

The certbot container above renews every 12 hours and writes certs into a shared volume that Nginx reads.

Bootstrap (run once, on the server):

```bash
$ docker compose -f docker-compose.prod.yml run --rm certbot certonly \
    --webroot --webroot-path=/var/www/certbot \
    -d example.com -d api.example.com \
    --email you@example.com --agree-tos --no-eff-email
```

Now Nginx has valid certs and the certbot container keeps them renewed forever.

---

## 9. CI/CD with GitHub Actions

Three triggers, three jobs:

- **Every push to `main`** → build images, run smoke tests, push to GHCR, deploy to staging.
- **Every pull request** → build images, run tests, scan for vulnerabilities, **do not deploy**.
- **Every tag matching `v*`** → build images, run tests, push with the tag, deploy to production.

### The workflow

`.github/workflows/deploy.yml`:

```yaml
name: build-and-deploy

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_API: ${{ github.repository }}/saas-api
  IMAGE_WEB: ${{ github.repository }}/saas-web

jobs:
  test:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: apps/api/package-lock.json

      - name: Install & test API
        working-directory: apps/api
        run: |
          npm ci
          npm test

      - name: Install & build web
        working-directory: apps/web
        run: |
          npm ci
          npm run build

  scan:
    runs-on: ubuntu-22.04
    needs: test
    permissions:
      contents: read
      packages: read
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: Build api image
        uses: docker/build-push-action@v6
        with:
          context: apps/api
          file: docker/api.Dockerfile
          push: false
          tags: ${{ env.IMAGE_API }}:scan-${{ github.sha }}
          load: true

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.IMAGE_API }}:scan-${{ github.sha }}
          format: table
          exit-code: "1"
          ignore-unfixed: true
          vuln-type: os,library
          severity: CRITICAL,HIGH

  build-and-push:
    runs-on: ubuntu-22.04
    needs: [test, scan]
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v'))
    permissions:
      contents: read
      packages: write
    outputs:
      tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract API metadata
        id: meta-api
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_API }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,format=short

      - name: Build and push API
        uses: docker/build-push-action@v6
        with:
          context: apps/api
          file: docker/api.Dockerfile
          push: true
          tags: ${{ steps.meta-api.outputs.tags }}
          labels: ${{ steps.meta-api.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Extract web metadata
        id: meta-web
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_WEB }}

      - name: Build and push web
        uses: docker/build-push-action@v6
        with:
          context: apps/web
          file: docker/web.Dockerfile
          push: true
          tags: ${{ steps.meta-web.outputs.tags }}
          labels: ${{ steps.meta-web.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-prod:
    runs-on: ubuntu-22.04
    needs: build-and-push
    if: startsWith(github.ref, 'refs/tags/v')
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4

      - name: Copy compose file to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.VPS_HOST }}
          username: deploy
          key: ${{ secrets.VPS_SSH_KEY }}
          source: "docker-compose.prod.yml,nginx/"
          target: ~/saas/

      - name: Run deploy script
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: deploy
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd ~/saas
            export RELEASE_TAG=${GITHUB_REF_NAME}
            docker compose -f docker-compose.prod.yml pull api web
            docker compose -f docker-compose.prod.yml up -d --remove-orphans
            docker image prune -f
            echo "Deploy complete at $(date -u)"
```

### GitHub Environment setup

1. Repo → Settings → Environments → New environment → `production`.
2. Add secrets:
   - `VPS_HOST` — your server's IP or hostname.
   - `VPS_SSH_KEY` — the private SSH key for the `deploy` user (not your personal key).
3. (Optional) Add required reviewers for `production` so a teammate must approve each deploy.

### Secrets management

In `.env.production` on the server, secrets come from environment variables you set manually. In GitHub, secrets are repo- or environment-scoped. **Never put a real secret in a YAML file committed to git.**

If you want centralized secrets, integrate with **Doppler**, **Infisical**, or **AWS Secrets Manager**. Out of scope for this mission; mentioned so you know where to go next.

---

## 10. Environment variables and upgrading Nginx on the VPS

### Adding a new env var to production

1. Edit `.env.production` on the server.
2. Restart the relevant service:
   ```bash
   $ docker compose -f docker-compose.prod.yml up -d api
   ```
3. Confirm the new env is in the container:
   ```bash
   $ docker compose -f docker-compose.prod.yml exec api env | grep NEW_VAR
   ```

### Upgrading Nginx config

Edit `nginx/conf.d/saas.conf` on the server (or push via GitHub Actions). Then:

```bash
$ docker compose -f docker-compose.prod.yml exec nginx nginx -t
nginx: configuration file /etc/nginx/nginx.conf test is successful
$ docker compose -f docker-compose.prod.yml exec nginx nginx -s reload
```

That's it. No container restart, no downtime, no SSL handshake drops.

If the syntax test fails, you broke something. Nginx will keep running with the old config; fix the file and try again.

---

## 11. CI/CD pipeline recap

```
        ┌──────────────────────────────────────────────────────┐
        │  GitHub Actions (ubuntu-22.04)                       │
        │                                                      │
PR / push ──► test ──► scan ──► build & push ──► (tag only)  │
                                              │                │
                                              ▼                │
                                       deploy-prod             │
                                              │                │
                                              ▼                │
                                  ┌──────────────────┐         │
                                  │  VPS (Ubuntu)    │         │
                                  │  docker compose  │         │
                                  │  pull + up -d    │         │
                                  └──────────────────┘         │
                                          │                    │
                                   ┌──────┴──────┐             │
                                   ▼             ▼             │
                              web (port 3000) api (3000)       │
                                   │             │             │
                                   └──────┬──────┘             │
                                          ▼                    │
                                    Nginx (80/443) ◄───────────┘
                                          │
                                          ▼
                                       Browser
```

The pipeline is idempotent. Re-running it produces the same result.

---

## 12. Conceptual session 1: tracing a request

**Scenario:** A user opens `https://example.com/dashboard` in their browser.

Step by step, with no gaps:

1. **Browser** resolves `example.com` via DNS. The resolver returns your VPS IP.
2. **Browser** opens a TCP connection to `<vps-ip>:443`.
3. **TLS handshake.** Nginx presents the Let's Encrypt cert for `example.com`. Browser validates it.
4. **HTTP request.** `GET /dashboard`, headers `Host: example.com`, `User-Agent: ...`, etc.
5. **Nginx** matches the request to the `server` block with `server_name example.com`.
6. **Nginx** proxies to `http://web:3000` (the Next.js container), rewriting `Host`, adding `X-Real-IP`, etc.
7. **Docker networking.** `web` resolves to the container's IP on the `frontend` network. Nginx's request lands on port 3000 of the `web` container.
8. **Next.js** handles the request. For a server component, it may render HTML. For a client component, it may return JS + serialized props. It may call `https://api.example.com/...` from inside the container.
9. **If the page calls `/api/...`:** the browser makes a separate request to `api.example.com`, which goes back through the same path, but matches the second `server` block, which proxies to `http://api:3000`.
10. **The API container** processes the request. It might query Postgres over the `backend` network at `db:5432`.
11. **Postgres** returns rows over the same network.
12. **API returns** JSON, which Nginx returns to the browser.
13. **Browser** renders the page.

Every step is replaceable. Every step can fail. Knowing the path is how you debug.

---

## 13. Conceptual session 2: "it broke in production"

**Scenario:** You pushed `v1.4.0`. The CI passed. The deploy step ran. But the homepage returns 502. Let's diagnose.

### Step 1: read the deploy logs

In GitHub → Actions → the workflow run, check the `deploy-prod` job output. Did `docker compose pull` succeed? Did `up -d` succeed? Did the script exit cleanly?

```bash
# On the server
$ journalctl -u docker --since "1 hour ago" --no-pager | tail
$ cd ~/sa~/saas && docker compose -f docker-compose.prod.yml ps
```

If a container is `Exit 1` or restarting, that's the suspect.

### Step 2: read the broken container's logs

```bash
$ docker compose -f docker-compose.prod.yml logs --tail=200 web
$ docker compose -f docker-compose.prod.yml logs --tail=200 api
$ docker compose -f docker-compose.prod.yml logs --tail=200 nginx
$ docker compose -f docker-compose.prod.yml logs --tail=200 db
```

Common patterns:

- `web` exited with code 1 → application error (bad env var, bad build, missing file).
- `web` is restarting → crash loop; check `docker inspect` for `RestartCount` and the previous log block.
- `nginx` says `connect() failed (111: Connection refused) while connecting to upstream` → the upstream container isn't listening on the expected port.
- `db` says `FATAL: password authentication failed for user "app"` → wrong env var was passed.
- `db` says `connection limit exceeded for non-superusers` → too many clients; you forgot to close a connection pool.

### Step 3: check env vars

```bash
$ docker compose -f docker-compose.prod.yml exec api env | sort
```

Is `DATABASE_URL` what you think it is? Is `JWT_SECRET` set? Did `.env.production` get clobbered by an earlier deploy?

### Step 4: check the network

```bash
$ docker network inspect saas-prod_frontend
$ docker network inspect saas-prod_backend
```

Are all expected containers attached? Is the `web` container on `frontend`?

### Step 5: roll back

If you can't figure it out in 15 minutes, roll back to the previous tag:

```bash
$ export RELEASE_TAG=1.3.0
$ docker compose -f docker-compose.prod.yml pull api web
$ docker compose -f docker-compose.prod.yml up -d --remove-orphans
```

If `1.3.0` was working and `1.4.0` is broken, you've reduced your surface area to the diff between the two tags. Bisect from there.

### Step 6: fix forward

Once you've found the bug, push a `v1.4.1` tag. CI runs, deploy happens, system is back. Document the incident in your team's runbook. Move on.

---

## 14. Common mistakes

- **Publishing Postgres to the host.** `ports: ["5432:5432"]` on the db service. Don't.
- **No firewall on the VPS.** Without UFW, your database is one stray `-p` away from being public.
- **Plain HTTP in prod.** Browsers will warn; SEO will suffer; you'll lose data integrity at scale. Use HTTPS.
- **Self-signed certs in production.** Real users will get scary warnings. Use Let's Encrypt.
- **Storing secrets in `docker-compose.yml`.** Use env vars, a `.env.production` file with `chmod 600`, or a secret manager.
- **Deploying on every push to `main`.** Sometimes you want a manual approval step. Use GitHub Environments with required reviewers.
- **`docker compose down -v` in prod.** You will lose your database. Do not.
- **Restarting containers to reload config.** Nginx supports `nginx -s reload`; the API typically needs a restart for env changes; both should be possible without touching the other.
- **Forgetting to prune old images.** `docker image prune -f` in the deploy script keeps disk usage from filling up.
- **No structured logs.** You will be sad at 3am. Use JSON logs and ship them somewhere (CloudWatch, Loki, Datadog).

---

## 15. Exercise

**Goal:** Deploy your SaaS to a real VPS and set up the CI/CD pipeline.

**Steps:**

1. Provision a small Ubuntu 22.04 VPS (Hetzner, DigitalOcean, Vultr).
2. Harden per section 6.
3. Install Docker.
4. Point a domain's A record to the VPS IP. (Buy a cheap domain, or use a free DuckDNS / No-IP host for testing.)
5. Clone your SaaS repo onto the server.
6. Copy `docker-compose.prod.yml` and `nginx/` to the server.
7. Bootstrap TLS with certbot (one-time).
8. `docker compose -f docker-compose.prod.yml up -d`.
9. Verify `https://yourdomain.com` works.
10. Add GitHub secrets (`VPS_HOST`, `VPS_SSH_KEY`).
11. Push a `v1.0.0` tag. Watch the CI/CD pipeline run. Verify the live site updates.
12. Intentionally break something (e.g. wrong env var). Watch it fail in the browser. Debug using the steps in section 13. Roll back. Fix forward.

**Deliverable:** A live URL serving your SaaS over HTTPS, plus a successful tag-triggered redeploy.

---

## 16. AI Industry Note (2026)

Deploying AI products has a few wrinkles the SaaS world doesn't:

- **Model weights.** Pull them at runtime from S3 or Hugging Face into a volume; don't bake them into the image.
- **GPU memory.** A single container with access to all of them (or `nvidia-container-runtime`) is usually how you run an LLM server. Treat GPU memory like RAM — a container that holds onto it is not "stateless."
- **Cold starts.** A 70B-parameter model takes ~30s to load into VRAM. Your healthcheck needs a long `start_period` (60–120s) or your orchestrator will kill it as "unhealthy."
- **Streaming.** Most LLM APIs stream tokens via SSE or WebSockets. Make sure Nginx's `proxy_buffering off;` and long `proxy_read_timeout` are set, or your stream will buffer and time out.
- **Cost observability.** Every AI startup in 2026 tracks cost-per-request. Bake the cost counter into the API (log tokens in, tokens out, USD) so you can attribute spend per user.

The deploy pipeline you built in this module — build, scan, push, deploy, healthcheck — is exactly the shape of every serious AI product deploy. The fact that the artifact happens to be 5GB and holds a GPU is incidental; the engineering is the same.

Also in 2026: **GitHub Actions OIDC + cloud IAM** lets your workflow assume a cloud role without storing long-lived secrets. Worth setting up when you graduate to AWS/GCP. **Cosign + Sigstore** signs your images so consumers can verify provenance. **SBOMs** (Software Bills of Materials) are now mandatory in many enterprise deals. Trivy and Syft both emit them; include them as workflow outputs.

---

## 17. Where you are now

Take a breath. Two missions ago you had a Next.js app and a Node API on your laptop. Today you have:

- A reproducible multi-container app, declared in a `docker-compose.yml`.
- A production Dockerfile pipeline with multi-stage builds, non-root users, healthchecks, and tini.
- A reverse proxy with TLS termination and load balancing.
- A VPS hardened with UFW and fail2ban.
- A CI/CD pipeline that builds, scans, signs, and ships on tag.
- The debugging vocabulary to fix it.

That is the same stack used by real production teams shipping to paying customers. The only differences at scale are operational (multiple servers, Kubernetes, observability, secrets management, autoscaling). The vocabulary is identical.

Mission 6 — **AI Systems** — will build on this foundation. Every model you fine-tune, every RAG pipeline you build, every agent you deploy will be a Docker image behind a reverse proxy on infrastructure you set up exactly like this. You're ready.