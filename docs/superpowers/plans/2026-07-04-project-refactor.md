# 项目重构实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将项目从 TypeScript + Tailwind + antd-mobile 重构为 JS/JSX + 普通 CSS + Ant Design PC 版，去掉移动端适配。

**Architecture:** 分 4 阶段逐层迁移：基础设施 → 基础层（stores/hooks/utils）→ 组件层 → 页面层。每阶段完成后 `pnpm dev` 应正常运行。后端（server/）不改动。

**Tech Stack:** React 19, Vite 8, Zustand 5, Ant Design 5, react-router-dom 7, react-markdown 10

## Global Constraints

- 所有 `.ts` → `.js`，`.tsx` → `.jsx`
- 不使用 Tailwind，所有样式用普通 CSS 文件 + BEM 命名
- 不做移动端适配，固定桌面端布局（max-width: 1200px 居中）
- UI 组件库用 Ant Design 5（PC 版），主题色 `#6366f1`
- 代码注释使用中文
- 包管理器使用 pnpm
- 后端代码（server/）不改动

---

## 阶段 1：基础设施

### Task 1: 清理 package.json

**Files:**
- Modify: `package.json`

- [ ] **Step 1: 更新 package.json 依赖**

移除 TypeScript、Tailwind、antd-mobile、vitest 相关依赖，加入 antd 和 @ant-design/icons。

```json
{
  "name": "react-travel",
  "type": "module",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "lint": "eslint .",
    "preview": "vite preview"
  },
  "dependencies": {
    "@ant-design/icons": "^5.6.1",
    "antd": "^5.25.3",
    "html-to-image": "^1.11.13",
    "react": "^19.2.6",
    "react-dom": "^19.2.6",
    "react-markdown": "^10.1.0",
    "react-router-dom": "^7.15.1",
    "zustand": "^5.0.13"
  },
  "devDependencies": {
    "@eslint/js": "^10.0.1",
    "@testing-library/jest-dom": "^6.9.1",
    "@testing-library/react": "^16.3.2",
    "@testing-library/user-event": "^14.6.1",
    "@vitejs/plugin-react": "^6.0.1",
    "eslint": "^10.3.0",
    "eslint-plugin-react-hooks": "^7.1.1",
    "eslint-plugin-react-refresh": "^0.5.2",
    "globals": "^17.6.0",
    "jsdom": "^29.1.1",
    "vite": "^8.0.12"
  }
}
```

- [ ] **Step 2: 安装依赖**

```bash
cd /Users/joygy/Documents/react/react-travel && pnpm install
```

- [ ] **Step 3: 提交**

```bash
git add package.json pnpm-lock.yaml
git commit -m "refactor: 清理依赖，移除 TS/tailwind/antd-mobile，加入 antd"
```

---

### Task 2: 删除 TS 配置文件和测试配置

**Files:**
- Delete: `tsconfig.json`
- Delete: `tsconfig.app.json`
- Delete: `tsconfig.node.json`
- Delete: `vitest.config.ts`

- [ ] **Step 1: 删除文件**

```bash
cd /Users/joygy/Documents/react/react-travel
rm tsconfig.json tsconfig.app.json tsconfig.node.json vitest.config.ts
```

- [ ] **Step 2: 提交**

```bash
git add -A
git commit -m "refactor: 删除 tsconfig 和 vitest 配置文件"
```

---

### Task 3: Vite 配置转 JS

**Files:**
- Delete: `vite.config.ts`
- Create: `vite.config.js`

- [ ] **Step 1: 创建 vite.config.js**

```js
import path from 'node:path'
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vite'

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    host: true,
    port: 5181,
    proxy: {
      '/api': {
        target: 'http://localhost:3030',
        changeOrigin: true,
      },
    },
  },
  plugins: [react()],
})
```

- [ ] **Step 2: 删除旧文件**

```bash
rm /Users/joygy/Documents/react/react-travel/vite.config.ts
```

- [ ] **Step 3: 验证 dev server 启动**

```bash
cd /Users/joygy/Documents/react/react-travel && timeout 5 pnpm dev || true
```

- [ ] **Step 4: 提交**

```bash
git add vite.config.js
git commit -m "refactor: vite.config 转 JS，移除 tailwind 插件"
```

---

### Task 4: ESLint 配置简化

**Files:**
- Modify: `eslint.config.js`

- [ ] **Step 1: 更新 eslint.config.js**

