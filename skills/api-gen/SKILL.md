---
name: api-gen
description: "Generate a type-safe TypeScript API client from a Swagger/OpenAPI spec using the qxun-api-generator npm package. Creates an api-generator folder, writes api.json config, asks the user for their Swagger URL, then runs npx qxun-api-generator. Use when the user wants to generate API client from swagger, create typed API hooks, auto-generate frontend API code from OpenAPI spec."
---

# API Generator Skill

Generate a type-safe TypeScript API client from a Swagger/OpenAPI spec using the `qxun-api-generator` npm package.

## Usage

```
/api-gen [swagger-url] [service-name]
```

**Arguments** (from `$ARGUMENTS`):
- `swagger-url` — Remote Swagger JSON URL (optional; will ask if not provided)
- `service-name` — Name of the backend service, e.g. `consumer`, `user`, `order` (default: `api`)

---

## Instructions

Follow each step in order.

### Step 0 — Parse Arguments

Parse `$ARGUMENTS`:
- First token = swagger URL (may be empty)
- Second token = service name (default: `api`)

If no swagger URL is provided in `$ARGUMENTS`, ask the user:
> "请提供 Swagger API 文档地址（例如：https://your-server.com/v3/api-docs/service）"

Wait for the user's answer before continuing.

---

### Step 1 — Locate the Project Root and HTTP Utility

Before creating any files, determine two things:

1. **Project root** — look for `package.json` in the current directory or its parents. Use that directory as the project root.

2. **HTTP utility import path** — check if `src/utils/http.ts` or `src/utils/request.ts` exists in the project root.
   - If found, calculate the relative import path from `api-generator/` to that file. For example, if the file is at `src/utils/http.ts` and `api-generator/` is at the project root, the import is `../../src/utils/http`.
   - If NOT found, create `src/utils/http.ts` with the content from the **Appendix** below, and use `../../src/utils/http` as the import path.

Set these two variables for use in later steps:
- `HTTP_IMPORT_PATH` = the relative path calculated above (without extension)
- `API_GENERATOR_DIR` = `<project-root>/api-generator`

---

### Step 1.5 — Ask for SIT and PROD Base URLs

Ask the user:
> "请提供两个环境的 API base URL：
> - SIT（测试环境）base URL，例如 http://sit.example.com
> - PROD（生产环境）base URL，例如 https://api.example.com
>
> 如果暂时不知道可以直接回车跳过，之后手动填写 .env 文件。"

After getting the answers, write/update `.env` at the project root. Only add or update `VITE_API_BASE_URL_SIT` and `VITE_API_BASE_URL_PROD` — do not remove other variables.

```
VITE_API_BASE_URL_SIT=<sit-url>
VITE_API_BASE_URL_PROD=<prod-url>
```

Also update `src/utils/http.ts` (or create it from the Appendix) so it picks the right URL based on the Vite mode:

```typescript
baseURL: import.meta.env.MODE === 'production'
  ? import.meta.env.VITE_API_BASE_URL_PROD
  : import.meta.env.VITE_API_BASE_URL_SIT,
```

---

### Step 2 — Create `api-generator/` Directory and `api.json`

Create the `api-generator/` directory at the project root if it does not already exist.

Then create or update `api-generator/api.json`. If the file already exists, read it first and add the new service entry to the `apis` array without removing existing entries.

The `api.json` content:

```json
{
  "$schema": "../node_modules/qxun-api-generator/lib/schema.json",
  "apis": [
    {
      "service": "<service-name>",
      "outputDir": "../src/apis",
      "path": "<swagger-url>",
      "httpPath": "import { http as globalAxios } from '<HTTP_IMPORT_PATH>'",
      "baseUrl": ""
    }
  ]
}
```

Replace:
- `<service-name>` with the service name (e.g. `consumer`) — only in the `service` field
- `<swagger-url>` with the full Swagger JSON URL
- `<HTTP_IMPORT_PATH>` with the relative import path calculated in Step 1

**Important:**
- `outputDir` is always `"../src/apis"` — the generator automatically creates `src/apis/<service-name>/` using the `service` field value
- `baseUrl` is always `""` — the actual base URL is read at runtime from `VITE_API_BASE_URL` in the axios instance
- The `httpPath` value is a full TypeScript import statement, not a file path

---

### Step 3 — Install qxun-api-generator if Needed

Check if the package is available:

```bash
npx qxun-api-generator --version 2>/dev/null
```

- If it prints a version → already installed, skip.
- If it fails → run: `npm install qxun-api-generator --save-dev` in the project root.

---

### Step 4 — Run the Generator

Change into the `api-generator/` directory and run the generator:

```bash
cd <API_GENERATOR_DIR> && npx qxun-api-generator
```

The generator reads `api.json` from the current directory and outputs the generated TypeScript files into `src/apis/<service-name>/`.

If the command fails with an auth error, ask the user for an Authorization token and retry:

```bash
cd <API_GENERATOR_DIR> && npx qxun-api-generator --auth "Bearer <token>"
```

Report whether the generation succeeded or failed, and list the output files created.

---

### Step 5 — Final Summary

Tell the user:
- The `api.json` location: `api-generator/api.json`
- The generated files location: `src/apis/<service-name>/`
- The HTTP client location (created or existing)
- How to regenerate: run `/api-gen <swagger-url> <service-name>` again whenever the backend updates the Swagger spec, or manually run `cd api-generator && npx qxun-api-generator`

---

## Appendix — Default `src/utils/http.ts`

Only create this file if no HTTP utility file exists in the project.

```typescript
import axios, {
  type AxiosInstance,
  type AxiosResponse,
} from 'axios'

const ERROR_MESSAGES: Record<number, string> = {
  400: '请求参数错误',
  401: '未授权，请重新登录',
  403: '拒绝访问',
  404: '请求资源不存在',
  408: '请求超时',
  409: '数据冲突，请刷新后重试',
  422: '请求数据验证失败',
  429: '请求过于频繁，请稍后再试',
  500: '服务器内部错误',
  502: '网关错误',
  503: '服务不可用',
  504: '网关超时',
}

export const http: AxiosInstance = axios.create({
  baseURL: import.meta.env.MODE === 'production'
    ? import.meta.env.VITE_API_BASE_URL_PROD
    : import.meta.env.VITE_API_BASE_URL_SIT,
  timeout: 15_000,
  headers: { 'Content-Type': 'application/json' },
})

http.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

http.interceptors.response.use(
  (response: AxiosResponse) => response,
  (error) => {
    const status: number | undefined = error.response?.status
    const serverMessage: string | undefined = error.response?.data?.message

    const message =
      serverMessage ?? (status ? ERROR_MESSAGES[status] : undefined) ?? '网络错误，请稍后重试'

    window.dispatchEvent(new CustomEvent('api-error', { detail: { status, message } }))

    if (status === 401) {
      localStorage.removeItem('token')
      window.location.href = '/login'
    }

    return Promise.reject(new Error(message))
  },
)
```
