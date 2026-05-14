---
name: skill-repo-management
description: "Manage the skill monorepo at github.com/andy304yang/skill. Clone, add skills, commit, push. Use when user asks to add, update, or sync skills to the skill monorepo."
---

# Skill Repository Management

Manage skills in the monorepo `github.com/andy304yang/skill`. Each skill lives in `skills/<skill-name>/SKILL.md`.

## GitHub Auth

The GitHub PAT is stored in `~/.git-credentials`. Read it before git operations:

```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
echo "Token: ${GITHUB_PAT:0:8}..."
```

## Clone the Repo

```bash
git clone https://github.com/andy304yang/skill.git /tmp/skill-repo
cd /tmp/skill-repo
git config credential.helper store
```

Or with token:
```bash
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
git clone https://${GITHUB_PAT}@github.com/andy304yang/skill.git /tmp/skill-repo
```

## Add / Update a Skill

```bash
# Create or update skill directory
mkdir -p skills/<skill-name>
# Write SKILL.md into skills/<skill-name>/

# Commit and push
cd /tmp/skill-repo
git add skills/<skill-name>/
git commit -m "feat: add <skill-name> skill"

GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
git remote set-url origin https://${GITHUB_PAT}@github.com/andy304yang/skill.git
git push origin main
```

## Sync Latest from Remote

```bash
cd /tmp/skill-repo
GITHUB_PAT=$(grep -o 'ghp_[^@]*' ~/.git-credentials | head -1)
git remote set-url origin https://${GITHUB_PAT}@github.com/andy304yang/skill.git
git pull origin main
```

## Skill Format

Each skill needs:
- `SKILL.md` with YAML frontmatter (name, description)
- Clear trigger conditions in description
- Numbered steps with exact commands
- Verification steps
- **NO secrets/tokens in the skill file** — reference ~/.git-credentials instead

Example `SKILL.md` frontmatter:
```yaml
---
name: my-skill
description: "Do X. Use when user wants X. Prerequisites: A, B."
---
```

## Important: No Secrets in Skill Files

Do NOT write actual tokens or secrets into skill markdown files. GitHub will block the push. Use placeholders and reference `~/.git-credentials` at runtime.
