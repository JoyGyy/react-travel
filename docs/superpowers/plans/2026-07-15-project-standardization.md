# Project Standardization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 React Travel 整理到适合个人长期维护、作品集展示和阿里云 ECS 裸机部署的规范程度。

**Architecture:** 保持当前 Vite SPA + Express API 的前后端分离结构。前端新增统一 API 边界并增强 SSE Hook，后端新增轻量校验、认证和错误处理工具，部署采用 Nginx 托管前端静态资源、PM2 守护后端 Node 服务。

**Tech Stack:** React 19、Vite 8、React Router 7、Ant Design 5、Zustand 5、Node.js、Express 4、better-sqlite3、JWT、pnpm、PM2、Nginx。

## Global Constraints

- 所有回复、文档说明和代码注释使用中文。
- 包管理器使用 pnpm，不使用 npm 或 yarn。
- 当前不改成 pnpm workspace。
- 当前不强制全量 TypeScript 迁移。
- 当前不引入 Docker Compose。
- 当前不重写 RAG 或更换数据库。
- 当前不引入完整测试体系，以 `pnpm check`、`cd server && pnpm check` 和运行时冒烟验证为主。
- Git 提交信息使用中文，不添加 `Co-Authored-By` 或任何 Claude 标记。
- 生产部署目标为阿里云 ECS：Nginx 托管前端静态资源，PM2 守护后端 Node 服务。

---

## File Structure

### 新增文件

- `docs/deployment/aliyun-ecs.md`：阿里云 ECS 裸机部署说明，覆盖 Node、pnpm、PM2、Nginx、环境变量、健康检查和排错。
- `deploy/nginx/react-travel.conf.example`：Nginx 站点配置示例，前端 SPA fallback，`/api` 反代后端。
- `ecosystem.config.cjs`：PM2 示例配置，运行 `server/index.js`。
- `src/api/client.js`：前端统一 JSON 请求客户端，负责认证头、错误解析和 401 处理。
- `src/api/auth.js`：认证接口封装。
- `src/api/weather.js`：天气接口封装。
- `server/utils/http.js`：后端 HTTP 错误对象、异步路由包装、统一错误响应。
- `server/utils/validation.js`：后端轻量字段校验、字符串和数字约束。
- `server/middleware/auth.js`：JWT 认证中间件，向 `req.user` 写入当前用户。

### 修改文件

- `.gitignore`：移除 `docs` 忽略规则，让项目文档可正常纳入版本控制。
- `.env.example`：补充生产部署相关注释和阿里云域名配置示例。
- `README.md`：补充项目亮点、部署入口和规范化后的脚本说明。
- `src/stores/auth.js`：改用 `src/api/auth.js`，保留 Zustand 持久化行为。
- `src/hooks/useWeather.js`：改用 `src/api/weather.js`，保留现有取消请求逻辑。
- `src/hooks/useSSE.js`：增强 Abort、重复请求保护、SSE 错误事件和解析容错。
- `src/pages/Chat/index.jsx`：接入增强后的 SSE 错误信息和重试状态。
- `src/pages/Detail/index.jsx`：接入增强后的 SSE 错误信息和加载收尾逻辑。
- `server/index.js`：挂载统一错误处理中间件，统一 404 响应格式。
- `server/routes/auth.js`：使用校验工具、认证中间件和统一错误处理。
- `server/routes/weather.js`：使用校验工具和统一错误处理。
- `server/routes/travel.js`：在初始化 SSE 前校验推荐请求参数。
- `server/routes/chat.js`：在初始化 SSE 前校验聊天请求参数和历史消息大小。
- `server/routes/share.js`：使用认证中间件、校验工具和统一错误处理。
- `server/package.json`：更新 `check` 脚本，覆盖新增后端工具文件。

---

### Task 1: 工程文档与部署基础

**Files:**
- Modify: `.gitignore`
- Modify: `.env.example`
- Modify: `README.md`
- Create: `docs/deployment/aliyun-ecs.md`
- Create: `deploy/nginx/react-travel.conf.example`
- Create: `ecosystem.config.cjs`

