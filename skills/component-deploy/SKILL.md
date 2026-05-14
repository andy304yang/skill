---
name: component-deploy
description: "Deploy and manage the Component library (Vite + React + Storybook) on Tencent Cloud. Repo: github.com/andy304yang/Component. Server: 81.71.29.84. Container: component-app (port 3002). URL: https://mclarenai.cn/component/. Use when user asks to deploy or update the component library."
---

# Component Library Deploy Skill

Deploy, update, and manage the component library at `https://mclarenai.cn/component/`.

## Component Info

- **Type**: Vite + React component library with Storybook
- **GitHub**: `github.com/andy304yang/Component` (branch: `main`)
- **Server IP**: `81.71.29.84`
- **SSH**: `ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84`
- **Deploy path on server**: `/opt/component/`
- **Docker container**: `component-app` (port 3002:80, network: `dream_web`)
- **URL**: `https://mclarenai.cn/component/`

## Build + Deploy Flow

### 1. Build locally

```bash
cd /path/to/Component
npm install
npm run build   # outputs to dist/
```

### 2. Upload to server

```bash
mkdir -p /tmp/component-deploy
cp -r /path/to/Component/dist /tmp/component-deploy/
# Create Dockerfile
cat > /tmp/component-deploy/Dockerfile << 'DOCKERFILE'
FROM nginx:alpine
COPY dist/ /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
DOCKERFILE

# Upload
scp -i /home/agentuser/ssh.pem -r /tmp/component-deploy/* ubuntu@81.71.29.84:/opt/component/
```

### 3. Build Docker image on server

```bash
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84
sudo mkdir -p /opt/component && sudo chmod 777 /opt/component
cd /opt/component && sudo docker build -t component-app:latest .
```

### 4. Run container (on dream_web network)

```bash
sudo docker run -d --name component-app --network dream_web -p 3002:80 component-app:latest
```

### 5. Update Nginx routing

Add to `/etc/nginx/http.d/default.conf` inside `dream-nginx` container, inside the `server { listen 443 ssl; }` block:

```nginx
    location /component/ {
        proxy_pass http://component-app:80/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
```

Then reload Nginx:
```bash
sudo docker exec dream-nginx nginx -t && sudo docker exec dream-nginx nginx -s reload
```

## Update Nginx Config from Local

```bash
# Download current config
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84   "sudo docker cp dream-nginx:/etc/nginx/http.d/default.conf /home/ubuntu/default.conf"

# Add the /component/ location block (after the last location block in the 443 server)
# Then upload:
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84   "sudo docker cp /home/ubuntu/default.conf dream-nginx:/etc/nginx/http.d/default.conf &&    sudo docker exec dream-nginx nginx -t &&    sudo docker exec dream-nginx nginx -s reload"
```

## Verify Deployment

```bash
curl -sI https://mclarenai.cn/component/ | head -3
curl -s http://localhost:3002 | head -5
```

## Troubleshooting

- **404 on /component/**: Check Nginx config was reloaded, component-app container running
- **Container won't start**: `sudo docker logs component-app`
- **Wrong port**: Ensure container is on `dream_web` network (same as dream-nginx)
- **Build fails**: Check Node version, TypeScript errors in src/stories/