移除 typescript-eslint 相关配置，改为只检查 JS/JSX。

```js
import js from '@eslint/js'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import { defineConfig, globalIgnores } from 'eslint/config'

export default defineConfig([
  globalIgnores(['dist']),
  {
    files: ['**/*.{js,jsx}'],
    extends: [
      js.configs.recommended,
      reactHooks.configs.flat.recommended,
      reactRefresh.configs.vite,
    ],
    languageOptions: {
      globals: globals.browser,
    },
  },
  {
    files: ['src/router/**'],
    rules: {
      'react-refresh/only-export-components': 'off',
    },
  },
])
```

- [ ] **Step 2: 提交**

```bash
git add eslint.config.js
git commit -m "refactor: ESLint 移除 TypeScript 相关配置"
```

---

### Task 5: 入口文件 main.jsx + App.jsx

**Files:**
- Delete: `src/main.tsx`
- Delete: `src/App.tsx`
- Create: `src/main.jsx`
- Create: `src/App.jsx`

- [ ] **Step 1: 创建 src/main.jsx**

```jsx
/**
 * 应用入口文件
 * 负责初始化 React 应用并挂载到 DOM
 */
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

- [ ] **Step 2: 创建 src/App.jsx**

```jsx
/**
 * 应用根组件
 * 使用 Ant Design ConfigProvider 配置主题，React Router 提供路由功能
 */
import { ConfigProvider } from 'antd'
import { useEffect } from 'react'
import { RouterProvider } from 'react-router-dom'
import router from '@/router'
import { useAuthStore } from '@/stores/auth'

export default function App() {
  // 页面刷新时，如果用户已登录，加载对应的历史记录
  useEffect(() => {
    const unsub = useAuthStore.persist.onFinishHydration(() => {
      const user = useAuthStore.getState().user
      if (user) {
        // history store 已删除，此处不再加载历史
      }
    })
    if (useAuthStore.persist.hasHydrated()) {
      const user = useAuthStore.getState().user
      if (user) {
        // history store 已删除
      }
    }
    return unsub
  }, [])

  return (
    <ConfigProvider
      theme={{
        token: {
          colorPrimary: '#6366f1',
          borderRadius: 8,
          fontFamily: "'Noto Sans SC', -apple-system, BlinkMacSystemFont, sans-serif",
        },
      }}
    >
      <RouterProvider router={router} />
    </ConfigProvider>
  )
}
```

- [ ] **Step 3: 删除旧文件**

```bash
rm /Users/joygy/Documents/react/react-travel/src/main.tsx /Users/joygy/Documents/react/react-travel/src/App.tsx
```

- [ ] **Step 4: 提交**

```bash
git add src/main.jsx src/App.jsx
git commit -m "refactor: 入口文件转 JSX，加入 Ant Design ConfigProvider"
```

---

### Task 6: 全局样式 global.css 去掉 Tailwind

**Files:**
- Modify: `src/styles/global.css`

- [ ] **Step 1: 重写 global.css**

移除 `@import "tailwindcss"` 和 `@theme` 块，保留所有 CSS 变量、reset、动画、Markdown 样式。

```css
@import url('https://fonts.googleapis.com/css2?family=Noto+Serif+SC:wght@400;600;700&family=Noto+Sans+SC:wght@300;400;500;600;700&display=swap');

:root {
  --c-terracotta: #6366f1;
  --c-terracotta-light: #818cf8;
  --c-sand: #f1f5f9;
  --c-cream: #f8fafc;
  --c-forest: #1e293b;
  --c-forest-light: #334155;
  --c-gold: #94a3b8;
  --c-gold-light: #cbd5e1;
  --c-ink: #0f172a;
  --c-ink-light: #64748b;
  --c-paper: #f8fafc;
  --c-paper-dark: #e2e8f0;
  --c-white: #ffffff;

  --font-serif: 'Noto Serif SC', 'Songti SC', Georgia, serif;
  --font-sans: 'Noto Sans SC', -apple-system, BlinkMacSystemFont, sans-serif;

  --shadow-xs: 0 1px 2px rgba(15, 23, 42, 0.04);
  --shadow-sm: 0 1px 3px rgba(15, 23, 42, 0.06), 0 1px 2px rgba(15, 23, 42, 0.03);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.06), 0 1px 3px rgba(15, 23, 42, 0.03);
  --shadow-lg: 0 8px 28px rgba(15, 23, 42, 0.08), 0 2px 6px rgba(15, 23, 42, 0.03);
  --shadow-xl: 0 16px 48px rgba(15, 23, 42, 0.1), 0 4px 12px rgba(15, 23, 42, 0.04);
  --shadow-inset: inset 0 1px 3px rgba(15, 23, 42, 0.04), inset 0 0 0 1px rgba(15, 23, 42, 0.04);
}

