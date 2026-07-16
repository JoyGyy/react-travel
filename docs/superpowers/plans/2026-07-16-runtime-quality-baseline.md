# Runtime Quality Baseline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 消除 Ant Design React 19 运行时警告，建立最小 E2E、API/SSE 类型基线和包体积观察基线。

**Architecture:** 保持当前 React 19 + Vite SPA 架构，在入口层接入 AntD React 19 patch，并通过 AntD App 上下文替换静态反馈 API。新增 Playwright E2E 作为独立质量门禁，不纳入默认 `pnpm check`；类型基线采用 JSDoc 和共享 typedef，避免全量 TypeScript 迁移。

**Tech Stack:** React 19、Vite 8、Ant Design 5、React Router 7、Zustand 5、Playwright、Node.js、Express、pnpm。

## Global Constraints

- 所有回复、文档说明和代码注释使用中文。
- 包管理器使用 pnpm，不使用 npm 或 yarn。
- 当前不做全量 TypeScript 迁移。
- 当前不把 E2E 强制纳入 `pnpm check`。
- 当前不做大规模拆包、虚拟列表或 UI 重构。
- 当前不迁移到 Ant Design v6。
- 当前不修改后端核心 Agent/RAG 逻辑。
- Git 提交信息使用中文，不添加 `Co-Authored-By` 或任何 Claude 标记。

---

## File Structure

### 新增文件

- `src/hooks/useAppMessage.js`：封装 `App.useApp().message`，统一提供上下文安全的 message API。
- `src/types/api.js`：JSDoc typedef 类型基线，描述认证、天气、SSE 和 API 错误结构。
- `playwright.config.js`：Playwright E2E 配置，自动启动后端和前端。
- `e2e/app.spec.js`：最小端到端验证，覆盖首页、注册登录、天气、聊天。
- `docs/performance/bundle-baseline.md`：构建产物体积观察文档。

### 修改文件

- `package.json`：新增 `@ant-design/v5-patch-for-react-19`、`@playwright/test`，新增 `e2e` 和 `e2e:headed` 脚本。
- `pnpm-lock.yaml`：依赖锁文件更新。
- `src/main.jsx`：入口引入 AntD React 19 patch。
- `src/App.jsx`：在 `ConfigProvider` 内包裹 AntD `App` 容器。
- `src/components/ProtectedRoute/index.jsx`：替换静态 `message.warning`。
- `src/pages/Home/index.jsx`：替换静态 `message.loading`。
- `src/pages/Login/index.jsx`：替换静态 `message.success`。
- `src/pages/Chat/index.jsx`：将 `Modal.confirm` 迁移为 `Modal.useModal`。
- `src/api/client.js`、`src/api/auth.js`、`src/api/weather.js`、`src/hooks/useSSE.js`：补充 JSDoc 类型引用。
- `docs/deployment/checklist.md`：补充 E2E 验证项。

---

### Task 1: AntD React 19 兼容与上下文反馈 API

**Files:**
- Modify: `package.json`
- Modify: `pnpm-lock.yaml`
- Modify: `src/main.jsx`
- Modify: `src/App.jsx`
- Create: `src/hooks/useAppMessage.js`
- Modify: `src/components/ProtectedRoute/index.jsx`
- Modify: `src/pages/Home/index.jsx`
- Modify: `src/pages/Login/index.jsx`
- Modify: `src/pages/Chat/index.jsx`

**Interfaces:**
- Produces: `useAppMessage()`，返回 AntD `App.useApp().message`。
- Consumes: AntD `App` 容器必须位于需要使用 `App.useApp()` 的组件上层。

- [ ] **Step 1: 安装 AntD React 19 patch**

Run:

```bash
pnpm add @ant-design/v5-patch-for-react-19
```

Expected: `package.json` dependencies 增加 `@ant-design/v5-patch-for-react-19`，`pnpm-lock.yaml` 更新。

- [ ] **Step 2: 在入口引入 patch**

修改 `src/main.jsx`：

```jsx
/**
 * 应用入口文件
 * 负责初始化 React 应用并挂载到 DOM
 */
import '@ant-design/v5-patch-for-react-19'
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './styles/global.css'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

- [ ] **Step 3: 在 App 中加入 AntD App 容器**

修改 `src/App.jsx`：

```jsx
/**
 * 应用根组件
 * 使用 Ant Design ConfigProvider 配置主题，React Router 提供路由功能
 */