**Interfaces:**
- Consumes: 当前项目端口约定：前端 5181，后端 3030，生产 `/api` 由 Nginx 反代。
- Produces: 可提交的项目文档目录、PM2 配置 `apps[0].name = 'react-travel-server'`、Nginx 示例中的站点根目录 `/var/www/react-travel/dist`。

- [ ] **Step 1: 允许提交项目文档**

修改 `.gitignore`，删除独立的 `docs` 忽略规则，保留 `.DS_Store`、`.env`、`.playwright-mcp` 等本地文件忽略。

预期关键片段：

```gitignore
# 本地 AI 助手配置，不提交到仓库
CLAUDE.md
AGENTS.md
.claude/

server/data/*
!server/data/shared_itineraries.json
.playwright-mcp
```

- [ ] **Step 2: 补充生产环境变量示例**

更新 `.env.example`，保留现有变量，补充生产说明。关键内容：

```env
# 生产环境示例：部署到阿里云 ECS 时改为 production
NODE_ENV=development

# 生产环境建议填写真实域名，多个域名用英文逗号分隔
# CORS_ORIGIN=https://your-domain.com,https://www.your-domain.com
CORS_ORIGIN=http://localhost:5181,http://localhost:3000
```

- [ ] **Step 3: 创建 PM2 配置**

创建 `ecosystem.config.cjs`：

```js
module.exports = {
  apps: [
    {
      name: 'react-travel-server',
      script: 'index.js',
      cwd: './server',
      instances: 1,
      exec_mode: 'fork',
      env: {
        NODE_ENV: 'production',
        PORT: 3030,
      },
      max_memory_restart: '300M',
      error_file: './logs/server-error.log',
      out_file: './logs/server-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss',
    },
  ],
}
```

- [ ] **Step 4: 创建 Nginx 示例配置**

创建 `deploy/nginx/react-travel.conf.example`：

```nginx
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;

    root /var/www/react-travel/dist;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:3030/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_read_timeout 180s;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|webp)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }
}
```

- [ ] **Step 5: 创建阿里云 ECS 部署文档**

创建 `docs/deployment/aliyun-ecs.md`，必须覆盖以下命令：

```bash
pnpm install
cd server && pnpm install
cd ..
pnpm build
pm2 start ecosystem.config.cjs
pm2 save
curl http://127.0.0.1:3030/api/health
```

文档必须说明：

- 备案完成前可以先用 ECS 公网 IP 验证。
- 备案完成后将域名解析到 ECS 公网 IP。
- 安全组至少开放 80、443、SSH 端口。
- `.env` 必须放在仓库根目录，因为后端从根目录读取。
- `JWT_SECRET` 生产环境至少 32 个字符。

- [ ] **Step 6: 更新 README**

更新 `README.md`，新增“部署到阿里云 ECS”入口，链接到 `[docs/deployment/aliyun-ecs.md](docs/deployment/aliyun-ecs.md)`，并说明：

```markdown
## 部署

推荐个人项目首版使用阿里云 ECS 裸机部署：

- Nginx 托管 `dist/` 静态资源
- `/api` 反向代理到本机 `3030` 端口
- PM2 守护 Express 后端

详细步骤见：[阿里云 ECS 部署说明](docs/deployment/aliyun-ecs.md)。
```

- [ ] **Step 7: 运行文档阶段检查**

Run:

```bash
git diff --check
pnpm check
cd server && pnpm check
```

Expected:

- `git diff --check` 无空白错误。
- 前端 lint、typecheck、build 通过。
- 后端 Node 语法检查通过。

- [ ] **Step 8: 提交文档与部署基础**

```bash
git add .gitignore .env.example README.md docs/deployment/aliyun-ecs.md deploy/nginx/react-travel.conf.example ecosystem.config.cjs
git commit -m "docs: 补充阿里云部署说明"
```

---

### Task 2: 前端统一 API 层

**Files:**
- Create: `src/api/client.js`
- Create: `src/api/auth.js`
- Create: `src/api/weather.js`
- Modify: `src/stores/auth.js`
- Modify: `src/hooks/useWeather.js`

