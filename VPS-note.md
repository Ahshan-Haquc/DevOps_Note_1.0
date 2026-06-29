# Hostinger VPS-এ Docker Project Deploy — সম্পূর্ণ গাইড (বাংলা)

## এই গাইডে কী শিখবেন?

```
আপনার কম্পিউটার                    Hostinger VPS
─────────────────                   ─────────────────────────────────
Next.js project    ──── Deploy ───▶  Docker Container চলছে
Node.js API        ──── Deploy ───▶  Docker Container চলছে
                                     Nginx (reverse proxy)
                                     SSL Certificate (HTTPS)
                                          │
                                     yourdomain.com ◀── ইন্টারনেট
```

---

## অংশ ১: Hostinger Plan কেনা (সংক্ষিপ্ত নোট)

### কোন Plan নেবেন?

| Plan | RAM | CPU | Storage | মাসিক | কীসের জন্য |
|---|---|---|---|---|---|
| **KVM 1** | 4GB | 2 vCPU | 50GB NVMe | ~$5-6 | ছোট project (আপনি এটাই নিয়েছেন) |
| KVM 2 | 8GB | 4 vCPU | 100GB | ~$9 | Medium project |
| KVM 4 | 16GB | 4 vCPU | 200GB | ~$18 | বড় project |

KVM-1 দিয়ে Node.js API + Next.js frontend + MongoDB Atlas চালানো যাবে।

### কেনার ধাপ
```
hostinger.com → VPS Hosting → Plan select
→ OS: Ubuntu 22.04 LTS (অবশ্যই এটা বেছে নিন)
→ Location: Singapore (Bangladesh থেকে কাছে)
→ Payment (VISA/Bkash)
→ Email-এ credentials আসবে:
   IP Address: 41.112.107.1      ← আপনার server-এর IP
   Root Password: +5Z+Rsji8@d/ert7  ← প্রথম login-এর password
   SSH Command: ssh root@41.112.107.1
```

> ⚠️ এই password এবং IP আপনার কাছে secret রাখুন।

---

## অংশ ২: VPS-এ প্রথমবার Connect করা (SSH)

### SSH কী?

SSH (Secure Shell) হলো আপনার কম্পিউটার থেকে দূরের একটা server-এ নিরাপদে connect করার উপায়। এটা দিয়ে আপনি Dhaka-তে বসে Hostinger-এর Singapore server-এ command চালাতে পারবেন।

```
আপনার Laptop                         Hostinger Server (Singapore)
──────────────                        ─────────────────────────────
Terminal খুলুন  ──── SSH (Encrypted) ──▶  Ubuntu Linux চলছে
                                          আপনি এখানে command লিখছেন
```

### Windows-এ Connect করবেন যেভাবে

**Option A: Windows Terminal (Windows 10/11 — সহজ)**
```
Start Menu → "Windows Terminal" বা "PowerShell" খুলুন
```

**Option B: PuTTY (পুরনো Windows)**
```
1. putty.org থেকে PuTTY download করুন
2. Host Name: 41.112.107.1
3. Port: 22
4. Connection Type: SSH
5. Open → Yes → login as: root → password দিন
```

### Mac/Linux-এ Connect করবেন যেভাবে
Terminal app খুলুন (Mac: Spotlight → Terminal)

### এখন Connect করুন

আপনার terminal-এ এই command লিখুন (IP আপনারটা দিন):

```bash
ssh root@41.112.107.1
```

প্রথমবার এই message আসবে:
```
The authenticity of host '41.112.107.1' can't be established.
Are you sure you want to continue connecting (yes/no)?
```
**`yes` লিখে Enter চাপুন।**

তারপর password চাইবে:
```
root@41.112.107.1's password:
```
Hostinger-এর দেওয়া password টাইপ করুন (`+5Z+Rsji8@d/ert7`)।
> ⚠️ টাইপ করার সময় কিছু দেখাবে না — এটা স্বাভাবিক। টাইপ করে Enter চাপুন।