import { App as AntdApp, ConfigProvider } from 'antd'
import { RouterProvider } from 'react-router-dom'
import router from '@/router'

export default function App() {
  return (
    <ConfigProvider
      theme={{
        token: {
          colorPrimary: '#FF6B35',
          colorInfo: '#3B82F6',
          colorBgLayout: '#FAFAF8',
          colorBgElevated: 'rgba(255,255,255,0.92)',
          colorBgContainer: 'rgba(255,255,255,0.96)',
          colorText: '#1C1917',
          colorError: '#DC2626',
          colorWarning: '#D97706',
          colorSuccess: '#059669',
          borderRadius: 12,
          fontFamily: '\'Plus Jakarta Sans\', \'Noto Sans SC\', -apple-system, BlinkMacSystemFont, sans-serif',
        },
      }}
    >
      <AntdApp>
        <RouterProvider router={router} />
      </AntdApp>
    </ConfigProvider>
  )
}
```

- [ ] **Step 4: 创建上下文安全 message Hook**

创建 `src/hooks/useAppMessage.js`：

```js
/**
 * 获取 Ant Design App 上下文中的 message API
 * 避免静态 message.* 无法读取 ConfigProvider 主题上下文的问题
 */
import { App as AntdApp } from 'antd'

export function useAppMessage() {
  const { message } = AntdApp.useApp()
  return message
}
```

- [ ] **Step 5: 改造 ProtectedRoute**

修改 `src/components/ProtectedRoute/index.jsx`：

```jsx
import { useEffect } from 'react'
import { useNavigate } from 'react-router-dom'
import { useAppMessage } from '@/hooks/useAppMessage'
import { useAuthStore } from '@/stores/auth'

export function ProtectedRoute({ children }) {
  const navigate = useNavigate()
  const message = useAppMessage()
  const user = useAuthStore(state => state.user)
  const hasHydrated = useAuthStore(state => state._hasHydrated)

  useEffect(() => {
    if (hasHydrated && !user) {
      message.warning('请先登录')
      navigate('/login', { replace: true })
    }
  }, [user, hasHydrated, navigate, message])

  // 保留原有渲染逻辑
}
```

- [ ] **Step 6: 改造 Home**

修改 `src/pages/Home/index.jsx`：

- 删除 `import { message } from 'antd'`。
- 新增 `import { useAppMessage } from '@/hooks/useAppMessage'`。
- 在组件中新增：

```jsx
const message = useAppMessage()
```

`onStart()` 中继续使用：

```jsx
message.loading('加载中...')
```

- [ ] **Step 7: 改造 Login**

修改 `src/pages/Login/index.jsx`：

- 删除 `import { message } from 'antd'`。
- 新增 `import { useAppMessage } from '@/hooks/useAppMessage'`。
- 在组件中新增：

```jsx
const message = useAppMessage()
```

提交成功后继续使用：

```jsx
message.success(tab === 'login' ? '登录成功' : '注册成功')
```

- [ ] **Step 8: 改造 Chat Modal**

修改 `src/pages/Chat/index.jsx`：

```jsx
const [modal, modalContextHolder] = Modal.useModal()

