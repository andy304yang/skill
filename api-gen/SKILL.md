---
name: api-gen
description: Generate a type-safe TypeScript API client from a Swagger/OpenAPI spec. Sets up a custom Axios HTTP client with auth, error handling, and global error events. Configures TanStack Query (useQuery/useMutation) patterns. Use when the user wants to: generate API client from swagger, create typed API hooks, set up axios interceptors, integrate OpenAPI with React Query, auto-generate frontend API code.
---

# API Generator Skill

Generate a type-safe TypeScript API client from a Swagger/OpenAPI spec, set up a custom HTTP client with error handling, and configure TanStack Query patterns.

## Usage

```
/api-gen <swagger-url> [service-name] [output-dir]
```

**Arguments** (from `$ARGUMENTS`):
- `swagger-url` — Remote Swagger JSON/YAML URL (required)
- `service-name` — Name of the backend service, e.g. `user`, `order` (default: `api`)
- `output-dir` — Where to write generated files (default: `src/api/<service-name>`)

---

## Instructions

You are helping the user generate a frontend API client from a Swagger spec. Follow each step in order.

### Step 0 — Parse Arguments

Parse `$ARGUMENTS`:
- First token = swagger URL
- Second token = service name (default: `api`)
- Third token = output dir (default: `src/api/<service-name>`)

If no swagger URL is provided, ask the user for it before continuing.

---

### Step 1 — Check Prerequisites

Run the following checks and fix any issues before proceeding:

```bash
# Check openapi-generator
which openapi-generator || brew install openapi-generator

# Check qxun-api-generator
npx qxun-api-generator --version 2>/dev/null || npm install -g qxun-api-generator
```

---

### Step 2 — Create HTTP Client File

Check if `src/utils/http.ts` (or `src/utils/request.ts`) already exists. If it does, skip this step and use the existing path as `httpPath`.

If it does NOT exist, create `src/utils/http.ts` with this content:

```typescript
import axios, {
  type AxiosInstance,
  type AxiosRequestConfig,
  type AxiosResponse,
} from 'axios'

// ---- Error code messages ----
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

const http: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL ?? '',
  timeout: 15_000,
  headers: { 'Content-Type': 'application/json' },
})

// ---- Request interceptor ----
http.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

// ---- Response interceptor ----
http.interceptors.response.use(
  (response: AxiosResponse) => response,
  (error) => {
    const status: number | undefined = error.response?.status
    const serverMessage: string | undefined = error.response?.data?.message

    const message =
      serverMessage ?? (status ? ERROR_MESSAGES[status] : undefined) ?? '网络错误，请稍后重试'

    // Emit a global error event — UI layer can listen and show toast/modal
    window.dispatchEvent(new CustomEvent('api-error', { detail: { status, message } }))

    // Redirect to login on 401
    if (status === 401) {
      localStorage.removeItem('token')
      window.location.href = '/login'
    }

    return Promise.reject(new Error(message))
  },
)

export default http
```

---

### Step 3 — Create or Update `api.json`

Read the existing `api.json` at the project root (if any). Add or update the entry for this service. Do NOT remove existing entries.

The new entry:

```json
{
  "service": "<service-name>",
  "outputDir": "<output-dir>",
  "path": "<swagger-url>",
  "httpPath": "src/utils/http.ts"
}
```

Full `api.json` shape:

```json
{
  "apis": [
    {
      "service": "<service-name>",
      "outputDir": "<output-dir>",
      "path": "<swagger-url>",
      "httpPath": "src/utils/http.ts"
    }
  ]
}
```

---

### Step 4 — Run the Generator

```bash
npx qxun-api-generator
```

If the Swagger endpoint requires auth, prompt the user for the Authorization token and run:

```bash
npx qxun-api-generator --auth "Bearer <token>"
```

Report which services succeeded and which failed.

---

### Step 5 — Show TanStack Query Usage Pattern

After generation succeeds, print the following usage guide for the user and any future AI agent working in this project:

---

#### How to call APIs in this project

**Rule: always look up available APIs in the Swagger docs first (`<swagger-url>`), then use the generated typed client in `<output-dir>`.**

Never hand-write fetch/axios calls. Use the generated API classes.

##### GET request — `useQuery`

```typescript
import { useQuery } from '@tanstack/react-query'
import { UserApi } from '@/api/<service-name>'

const userApi = new UserApi()

export function useUserProfile(userId: string) {
  return useQuery({
    queryKey: ['user', 'profile', userId],
    queryFn: () => userApi.getUserById(userId).then((res) => res.data),
  })
}
```

##### POST / PUT / DELETE — `useMutation`

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { UserApi } from '@/api/<service-name>'

const userApi = new UserApi()

export function useUpdateUser() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (payload: UpdateUserDto) => userApi.updateUser(payload).then((res) => res.data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['user'] })
    },
  })
}
```

##### Error handling

HTTP errors are intercepted globally in `src/utils/http.ts`. The `api-error` custom event fires on every error — wire it up in your root component:

```typescript
useEffect(() => {
  const handler = (e: CustomEvent) => toast.error(e.detail.message)
  window.addEventListener('api-error', handler as EventListener)
  return () => window.removeEventListener('api-error', handler as EventListener)
}, [])
```

---

### Step 6 — Final summary

Tell the user:
- Which files were created or updated
- The path to the generated API client
- Reminder: run the generator again (`/api-gen <swagger-url>`) whenever the backend updates the Swagger spec