**Interfaces:**
- Produces: `request(path, options)`、`getAuthHeader()`、`loginApi(username, password)`、`registerApi(username, password)`、`getWeatherApi(city, options)`。
- Consumes: Zustand persist key `travel_auth`，后端响应 `{ success, message, token, user }`。

- [ ] **Step 1: 创建统一请求客户端**

创建 `src/api/client.js`：

```js
const AUTH_STORAGE_KEY = 'travel_auth'

export class ApiError extends Error {
  constructor(message, { status, data } = {}) {
    super(message)
    this.name = 'ApiError'
    this.status = status
    this.data = data
  }
}

function readPersistedToken() {
  try {
    const raw = localStorage.getItem(AUTH_STORAGE_KEY)
    if (!raw)
      return ''
    const parsed = JSON.parse(raw)
    return parsed?.state?.token || ''
  }
  catch {
    return ''
  }
}

export function getAuthHeader() {
  const token = readPersistedToken()
  return token ? { Authorization: `Bearer ${token}` } : {}
}

async function parseResponse(res) {
  const contentType = res.headers.get('content-type') || ''
  if (!contentType.includes('application/json'))
    return null
  try {
    return await res.json()
  }
  catch {
    return null
  }
}

export async function request(path, options = {}) {
  const {
    method = 'GET',
    body,
    headers,
    auth = false,
    signal,
  } = options

  const res = await fetch(path, {
    method,
    headers: {
      ...(body === undefined ? {} : { 'Content-Type': 'application/json' }),
      ...(auth ? getAuthHeader() : {}),
      ...headers,
    },
    body: body === undefined ? undefined : JSON.stringify(body),
    signal,
  })

  const data = await parseResponse(res)

  if (!res.ok || data?.success === false) {
    const message = data?.message || data?.error || `请求失败: HTTP ${res.status}`
    throw new ApiError(message, { status: res.status, data })
  }

  return data
}
```

- [ ] **Step 2: 创建认证 API 封装**

创建 `src/api/auth.js`：

```js
import { request } from './client'

export function loginApi(username, password) {
  return request('/api/auth/login', {
    method: 'POST',
    body: { username, password },
  })
}

export function registerApi(username, password) {
  return request('/api/auth/register', {
    method: 'POST',
    body: { username, password },
  })
}

export function getMeApi() {
  return request('/api/auth/me', {
    auth: true,
  })
}
```

- [ ] **Step 3: 创建天气 API 封装**

创建 `src/api/weather.js`：

```js
import { request } from './client'

export function getWeatherApi(city, options = {}) {
  return request(`/api/weather?city=${encodeURIComponent(city)}`, {
    signal: options.signal,
  })
}
```

- [ ] **Step 4: 更新认证 store**

修改 `src/stores/auth.js`，把 fetch 调用替换为 API 封装。关键实现：

```js
import { loginApi, registerApi } from '@/api/auth'
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

// ...保留 persist 结构
async login(username, password) {
  const data = await loginApi(username, password)
  set({ user: data.user, token: data.token })
},

async register(username, password) {
  const data = await registerApi(username, password)
  set({ user: data.user, token: data.token })
},
```

- [ ] **Step 5: 更新天气 Hook**

修改 `src/hooks/useWeather.js`，把 fetch 调用替换为 `getWeatherApi`。关键实现：

```js
import { getWeatherApi } from '@/api/weather'

// fetchWeather 内部：
getWeatherApi(city, { signal: controller.signal })
  .then((data) => {
    if (!controller.signal.aborted) {
      setWeather(data)
      setLoading(false)
    }
  })
  .catch((err) => {
    if (err.name !== 'AbortError') {
      setError(err.message || '天气查询失败')
      setLoading(false)
    }
  })
```

- [ ] **Step 6: 运行前端 API 阶段检查**

Run:

```bash
pnpm check
```

Expected: lint、typecheck、build 全部通过。

- [ ] **Step 7: 提交前端 API 层**

```bash
git add src/api/client.js src/api/auth.js src/api/weather.js src/stores/auth.js src/hooks/useWeather.js
git commit -m "refactor: 统一前端接口请求封装"
```

