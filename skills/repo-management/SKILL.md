---
name: repo-management
description: 仓库部署清单 - 管理所有 GitHub 仓库对应的本地路径、部署方式、服务器信息。当需要推送/部署代码时，从此 skill 读取目标仓库的部署配置。格式：仓库名 → 本地路径 + 部署方式 + 服务器信息。
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [repo, deploy, management, infrastructure]
    maintainer: Hermes Agent
---

# Repo Management Skill

记录所有 GitHub 仓库的本地路径、部署方式、服务器信息。

## 格式说明

```yaml
仓库名:
  github: owner/repo-name       # GitHub 仓库
  local: /path/to/local/dir     # 本地代码路径
  branch: master                # 部署分支
  deploy_method: docker|compose|ssh-pip|single-binary  # 部署方式
  server:                       # 服务器信息（可选）
    ip: 81.71.29.84
    ssh_key: /home/agentuser/ssh.pem
    ssh_user: ubuntu
  docker:                       # Docker 部署（可选）
    container: blog-app
    compose_file: docker-compose.yml
    port: "3001:3000"
    network: dream_web
  build_commands:               # 构建命令（可选）
    - docker build --no-cache -t app:latest .
  deploy_commands:               # 部署命令（可选）
    - docker compose down
    - docker compose up --build -d
```

## 部署方式说明

| deploy_method | 说明 |
|---|---|
| `docker` | 单容器 docker build + docker run |
| `compose` | docker compose up --build |
| `ssh-pip` | SSH 到服务器 git pull + 手动构建 |
| `single-binary` | 单 binary 直接运行 |

---

## 仓库清单

### Blog (个人博客)

```yaml
github: andy304yang/Blog
local: /home/agentuser/blog-temp
branch: master
deploy_method: docker
server:
  ip: 81.71.29.84
  ssh_key: /home/agentuser/ssh.pem
  ssh_user: ubuntu
docker:
  container: blog-app
  port: "3001:3000"
  network: dream_web
  hostname: saylo-blog  # Nginx 网络中的主机名
build_commands:
  - docker build --no-cache -t blog-app:latest .
deploy_commands:
  - docker stop blog-app || true
  - docker rm blog-app || true
  - docker run -d --name blog-app --network dream_web -p 3001:3000 blog-app:latest
nginx_route: /  # Nginx 路由 /
```

### Dream (Excel 智能处理平台)

```yaml
github: andy304yang/dream
local: /home/agentuser/dream-temp
branch: master
deploy_method: compose
server:
  ip: 81.71.29.84
  ssh_key: /home/agentuser/ssh.pem
  ssh_user: ubuntu
docker:
  compose_file: docker-compose.yml
  services:
    - dream-app      # Next.js 前端
    - dream-backend  # FastAPI 后端
    - dream-nginx    # Nginx 反向代理
nginx_routes:
  /: dream-app:3000
  /api/: dream-backend:8000
```

### Component (组件库)

```yaml
github: andy304yang/component
local: /home/agentuser/component-temp
branch: master
deploy_method: docker
server:
  ip: 81.71.29.84
  ssh_key: /home/agentuser/ssh.pem
  ssh_user: ubuntu
docker:
  container: component-app
  port: "3002:80"
  network: dream_web
build_commands:
  - docker build --no-cache -t component-app:latest .
deploy_commands:
  - docker stop component-app || true
  - docker rm component-app || true
  - docker run -d --name component-app --network dream_web -p 3002:80 component-app:latest
nginx_route: /component/  # Nginx 路由 /component/
```

---

## 查询命令

### 列出所有仓库

```bash
grep -A 20 "^## " ~/.hermes/skills/github/repo-management/SKILL.md | grep -E "^### |^github:|^local:|^deploy_method:"
```

### 查找特定仓库

```bash
# 替换 REPO_NAME 为要查找的仓库名（如 Blog, Dream, Component）
awk '/^### REPO_NAME/,/^---/' ~/.hermes/skills/github/repo-management/SKILL.md
```

### 从当前目录推断仓库

```bash
# 从 .git 目录推断当前仓库信息
cd /home/agentuser/blog-temp
OWNER_REPO=$(git remote get-url origin | sed -E 's|.*github\.com[:/]||; s|\.git$||')
echo "Current repo: $OWNER_REPO"
```

---

## 添加新仓库

当用户告知新项目需要部署时：

1. 询问：GitHub 仓库地址、本地路径、部署方式、服务器信息
2. 在此 skill 的 `仓库清单` 部分追加新条目
3. patch 此文件，追加新仓库配置

## 更新仓库配置

当仓库部署信息变更时（如端口改动、路径变化）：

1. patch 此文件中对应仓库的条目
2. 同步更新 `personal-blog-deploy` 或其他相关 deploy skill

## 部署流程（调用其他 skill）

1. 从本 skill 读取目标仓库的 `deploy_method` + `server` 信息
2. 调用对应 deploy skill（如 `personal-blog-deploy`、`dream-deploy`）
3. 或直接执行 `github-push` skill 进行一键推送+部署
