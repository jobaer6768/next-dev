# Reward Course 04 — AWS Deployment

> Reward module. Builds on Mission 5 (production, Docker) and prepares for the Capstone. Estimated effort: 14–18 hours.

---

## Why this module matters

In Mission 5 you packaged the SaaS into a Docker stack and ran it on your laptop. In this module you ship that same stack to a real cloud: AWS. By the end you will have a working EC2 instance with a domain, HTTPS, a managed Postgres, S3 for files, and CloudFront in front of it.

This is the difference between "I built a project" and "I shipped a product." The hiring manager in 2026 doesn't care that you can write code; they care that you can run it in production without paging them at 3am.

AWS is the largest cloud by market share and the one most likely to show up in your first job. We will use it pragmatically: the minimum services to ship, the maximum understanding of why each is there.

---

## Learning objectives

By the end of this module you will be able to:

1. Describe the AWS global infrastructure (regions, AZs, edge).
2. Apply IAM least-privilege in a real AWS account.
3. Store and serve static files from S3.
4. Administer a Linux server on EC2 (ssh, systemd, journalctl, ufw).
5. Manage an EC2 instance as a deployable target.
6. Configure a domain and DNS in Route 53.
7. Provision and auto-renew an SSL certificate (Let's Encrypt).
8. Monitor a running server with CloudWatch.
9. Apply the AWS Well-Architected Framework pillars to a real design.
10. Deploy the Mission 5 Docker stack to EC2 + RDS + S3 + CloudFront.

---

## 1. Introduction to Cloud Computing

Cloud computing is renting someone else's computer, network, and disk. The "someone else" is AWS (or GCP, Azure, etc.), and the rental is by the second, with no contract.

### Service models

| Model | You manage | Provider manages | Example |
|---|---|---|---|
| **IaaS** (EC2) | OS, runtime, app | Hardware, network, hypervisor | A raw VM |
| **PaaS** (Elastic Beanstalk, App Runner) | App + data | Everything else | `git push` deploys |
| **SaaS** (most AWS services) | Configuration, data | Everything | S3, RDS, Lambda |

We will mostly use **IaaS** (EC2) because that's what the Capstone needs. We will also use managed services (RDS, S3, CloudFront) for the things where managing them yourself is a waste of time.

### The pricing mental model

You pay for: compute (per second, by instance size), storage (per GB-month), data transfer *out* (per GB, free in), and API calls (per 1M). Data transfer in is free. Data transfer out is the silent killer — a busy site can owe more in bandwidth than in compute.

**Cost rule for this module:** stay in the AWS Free Tier. Use `t2.micro` or `t3.micro` instances, single-AZ RDS, S3 with lifecycle rules.

---

## 2. AWS Global Infrastructure

```text
                        ┌──────────────┐
                        │ Edge Location│ (CloudFront, Route 53)
                        │  (400+ WW)   │
                        └──────────────┘
                                │
                ┌───────────────┴───────────────┐
                ▼                               ▼
        ┌──────────────┐                 ┌──────────────┐
        │   Region     │                 │   Region     │
        │ (e.g. us-e-1)│                 │ (e.g. eu-w-1)│
        │              │                 │              │
        │ ┌──────────┐ │                 │ ┌──────────┐ │
        │ │   AZ-a   │ │                 │ │   AZ-c   │ │
        │ │ ┌──────┐ │ │                 │ │ ┌──────┐ │ │
        │ │ │ DC 1 │ │ │                 │ │ │ DC 5 │ │ │
        │ │ └──────┘ │ │                 │ │ └──────┘ │ │
        │ └──────────┘ │                 │ └──────────┘ │
        │ ┌──────────┐ │                 │ ┌──────────┐ │
        │ │   AZ-b   │ │                 │ │   AZ-d   │ │
        │ │ ┌──────┐ │ │                 │ │ ┌──────┐ │ │
        │ │ │ DC 2 │ │ │                 │ │ │ DC 6 │ │ │
        │ │ └──────┘ │ │                 │ │ └──────┘ │ │
        │ └──────────┘ │                 │ └──────────┘ │
        └──────────────┘                 └──────────────┘
```

- **Region** — a geographic area (e.g., `us-east-1` is N. Virginia). Each region is fully independent.
- **Availability Zone (AZ)** — one or more discrete data centers in a region, each with independent power, network, and cooling. A region has 2–6 AZs.
- **Edge Location** — a CDN point of presence. Used by CloudFront (cache) and Route 53 (DNS).

**Rules:**

- **Pick one region.** Most teams pick the region closest to their users. For this course, `us-east-1` is fine and has the most services.
- **Run in at least 2 AZs** for any production service that should survive a data center failure.
- **Data transfer across regions is expensive.** Don't put your API in `us-east-1` and your DB in `eu-west-1` unless you have a real reason.

---

## 3. Identity & Access Management (IAM)

IAM is how you say *who* can do *what* on *which* resource. If you skip this section, you will ship a credential in your repo and your account will be hijacked for crypto mining within a week.

### The vocabulary

- **User** — a person or service. Has long-lived credentials. Avoid for services.
- **Group** — a collection of users. Easier to grant the same policy to 5 engineers.
- **Role** — a set of permissions that something *assumes* temporarily. Preferred for services.
- **Policy** — a JSON document describing allowed/denied actions. Attached to users, groups, or roles.

### The principle of least privilege

Grant the *minimum* permission needed. Not "admin." Not "power user." Just the specific actions on the specific resources.

```json
// policy: read-only on a specific S3 bucket
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-saas-uploads/*"
    }
  ]
}
```

### The credential hierarchy (best → worst)

1. **IAM Role + instance profile** (for EC2) — no credentials in your code
2. **IAM Role + IRSA** (for EKS) — same, for Kubernetes
3. **IAM Role + OIDC** (for GitHub Actions) — deploy without static keys
4. **Environment variables from Secrets Manager / SSM**
5. **Long-lived access keys in env vars** — last resort, rotate aggressively

### The root account

Lock it down immediately:

1. Enable MFA (preferably a hardware key, otherwise TOTP).
2. Create an admin IAM user. Use that.
3. Never use the root account for anything except billing or account-recovery.

---

## 4. Amazon S3 Object Storage

S3 is a key-value store where keys are strings and values are bytes (up to 5TB per object). It has 11 nines of durability and is the canonical place for static files.

### Concepts

- **Bucket** — a namespace. Globally unique name.
- **Object** — a file + metadata + a key.
- **Key** — the path-like name (`uploads/user-42/photo.jpg`).
- **Versioning** — keep old versions of objects. Costs more; worth it for important data.
- **Lifecycle rule** — automatically move objects to cheaper storage or delete them after N days.

### Storage classes (cheapest to most expensive)

| Class | Use case | Cost/GB-month |
|---|---|---|
| Standard | Hot data, frequent access | $0.023 |
| Intelligent-Tiering | Unknown access pattern | $0.023 + monitoring |
| Standard-IA | Infrequent (≥30d) | $0.0125 |
| One Zone-IA | Infrequent, can be lost | $0.01 |
| Glacier Instant | Archive, instant retrieval | $0.004 |
| Glacier Flexible | Archive, hours to retrieve | $0.0036 |
| Glacier Deep | Archive, 12+ hours | $0.00099 |

### The CLI basics

```bash
aws s3 mb s3://my-saas-uploads                    # create bucket
aws s3 cp file.txt s3://my-saas-uploads/         # upload
aws s3 sync ./dist s3://my-saas-static/           # deploy static site
aws s3 presign s3://my-saas-uploads/photo.jpg    # 15-min signed URL
```

### For the SaaS

- Bucket `my-saas-uploads` — private, signed URLs for downloads.
- Bucket `my-saas-static` — public-read via CloudFront, for the Next.js frontend.
- Lifecycle rule: move uploads older than 90 days to Standard-IA.

---

## 5. Linux Essentials for Cloud Servers

You will spend a lot of time on a Linux shell. Here is the minimum.

### ssh

```bash
ssh -i ~/keys/my-key.pem ubuntu@1.2.3.4
# or, with ssh config:
Host mysaas
  HostName 1.2.3.4
  User ubuntu
  IdentityFile ~/keys/my-key.pem
ssh mysaas
```

### scp / rsync

```bash
scp ./docker-compose.yml mysaas:~/
rsync -avz ./app mysaas:~/app
```

### systemd

Every long-running process should be a systemd unit. Don't `npm start` and close your terminal.

```ini
# /etc/systemd/system/my-saas.service
[Unit]
Description=My SaaS API
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/app
ExecStart=/usr/bin/docker compose up
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my-saas
sudo systemctl status my-saas
```

### journalctl

```bash
sudo journalctl -u my-saas -f           # follow logs
sudo journalctl -u my-saas --since "1h ago"
```

### ufw (the uncomplicated firewall)

```bash
sudo ufw allow 22/tcp     # ssh
sudo ufw allow 80/tcp     # http
sudo ufw allow 443/tcp    # https
sudo ufw enable
sudo ufw status
```

Default-deny everything else. This is the easiest security win in cloud.

---

## 6. EC2 Instance Management

EC2 is a VM. You pick an instance type (CPU + RAM), an AMI (image), and a key pair. AWS gives you an IP.

### Instance types you'll use

| Type | vCPU | RAM | Cost/hour (t3, us-east-1) | Use |
|---|---|---|---|---|
| `t3.micro` | 2 | 1 GB | $0.0104 | Free-tier, dev, hobby |
| `t3.small` | 2 | 2 GB | $0.0208 | Capstone, low traffic |
| `t3.medium` | 2 | 4 GB | $0.0416 | Capstone, real traffic |
| `m6i.large` | 2 | 8 GB | $0.096 | Production |

### The minimum setup

1. Launch `t3.micro` (or `t3.small` for the Capstone) with Ubuntu 24.04 LTS.
2. Storage: 20GB gp3 (general-purpose SSD).
3. Security group: 22 (your IP only), 80, 443.
4. Attach an IAM role with `AmazonS3FullAccess` *or* a more specific policy.
5. SSH in. `sudo apt update && sudo apt upgrade -y`.
6. Install Docker, Node 22, your deploy key for GitHub.

### Elastic IP

EC2 instances get a public IP, but it changes when you stop/start. For a stable endpoint, allocate an **Elastic IP** and associate it. Free while attached.

### User data

You can bootstrap an instance with a script passed at launch:

```bash
#!/bin/bash
apt update -y
apt install -y docker.io docker-compose-plugin
systemctl enable --now docker
usermod -aG docker ubuntu
```

This runs *once* on first boot.

---

## 7. Domain & DNS Configuration

You need a domain. Buy one from Route 53, Namecheap, Cloudflare, or any registrar. For this course, Route 53 is fine.

### DNS records you'll set

| Record | When | Example |
|---|---|---|
| **A** | Domain → IPv4 | `mysaas.com → 1.2.3.4` |
| **AAAA** | Domain → IPv6 | `mysaas.com → ::1` |
| **CNAME** | Subdomain → another domain | `www.mysaas.com → mysaas.com` |
| **TXT** | Verification, SPF | `mysaas.com → "v=spf1 -all"` |
| **MX** | Mail | `mysaas.com → 10 mail.mysaas.com` |
| **NS** | Authoritative servers | `mysaas.com → ns-1.awsdns-1.com` |

### TTL

How long resolvers cache your record. 300s (5 min) for development, 86400s (1 day) for stable production. The lower the TTL, the faster changes propagate.

### Route 53 hosted zone

```bash
# create zone
aws route53 create-hosted-zone --name mysaas.com --caller-reference $(date +%s)
# get the 4 NS records
aws route53 list-resource-record-sets --hosted-zone-id Z123...
# point your registrar's NS records to those 4
```

### Latency-based routing (optional)

If you go multi-region later, Route 53 can route users to the closest region. Not needed for the Capstone.

---

## 8. SSL/HTTPS with Let's Encrypt

HTTP is fine for development. Production is HTTPS. Always.

### The plan

1. Get a free cert from Let's Encrypt.
2. Install it on the server (or on NGINX in front of the app).
3. Auto-renew it (cert expires every 90 days).

### Using certbot with NGINX

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d mysaas.com -d www.mysaas.com
# enter email, agree to ToS
# certbot edits /etc/nginx/sites-enabled/mysaas automatically
```

The cert lives at `/etc/letsencrypt/live/mysaas.com/fullchain.pem`.

### Auto-renewal

Certbot installs a systemd timer. Verify:

```bash
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

### The ACM alternative (AWS-only)

ACM (AWS Certificate Manager) issues free certs, but only for AWS resources (ALB, CloudFront, API Gateway). If your cert is for an EC2 instance directly, you can't use ACM. You can use ACM *behind* an ALB, which is the production pattern. For the Capstone, certbot is simpler.

### Force HTTPS

In NGINX:

```nginx
server {
  listen 80;
  server_name mysaas.com www.mysaas.com;
  return 301 https://$host$request_uri;
}
server {
  listen 443 ssl http2;
  server_name mysaas.com;
  ssl_certificate /etc/letsencrypt/live/mysaas.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/mysaas.com/privkey.pem;
  # ... proxy to your app
}
```

---

## 9. Monitoring & Server Maintenance

You cannot fix what you cannot see. CloudWatch is the default.

### CloudWatch metrics (free for EC2)

- CPUUtilization, NetworkIn, NetworkOut, StatusCheckFailed
- Memory, disk — *not* free; install the CloudWatch agent.

### CloudWatch logs

Forward your app's logs to CloudWatch:

```bash
# install the agent, configure
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Or, simpler, ship Docker logs with the `awslogs` driver:

```yaml
# docker-compose.yml
services:
  api:
    image: my-saas-api
    logging:
      driver: awslogs
      options:
        awslogs-group: /my-saas/api
        awslogs-region: us-east-1
        awslogs-create-group: "true"
```

### Alarms

You don't watch dashboards; you get paged.

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name api-high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123:alerts
```

When the alarm fires, SNS publishes to an email, Slack webhook, or PagerDuty.

### The "is it alive?" check

The simplest, highest-value alarm:

```bash
curl -fs https://mysaas.com/healthz
```

If this returns non-200 for 2 minutes, page someone. Use a synthetic monitor (UptimeRobot, Pingdom, or CloudWatch Synthetics).

---

## 10. AWS Best Practices: the Well-Architected Framework

AWS has codified its hard-won lessons into six pillars. Every architecture you ever design should be checked against them.

| Pillar | Question | Practices |
|---|---|---|
| **Operational Excellence** | Are you running it well? | IaC (Terraform/CDK), runbooks, CI/CD, monitoring |
| **Security** | Is it safe? | Least-privilege IAM, encryption at rest & in transit, MFA, logging |
| **Reliability** | Does it recover? | Multi-AZ, backups, auto-scaling, health checks |
| **Performance Efficiency** | Is it fast? | Right-sized instances, caching (CloudFront, ElastiCache), profiling |
| **Cost Optimization** | Is it cheap? | Right-sized, lifecycle policies, tagging, scheduled scaling |
| **Sustainability** | Is it green? | Right-sized regions, efficient code, serverless where it fits |

### How to actually apply them in the Capstone

- **Operational:** the `docker-compose.yml` is in a Git repo, deployed by a script in GitHub Actions, with a rollback command.
- **Security:** IAM role on the instance, no static keys, security group with only 22/80/443, ufw, certbot auto-renew, RDS in a private subnet.
- **Reliability:** RDS has automated backups and a 7-day retention. The instance has a snapshot on every deploy. Health check at `/healthz`.
- **Performance:** CloudFront in front of S3 for static assets. NGINX gzip on responses.
- **Cost:** all `t3.micro` or `t3.small`, lifecycle rule on S3, single-AZ RDS, alarms for billing.
- **Sustainability:** single region, smallest instance that handles the load.

---

## 11. Worked Example: Deploy the Mission 5 Docker Stack

The Mission 5 stack is:

```yaml
# docker-compose.yml
services:
  api:
    build: ./apps/api
    environment:
      DATABASE_URL: postgres://...
      S3_BUCKET: my-saas-uploads
    depends_on: [db]
  web:
    build: ./apps/web
    environment:
      NEXT_PUBLIC_API_URL: https://api.mysaas.com
  db:
    image: postgres:16
    volumes: [pgdata:/var/lib/postgresql/data]
volumes:
  pgdata:
```

### Step 1 — Provision the AWS resources

You can do this by clicking, but for reproducibility, use the CLI or Terraform.

```bash
# RDS (managed Postgres)
aws rds create-db-instance \
  --db-instance-identifier my-saas-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16 \
  --master-username mysaas \
  --master-user-password $(openssl rand -base64 24) \
  --allocated-storage 20 \
  --db-name mysaas \
  --no-publicly-accessible \
  --backup-retention-period 7

# S3 buckets
aws s3 mb s3://my-saas-uploads
aws s3 mb s3://my-saas-static

# EC2 instance (Ubuntu 24.04, t3.small, in a public subnet with an Elastic IP)
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.small \
  --key-name my-key \
  --security-group-ids sg-xxx \
  --iam-instance-profile Name=my-saas-instance-role
```

### Step 2 — Set up the EC2 instance

```bash
ssh ubuntu@1.2.3.4
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-plugin nginx certbot python3-certbot-nginx
sudo usermod -aG docker ubuntu
# log out and back in
```

### Step 3 — Deploy the stack

```bash
git clone https://github.com/you/my-saas.git
cd my-saas
cp .env.example .env
# fill in DATABASE_URL, S3 keys, JWT_SECRET
docker compose up -d
```

### Step 4 — NGINX as reverse proxy

```nginx
# /etc/nginx/sites-enabled/mysaas
server {
  listen 80;
  server_name api.mysaas.com;

  location / {
    proxy_pass http://localhost:4000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

```bash
sudo certbot --nginx -d api.mysaas.com
```

### Step 5 — CloudFront in front of S3 for the frontend

```bash
# create distribution
aws cloudfront create-distribution \
  --origin-domain-name my-saas-static.s3.amazonaws.com \
  --default-root-object index.html
```

Point `mysaas.com` (A record, alias) to the CloudFront distribution.

### Step 6 — CI/CD (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: SSH and deploy
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ~/my-saas
            git pull
            docker compose pull
            docker compose up -d --build
            docker system prune -f
```

### Step 7 — DNS in Route 53

| Record | Type | Value |
|---|---|---|
| `mysaas.com` | A (alias) | CloudFront distribution |
| `api.mysaas.com` | A | EC2 Elastic IP |
| `mysaas.com` | TXT | `"v=spf1 -all"` |

### Step 8 — Verify

```bash
curl -fs https://mysaas.com             # 200
curl -fs https://api.mysaas.com/healthz # 200
```

If both 200, you're live. The SaaS is on the internet. Take a screenshot, save it in the README, and tell people.

---

## Common mistakes

1. **Root account with no MFA.** First thing on day 1.
2. **0.0.0.0/0 on port 22.** Open SSH to the world. Restrict to your IP.
3. **Long-lived access keys in code.** Use IAM roles or Secrets Manager.
4. **No backups.** Enable RDS automated backups. Test the restore.
5. **Single AZ in production.** RDS Multi-AZ costs 2x but saves you from a 4-hour outage.
6. **Forgot to set up billing alarms.** A misconfigured S3 bucket with public-write will be found and filled with crypto mining data within hours.
7. **`chmod 777` to fix a permission error.** Stop, add the user to the right group.
8. **No domain at all.** IP-address-in-the-URL is not a product.
9. **Cert expired because no auto-renew.** Set up certbot.timer. Test with `certbot renew --dry-run`.
10. **CloudFront pointing at the wrong bucket.** Always check the origin domain in the console.

---

## Exercise

**Deploy the Mission 5 SaaS to AWS.**

1. Create an AWS account (use the Free Tier).
2. Lock down the root account with MFA. Create an admin IAM user.
3. Create two S3 buckets: `yourname-saas-uploads` (private) and `yourname-saas-static` (via CloudFront).
4. Launch a `t3.small` Ubuntu 24.04 instance in `us-east-1` with an Elastic IP.
5. Set up ufw (22, 80, 443 only).
6. Install Docker, NGINX, and certbot on the instance.
7. Deploy the Mission 5 stack via `docker compose`.
8. Configure NGINX as a reverse proxy for the API.
9. Get a Let's Encrypt cert for both `mysaas.yourname.com` and `api.mysaas.com`.
10. Configure Route 53 to point your subdomain at the instance.
11. Put CloudFront in front of the static-asset S3 bucket.
12. Set up a CloudWatch alarm: page yourself (email) if CPU > 80% for 5 minutes.
13. Add a `scripts/smoke.sh` that curls `/healthz` and exits non-zero on failure.
14. **Stretch:** add a GitHub Actions workflow that deploys on every push to `main`.

Submit: the deployed URL, a screenshot of the running app, the contents of your `docker-compose.yml`, your NGINX config (redacted), and a 1-paragraph postmortem of any failures along the way.

---

## AI Industry Note (2026)

In 2026, every senior backend engineer is expected to be able to ship to a cloud without ceremony. The tools have changed the ergonomics — Claude Code can write a Terraform module, a GitHub Action, and an NGINX config in one prompt — but the *concepts* (IAM, regions, AZs, DNS, TLS) are still the same. The model can write the YAML; you have to know whether the YAML is *correct* and *secure*.

The hiring signal in 2026: "I shipped a real product to a real cloud, with a real domain, and I can debug it from a terminal." That story is worth more than any certificate. This module is the shortest path to that story.

---

*Next: the Capstone — a multi-tenant AI SaaS, end to end.*