function clearChat() {
  modal.confirm({
    title: '确定要清空所有对话记录吗？',
    content: '清空后当前对话无法恢复。',
    okText: '确定清空',
    cancelText: '取消',
    okButtonProps: { danger: true },
    onOk: () => {
      clearMessages()
      setRagSources([])
      setNotice('')
      setChatError('')
    },
  })
}
```

在 JSX 的 `<main>` 内最前面渲染：

```jsx
{modalContextHolder}
```

- [ ] **Step 9: 运行兼容阶段检查**

Run:

```bash
pnpm check
pnpm --dir server check
```

Expected: 两个命令都通过。

- [ ] **Step 10: 提交兼容修复**

```bash
git add package.json pnpm-lock.yaml src/main.jsx src/App.jsx src/hooks/useAppMessage.js src/components/ProtectedRoute/index.jsx src/pages/Home/index.jsx src/pages/Login/index.jsx src/pages/Chat/index.jsx
git commit -m "fix: 适配 Ant Design React 19 运行时兼容"
```

---

### Task 2: 最小 Playwright E2E

**Files:**
- Modify: `package.json`
- Modify: `pnpm-lock.yaml`
- Create: `playwright.config.js`
- Create: `e2e/app.spec.js`
- Modify: `docs/deployment/checklist.md`

**Interfaces:**
- Produces: `pnpm e2e` 和 `pnpm e2e:headed`。
- Consumes: 后端从根目录 `.env` 读取配置；前端 Vite 端口 5181；后端端口 3030。

- [ ] **Step 1: 安装 Playwright 测试依赖**

Run:

```bash
pnpm add -D @playwright/test
```

Expected: `devDependencies` 增加 `@playwright/test`。

- [ ] **Step 2: 新增 E2E 脚本**

修改 `package.json` scripts，增加：

```json
"e2e": "playwright test",
"e2e:headed": "playwright test --headed"
```

保留原有 `check` 不变。

- [ ] **Step 3: 创建 Playwright 配置**

创建 `playwright.config.js`：

```js
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  timeout: 60_000,
  expect: {
    timeout: 15_000,
  },
  fullyParallel: false,
  workers: 1,
  reporter: [['list']],
  use: {
    baseURL: 'http://127.0.0.1:5181',
    trace: 'on-first-retry',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
  webServer: [
    {
      command: 'pnpm --dir server dev',
      url: 'http://127.0.0.1:3030/api/health',
      reuseExistingServer: true,
      timeout: 30_000,
    },
    {
      command: 'pnpm dev',
      url: 'http://127.0.0.1:5181',
      reuseExistingServer: true,
      timeout: 30_000,
    },
  ],
})
```

- [ ] **Step 4: 创建 E2E 用例**

创建 `e2e/app.spec.js`：

```js
import { expect, test } from '@playwright/test'

function uniqueUser() {
  return `e2e_${Date.now()}`
}

test('核心路径：注册、天气查询、AI 咨询', async ({ page }) => {
  await page.goto('/')
  await expect(page.getByRole('heading', { name: /AI 旅行规划师/ })).toBeVisible()

  const username = uniqueUser()
  await page.getByRole('link', { name: /登录|注册/ }).click()
  await expect(page).toHaveURL(/\/login$/)
  await page.getByRole('button', { name: '注册' }).click()
  await page.getByLabel('用户名').fill(username)
  await page.getByLabel('密码').fill('e2e123456')
  await page.getByRole('button', { name: '创建账号' }).click()
  await expect(page).toHaveURL(/\/$/)
  await expect(page.getByText(username)).toBeVisible()

  await page.goto('/weather')
  await page.getByRole('combobox', { name: '城市名称' }).fill('北京')
  await page.getByRole('combobox', { name: '城市名称' }).press('Enter')
  await expect(page.getByRole('region', { name: /北京 天气概览/ })).toBeVisible()

  await page.goto('/chat')
  await expect(page.getByRole('heading', { name: '旅行顾问' })).toBeVisible()
  await page.getByRole('button', { name: '北京有哪些必去的景点？' }).click()
  await expect(page.getByText('故宫博物院').first()).toBeVisible({ timeout: 30_000 })
  await expect(page.getByText('参考来源')).toBeVisible()
})
```

- [ ] **Step 5: 更新部署检查清单**

修改 `docs/deployment/checklist.md`，本地检查增加：

```markdown
- [ ] 根目录执行 `pnpm e2e` 通过
```

- [ ] **Step 6: 安装 Playwright 浏览器**

Run:

```bash
pnpm exec playwright install chromium
```

Expected: Chromium 浏览器安装完成；如果本机已安装则快速完成。

- [ ] **Step 7: 运行 E2E**

Run:

```bash
pnpm e2e
```

Expected: 1 个测试通过。

- [ ] **Step 8: 运行静态检查**

Run:

```bash
pnpm check
pnpm --dir server check
```

Expected: 两个命令都通过。

- [ ] **Step 9: 提交 E2E 基线**

```bash
git add package.json pnpm-lock.yaml playwright.config.js e2e/app.spec.js docs/deployment/checklist.md
git commit -m "test: 添加核心路径 E2E 验证"
```

---

### Task 3: API/SSE 类型基线

**Files:**
- Create: `src/types/api.js`
- Modify: `src/api/client.js`
- Modify: `src/api/auth.js`
- Modify: `src/api/weather.js`
- Modify: `src/hooks/useSSE.js`

**Interfaces:**
- Produces JSDoc typedef: `ApiSuccess`, `AuthUser`, `AuthResponse`, `WeatherResponse`, `SSEEvent`, `SSECallbacks`。
- Consumes: 现有 API 封装和 `useSSE()` 返回 `{ sendRequest, abort }`。

- [ ] **Step 1: 创建类型基线文件**

创建 `src/types/api.js`：

```js
/**
 * API 与 SSE 结构类型基线。
 * 这是 JSDoc 类型声明文件，不包含运行时代码。
 */