---

### Task 3: SSE 流式请求稳定性

**Files:**
- Modify: `src/hooks/useSSE.js`
- Modify: `src/pages/Chat/index.jsx`
- Modify: `src/pages/Detail/index.jsx`

**Interfaces:**
- Produces: `sendRequest(url, body, callbacks)` 会在发起新请求前终止旧请求；支持 `callbacks.onError(error)`、`callbacks.onFinally()`、`callbacks.onComplete(data)`。
- Consumes: 后端 SSE 行格式 `data: {"type":"chunk|step|notice|complete|error"}`。

- [ ] **Step 1: 增强 useSSE**

修改 `src/hooks/useSSE.js`，关键行为：

```js
import { getAuthHeader } from '@/api/client'
import { useCallback, useRef } from 'react'

export function useSSE() {
  const abortControllerRef = useRef(null)
  const requestIdRef = useRef(0)

  const sendRequest = useCallback(async (url, body, callbacks = {}) => {
    abortControllerRef.current?.abort()
    const requestId = requestIdRef.current + 1
    requestIdRef.current = requestId

    const controller = new AbortController()
    abortControllerRef.current = controller

    const isCurrentRequest = () => requestIdRef.current === requestId && !controller.signal.aborted

    try {
      const res = await fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          ...getAuthHeader(),
        },
        body: JSON.stringify(body),
        signal: controller.signal,
      })

      if (!res.ok) {
        let msg = `请求失败: HTTP ${res.status}`
        try {
          const errData = await res.json()
          msg = errData.message || errData.error || msg
        }
        catch {
          // 错误响应不是 JSON 时，保留默认 HTTP 错误信息
        }
        throw new Error(msg)
      }

      if (!res.body)
        throw new Error('响应体为空')

      const reader = res.body.getReader()
      const decoder = new TextDecoder('utf-8')
      let full = ''
      let remainder = ''

      async function handleLine(line) {
        if (!line.startsWith('data: ') || !isCurrentRequest())
          return

        let data
        try {
          data = JSON.parse(line.slice(6))
        }
        catch {
          throw new Error('服务端返回了无法解析的流式数据')
        }

        if (data.type === 'error')
          throw new Error(data.message || '流式请求失败')
        if (data.type === 'chunk' && callbacks.onChunk) {
          full += data.content || ''
          callbacks.onChunk(full)
        }
        if (data.type === 'step' && callbacks.onStep)
          callbacks.onStep(data)
        if (data.type === 'notice' && callbacks.onNotice)
          callbacks.onNotice(data.message)
        if (data.type === 'complete' && callbacks.onComplete)
          callbacks.onComplete(data.data)
      }

      while (true) {
        const { done, value } = await reader.read()
        if (done) {
          remainder += decoder.decode()
          break
        }

        const chunk = remainder + decoder.decode(value, { stream: true })
        const lines = chunk.split('\n')
        remainder = lines.pop() || ''

        for (const line of lines) {
          await handleLine(line)
        }
      }

      if (remainder.trim())
        await handleLine(remainder)
    }
    catch (err) {
      if (err.name !== 'AbortError' && isCurrentRequest()) {
        callbacks.onError?.(err)
        throw err
      }
    }
    finally {
      if (isCurrentRequest()) {
        abortControllerRef.current = null
        callbacks.onFinally?.()
      }
    }
  }, [])

  const abort = useCallback(() => {
    abortControllerRef.current?.abort()
    abortControllerRef.current = null
    requestIdRef.current += 1
  }, [])

  return { sendRequest, abort }
}
```

- [ ] **Step 2: 更新 Chat 错误回调**

修改 `src/pages/Chat/index.jsx`，给 `sendRequest` 增加 `onError`，避免错误来源丢失：

```jsx
await sendRequest('/api/travel/chat', { message: text, messages: historyForBackend }, {
  onChunk: full => updateLastMessage(full),
  onStep: (data) => {
    setCurrentAgentStep(data.step)
    updateLastMessageSteps(data)
  },
  onNotice: msg => setNotice(msg),
  onComplete: (data) => {
    if (data?.sources?.length)
      setRagSources(data.sources)
  },
  onError: (err) => {
    setLastFailedText(text)
    setChatError(err.message || '请求失败，请检查网络后重试。')
  },
})
```

