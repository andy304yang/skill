---
name: github-push
description: 通用 GitHub 推送 + 服务器部署 skill。读取 repo-management 获取目标仓库的部署配置，通过 GitHub API 推送代码，然后 SSH 到服务器执行构建和部署。适用于任何在 repo-management 中注册过的仓库。
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [GitHub, push, deploy, CI/CD, automation]
    requires: [repo-management]
---

# GitHub Push Skill

通用的一键推送 + 部署流程。读取 `repo-management` 获取仓库配置，通过 GitHub API 推送代码，然后部署到服务器。

## 前置条件

1. 目标仓库已在 `repo-management` skill 中注册
2. GitHub Token 存在于 `~/.hermes/.env` 的 `GITHUB_TOKEN` 字段

## 快速使用

```bash
# 用户说"推送 Blog"时，执行以下完整流程
# 1. 读取 repo-management 获取 Blog 配置
# 2. 读取 token
# 3. GitHub API 推送
# 4. SSH 部署
```
## 快速使用

```bash
# 用户说"推送 Blog"时，执行以下完整流程
# 1. 读取 repo-management 获取仓库配置
# 2. 先 SSH 部署（服务器已有最新代码，通过 git fetch/reset 拉取）
# 3. GitHub API 推送
# 4. 部署成功后通知用户
```

## 完整流程详解

### Step 0: 确定目标仓库

用户可能说：
- "推送 Blog" → 查 repo-management → Blog 配置
- "部署 Dream" → 查 repo-management → Dream 配置
- 或从当前目录 `.git` 推断

```bash
# 从当前目录推断
cd /home/agentuser/blog-temp
REMOTE_URL=$(git remote get-url origin)
OWNER_REPO=$(echo "$REMOTE_URL" | sed -E 's|.*github\.com[:/]||; s|\.git$||')
# 结果: andy304yang/Blog
```

### Step 1: SSH 先部署（服务器拉取最新代码）

**先部署，后推送。** 服务器代码更新后再推送 GitHub，确保 GitHub 和服务器同步。

根据 repo-management 中的 `deploy_method` 选择部署方式：

#### 方式 A: 单容器 docker (Blog 专用)

```bash
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84 << 'EOF'
cd /home/ubuntu/blog

# 同步代码（如果服务器有本地 commit 分歧，用 reset --hard）
git fetch origin
git reset --hard origin/master

# 清理 builder cache 防止磁盘空间不足
sudo docker builder prune -f

# 构建镜像
sudo docker build --no-cache -t blog-app-new /home/ubuntu/blog

# 重启容器（--network dream_web + --network-alias saylo-blog 供 nginx 解析）
sudo docker stop blog-app && sudo docker rm blog-app
sudo docker run -d --name blog-app \
  --network dream_web \
  --network-alias saylo-blog \
  -p 3001:3000 \
  blog-app-new

# 重启 nginx 让 upstream 生效
sudo docker restart dream-nginx

# 验证域名访问
HTTP_CODE=$(curl -s -o /dev/null -w '%{http_code}' https://mclarenai.cn)
echo "HTTP: $HTTP_CODE"
EOF
```

**⚠️ 关键：`--network-alias saylo-blog` 必须加**，否则 nginx upstream `saylo-blog` 无法解析，会 502。

#### 方式 B: docker compose

```bash
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84 << 'EOF'
cd /home/ubuntu/blog
git pull origin master
sudo docker compose down
sudo docker compose up --build -d
sudo docker ps | grep blog-app
EOF
```

#### 方式 C: git pull + 重启服务

```bash
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84 << 'EOF'
cd /path/to/repo
git pull origin master
sudo systemctl restart your-service
EOF
```

### Step 2: 验证部署成功后再推送 GitHub

部署成功（HTTP 200）后，再推送 GitHub。如果部署失败，**不要推送**，先排查问题。

```bash
# 检查容器状态
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84 "sudo docker ps | grep blog-app"

# 检查域名访问（通过 Nginx，验证 200 才算成功）
curl -sI https://mclarenai.cn/ | head -3
# 期望: HTTP/1.1 200 OK

# 如果域名 502，先验证 IP 直连是否正常
curl -sI http://81.71.29.84:3001 | head -3
```

### Step 3: 读取 Token

```bash
python3 -c "nwith open('/home/agentuser/.hermes/.env') as f:n    for line in f:n        if line.startswith('GITHUB_TOKEN='):n            token = line.split('=', 1)[1].strip()n            with open('/tmp/token.txt', 'w') as out:n                out.write(token)n            print(f'Token ready, length: {len(token)}')n            breakn"
```

### Step 4: GitHub API 推送

#### 推送单个文件

