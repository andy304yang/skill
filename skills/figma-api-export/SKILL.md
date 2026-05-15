---
name: figma-api-export
description: Extract nodes and export images from Figma files via REST API. Requires a Figma personal access token. Use when user shares a Figma file URL or node ID and wants to extract/download images programmatically.
triggers:
  - figma
  - extract figma
  - download figma image
  - figma api
  - figd_
---

# Figma API Export

Extract nodes, inspect designs, and download rendered images from Figma files using the REST API.

## 1. Get a Figma Personal Access Token

1. Go to Figma → Account Settings → Account → **Personal access tokens**
2. Click **Generate new token** → name it → copy the token (starts with `figd_`)
3. Provide the token to the agent

Token 从 `~/.figma-token` 文件读取（每行一个 token，支持多行追加）
```bash
FIGMA_TOKEN=$(cat ~/.figma-token)
```

## 2. API Base

- Base URL: `https://api.figma.com/v1`
- Auth: `-H "X-Figma-Token: <token>"`

## 3. Find the File Key

From URL: `https://www.figma.com/design/<FILE_KEY>/...`
Example: `N6bUNhjD2efx8x2B3xjWMC` from `https://www.figma.com/design/N6bUNhjD2efx8x2B3xjWMC/...`

## 4. Key Endpoints

### Get file structure (check node exists)
```bash
curl -s -H "X-Figma-Token: <token>" \
  "https://api.figma.com/v1/files/<FILE_KEY>?depth=2"
```

### Get specific nodes (returns full document tree)
```bash
curl -s -H "X-Figma-Token: <token>" \
  "https://api.figma.com/v1/files/<FILE_KEY>/nodes?ids=<NODE_ID>&depth=10"
```

### Export node as image
```bash
curl -s -G "https://api.figma.com/v1/images/<FILE_KEY>" \
  --data-urlencode "ids=<NODE_ID>" \
  --data-urlencode "format=png" \
  --data-urlencode "scale=2" \
  -H "X-Figma-Token: <token>"
```
Response: `{"err":null,"images":{"<NODE_ID>":"https://..."}}`

## 5. Node ID Format

Figma URL node-id: `107-4728` → API format: `107:4728`
File key is **not** part of the node ID in the API.

## 6. Identify Image Nodes

- **`fills` with `type: "IMAGE"`** → raster images in RECTANGLE/COMPONENT/INSTANCE nodes
- **`type: "VECTOR"`** → SVG paths, not raster images
- **`type: "TEXT"`** → text layers
- **`type: "FRAME"`** → frames/groups

Walk the document tree recursively and collect fills of type IMAGE to find all embedded images.

## 7. Download and Upload

```bash
# Download
curl -sL "<image_url>" -o /tmp/figma_export.png

# Upload to COS
python3 /tmp/cos_upload.py /tmp/figma_export.png <dest_path>
```

## Common Issues

- **403 from browser**: Figma blocks CloudFront bots. Use API instead.
- **Node not found**: Try higher depth (depth=10) or query the file first to locate the node.
- **Empty image export**: Node might be VECTOR (SVG), not raster. Vectors can't be exported as PNG — they're part of the SVG response.
- **`role: viewer`**: Token has read-only access, sufficient for export.