`catch` 中保留兜底，但不要覆盖已经设置的明确错误：

```jsx
catch {
  setLastFailedText(text)
  setChatError(prev => prev || '请求失败，请检查网络后重试。')
  updateLastMessage('抱歉，刚刚没有连接成功。你可以点击下方按钮重试。')
}
```

- [ ] **Step 3: 更新 Detail 错误收尾**

修改 `src/pages/Detail/index.jsx`，保留 `dataReceived`，但错误回调使用后端文案：

```jsx
onError: (err) => {
  setErrorMessage(err.message || '生成行程失败，请稍后重试')
},
onFinally: () => {
  if (!dataReceived)
    setShowLoading(false)
},
```

`catch` 中保留：

```jsx
.catch((err) => {
  setErrorMessage(err.message || '生成行程失败，请稍后重试')
})
```

- [ ] **Step 4: 运行 SSE 阶段检查**

Run:

```bash
pnpm check
```

Expected: lint、typecheck、build 全部通过。

- [ ] **Step 5: 提交 SSE 稳定性改造**

```bash
git add src/hooks/useSSE.js src/pages/Chat/index.jsx src/pages/Detail/index.jsx
git commit -m "refactor: 增强流式请求稳定性"
```

---

### Task 4: 后端校验、认证与错误处理

**Files:**
- Create: `server/utils/http.js`
- Create: `server/utils/validation.js`
- Create: `server/middleware/auth.js`
- Modify: `server/index.js`
- Modify: `server/routes/auth.js`
- Modify: `server/routes/weather.js`
- Modify: `server/routes/travel.js`
- Modify: `server/routes/chat.js`
- Modify: `server/routes/share.js`
- Modify: `server/package.json`

**Interfaces:**
- Produces: `HttpError`、`httpError(status, message)`、`asyncHandler(handler)`、`notFoundHandler`、`errorHandler`。
- Produces: `requireAuth(req, res, next)`，成功时写入 `req.user = { id, username }`。
- Produces: `readRequiredString(value, fieldName, options)`、`readPositiveInteger(value, fieldName, options)`。
- Consumes: 现有 `verifyToken(token)`、SSE `sendError(res, message)`。

- [ ] **Step 1: 创建 HTTP 工具**

创建 `server/utils/http.js`：

```js
export class HttpError extends Error {
  constructor(status, message) {
    super(message)
    this.name = 'HttpError'
    this.status = status
  }
}

export function httpError(status, message) {
  return new HttpError(status, message)
}

export function asyncHandler(handler) {
  return async (req, res, next) => {
    try {
      await handler(req, res, next)
    }
    catch (err) {
      next(err)
    }
  }
}

export function notFoundHandler(req, res) {
  res.status(404).json({ success: false, message: '接口不存在' })
}

export function errorHandler(err, req, res, next) {
  if (res.headersSent) {
    next(err)
    return
  }

  if (err.type === 'entity.too.large') {
    res.status(413).json({ success: false, message: '请求数据过大' })
    return
  }

  if (err.message === '不允许的跨域来源') {
    res.status(403).json({ success: false, message: err.message })
    return
  }

  if (err.name === 'JsonWebTokenError' || err.name === 'TokenExpiredError') {
    res.status(401).json({ success: false, message: '令牌无效或已过期' })
    return
  }

  if (err instanceof HttpError) {
    res.status(err.status).json({ success: false, message: err.message })
    return
  }

  console.error('服务器错误:', err)
  res.status(500).json({ success: false, message: '服务器内部错误' })
}
```

- [ ] **Step 2: 创建校验工具**

创建 `server/utils/validation.js`：