/* 基础样式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: var(--font-sans);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background-color: var(--c-paper);
  color: var(--c-ink);
}

/* 滚动条 */
::-webkit-scrollbar {
  width: 4px;
  height: 4px;
}

::-webkit-scrollbar-track {
  background: transparent;
}

::-webkit-scrollbar-thumb {
  background: rgba(15, 23, 42, 0.1);
  border-radius: 100px;
}

::-webkit-scrollbar-thumb:hover {
  background: rgba(15, 23, 42, 0.18);
}

/* 选中文本 */
::selection {
  background: rgba(99, 102, 241, 0.12);
  color: var(--c-ink);
}

/* 动画 */
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes fadeUp {
  from {
    opacity: 0;
    transform: translateY(8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes scaleIn {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

@keyframes pulseGlow {
  0%, 100% {
    box-shadow: 0 0 0 0 rgba(99, 102, 241, 0.2);
  }
  50% {
    box-shadow: 0 0 0 10px rgba(99, 102, 241, 0);
  }
}

@keyframes dotBounce {
  0%, 80%, 100% {
    transform: scale(0.5);
    opacity: 0.3;
  }
  40% {
    transform: scale(1);
    opacity: 1;
  }
}

/* Markdown 内容样式 */
.markdown-body h1,
.markdown-body h2,
.markdown-body h3 {
  font-family: var(--font-serif);
  font-weight: 600;
  color: var(--c-ink);
  margin-top: 1em;
  margin-bottom: 0.5em;
  line-height: 1.4;
}

.markdown-body h1 { font-size: 1.25rem; }
.markdown-body h2 { font-size: 1.1rem; }
.markdown-body h3 { font-size: 1rem; }

.markdown-body p {
  margin-bottom: 0.5em;
  line-height: 1.75;
}

.markdown-body ul,
.markdown-body ol {
  padding-left: 1.25em;
  margin-bottom: 0.5em;
}

.markdown-body li {
  margin-bottom: 0.25em;
  line-height: 1.7;
}

.markdown-body strong {
  font-weight: 600;
  color: var(--c-ink);
}

.markdown-body code {
  background: rgba(15, 23, 42, 0.04);
  padding: 0.1em 0.35em;
  border-radius: 4px;
  font-size: 0.875em;
  font-family: 'SF Mono', 'Fira Code', monospace;
}

.markdown-body table {
  width: 100%;
  border-collapse: collapse;
  margin: 0.75em 0;
  font-size: 0.85em;
}

.markdown-body th {
  background: rgba(15, 23, 42, 0.03);
  font-weight: 600;
  text-align: left;
  padding: 0.5em 0.75em;
  border-bottom: 2px solid var(--c-paper-dark);
}

.markdown-body td {
  padding: 0.4em 0.75em;
  border-bottom: 1px solid rgba(226, 232, 240, 0.6);
}

.markdown-body tr:last-child td {
  border-bottom: none;
}

.markdown-body blockquote {
  border-left: 3px solid #818cf8;
  padding: 0.5em 1em;
  margin: 0.75em 0;
  background: rgba(99, 102, 241, 0.04);
  border-radius: 0 8px 8px 0;
  color: var(--c-ink-light);
  font-style: italic;
}

.markdown-body hr {
  border: none;
  height: 1px;
  background: linear-gradient(90deg, transparent, var(--c-paper-dark), transparent);
  margin: 1em 0;
}

.markdown-body em {
  font-style: italic;
  color: var(--c-ink-light);
}
```

- [ ] **Step 2: 验证**

```bash
cd /Users/joygy/Documents/react/react-travel && timeout 5 pnpm dev || true
```

- [ ] **Step 3: 提交**

```bash
git add src/styles/global.css
git commit -m "refactor: global.css 移除 tailwind，保留 CSS 变量和动画"
```

---

## 阶段 2：基础层

### Task 7: 删除废弃文件

**Files:**
- Delete: `src/stores/history.ts`
- Delete: `src/types/index.ts`
- Delete: `src/test/setup.ts`
- Delete: `src/stores/__tests__/chat.test.ts`
- Delete: `src/utils/__tests__/storage.test.ts`
- Delete: `src/components/__tests__/AgentSteps.test.tsx`
- Delete: `server/services/__tests__/agent.test.ts`
- Delete: `server/services/__tests__/tfidf.test.ts`
- Delete: `server/services/__tests__/rag.test.ts`

- [ ] **Step 1: 删除文件**

```bash
cd /Users/joygy/Documents/react/react-travel
rm -f src/stores/history.ts src/types/index.ts src/test/setup.ts
rm -rf src/stores/__tests__ src/utils/__tests__ src/components/__tests__
rm -rf server/services/__tests__
```

- [ ] **Step 2: 提交**

```bash
git add -A
git commit -m "refactor: 删除 history store、类型文件和所有测试文件"
```

---

### Task 8: auth store 转 JS

**Files:**
- Delete: `src/stores/auth.ts`
- Create: `src/stores/auth.js`

- [ ] **Step 1: 创建 src/stores/auth.js**

```js
/**
 * 认证状态管理
 * 使用 Zustand + persist 持久化用户登录状态
 */
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

export const useAuthStore = create(
  persist(
    set => ({
      user: null,
      token: null,
      _hasHydrated: false,
      setHasHydrated: v => set({ _hasHydrated: v }),

      async login(username, password) {
        const res = await fetch('/api/auth/login', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ username, password }),
        })
        const data = await res.json()
        if (!data.success) throw new Error(data.message)
        set({ user: data.user, token: data.token })
      },

      async register(username, password) {
        const res = await fetch('/api/auth/register', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ username, password }),
        })
        const data = await res.json()
        if (!data.success) throw new Error(data.message)
        set({ user: data.user, token: data.token })
      },

      logout() {
        set({ user: null, token: null })
      },
    }),
    {
      name: 'travel_auth',
      onRehydrateStorage: () => (state) => {
        state?.setHasHydrated(true)
      },
    },
  ),
)
```

- [ ] **Step 2: 删除旧文件**

```bash
rm /Users/joygy/Documents/react/react-travel/src/stores/auth.ts
```

- [ ] **Step 3: 提交**

```bash
git add src/stores/auth.js
git commit -m "refactor: auth store 转 JS，移除 TS 类型和 history 依赖"
```

---

### Task 9: chat store 转 JS

**Files:**
- Delete: `src/stores/chat.ts`
- Create: `src/stores/chat.js`

- [ ] **Step 1: 创建 src/stores/chat.js**

```js
/**
 * 聊天状态管理
 * 进入页面时自动清空，不持久化
 */
