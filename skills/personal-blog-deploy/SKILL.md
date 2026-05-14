---
name: personal-blog-deploy
description: "Deploy and manage the personal blog (Next.js 15) on Tencent Cloud. Blog repo: github.com/andy304yang/Blog. Server: 81.71.29.84, SSH via /home/agentuser/ssh.pem. Container: blog-app (port 3001), Nginx reverse proxy dream-nginx. Use when user asks to deploy, update, or manage the personal blog site."
---

# Personal Blog Deploy Skill

Deploy, update, and manage the personal blog at `https://mclarenai.cn`.

## Blog Info

- **Type**: Next.js 15 personal portfolio/blog (standalone output)
- **GitHub**: `github.com/andy304yang/Blog` (branch: `master`)
- **Server IP**: `81.71.29.84`
- **SSH**: `ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84`
- **Blog path on server**: `/home/ubuntu/blog/`
- **Docker container**: `blog-app` (port 3001:3000, network: `dream_web`)
- **Nginx hostname**: `saylo-blog` (aliased to blog-app on dream_web network)
- **Nginx container**: `dream-nginx` (HTTPS on 443, proxies `/` → `saylo-blog:3000`)
- **ICP备案**: 粤ICP备2026057233号-1

## GitHub Auth

The GitHub PAT is stored in `~/.git-credentials`. Read it before use:

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
```

## Deploy / Update Flow

### Option A: Pull from GitHub on server + rebuild (recommended)

```bash
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84 << 'EOF'
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
cd /home/ubuntu/blog
git remote set-url origin https://${GITHUB_PAT}@github.com/andy304yang/Blog.git
git pull origin master

# Rebuild Docker image
sudo docker build --no-cache -t blog-app:latest .

# Restart container (on dream_web network)
sudo docker stop blog-app && sudo docker rm blog-app
sudo docker run -d --name blog-app --network dream_web -p 3001:3000 blog-app:latest
EOF
```

### Option B: Local code → push to GitHub → rebuild on server

```bash
# 1. Edit code locally (e.g. /tmp/Blog)
# 2. Commit and push to GitHub
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
cd /path/to/Blog
git remote set-url origin https://${GITHUB_PAT}@github.com/andy304yang/Blog.git
git add . && git commit -m "update" && git push origin master

# 3. Pull on server and rebuild (same as Option A)
```

## Nginx Config Location

Inside container: `/etc/nginx/http.d/default.conf`

To update Nginx config:
```bash
# Copy config out
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84   "sudo docker cp dream-nginx:/etc/nginx/http.d/default.conf /home/ubuntu/default.conf"

# Edit /home/ubuntu/default.conf locally, then:
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84   "sudo docker cp /home/ubuntu/default.conf dream-nginx:/etc/nginx/http.d/default.conf &&    sudo docker exec dream-nginx nginx -t &&    sudo docker exec dream-nginx nginx -s reload"
```

Key routing in Nginx:
- `/` → `saylo-blog:3000` (blog)
- `/dream/` → `dream-app:3000`
- `/component/` → `component-app:80`
- `/api/` → `dream-backend:8000`

## Verify Deployment

```bash
curl -sI https://mclarenai.cn/ | head -3
curl -s https://mclarenai.cn/ | grep -o '粤ICP备[^<]*'
```

## Troubleshooting

- **Container won't start**: `sudo docker logs blog-app`
- **Port conflict**: `sudo docker ps | grep 3001`
- **Build fails**: Check Node version (needs Node 18+), clear `.next` cache
- **Nginx 502**: Check if `blog-app` container is on `dream_web` network and running
- **saylo-blog not resolved**: `sudo docker exec dream-nginx getent hosts saylo-blog`

## Adding ICP备案号 to Footer

In `src/app/page.tsx`, footer section:
```tsx
<p className="text-xs text-slate-700">
  © {new Date().getFullYear()} Saylo · Built with Next.js · 
  <a href="https://beian.miit.gov.cn/" target="_blank" rel="noopener noreferrer" className="hover:text-slate-500 transition-colors">粤ICP备2026057233号-1</a>
</p>
```