```js
import { httpError } from './http.js'

export function readRequiredString(value, fieldName, options = {}) {
  const { min = 1, max = 2000 } = options
  if (typeof value !== 'string')
    throw httpError(400, `${fieldName}必须是文本`)

  const trimmed = value.trim()
  if (trimmed.length < min)
    throw httpError(400, `请输入${fieldName}`)
  if (trimmed.length > max)
    throw httpError(400, `${fieldName}不能超过 ${max} 个字符`)

  return trimmed
}

export function readPositiveInteger(value, fieldName, options = {}) {
  const { min = 1, max = 30 } = options
  const number = Number(value)
  if (!Number.isInteger(number) || number < min || number > max)
    throw httpError(400, `${fieldName}必须是 ${min}-${max} 之间的整数`)
  return number
}

export function readPositiveNumber(value, fieldName, options = {}) {
  const { min = 1, max = 1_000_000 } = options
  const number = Number(value)
  if (!Number.isFinite(number) || number < min || number > max)
    throw httpError(400, `${fieldName}必须是 ${min}-${max} 之间的数字`)
  return number
}

export function ensureArray(value, fieldName, options = {}) {
  const { max = 20 } = options
  if (value === undefined)
    return []
  if (!Array.isArray(value))
    throw httpError(400, `${fieldName}必须是数组`)
  if (value.length > max)
    throw httpError(400, `${fieldName}最多支持 ${max} 条`)
  return value
}
```

- [ ] **Step 3: 创建认证中间件**

创建 `server/middleware/auth.js`：

```js
import { verifyToken } from '../services/auth.js'
import { httpError } from '../utils/http.js'

export function requireAuth(req, res, next) {
  try {
    const authHeader = req.headers.authorization
    if (!authHeader || !authHeader.startsWith('Bearer '))
      throw httpError(401, '未登录')

    const token = authHeader.slice(7)
    req.user = verifyToken(token)
    next()
  }
  catch (err) {
    next(err)
  }
}
```

- [ ] **Step 4: 更新 Express 入口错误处理**

修改 `server/index.js`：

```js
import { errorHandler, notFoundHandler } from './utils/http.js'

// 替换原 404 与全局错误处理中间件：
app.use(notFoundHandler)
app.use(errorHandler)
```

保留原有 `helmet`、`cors`、`express.json`、限流和路由挂载顺序。

- [ ] **Step 5: 更新认证路由**

修改 `server/routes/auth.js`：

```js
import { Router } from 'express'
import { requireAuth } from '../middleware/auth.js'
import { login, register } from '../services/auth.js'
import { asyncHandler } from '../utils/http.js'
import { readRequiredString } from '../utils/validation.js'

const router = Router()

router.post('/register', asyncHandler(async (req, res) => {
  const username = readRequiredString(req.body.username, '用户名', { min: 2, max: 20 })
  const password = readRequiredString(req.body.password, '密码', { min: 6, max: 72 })
  const result = await register(username, password)
  res.json({ success: true, ...result })
}))

router.post('/login', asyncHandler(async (req, res) => {
  const username = readRequiredString(req.body.username, '用户名', { min: 1, max: 20 })
  const password = readRequiredString(req.body.password, '密码', { min: 1, max: 72 })
  const result = await login(username, password)
  res.json({ success: true, ...result })
}))

router.get('/me', requireAuth, (req, res) => {
  res.json({ success: true, user: { id: req.user.id, username: req.user.username } })
})

export default router
```

- [ ] **Step 6: 更新天气路由**

修改 `server/routes/weather.js`，用 `asyncHandler` 和校验工具：

```js
import { Router } from 'express'
import { getWeather } from '../services/weather.js'
import { asyncHandler, httpError } from '../utils/http.js'
import { readRequiredString } from '../utils/validation.js'

const router = Router()

router.get('/weather', asyncHandler(async (req, res) => {
  const city = readRequiredString(req.query.city, '城市名称', { min: 1, max: 50 })
  const weather = await getWeather(city)
  if (!weather)
    throw httpError(404, '未找到该城市天气信息')
  res.json(weather)
}))

export default router
```

- [ ] **Step 7: 更新推荐路由校验**

修改 `server/routes/travel.js`，在 `initSSE(res)` 前完成参数校验：

