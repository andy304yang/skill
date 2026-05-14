---
name: personal-blog-deploy
description: "Deploy and manage the personal blog (Next.js 15) on Tencent Cloud. Blog repo: github.com/andy304yang/Blog. Server: 81.71.29.84, SSH via /home/agentuser/ssh.pem. Container: blog (port 3000), Nginx reverse proxy. Use when user asks to deploy, update, or manage the personal blog site."
---

# Personal Blog Deploy Skill

Deploy, update, and manage the personal blog at `https://mclarenai.cn` (or `http://81.71.29.84`).

## Blog Info

- **Type**: Next.js 15 personal portfolio/blog
- **GitHub**: `github.com/andy304yang/Blog`
- **Server IP**: `81.71.29.84`
- **SSH**: `ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84`
- **Blog path on server**: `/opt/blog/`
- **Docker container**: `blog` (port 3000)
- **Nginx container**: `dream-nginx` (proxies to `blog:3000`)

## GitHub Auth (for git push from server)

The GitHub PAT is stored in `~/.git-credentials`. Read it before use:

```bash
GITHUB_PAT=$(cat ~/.git-credentials | sed 's|https://||; s|@github.com||')
# Or extract just the token:
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
echo "Token loaded: ${GITHUB_PAT:0:8}..."
```

If credentials file doesn't exist, you need to ask the user for the GitHub PAT.

## Deploy / Update Flow

### Option A: Pull latest from GitHub on server

```bash
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84 << 'EOF'
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
cd /opt/blog
git remote set-url origin https://${GITHUB_PAT}@github.com/andy304yang/Blog.git
git pull origin main

# Rebuild Docker container
sudo docker compose -f /opt/blog/docker-compose.yml down
sudo docker compose -f /opt/blog/docker-compose.yml up -d --build

# Check logs if needed
sudo docker logs blog --tail 20
EOF
```

### Option B: Full redeploy from local build

```bash
# 1. Build locally (from /tmp/blog-site or wherever the code is)
cd /path/to/blog
npm install
npm run build

# 2. Tar and upload to server
tar czf blog-code.tar.gz -C /path/to/blog --exclude='node_modules' --exclude='.next' .
scp -i /home/agentuser/ssh.pem blog-code.tar.gz ubuntu@81.71.29.84:/home/ubuntu/

# 3. Deploy on server
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84 << 'EOF'
cd /opt/blog
sudo rm -rf *
sudo tar xzf /home/ubuntu/blog-code.tar.gz --strip-components=1

# Rebuild and restart
sudo docker compose down
sudo docker compose up -d --build
EOF
```

### Option C: Direct git push to GitHub from local/agent

```bash
# Get token from credentials
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)

cd /path/to/blog
git remote set-url origin https://${GITHUB_PAT}@github.com/andy304yang/Blog.git
git add .
git commit -m "update"
git push origin main
```

## Nginx Config (if needed)

The Nginx container (`dream-nginx`) proxies `mclarenai.cn` to `blog:3000`. Config location inside container: `/etc/nginx/http.d/default.conf`.

If you need to update Nginx config:
```bash
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84
sudo docker cp dream-nginx:/etc/nginx/http.d/default.conf /tmp/default.conf
# Edit /tmp/default.conf, change upstream blog:3000 if needed
sudo docker cp /tmp/default.conf dream-nginx:/etc/nginx/http.d/default.conf
sudo docker exec dream-nginx nginx -s reload
```

## Verify Deployment

```bash
curl -s http://localhost:3000 | head -20
# Or from outside:
curl -s -k https://mclarenai.cn | head -20
```

## Troubleshooting

- **Container won't start**: `sudo docker logs blog`
- **Port 3000 not responding**: Check if container is running `sudo docker ps | grep blog`
- **Build fails**: Check Node version - needs Node 18+
- **Nginx 502**: Check if `blog` container is healthy `sudo docker inspect blog`