import { create } from 'zustand'

export const useChatStore = create(set => ({
  messages: [],
  isLoading: false,
  currentAgentStep: 0,

  addMessage: msg =>
    set(state => ({ messages: [...state.messages, msg] })),

  updateLastMessage: content =>
    set((state) => {
      const messages = [...state.messages]
      const last = messages[messages.length - 1]
      if (last) {
        messages[messages.length - 1] = { ...last, content }
      }
      return { messages }
    }),

  updateLastMessageSteps: step =>
    set((state) => {
      const messages = [...state.messages]
      const last = messages[messages.length - 1]
      if (last) {
        const steps = [...(last.steps || [])]
        const idx = steps.findIndex(s => s.step === step.step)
        if (idx >= 0) {
          steps[idx] = step
        } else {
          steps.push(step)
        }
        messages[messages.length - 1] = { ...last, steps }
      }
      return { messages }
    }),

  clearMessages: () => set({ messages: [], isLoading: false, currentAgentStep: 0 }),
  setLoading: isLoading => set({ isLoading }),
  setCurrentAgentStep: currentAgentStep => set({ currentAgentStep }),
}))
```

- [ ] **Step 2: 删除旧文件**

```bash
rm /Users/joygy/Documents/react/react-travel/src/stores/chat.ts
```

- [ ] **Step 3: 提交**

```bash
git add src/stores/chat.js
git commit -m "refactor: chat store 转 JS"
```

---

### Task 10: itinerary store 转 JS

**Files:**
- Delete: `src/stores/itinerary.ts`
- Create: `src/stores/itinerary.js`

- [ ] **Step 1: 创建 src/stores/itinerary.js**

```js
/**
 * 行程状态管理
 * 管理行程规划页面的数据和加载状态
 */
import { create } from 'zustand'