/**
 * @typedef {{ success?: boolean, message?: string, error?: string }} ApiSuccess
 */

/**
 * @typedef {{ id: string, username: string, createdAt?: string }} AuthUser
 */

/**
 * @typedef {{ success: true, token: string, user: AuthUser }} AuthResponse
 */

/**
 * @typedef {{ date: string, maxTemp: number, minTemp: number, weatherCode: number, weatherDesc: string }} WeatherForecast
 */

/**
 * @typedef {{ city: string, temperature: number, feelsLike?: number, humidity?: number, windSpeed?: number, weatherCode?: number, weatherDesc: string, forecast?: WeatherForecast[] }} WeatherResponse
 */

/**
 * @typedef {{ type: 'chunk', content: string } | { type: 'step', step: number, name: string, status: 'start' | 'complete', data?: object } | { type: 'notice', message: string } | { type: 'complete', data?: object } | { type: 'error', message?: string }} SSEEvent
 */

/**
 * @typedef {{ onChunk?: (content: string) => void, onStep?: (event: Extract<SSEEvent, { type: 'step' }>) => void, onNotice?: (message: string) => void, onComplete?: (data?: object) => void, onError?: (error: Error) => void, onFinally?: () => void }} SSECallbacks
 */

export {}
```

- [ ] **Step 2: 给 API client 补类型**

在 `src/api/client.js` 顶部补充：

```js
/** @typedef {import('@/types/api').ApiSuccess} ApiSuccess */
```

给 `request` 补 JSDoc：

```js
/**
 * 发起 JSON API 请求。
 * @param {string} path
 * @param {{ method?: string, body?: unknown, headers?: Record<string, string>, auth?: boolean, signal?: AbortSignal }} [options]
 * @returns {Promise<ApiSuccess | null | object>}
 */
```

- [ ] **Step 3: 给 auth API 补类型**

在 `src/api/auth.js` 顶部补充：

```js
/** @typedef {import('@/types/api').AuthResponse} AuthResponse */
```

给 `loginApi` 和 `registerApi` 补 JSDoc：

```js
/**
 * @param {string} username
 * @param {string} password
 * @returns {Promise<AuthResponse>}
 */
```

- [ ] **Step 4: 给 weather API 补类型**

在 `src/api/weather.js` 顶部补充：

```js
/** @typedef {import('@/types/api').WeatherResponse} WeatherResponse */
```

给 `getWeatherApi` 补 JSDoc：

```js
/**
 * @param {string} city
 * @param {{ signal?: AbortSignal }} [options]
 * @returns {Promise<WeatherResponse>}
 */
```

- [ ] **Step 5: 给 useSSE 补类型**

在 `src/hooks/useSSE.js` 顶部补充：

```js
/** @typedef {import('@/types/api').SSECallbacks} SSECallbacks */
```

给 `sendRequest` 内部函数附近补充：

```js
/**
 * @param {string} url
 * @param {object} body
 * @param {SSECallbacks} [callbacks]
 * @returns {Promise<void>}
 */
```

- [ ] **Step 6: 运行类型阶段检查**

Run:

```bash
pnpm check
```

Expected: lint、typecheck、build 全部通过。

- [ ] **Step 7: 提交类型基线**

```bash
git add src/types/api.js src/api/client.js src/api/auth.js src/api/weather.js src/hooks/useSSE.js
git commit -m "chore: 添加 API 和 SSE 类型基线"
```

---

### Task 4: 包体积观察文档

**Files:**
- Create: `docs/performance/bundle-baseline.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: `pnpm build` 输出中的 chunk 名称和 gzip 大小。
- Produces: 包体积观察基线和下一轮优化方向。