```js
import { Router } from 'express'
import { executeAgent } from '../services/agent.js'
import { asyncHandler } from '../utils/http.js'
import { readPositiveInteger, readPositiveNumber, readRequiredString } from '../utils/validation.js'
import { initSSE, sendError } from '../utils/sse.js'

const router = Router()

router.post('/recommend', asyncHandler(async (req, res) => {
  const city = readRequiredString(req.body.city, '目的地城市', { min: 1, max: 50 })
  const budget = readPositiveNumber(req.body.budget, '预算', { min: 1, max: 1_000_000 })
  const days = readPositiveInteger(req.body.days, '行程天数', { min: 1, max: 30 })

  try {
    initSSE(res)
    await executeAgent(res, { city, budget, days })
  }
  catch (err) {
    console.error('行程推荐失败:', err)
    sendError(res, '生成行程时出现错误，请稍后重试')
  }
  finally {
    res.end()
  }
}))

export default router
```

- [ ] **Step 8: 更新聊天路由校验**

修改 `server/routes/chat.js` 的 `router.post('/chat'...)` 开头，保证校验在 `initSSE(res)` 前：

```js
import { asyncHandler } from '../utils/http.js'
import { ensureArray, readRequiredString } from '../utils/validation.js'

router.post('/chat', asyncHandler(async (req, res) => {
  const message = readRequiredString(req.body.message, '问题', { min: 1, max: 2000 })
  const historyMessages = ensureArray(req.body.messages, '历史消息', { max: 20 }).map((item) => {
    const role = item?.role === 'assistant' ? 'assistant' : 'user'
    const content = typeof item?.content === 'string' ? item.content.slice(0, 2000) : ''
    return { role, content }
  }).filter(item => item.content)

  try {
    initSSE(res)
    // 后续保留原有 Agent 与 Mock 流程，删除原来的 const { message, messages: historyMessages } = req.body
  }
  catch (err) {
    console.error('对话失败:', err)
    sendError(res, '处理消息时出现错误，请稍后重试')
  }
  finally {
    res.end()
  }
}))
```

注意：只替换路由包装和参数读取，不改动 `CHAT_TOOLS`、`executeChatTool`、`buildMessagesWithMemory`、Mock 回复逻辑。

- [ ] **Step 9: 更新分享路由**

修改 `server/routes/share.js`，用 `requireAuth` 替代手写 token 校验，并保留分享数据大小限制：

```js
import { requireAuth } from '../middleware/auth.js'
import { asyncHandler, httpError } from '../utils/http.js'
import { readPositiveInteger, readRequiredString } from '../utils/validation.js'

function validateSharePayload(payload) {
  if (Buffer.byteLength(JSON.stringify(payload), 'utf8') > MAX_SHARE_BODY_SIZE)
    throw httpError(400, '请求数据过大')

  const city = readRequiredString(payload.city, '城市名称', { min: 1, max: 50 })
  const days = readPositiveInteger(payload.days, '行程天数', { min: 1, max: 30 })
  const budget = readRequiredString(String(payload.budget ?? ''), '预算', { min: 1, max: 50 })

  if (!payload.itinerary)
    throw httpError(400, '缺少行程数据')

  return { city, days, budget, itinerary: payload.itinerary }
}

router.post('/share', sharePostLimiter, requireAuth, asyncHandler(async (req, res) => {
  const payload = validateSharePayload(req.body)
  const shareId = createShare(payload)
  res.json({ success: true, shareId, shareUrl: `/share/${shareId}` })
}))

router.get('/share/:id', shareGetLimiter, asyncHandler(async (req, res) => {
  const share = getShare(req.params.id)
  if (!share)
    throw httpError(404, '分享不存在')
  res.json({ success: true, data: share })
}))
```

- [ ] **Step 10: 更新后端 check 脚本**

修改 `server/package.json` 的 `check` 脚本，加入新增文件：

```json
"check": "node --check index.js && node --check services/llm.js && node --check services/auth.js && node --check middleware/rateLimit.js && node --check middleware/auth.js && node --check routes/auth.js && node --check routes/chat.js && node --check routes/share.js && node --check routes/travel.js && node --check routes/weather.js && node --check config/env.js && node --check utils/http.js && node --check utils/validation.js"
```