```python
import urllib.request
import json
import base64
import os

token = open('/tmp/token.txt').read().strip()
repo = 'andy304yang/Blog'
file_path = 'src/lib/posts.ts'
local_full_path = '/home/agentuser/blog-temp/src/lib/posts.ts'

# 1. 获取当前 SHA
def get_sha(path):
    req = urllib.request.Request(
        f'https://api.github.com/repos/{repo}/contents/{path}',
        headers={'Authorization': f'token {token}', 'Accept': 'application/json'}
    )
    try:
        with urllib.request.urlopen(req) as resp:
            return json.loads(resp.read())['sha']
    except Exception as e:
        print(f'File not found or error: {e}')
        return None

# 2. 推送文件
def push_file(github_path, local_path, message):
    sha = get_sha(github_path)
    content = open(local_path).read()
    encoded = base64.b64encode(content.encode()).decode()
    payload = json.dumps({
        'message': message,
        'content': encoded,
        'sha': sha or ''
    }).encode()
    req = urllib.request.Request(
        f'https://api.github.com/repos/{repo}/contents/{github_path}',
        data=payload,
        headers={'Authorization': f'token {token}', 'Accept': 'application/json'},
        method='PUT'
    )
    with urllib.request.urlopen(req) as resp:
        result = json.loads(resp.read())
        print(f"✅ Pushed: {result['commit']['html_url']}")
        return result['commit']['html_url']

push_file('src/lib/posts.ts', '/home/agentuser/blog-temp/src/lib/posts.ts', '更新博客')
```

#### 推送多个文件

```python
import urllib.request, json, base64, os

token = open('/tmp/token.txt').read().strip()
repo = 'andy304yang/Blog'
local_base = '/home/agentuser/blog-temp'

files_to_push = [
    'src/lib/posts.ts',
    'src/app/page.tsx',
]

def get_sha(path):
    req = urllib.request.Request(
        f'https://api.github.com/repos/{repo}/contents/{path}',
        headers={'Authorization': f'token {token}', 'Accept': 'application/json'}
    )
    try:
        with urllib.request.urlopen(req) as resp:
            return json.loads(resp.read())['sha']
    except:
        return None

def push_file(github_path, local_path, message):
    sha = get_sha(github_path)
    content = open(local_path).read()
    encoded = base64.b64encode(content.encode()).decode()
    payload = json.dumps({
        'message': message,
        'content': encoded,
        'sha': sha or ''
    }).encode()
    req = urllib.request.Request(
        f'https://api.github.com/repos/{repo}/contents/{github_path}',
        data=payload,
        headers={'Authorization': f'token {token}', 'Accept': 'application/json'},
        method='PUT'
    )
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())

for file_path in files_to_push:
    full_path = os.path.join(local_base, file_path)
    if os.path.exists(full_path):
        result = push_file(file_path, full_path, f'Update {file_path}')
        print(f"✅ {file_path}")
    else:
        print(f"⏭️  {file_path} not found, skipping")
```

## 错误处理

### 部署失败

- **不要推送 GitHub**，先排查部署问题
- 常见问题：Docker 构建超时、端口被占用、网络配置错误
- 排查顺序：容器日志 → Docker 状态 → Nginx 配置 → 代码变更

### Token 无效

```python
# 测试 token 是否有效
req = urllib.request.Request(
    'https://api.github.com/user',
    headers={'Authorization': f'token {token}'}
)
with urllib.request.urlopen(req) as resp:
    user = json.loads(resp.read())
    print(f"Authenticated as: {user['login']}")
# 如果返回 401，说明 token 无效或过期
```

### SHA 不匹配（文件已被修改）

```python
# 如果 push 返回 409 Conflict，重新获取 SHA 后再 push
sha = get_sha('src/lib/posts.ts')
```

### 部署失败时应急方案：本地构建 + scp 镜像

如果服务器访问 GitHub 网络不稳定（git fetch 超时），改用本地构建 + scp 传输镜像：

```bash
# 1. 本地构建镜像
cd /home/agentuser/blog-temp
git fetch origin && git reset --hard origin/master  # 先同步本地代码
sudo docker build --no-cache -t blog-app-new .

# 2. 导出镜像并传输到服务器
sudo docker save blog-app-new -o /tmp/blog-app-new.tar
cp /tmp/blog-app-new.tar /home/agentuser/blog-app-new.tar
scp -i /home/agentuser/ssh.pem /home/agentuser/blog-app-new.tar ubuntu@81.71.29.84:/home/ubuntu/blog-app-new.tar

# 3. 服务器加载镜像并部署
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84 << 'EOF'
sudo docker load -i /home/ubuntu/blog-app-new.tar
sudo docker stop blog-app && sudo docker rm blog-app
sudo docker run -d --name blog-app \
  --network dream_web \
  --network-alias saylo-blog \
  -p 3001:3000 \
  blog-app-new
sudo docker restart dream-nginx
curl -sI https://mclarenai.cn/ | head -3
EOF
```

**注意**：scp 上传到 `/tmp/` 会有权限问题，服务器路径用 `/home/ubuntu/`。

```bash
# 测试 SSH 连接
ssh -i /home/agentuser/ssh.pem -o ConnectTimeout=10 ubuntu@81.71.29.84 "echo connected"
```