Successfully connect হলে দেখবেন:
```
root@srv123456:~#
```
**অভিনন্দন! আপনি এখন আপনার VPS-এর ভিতরে আছেন।**

---

## অংশ ৩: VPS Initial Setup (একবারের কাজ)

এই setup শুধু **একবার** করতে হবে। এরপর থেকে শুধু deploy করলেই হবে।

### ধাপ ১: System Update করুন

```bash
apt update && apt upgrade -y
```

এটা সব installed software-এর latest security patches নামাবে। কিছুক্ষণ সময় লাগবে।

### ধাপ ২: নতুন User তৈরি করুন (root ছাড়া)

`root` হিসেবে সবসময় কাজ করা বিপজ্জনক। একটা নতুন user তৈরি করবো:

```bash
# নতুন user তৈরি (আপনার নাম দিন, যেমন: ahsan)
adduser ahsan
```

কিছু প্রশ্ন করবে — password দিন, বাকি Enter চাপুন।

```bash
# user-কে sudo permission দিন (admin কাজ করার ক্ষমতা)
usermod -aG sudo ahsan

# নিশ্চিত করুন user তৈরি হয়েছে
id ahsan
```

### ধাপ ৩: Firewall Setup (UFW)

Firewall শুধু দরকারি ports খোলা রাখবে, বাকি সব বন্ধ:

```bash
# ufw install (সাধারণত থাকে, না থাকলে install)
apt install ufw -y

# Default: সব incoming বন্ধ, সব outgoing খোলা
ufw default deny incoming
ufw default allow outgoing

# SSH port খোলা (না খুললে নিজেই locked out হয়ে যাবেন!)
ufw allow ssh        # Port 22

# HTTP এবং HTTPS খোলা (website-এর জন্য)
ufw allow 80         # HTTP
ufw allow 443        # HTTPS

# আপনার app-এর port (যদি directly expose করতে চান)
ufw allow 3000       # Node.js API
ufw allow 3001       # Frontend (যদি দরকার হয়)

# Firewall চালু করুন
ufw enable
# "Command may disrupt existing ssh connections. Proceed with operation (y|n)?" → y

# Status দেখুন
ufw status
```

Output:
```
Status: active

To                         Action      From
──                         ──────      ────
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
3000/tcp                   ALLOW       Anywhere
```

### ধাপ ৪: Docker Install করুন

Ubuntu-তে Docker install করার official পদ্ধতি:

```bash
# পুরনো version থাকলে সরান
apt remove docker docker-engine docker.io containerd runc -y

# Docker install-এর জন্য dependencies
apt install ca-certificates curl gnupg lsb-release -y

# Docker-এর official GPG key যোগ করুন
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# Docker repository যোগ করুন
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# Package list update করুন
apt update

# Docker install করুন
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Docker চলছে কিনা চেক করুন
docker --version
docker compose version
```

Output দেখাবে:
```
Docker version 25.0.0, build abcdef1
Docker Compose version v2.24.0
```

```bash
# আপনার user-কে docker group-এ যোগ করুন (sudo ছাড়া docker চালাতে)
usermod -aG docker ahsan

# Docker boot-এ auto-start হবে
systemctl enable docker
systemctl start docker

# Test করুন
docker run hello-world
```

### ধাপ ৫: নতুন User-এ Switch করুন

এখন থেকে `root`-এ কাজ না করে নতুন user-এ কাজ করবো:

```bash
su - ahsan
# অথবা পুরোপুরি logout করে নতুন user দিয়ে login করুন:
# ssh ahsan@41.112.107.1
```

---

## অংশ ৪: Project Deploy করা

### Deploy-এর কৌশল

দুটো পদ্ধতি আছে। Beginner-দের জন্য **পদ্ধতি ১** সহজ:

```
পদ্ধতি ১ (GitHub → VPS):
আপনার কম্পিউটার → push → GitHub → VPS-এ git pull → docker compose up

পদ্ধতি ২ (Docker Hub):
আপনার কম্পিউটার → docker build → docker push → VPS-এ docker pull → docker compose up
```

