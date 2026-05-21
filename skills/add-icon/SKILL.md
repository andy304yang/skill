---
name: add-icon
description: Add SVG icons to the component library. Accepts a Figma node URL (auto-downloads SVG via Figma REST API) or a local SVG file path, then runs SVGR + barrel generation automatically.
---

# Add Icon to Component Library

This skill adds icons to the component library at `/Users/andy/Component`.

## Trigger

Use when the user provides:
- A Figma URL (e.g. `figma.com/design/...?node-id=...`) and wants to add the icon
- A local SVG file path and wants to add it to the component library
- Says something like "add this icon", "把这个图标加进去", "更新图标库"

## Workflow

### Step 1: Detect input type

**Figma URL** — contains `figma.com`:
- Extract `fileKey` from URL path segment after `/design/`
- Extract `nodeId` from `node-id=` query param, convert `-` to `:` (e.g. `0-69` → `0:69`)
- Ask user for icon name if not provided (kebab-case, e.g. `icon-join`)

**Local SVG file** — a file path ending in `.svg`:
- If file is already in `src/icons/svg/`, proceed directly to Step 3
- If file is elsewhere, copy it to `src/icons/svg/<name>.svg`

### Step 2: Download SVG from Figma (Figma URL only)

Read the Figma access token from `~/.claude/mcp.json`:
```bash
FIGMA_TOKEN=$(cat ~/.claude/mcp.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['mcpServers']['figma']['env']['FIGMA_ACCESS_TOKEN'])")
```

Call Figma REST API to export node as SVG:
```bash
NODE_ID_ENCODED=$(python3 -c "import urllib.parse; print(urllib.parse.quote('NODE_ID'))")
RESPONSE=$(curl -s "https://api.figma.com/v1/images/FILE_KEY?ids=${NODE_ID_ENCODED}&format=svg" \
  -H "X-Figma-Token: ${FIGMA_TOKEN}")
SVG_URL=$(echo "$RESPONSE" | python3 -c "import json,sys; d=json.load(sys.stdin); imgs=d.get('images',{}); print(list(imgs.values())[0] if imgs else '')")
```

Download the SVG:
```bash
curl -s "$SVG_URL" -o "/Users/andy/Component/src/icons/svg/ICON_NAME.svg"
```

Verify the downloaded file is valid SVG (starts with `<svg`):
```bash
head -c 10 "/Users/andy/Component/src/icons/svg/ICON_NAME.svg"
```

### Step 3: Generate component

```bash
cd /Users/andy/Component && npm run generate:icons
```

### Step 4: Publish to npm

After generating the component, bump the patch version and publish:

```bash
# Bump patch version (0.0.1 → 0.0.2)
cd /Users/andy/Component && npm version patch --no-git-tag-version

# Build library
npm run build:lib

# Publish
npm publish --access public
```

If the user says "don't publish" or "先不发布", skip this step.

### Step 5: Confirm result

Show the user:
- Which `.svg` was saved
- Which `.tsx` component was generated
- The export name (e.g. `AttachmentIcon`) from `src/icons/index.ts`
- The new npm version published (e.g. `@andy304/icons@0.0.2`)

## Notes

- SVG files go in `src/icons/svg/` using kebab-case naming (e.g. `icon-join.svg`)
- `npm run generate:icons` runs SVGR + regenerates `src/icons/index.ts` automatically
- The SVGR template (`svgr.template.cjs`) handles `size`/`width`/`height` props correctly
- The Figma token is stored at `~/.claude/mcp.json` under `mcpServers.figma.env.FIGMA_ACCESS_TOKEN`
- If multiple node IDs are provided at once, pass them comma-separated to the Figma API and save each SVG with its own name
- For Figma URLs pointing to a frame containing multiple icons, ask the user which specific node to export, or export all icon children
