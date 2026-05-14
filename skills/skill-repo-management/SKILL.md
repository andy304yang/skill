---
name: skill-repo-management
description: "Manage the skill monorepo at github.com/andy304yang/skill. Clone, add skills, commit, push. Also covers Blog (andy304yang/Blog), Component (andy304yang/Component) management. Use when user asks to add, update, or sync skills to the skill monorepo, or manage GitHub repos."
---

# Skill Repository Management

Manage skills in the monorepo `github.com/andy304yang/skill`. Each skill lives in `skills/<skill-name>/SKILL.md`.

Also manages:
- `github.com/andy304yang/Blog` — Next.js 15 personal blog (branch: `master`)
- `github.com/andy304yang/Component` — Vite + React component library (branch: `main`)
- `github.com/andy304yang/skill` — Skill monorepo (branch: `main`)

## GitHub Auth

The GitHub PAT is stored in `~/.git-credentials`. Read it before git operations:

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
echo "Token: ${GITHUB_PAT:0:8}..."
```

## IMPORTANT: No Secrets in Skill Files

Do NOT write actual tokens or secrets into skill markdown files. GitHub has push protection that scans commits for secrets (including `ghp_` tokens) and will block the entire push if found. All secrets must be extracted from `~/.git-credentials` at runtime.

## Clone Any Repo

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
REPO=$1  # e.g. skill, Blog, Component
DEST=${2:-/tmp/$REPO}  # default destination

git clone https://${GITHUB_PAT}@github.com/andy304yang/$REPO.git "$DEST"
```

## Add / Update a Skill

```bash
REPO_DIR=/tmp/skill-repo
mkdir -p "$REPO_DIR/skills/<skill-name>"

# Write SKILL.md into skills/<skill-name>/

cd "$REPO_DIR"
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
git remote set-url origin https://${GITHUB_PAT}@github.com/andy304yang/skill.git

git config user.email "agent@hermes.ai" 2>/dev/null || true
git config user.name "Hermes Agent" 2>/dev/null || true

git add skills/<skill-name>/
git commit -m "feat: add <skill-name> skill"
git push origin main
```

## Sync Latest from Remote

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
cd /tmp/skill-repo
git remote set-url origin https://${GITHUB_PAT}@github.com/andy304yang/skill.git
git pull origin main --ff-only
```

## Pitfalls

- **Push rejected with "repository rule violations"**: You hardcoded a `ghp_` token in a skill file. Rewrite the skill to use `~/.git-credentials` + runtime extraction. Then `git reset --soft HEAD~1`, rewrite, re-add, and push.
- **Author identity unknown**: Set `git config user.email` and `user.name` before committing in fresh clones.
- **GnuTLS recv error (-110) when pushing**: `git config --global --unset http.sslBackend` if it was set to `openssl`. The agent environment may have a stale `openssl` backend config that conflicts with the system GnuTLS. Retry push after unsetting.
- **scp to /tmp/ fails with "Permission denied" on remote server**: Ubuntu's `/tmp/` on the cloud server often restricts write access. Use `/home/ubuntu/` as the scp destination instead (e.g. `scp file ubuntu@host:/home/ubuntu/file`), then `docker cp` from there.

## Skill Format

Each skill needs:
- `SKILL.md` with YAML frontmatter (`name`, `description`)
- Clear trigger conditions in description field
- Numbered steps with exact commands
- Verification steps
- **NO secrets/tokens** — reference `~/.git-credentials` instead

```yaml
---
name: my-skill
description: "Do X. Use when user wants X. Prerequisites: A, B."
---
```