**পদ্ধতি ১ (GitHub থেকে Deploy)** বিস্তারিত:

### ধাপ ১: VPS-এ Git Install ও Project Clone করুন

```bash
# Git install করুন (সাধারণত থাকে)
sudo apt install git -y

# আপনার project কোথায় রাখবেন তা ঠিক করুন
mkdir -p /home/ahsan/apps
cd /home/ahsan/apps

# GitHub থেকে project clone করুন
git clone https://github.com/yourusername/your-project.git

# Project folder-এ ঢুকুন
cd your-project

# ফাইলগুলো দেখুন
ls -la
```

### ধাপ ২: Environment Variables সেট করুন

```bash
# .env ফাইল তৈরি করুন (VPS-এ সরাসরি)
nano .env
```

`nano` একটা simple text editor। এখানে আপনার production values লিখুন:

```env
# Production .env

# MongoDB Atlas
MONGODB_URI=mongodb+srv://ahsan:yourpassword@cluster0.abc123.mongodb.net/myapp?retryWrites=true&w=majority

# App
NODE_ENV=production
PORT=3000

# Secrets
JWT_SECRET=your-very-long-random-secret-key-here
NEXTAUTH_SECRET=another-very-long-secret

# URLs
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
NEXTAUTH_URL=https://yourdomain.com
```

**Save করুন:** `Ctrl+X` → `Y` → `Enter`

```bash
# Permission ঠিক করুন (.env শুধু owner পড়তে পারবে)
chmod 600 .env

# .env তৈরি হয়েছে কিনা দেখুন
ls -la .env
```

### ধাপ ৩: Production Docker Compose File

VPS-এ `docker-compose.prod.yml` আছে কিনা দেখুন। না থাকলে তৈরি করুন:

```bash
cat docker-compose.prod.yml
```

আগের গাইডের `docker-compose.prod.yml` ব্যবহার করুন। মূল বিষয় হলো:
- MongoDB Atlas ব্যবহার করলে → MongoDB container নেই, শুধু API container
- Local MongoDB ব্যবহার করলে → সব container

### ধাপ ৪: Build ও Run করুন

```bash
# Production image build করুন ও চালান
docker compose -f docker-compose.prod.yml up -d --build

# চলছে কিনা দেখুন
docker compose -f docker-compose.prod.yml ps

# Log দেখুন
docker compose -f docker-compose.prod.yml logs -f
```

এখন আপনার app `http://41.112.107.1:3000` এ চলছে!

Browser-এ গিয়ে test করুন: `http://41.112.107.1:3000`

---

## অংশ ৫: Nginx দিয়ে Reverse Proxy Setup

এখন app port `3000`-এ চলছে। কিন্তু আমরা চাই:
- `http://yourdomain.com` → port 3000-এ forward হোক
- `https://` → SSL certificate

এর জন্য **Nginx** ব্যবহার করবো — এটাও Docker container হিসেবে চালাবো।

### Nginx কেন দরকার?

```
ইন্টারনেট                  VPS
──────────               ───────────────────────────────────────
Browser ──▶ yourdomain.com:80/443
                              │
                           Nginx (port 80/443)   ← SSL handle করে
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
              API Container         Frontend Container
              port 3001             port 3000
```

### Project Structure (Nginx সহ)

```
/home/ahsan/apps/your-project/
├── docker-compose.prod.yml
├── nginx/
│   └── default.conf        ← Nginx config
├── .env
└── ... (বাকি project files)
```

### `nginx/default.conf` তৈরি করুন

```bash
mkdir -p nginx
nano nginx/default.conf
```

**শুধু Backend API-র জন্য:**
```nginx
server {
    listen 80;
    server_name api.yourdomain.com;   # আপনার domain দিন

    location / {
        proxy_pass http://api:3000;   # docker service নাম
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**Next.js Frontend + Backend API দুটোর জন্য:**
```nginx
# Frontend
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://frontend:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}