const initialState = {
  itinerary: [],
  budgetBreakdown: null,
  tips: [],
  weather: null,
  accommodation: [],
  nightlife: [],
  agentSteps: [],
  currentAgentStep: 0,
  isLoading: false,
  shareId: null,
}

export const useItineraryStore = create(set => ({
  ...initialState,

  setItinerary: data => set({ itinerary: data }),
  setBudgetBreakdown: data => set({ budgetBreakdown: data }),
  setTips: tips => set({ tips }),
  setWeather: weather => set({ weather }),
  setAccommodation: data => set({ accommodation: data }),
  setNightlife: data => set({ nightlife: data }),

  addAgentStep: step =>
    set((state) => {
      const exists = state.agentSteps.find(s => s.step === step.step)
      if (exists) {
        return { agentSteps: state.agentSteps.map(s => s.step === step.step ? step : s) }
      }
      return { agentSteps: [...state.agentSteps, step] }
    }),

  setCurrentAgentStep: step => set({ currentAgentStep: step }),
  setLoading: loading => set({ isLoading: loading }),
  setShareId: id => set({ shareId: id }),
  reset: () => set(initialState),
}))
```

- [ ] **Step 2: 删除旧文件**

```bash
rm /Users/joygy/Documents/react/react-travel/src/stores/itinerary.ts
```

- [ ] **Step 3: 提交**

```bash
git add src/stores/itinerary.js
git commit -m "refactor: itinerary store 转 JS"
```

---

### Task 11: hooks 转 JS

**Files:**
- Delete: `src/hooks/useSSE.ts`
- Delete: `src/hooks/useWeather.ts`
- Create: `src/hooks/useSSE.js`
- Create: `src/hooks/useWeather.js`

- [ ] **Step 1: 创建 src/hooks/useSSE.js**

```js
/**
 * SSE (Server-Sent Events) 自定义 Hook
 * 用于处理流式 HTTP 响应，实现 AI 回复的实时流式显示
 */
import { useCallback, useRef } from 'react'

/**
 * SSE Hook
 * @returns {{ sendRequest: Function, abort: Function }}
 */
export function useSSE() {
  const abortControllerRef = useRef(null)

  const sendRequest = useCallback(async (url, body, callbacks) => {
    const controller = new AbortController()
    abortControllerRef.current = controller

    try {
      const res = await fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
        signal: controller.signal,
      })

      if (!res.ok) {
        let msg = `请求失败: HTTP ${res.status}`
        try {
          const errData = await res.json()
          if (errData.message) msg = errData.message
        } catch {}
        throw new Error(msg)
      }

      if (!res.body) {
        throw new Error('响应体为空')
      }

      const reader = res.body.getReader()
      const decoder = new TextDecoder('utf-8')
      let full = ''
      let remainder = ''

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
          if (line.startsWith('data: ')) {
            try {
              const data = JSON.parse(line.slice(6))
              if (data.type === 'chunk' && callbacks.onChunk) {
                full += data.content
                callbacks.onChunk(full)
              }
              if (data.type === 'step' && callbacks.onStep) {
                callbacks.onStep(data)
              }
              if (data.type === 'notice' && callbacks.onNotice) {
                callbacks.onNotice(data.message)
              }
              if (data.type === 'complete' && callbacks.onComplete) {
                callbacks.onComplete(data.data)
              }
            } catch {
              // SSE 解析错误，忽略
            }
          }
        }
      }
    } catch (e) {
      if (e.name !== 'AbortError' && callbacks.onError) {
        callbacks.onError(e)
      }
    } finally {
      callbacks.onFinally?.()
    }
  }, [])

  const abort = useCallback(() => {
    abortControllerRef.current?.abort()
  }, [])

  return { sendRequest, abort }
}
```

- [ ] **Step 2: 创建 src/hooks/useWeather.js**

```js
/**
 * 天气查询 Hook
 * 根据城市名获取天气信息
 */
import { useCallback, useEffect, useRef, useState } from 'react'

