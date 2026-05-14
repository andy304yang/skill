---
name: andy304yang-repos-management
description: "Manage all Andy Yang GitHub repos: andy304yang/Blog (Next.js 15 personal blog), andy304yang/Component (component library, currently empty), andy304yang/skill (skill monorepo). Clone, pull, push, deploy across all repos. GitHub PAT in ~/.git-credentials. Use when user wants to sync, update, or manage any of these repos."
---

# Andy304yang Repos Management

Manage all GitHub repos under `andy304yang` account. GitHub PAT is stored in `~/.git-credentials`.

## Quick Auth

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
echo "Token: ${GITHUB_PAT:0:8}..."
```

## Repo Overview

| Repo | Path (local) | Description |
|------|-------------|-------------|
| `andy304yang/Blog` | `/tmp/Blog` | Next.js 15 personal blog (mclarenai.cn) |
| `andy304yang/Component` | `/tmp/Component` | Component library (empty, under development) |
| `andy304yang/skill` | `/tmp/skill-repo` | Skill monorepo |

## Clone All Repos

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)

git clone https://${GITHUB_PAT}@github.com/andy304yang/Blog.git /tmp/Blog
git clone https://${GITHUB_PAT}@github.com/andy304yang/Component.git /tmp/Component
git clone https://${GITHUB_PAT}@github.com/andy304yang/skill.git /tmp/skill-repo
```

## Clone Single Repo

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
REPO=${1:-Blog}  # Blog, Component, or skill
TARGET="/tmp/$REPO"
git clone "https://${GITHUB_PAT}@github.com/andy304yang/${REPO}.git" "$TARGET"
```

## Push Changes

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
REPO=${1:-skill}
TARGET="/tmp/$REPO"

cd "$TARGET"
git remote set-url origin "https://${GITHUB_PAT}@github.com/andy304yang/${REPO}.git"
git add .
git commit -m "${2:-update}"
git push origin main
```

## Sync / Pull Latest

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
REPO=${1:-skill}
TARGET="/tmp/$REPO"

cd "$TARGET"
git remote set-url origin "https://${GITHUB_PAT}@github.com/andy304yang/${REPO}.git"
git pull origin main --rebase
```

## Blog Deploy (see `personal-blog-deploy` skill for full details)

```bash
# Quick deploy: pull latest + rebuild container on server
ssh -i /home/agentuser/ssh.pem ubuntu@81.71.29.84 << 'EOF'
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
cd /opt/blog
git remote set-url origin https://${GITHUB_PAT}@github.com/andy304yang/Blog.git
git pull origin main
sudo docker compose -f /opt/blog/docker-compose.yml up -d --build
sudo docker logs blog --tail 10
EOF
```

## Skill Repo: Add / Update a Skill

```bash
# 1. Clone if not exists
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
[ ! -d /tmp/skill-repo ] && git clone "https://${GITHUB_PAT}@github.com/andy304yang/skill.git" /tmp/skill-repo

# 2. Create or update skill
SKILL_NAME="my-new-skill"
mkdir -p /tmp/skill-repo/skills/$SKILL_NAME
# Write SKILL.md here...

# 3. Commit and push
cd /tmp/skill-repo
git add skills/$SKILL_NAME/
git commit -m "feat: add $SKILL_NAME skill"
git push origin main
```

## Component Repo

Currently empty. To populate it:

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
cd /tmp/Component

# Example: initialize with a component package structure
mkdir -p src/components
# Add your components...

git add .
git commit -m "feat: initial components"
git push origin main
```

## Important: No Secrets in Files

Never write actual tokens or PATs into any markdown/code files. Use `~/.git-credentials` at runtime. GitHub will block pushes containing secrets.