# Backend API
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://api:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Save: `Ctrl+X` → `Y` → `Enter`

### `docker-compose.prod.yml` — Nginx সহ Complete Version

```yaml
# docker-compose.prod.yml (Nginx + API + Optional Frontend)

services:

  # ── Nginx Reverse Proxy ─────────────────────────────────────────────────
  nginx:
    image: nginx:alpine
    container_name: nginx_proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      # SSL certificate (পরে certbot যোগ করবো)
      - certbot-etc:/etc/letsencrypt:ro
      - certbot-var:/var/lib/letsencrypt:ro
      - web-root:/var/www/html:ro
    depends_on:
      - api
    networks:
      - app-network
    restart: unless-stopped

  # ── Node.js API ─────────────────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: node_api
    # ports expose করছি না — Nginx দিয়েই access হবে
    # শুধু internal network-এ accessible
    environment:
      - NODE_ENV=production
      - PORT=3000
      - MONGODB_URI=${MONGODB_URI}
      - JWT_SECRET=${JWT_SECRET}
    networks:
      - app-network
    restart: unless-stopped

  # ── Certbot (SSL Certificate) ────────────────────────────────────────────
  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - certbot-var:/var/lib/letsencrypt
      - web-root:/var/www/html
    depends_on:
      - nginx
    # প্রথমবার certificate নেওয়ার command (নিচে দেখুন)

volumes:
  certbot-etc:
  certbot-var:
  web-root:

networks:
  app-network:
    driver: bridge
```

---

## অংশ ৬: Domain ও SSL Setup

### Domain Hostinger-এ Point করা

**যদি Hostinger থেকেই domain কিনে থাকেন:**
```
Hostinger Dashboard → Domains → আপনার domain → DNS Zone
→ A Record:
   Type: A
   Name: @          (root domain, যেমন yourdomain.com)
   Value: 41.112.107.1   (আপনার VPS IP)
   TTL: 3600

→ আরেকটা A Record:
   Type: A
   Name: api         (subdomain, যেমন api.yourdomain.com)
   Value: 41.112.107.1
   TTL: 3600

→ আরেকটা A Record:
   Type: A
   Name: www
   Value: 41.112.107.1
   TTL: 3600
```

DNS propagation-এ ৫ মিনিট থেকে ৪৮ ঘন্টা লাগতে পারে। Check করুন:
```bash
# VPS-এ বসে check করুন
nslookup yourdomain.com
# অথবা
ping yourdomain.com
```

### SSL Certificate নেওয়া (HTTPS)

Domain point হওয়ার পর SSL certificate নিন:

```bash
# প্রথমে app চালু করুন (Nginx চলতে হবে)
docker compose -f docker-compose.prod.yml up -d nginx api

# Certbot দিয়ে certificate নিন
docker compose -f docker-compose.prod.yml run --rm certbot \
  certonly --webroot \
  --webroot-path=/var/www/html \
  --email your@email.com \
  --agree-tos \
  --no-eff-email \
  -d yourdomain.com \
  -d www.yourdomain.com \
  -d api.yourdomain.com
```

সফল হলে দেখাবে:
```
Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/yourdomain.com/fullchain.pem
```

এখন `nginx/default.conf`-এ HTTPS যোগ করুন:

```nginx
# HTTP → HTTPS redirect
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com api.yourdomain.com;
    return 301 https://$host$request_uri;
}

# Frontend (HTTPS)
server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://frontend:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# API (HTTPS)
server {
    listen 443 ssl;
    server_name api.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://api:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Nginx reload করুন:
```bash
docker compose -f docker-compose.prod.yml restart nginx
```

এখন `https://yourdomain.com` কাজ করবে!

---

## অংশ ৭: Code Update Deploy করা (Redeploy)

নতুন code push করার পর VPS-এ আপডেট করতে:

```bash
# VPS-এ SSH করুন
ssh ahsan@41.112.107.1

# Project folder-এ যান
cd /home/ahsan/apps/your-project

# Latest code pull করুন
git pull origin main

# নতুন image build করে চালান
docker compose -f docker-compose.prod.yml up -d --build

# পুরনো unused image মুছুন (disk space বাঁচান)
docker image prune -f
```

---

## অংশ ৮: Daily Operations

### Log দেখা
```bash
# সব service-এর log
docker compose -f docker-compose.prod.yml logs

# নির্দিষ্ট service
docker compose -f docker-compose.prod.yml logs -f api

# শেষ ১০০ লাইন
docker compose -f docker-compose.prod.yml logs --tail 100 api
```

### Status দেখা
```bash
docker compose -f docker-compose.prod.yml ps

# Server resources
docker stats
```

### Container Restart করা
```bash
docker compose -f docker-compose.prod.yml restart api
```

### সব বন্ধ করা
```bash
docker compose -f docker-compose.prod.yml down
```

### Disk Space চেক করা
```bash
df -h                      # Server disk
docker system df           # Docker disk usage
docker system prune -f     # Unused resource মুছুন
```

---

## সম্পূর্ণ Flow — একনজরে

```
┌─────────────────────────────────────────────────────────────────────────┐
│ একবারের কাজ (Setup)                                                    │
│                                                                         │
│  1. ssh root@41.112.107.1                                               │
│  2. apt update && apt upgrade -y                                        │
│  3. adduser ahsan → usermod -aG sudo ahsan                              │
│  4. ufw allow ssh/80/443 → ufw enable                                   │
│  5. Docker install                                                       │
│  6. git clone <your-project>                                            │
│  7. .env file তৈরি                                                     │
│  8. docker compose up -d --build                                        │
│  9. Domain DNS point করুন                                              │
│ 10. SSL certificate নিন                                                 │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ প্রতিবার Deploy (Code Update)                                          │
│                                                                         │
│  1. Local-এ code লিখুন                                                 │
│  2. git push origin main                                                │
│  3. ssh ahsan@41.112.107.1                                              │
│  4. cd /home/ahsan/apps/your-project                                    │
│  5. git pull                                                            │
│  6. docker compose -f docker-compose.prod.yml up -d --build            │
│  7. docker image prune -f                                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Commands

```bash
# ── VPS Connect ──────────────────────────────────────────────────────────
ssh ahsan@41.112.107.1

# ── Project ──────────────────────────────────────────────────────────────
cd /home/ahsan/apps/your-project
git pull origin main

# ── Deploy ───────────────────────────────────────────────────────────────
docker compose -f docker-compose.prod.yml up -d --build   # Deploy/Update
docker compose -f docker-compose.prod.yml ps              # Status
docker compose -f docker-compose.prod.yml logs -f api     # Logs
docker compose -f docker-compose.prod.yml down            # বন্ধ করা

# ── Cleanup ──────────────────────────────────────────────────────────────
docker image prune -f       # পুরনো image মুছুন
docker system prune -f      # সব unused resource মুছুন
```

---

## সমস্যা ও সমাধান

| সমস্যা | কারণ | সমাধান |
|---|---|---|
| SSH connect হচ্ছে না | Password ভুল / IP ভুল | Hostinger dashboard থেকে IP/password confirm করুন |
| Port 3000 access হচ্ছে না | Firewall বন্ধ | `ufw allow 3000` |
| MongoDB connect হচ্ছে না | Atlas IP whitelist নেই | Atlas-এ VPS IP (41.112.107.1) add করুন |
| Domain কাজ করছে না | DNS propagate হয়নি | ১৫-৩০ মিনিট অপেক্ষা করুন |
| SSL error | Certificate নেওয়া হয়নি | Certbot command চালান |
| `docker: permission denied` | docker group-এ নেই | `usermod -aG docker ahsan` তারপর re-login |
| Disk full | পুরনো image জমে আছে | `docker system prune -f` |