- [ ] **Step 11: 运行后端阶段检查**

Run:

```bash
cd server && pnpm check
cd .. && pnpm check
```

Expected: 后端 Node 语法检查通过，前端检查通过。

- [ ] **Step 12: 提交后端规范化**

```bash
git add server/utils/http.js server/utils/validation.js server/middleware/auth.js server/index.js server/routes/auth.js server/routes/weather.js server/routes/travel.js server/routes/chat.js server/routes/share.js server/package.json
git commit -m "refactor: 规范后端校验和错误处理"
```

---

### Task 5: 运行时验收与最终评估

**Files:**
- Modify: `README.md` if validation reveals command or deployment wording needs correction.
- Create: `docs/deployment/checklist.md`

**Interfaces:**
- Consumes: Task 1-4 的文档、API、SSE 和后端错误处理。
- Produces: 最终验收清单，记录本轮已验证项目和剩余风险。

- [ ] **Step 1: 创建部署检查清单**

创建 `docs/deployment/checklist.md`：

```markdown
# 部署前检查清单

## 本地检查

- [ ] 根目录执行 `pnpm check` 通过
- [ ] `server/` 执行 `pnpm check` 通过
- [ ] `.env` 已配置 `JWT_SECRET`
- [ ] 未将真实 `.env` 提交到仓库

## ECS 检查

- [ ] 已安装 Node.js 20+ 和 pnpm
- [ ] 已安装 PM2
- [ ] 已安装 Nginx
- [ ] 安全组开放 80、443 和 SSH 端口
- [ ] 后端 `curl http://127.0.0.1:3030/api/health` 正常
- [ ] Nginx 访问首页正常
- [ ] `/api/health` 经过域名或 IP 访问正常

## 备案后检查

- [ ] 域名 A 记录指向 ECS 公网 IP
- [ ] `CORS_ORIGIN` 已改为正式域名
- [ ] 已准备 HTTPS 证书配置
```

- [ ] **Step 2: 执行静态检查**

Run:

```bash
pnpm check
cd server && pnpm check
```

Expected: 两个命令都通过。

- [ ] **Step 3: 启动后端冒烟验证**

Run:

```bash
cd server
pnpm dev
```

另一个 shell 执行：

```bash
curl http://127.0.0.1:3030/api/health
curl "http://127.0.0.1:3030/api/weather?city=北京"
```

Expected:

- health 返回 JSON，包含 `status: "ok"`。
- weather 返回天气 JSON 或明确错误 JSON。

- [ ] **Step 4: 启动前端冒烟验证**

Run:

```bash
pnpm dev
```

用浏览器访问：

```text
http://localhost:5181
```

验证：

- 首页可打开。
- 登录页可打开。
- 天气页输入北京后有结果或明确错误。
- AI 咨询页发送问题后不会永久 loading。

- [ ] **Step 5: 记录最终评估**

在最终回复中按以下结构汇报：

```markdown
## 本轮完成
- ...

## 验证结果
- `pnpm check`: ...
- `cd server && pnpm check`: ...
- 运行时冒烟验证: ...

## 剩余风险
- ...

## 下一轮建议
1. ...
2. ...
```

- [ ] **Step 6: 提交验收文档**

```bash
git add README.md docs/deployment/checklist.md
git commit -m "docs: 添加部署检查清单"
```

---

## Self-Review

- Spec coverage: 阶段一由 Task 1 覆盖，阶段二由 Task 2 覆盖，阶段三由 Task 3 覆盖，阶段四由 Task 4 覆盖，阶段五和最终评估由 Task 5 覆盖。
- Placeholder scan: 本计划未使用 TBD、TODO、implement later、similar to 等占位描述。
- Type consistency: 前端 API 函数名 `request`、`loginApi`、`registerApi`、`getWeatherApi` 在任务内定义并被后续步骤消费；后端工具函数 `asyncHandler`、`httpError`、`readRequiredString`、`readPositiveInteger`、`requireAuth` 在任务内定义并被后续步骤消费。
- Scope check: 本计划不引入 workspace、Docker、完整测试体系或 RAG 重写，符合轻量规范化与部署准备目标。