export function useWeather() {
  const [weather, setWeather] = useState(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState(null)
  const abortRef = useRef(null)

  const fetchWeather = useCallback((city) => {
    if (!city) {
      setWeather(null)
      return
    }

    abortRef.current?.abort()
    const controller = new AbortController()
    abortRef.current = controller

    setLoading(true)
    setError(null)

    fetch(`/api/weather?city=${encodeURIComponent(city)}`, { signal: controller.signal })
      .then(async (res) => {
        if (!res.ok) throw new Error('查询失败')
        return res.json()
      })
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
  }, [])

  useEffect(() => {
    return () => { abortRef.current?.abort() }
  }, [])

  return { weather, loading, error, fetchWeather }
}
```

- [ ] **Step 3: 删除旧文件**

```bash
rm /Users/joygy/Documents/react/react-travel/src/hooks/useSSE.ts /Users/joygy/Documents/react/react-travel/src/hooks/useWeather.ts
```

- [ ] **Step 4: 提交**

```bash
git add src/hooks/
git commit -m "refactor: hooks 转 JS，移除 TS 类型"
```

---

### Task 12: utils 和 constants 转 JS

**Files:**
- Delete: `src/utils/storage.ts`
- Delete: `src/constants/cities.ts`
- Create: `src/utils/storage.js`
- Create: `src/constants/cities.js`

- [ ] **Step 1: 创建 src/utils/storage.js**

history store 已删除，`saveToHistory` 函数需要移除或改为不依赖 history store。保留缓存相关函数。

```js
/**
 * 本地存储工具模块
 * 使用 localStorage 持久化行程缓存数据
 */

/**
 * 加载行程缓存
 * @param {string} city - 城市名
 * @param {number} budget - 预算
 * @param {number} days - 天数
 * @returns {object|null} 缓存的行程数据
 */
export function loadItineraryCache(city, budget, days) {
  const key = `detail_${city}_${budget}_${days}`
  const raw = localStorage.getItem(key)
  if (!raw) return null
  try {
    return JSON.parse(raw)
  } catch {
    return null
  }
}

/**
 * 保存行程到缓存
 * @param {string} city
 * @param {number} budget
 * @param {number} days
 * @param {object} data
 */
export function saveItineraryCache(city, budget, days, data) {
  const key = `detail_${city}_${budget}_${days}`
  localStorage.setItem(key, JSON.stringify(data))
}
```

- [ ] **Step 2: 创建 src/constants/cities.js**

```js
/**
 * 城市常量数据
 */

/** 所有可选城市列表 */
export const allCities = [
  '北京', '上海', '广州', '深圳', '成都', '杭州', '西安', '重庆',
  '南京', '武汉', '苏州', '长沙', '天津', '郑州', '济南', '青岛',
  '大连', '沈阳', '哈尔滨', '长春', '福州', '厦门', '南昌', '合肥',
  '昆明', '贵阳', '南宁', '桂林', '海口', '三亚', '丽江', '大理',
  '兰州', '乌鲁木齐', '拉萨', '呼和浩特', '太原', '石家庄',
]

/** 热门城市列表 */
export const hotCities = [
  '北京', '上海', '广州', '成都', '杭州', '西安', '重庆', '南京',
  '武汉', '苏州', '长沙', '三亚', '丽江', '大理', '厦门', '青岛',
]
```

- [ ] **Step 3: 删除旧文件**

```bash
rm /Users/joygy/Documents/react/react-travel/src/utils/storage.ts /Users/joygy/Documents/react/react-travel/src/constants/cities.ts
```

- [ ] **Step 4: 提交**

```bash
git add src/utils/ src/constants/
git commit -m "refactor: utils 和 constants 转 JS"
```

---

### Task 13: 新建 api/index.js

**Files:**
- Create: `src/api/index.js`

- [ ] **Step 1: 创建 src/api/index.js**

```js
/**
 * 统一 API 请求模块
 * 集中管理所有后端接口调用
 */
const BASE = '/api'

/**
 * 用户登录
 * @param {string} username
 * @param {string} password
 * @returns {Promise<{success: boolean, user: object, token: string, message?: string}>}
 */
export async function login(username, password) {
  const res = await fetch(`${BASE}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password }),
  })
  return res.json()
}

/**
 * 用户注册
 * @param {string} username
 * @param {string} password
 * @returns {Promise<{success: boolean, user: object, token: string, message?: string}>}
 */
export async function register(username, password) {
  const res = await fetch(`${BASE}/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password }),
  })
  return res.json()
}

/**
 * 查询天气
 * @param {string} city
 * @returns {Promise<object>}
 */
export async function fetchWeather(city) {
  const res = await fetch(`${BASE}/weather?city=${encodeURIComponent(city)}`)
  if (!res.ok) throw new Error('天气查询失败')
  return res.json()
}
```

- [ ] **Step 2: 提交**

```bash
git add src/api/
git commit -m "feat: 新建统一 API 请求模块"
```

---
