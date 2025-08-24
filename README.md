# DevOps Engineer Internship – MEAN App (CI/CD + Docker + EC2)

This project containerizes and deploys a full‑stack **MEAN** app (MongoDB, Express/Node backend, Angular frontend) using **Docker**, **Docker Compose**, **Nginx** (as reverse proxy), and **GitHub Actions** for CI/CD to an **Ubuntu EC2** instance.

* **Frontend image**: `slayerop15/frontend:latest` (Angular + Nginx with `/api` proxy)
* **Backend image**: `slayerop15/backend:latest`
* **DB**: `mongo:6` (official image, data persisted in a named volume)

---

## 🧭 Repository Structure

```
.
├─ frontend/
│  ├─ Dockerfile         # build Angular; final stage is Nginx serving SPA + proxy /api → backend
│  └─ nginx.conf         # copied into the frontend image at /etc/nginx/conf.d/app.conf
├─ backend/
│  └─ Dockerfile         # Node/Express app
├─ docker-compose.yml    # uses Docker Hub images + Mongo with a named volume
└─ .github/workflows/cicd.yml  # Build & push → SSH → compose pull/down/up (keeps data)
```

---

## 🌐 Architecture

* **Nginx** is bundled **inside the frontend image** and:

  * serves the Angular SPA on `/`
  * proxies API calls from `/api/*` → `backend:8080`
* **Mongo** runs as `mongo:6` with a named volume `mongo_data`
* Only the **frontend container** exposes a public port (`80:80`)

---

## 🚀 CI/CD (GitHub Actions)

Workflow: `.github/workflows/cicd.yml`

**On push to `main`:**

1. Build `frontend` and `backend` images
2. Push both to Docker Hub (`:latest`)
3. Wait briefly for tag propagation
4. SSH into EC2 and run:

   * `docker compose pull`
   * `docker compose down --remove-orphans` *(no `-v`, so data is kept)*
   * `docker compose up -d`

### Required Repository Secrets

* `DOCKER_HUB_TOKEN` – Docker Hub **access token** for `slayerop15`
* `EC2_SSH_KEY_PEM` – contents of your SSH private key (`.pem`)
* `EC2_TARGET` – **single** value of the form `ubuntu@<EC2-PUBLIC-DNS-OR-IP>`

> **Important:**
> We are **not** using an Elastic IP. If you **stop/start** the EC2 instance, its public IP/DNS will likely change.
> **You must update the `EC2_TARGET` secret** to the new `user@host` before the next deploy.

---

## 🧰 Deployment (what the workflow runs on EC2)

The workflow copies `docker-compose.yml` (if needed) or expects it at `/srv/app`, then executes:

```bash
docker compose pull
docker compose down --remove-orphans   # volumes are preserved
docker compose up -d
```

Your `docker-compose.yml` (images only) looks like:

```yaml
services:
  mongo:
    image: mongo:6
    restart: unless-stopped
    volumes:
      - mongo_data:/data/db

  backend:
    image: slayerop15/backend:latest
    restart: unless-stopped
    environment:
      MONGO_URL: mongodb://mongo:27017/dd_db
      NODE_ENV: production
    depends_on:
      - mongo
    expose:
      - "8080"    # internal only

  frontend:
    image: slayerop15/frontend:latest   # includes Nginx + nginx.conf
    restart: unless-stopped
    depends_on:
      - backend
    ports:
      - "80:80"   # public entrypoint

volumes:
  mongo_data:
```

---

## 🧪 How to Run **Locally** with Docker (no build required)

You can run the exact same stack locally using the published images:

```bash
# 1) Clone the repo
git clone https://github.com/SlayerK15/DevOps-Engineer-Internship-Assignment.git
cd DevOps-Engineer-Internship-Assignment

# 2) Start everything (will pull images if missing)
docker compose up -d

# 3) Verify
docker ps
# Open: http://localhost/
# API (through Nginx proxy): http://localhost/api/...
```

> This uses the **prebuilt** images from Docker Hub.
> If you want to build images locally instead, you’d need a compose file with `build:` entries; for this task we keep it simple and pull the published images.

---

## 📄 Nginx Reverse Proxy (inside frontend image)

The frontend image contains `nginx.conf` similar to:

```nginx
server {
  listen 80;
  server_name _;

  root /usr/share/nginx/html;
  index index.html;

  # Angular SPA fallback
  location / {
    try_files $uri $uri/ /index.html;
  }

  # Proxy API to backend container
  location /api/ {
    proxy_pass http://backend:8080/;    # or /api/ if backend mounts under /api
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

---

## 🔎 Verifications

After a deploy (or running locally):

```bash
docker ps
# expect: frontend (nginx), backend (node), mongo

curl -I http://<EC2-PUBLIC-IP>/
# 200 OK

# If you have a health route:
curl -i http://<EC2-PUBLIC-IP>/api/health
```

---

## 🖼️ Screenshots (attach in screenshots)

Include:

* ✅ GitHub Actions runs (build & deploy jobs green)
* ✅ Docker Hub repos updated (frontend/backend \:latest)
* ✅ EC2 `docker ps` output (containers up)
* ✅ App UI in the browser on `http://<EC2-IP>/`
* ✅ Nginx config snippet (or mention it’s baked into frontend image)

---

## 🔐 Notes on IP changes (important)

Because we’re **not** using an Elastic IP:

* Each time the EC2 instance is **stopped/started**, its public IP/DNS can change.
* **Update the `EC2_TARGET` secret** (e.g., `ubuntu@ec2-xx-xx-xx-xx.ap-south-1.compute.amazonaws.com`) before re‑running the workflow.
* Alternatives:

  * Allocate an **Elastic IP** and associate it with the instance.
  * Put the instance behind a **domain name** (Route53) and update DNS instead.

---

## 🧯 Troubleshooting

* **Actions step “Password required” for Docker login**
  Ensure `DOCKER_HUB_TOKEN` is a **Docker Hub Access Token** (not GitHub PAT) and set under **Secrets**.
* **SSH step shows `@host` missing**
  `EC2_TARGET` secret must be set to `user@host`. Also open **port 22** in the EC2 Security Group.
* **App not reachable**
  Open **port 80** in the Security Group. Check `docker logs` for `frontend` and `backend`.
* **Data lost after deploy**
  Ensure the workflow runs `docker compose down` **without** `-v`. The `mongo_data` named volume persists data.

---

## 📦 Deliverables (quick checklist)

* [x] Dockerfiles (frontend with Nginx, backend)
* [x] `docker-compose.yml` (images + Mongo volume)
* [x] CI/CD workflow (`.github/workflows/cicd.yml`)
* [x] Screenshots (Actions runs, Docker Hub, EC2 `docker ps`, UI)
* [x] README (this file)

---
