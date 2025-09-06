# 🚀 Next.js + Docker Deployment Guide

Hướng dẫn setup, deploy và CI/CD đơn giản cho dự án **Next.js** với **Docker + Docker Compose**.

---

## 📦 Yêu cầu
- [Docker](https://docs.docker.com/get-docker/) & [Docker Compose](https://docs.docker.com/compose/install/)
- Git
- VPS hoặc server Linux (Ubuntu, Arch Linux, etc...)

---

## ⚙️ 1. Cấu hình môi trường và file `.dockerignore`

Tạo file `.env` ở thư mục gốc:

```env

# Auth
NEXTAUTH_URL=https://ui.vhtech.vn # Production
#NEXTAUTH_URL=http://ip-local:3000 # Local
#NEXTAUTH_URL=http://localhost:3000 # Docker
NEXTAUTH_SECRET=your-secret-key # Tạo ngẫu nhiên bằng `openssl rand -base64 32`

# API backend
LINK_API=https://odoo.vhtech.vn # Production
#LINK_API=http://ip-local:8069 # Local
#LINK_API=http://localhost:8069 # Docker

```

Tạo file `.dockerignore` ở thư mục gốc:

```.dockerignore
node_modules
.git
Dockerfile
docker-compose*.yml
npm-debug.log
.next
```

---

## 2. Setup với Docker File multi-stage

Tạo file `Dockerfile` ở thư mục gốc:

```Dockerfile
# Stage 1: Build app
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

# Stage 2: Run (Node server)
FROM node:20-alpine AS runner

WORKDIR /app
ENV NODE_ENV=production

COPY package*.json ./
RUN npm install --omit=dev

# Copy only necessary files
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public

EXPOSE 3000

CMD ["npm", "start"]

```

---

## 3. Setup với Docker Compose

Tạo file `docker-compose.yml` ở thư mục gốc:

```docker-compose.yml

services:
  nextapp:
    container_name: nextapp
    build:
      context: .
      dockerfile: Dockerfile
    env_file:
      - .env
    ports:
      - "3000:3000"
    restart: always
    network_mode: "host"

```

---

## 4. Chạy ở local

```bash
# build
docker compose build

# run
docker compose up -d

# check logs
docker compose logs -f

# stop
docker compose down

```

Lưu ý: Khi thay đổi giá trị cấu hình trong file `.env` cần rebuild lại image để tránh container dùng lại image cũ → dẫn đến NEXTAUTH_URL, NEXTAUTH_SECRET, LINK_API vẫn giá trị cũ.  

Điều này sẽ khiến token sign/verify không trùng → "Invalid Token".

---

## 5. Chạy ở VPS

```bash

# Clone repo
git clone https://github.com/your-org/your-repo.git

# Chuyen vao thu muc
cd your-repo

# Tao file .env như bước 1
nano .env

# Build and run
docker compose build && docker compose up -d

# Update khi code thay đổi
git pull origin main
docker compose build && docker compose up -d

# Xoá image cũ
docker image prune -a -f

```

---

## 6. CI/CD và quản lý container với Watchtower and Portainer

Tham khảo [Watchtower](https://github.com/containrrr/watchtower) và [Portainer](https://docs.portainer.io/start/install-ce/server/docker/linux)  

Đầu tiên cần build image ở local với file `Dockerfile` như bước 2. Sau khi build xong, cần push image lên GHCR. (image nên để private)

```bash

# Login GitHub Container Registry
docker login ghcr.io -u your-username -p my-token

# Build and push
docker build -t ghcr.io/your-username/nextapp:latest .
docker push ghcr.io/your-username/nextapp:latest

```

Trên VPS cần có file `docker-compose.yml` với file `.env` và cần login với GHCR để pull image private về.

```bash

# Login GitHub Container Registry
docker login ghcr.io -u your-username -p my-token

```

File `docker-compose.yml` mới như sau:

```docker-compose.yml

services:
    watchtower:
        image: containrrr/watchtower
        restart: always
        command:
            - "--label-enable"
            - "--interval"
            - "30"
            - "--rolling-restart"
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
        networks:
            - next_network

    nextapp:
        container_name: nextapp
        image: ghcr.io/your-username/nextapp:latest
        env_file:
            - .env
        ports:
            - "3000:3000"
        restart: always
        labels:
            - "com.centurylinklabs.watchtower.enable=true"
        networks:
            - next_network
    # Tuỳ chọn không bắt buộc
    portainer:
        image: portainer/portainer-ce:latest
        container_name: portainer
        restart: always
        ports:
            - 9000:9000
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - portainer_data:/data
        networks:
            - next_network

networks:
    next_network:
        driver: bridge

volumes:
    portainer_data:

```

Lưu ý cần mở port 9000 cho Portainer với UFW:

```bash
sudo ufw allow 9000/tcp
```

Giờ đây mỗi khi update code chỉ cần build && push image lại thì Watchtower sẽ tự động phát hiện thay đổi của code trong image sau đó pull image về và tự động remove && run container.

```bash

# Build and push rút gọn
docker build . -t ghcr.io/your-username/nextapp:latest --push

```
---

## 7. Setup CI/CD với GitHub Actions

Tao file `.github/workflows/deploy.yml` ở thư mục gốc:

```yaml

name: CI/CD Next.js with Docker Compose (GHCR)

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      # Login GitHub Container Registry
      - name: Log in to GHCR
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build Docker image
        run: docker build -t ghcr.io/${{ github.repository_owner }}/nextapp:latest .

      - name: Push Docker image
        run: docker push ghcr.io/${{ github.repository_owner }}/nextapp:latest

  deploy:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Deploy to VPS via SSH
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /home/${{ secrets.VPS_USER }}/nextapp   # thư mục chứa docker-compose.yml và .env
            docker-compose pull nextapp
            docker-compose up -d --force-recreate

```

Khi đó file `docker-compose.yml` như sau:

```docker-compose.yml

services:
  nextapp:
    image: ghcr.io/your-github-username/nextapp:latest
    container_name: nextapp
    restart: always
    env_file:
      - .env
    ports:
      - "3000:3000"
    
```

---
