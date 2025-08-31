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
NEXTAUTH_URL=https://ui.vhtech.vn
NEXTAUTH_SECRET=your-secret-key

# API backend
LINK_API=https://odoo.vhtech.vn

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
# Stage 1: build app
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

# Stage 2: run app
FROM node:20-alpine AS runner
WORKDIR /app

COPY --from=builder /app ./

EXPOSE 3000

CMD ["npm", "run", "start"]

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
    environment:
      - NODE_ENV=production
    ports:
      - "3000:3000"
    restart: always

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
        build:
            context: .
            dockerfile: Dockerfile
        env_file:
            - .env
        environment:
            - NODE_ENV=production
        ports:
            - "3000:3000"
        restart: always
        labels:
            - "com.centurylinklabs.watchtower.enable=true"
        networks:
            - next_network
    
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
    environment:
      - NODE_ENV=production
    ports:
      - "3000:3000"
    
```