- [ ] **Step 1: 运行生产构建并记录输出**

Run:

```bash
pnpm build
```

Expected: 输出包含 `dist/assets/index-*.js`、`Chat-*.js`、`Home-*.js`、CSS chunk 和 gzip 大小。

- [ ] **Step 2: 创建包体积文档**

创建 `docs/performance/bundle-baseline.md`，结构如下：

```markdown
# 包体积观察基线

## 记录方式

执行：

```bash
pnpm build
```

## 当前主要产物

| 产物 | 观察重点 | 说明 |
| --- | --- | --- |
| `index-*.js` | 主包 gzip 约 140KB+ | 包含应用入口和共享依赖 |
| `Chat-*.js` | 页面包 gzip 约 70KB+ | 包含聊天页、Markdown 渲染和相关组件 |
| `Home-*.js` | 页面包 gzip 约 7KB+ | 首页已路由级懒加载 |
| `*.css` | 页面样式包 | 当前 CSS 按页面和组件拆分 |

## 下一轮优化方向

1. 优先分析 `Chat` 页面包：Markdown 渲染和聊天组件可能是主要来源。
2. 保持当前 React.lazy 路由级拆分。
3. 检查 AntD icons 是否存在重复或过宽导入。
4. 如需精确定位，再引入 bundle analyzer，不在本轮直接大改。
```

- [ ] **Step 3: README 增加性能文档入口**

在 `README.md` 的部署或测试状态附近增加：

```markdown
## 性能基线

当前包体积观察见：[包体积观察基线](docs/performance/bundle-baseline.md)。
```

- [ ] **Step 4: 运行文档阶段检查**

Run:

```bash
pnpm check
pnpm --dir server check
```

Expected: 两个命令都通过。

- [ ] **Step 5: 提交包体积基线**

```bash
git add docs/performance/bundle-baseline.md README.md
git commit -m "docs: 添加包体积观察基线"
```

---

### Task 5: 最终运行时验收

**Files:**
- Modify: `.claude/skills/verify/SKILL.md` if runtime verification flow changed locally; do not commit `.claude/` because project gitignore excludes it.

**Interfaces:**
- Consumes: Task 1-4 的所有改动。
- Produces: 最终验收报告。

- [ ] **Step 1: 执行完整检查**

Run:

```bash
pnpm check
pnpm --dir server check
pnpm e2e
```

Expected: 三个命令全部通过。

- [ ] **Step 2: 运行真实应用观察控制台**

Run:

```bash
pnpm --dir server dev
pnpm dev
```

浏览器访问：

```text
http://127.0.0.1:5181
```

验证：

- 首页正常打开。
- 登录/注册成功。
- 天气页查询“北京”成功。
- 登录后聊天页能返回“故宫博物院”。
- 控制台不再出现 AntD React 19 兼容警告。
- 控制台不再出现静态 `message.*` 上下文警告。

- [ ] **Step 3: 记录最终评估**

最终回复包含：

```markdown
## 本轮完成
- ...

## 验证结果
- `pnpm check`: ...
- `pnpm --dir server check`: ...
- `pnpm e2e`: ...
- 运行时验证: ...

## 剩余风险
- ...

## 下一轮建议
1. ...
```

- [ ] **Step 4: 确认工作区干净**

Run:

```bash
git status --short
```

Expected: 没有未提交变更；若只有 `.claude/` 本地技能文件被忽略，不需要处理。

---

## Self-Review

- Spec coverage: Task 1 覆盖 AntD React 19 patch、App 容器、message 和 Modal 迁移；Task 2 覆盖最小 E2E；Task 3 覆盖 API/SSE 类型基线；Task 4 覆盖包体积观察文档；Task 5 覆盖最终运行时验收。
- Placeholder scan: 未包含 TBD、TODO、implement later、类似 Task N 等占位描述。
- Type consistency: `useAppMessage()`、`SSECallbacks`、`AuthResponse`、`WeatherResponse` 均在任务内定义并被后续步骤使用。
- Scope check: 未引入 AntD v6、全量 TS、默认 check 中 E2E 或大规模拆包，符合设计范围。
