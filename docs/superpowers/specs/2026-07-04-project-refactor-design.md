# 项目重构设计文档

> 日期：2026-07-04
> 方案：分层迁移（方案 C）

## 1. 背景与目标

当前项目使用 React 19 + TypeScript + Tailwind CSS v4 + antd-mobile，同时做移动端和桌面端适配。

重构目标：
1. 去掉 Tailwind，改用普通 CSS 文件
2. 去掉 TypeScript，全部使用 JS/JSX
3. 去掉移动端适配，纯 Web 端
4. UI 组件库从 antd-mobile 换成 Ant Design（PC 版）
5. 封装程度适中，只提取全局公共组件
6. 保证代码可读性和可维护性

后端（server/ 目录）已经是纯 JS，**不改动**。

## 2. 页面范围

**保留（5 个页面）：**
- Home（首页）
- Detail（详情页）
- Weather（天气）
- Chat（AI 咨询）
- Login（登录页）

**删除（3 个页面）：**
- History — 历史记录功能不需要
- Profile — 个人中心不需要
- Share — 分享功能不需要

**认证系统保留**（JWT + ProtectedRoute）。

## 3. 目标目录结构

```
react-travel/
├── index.html
├── vite.config.js
├── eslint.config.js
├── package.json
├── server/                    # 不改动
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── api/                   # 统一 API 请求
│   │   └── index.js
│   ├── components/
│   │   ├── ProtectedRoute/
│   │   │   ├── index.jsx
│   │   │   └── style.css
│   │   └── Layout/
│   │       ├── index.jsx
│   │       └── style.css
│   ├── pages/
│   │   ├── Home/
│   │   │   ├── index.jsx
│   │   │   └── style.css
│   │   ├── Detail/
│   │   │   ├── index.jsx
│   │   │   └── style.css
│   │   ├── Weather/
│   │   │   ├── index.jsx
│   │   │   └── style.css
│   │   ├── Chat/
│   │   │   ├── index.jsx
│   │   │   └── style.css
│   │   └── Login/
│   │       ├── index.jsx
│   │       └── style.css
│   ├── stores/
│   │   ├── auth.js
│   │   ├── chat.js
│   │   └── itinerary.js
│   ├── hooks/
│   │   ├── useSSE.js
│   │   └── useWeather.js
│   ├── utils/
│   │   └── storage.js
│   ├── constants/
│   │   └── cities.js
│   └── styles/
│       └── global.css
```

## 4. CSS 架构

### 4.1 全局样式 `src/styles/global.css`

保留现有内容，做以下调整：
- 移除 `@import "tailwindcss"` 和 `@theme` 块
- 保留所有 CSS 变量（`--c-*`、`--font-*`、`--shadow-*`）
- 保留基础 reset、动画 keyframes、Markdown 样式
- 保留 scrollbar 和 selection 样式

### 4.2 组件/页面样式

- 每个组件/页面文件夹下有一个 `style.css`
- 使用 BEM 命名约定：`.block__element--modifier`
- 通过 `import './style.css'` 在 JSX 中引入
- 不使用任何 Tailwind utility class

### 4.3 布局策略

- 不做响应式，固定桌面端布局
- 主内容区 `max-width: 1200px`，水平居中
- 使用 Flexbox 布局

## 5. 组件迁移计划

### 5.1 保留并重写的组件

| 原组件 | 说明 |
|---|---|
| ProtectedRoute | 路由守卫，去掉 TS 类型 |
| ChatBubble | 聊天气泡，改用普通 CSS |
| AgentSteps | Agent 步骤展示 |
| ChatAgentSteps | Chat Agent 步骤 |
| WeatherCard | 天气卡片，可用 Ant Design Card |
| HomeWeather | 首页天气展示 |
| SpotItem | 景点列表项 |
| BudgetTable | 预算表格，可用 Ant Design Table |
| AccommodationCard | 住宿卡片 |
| RAGSource | RAG 来源展示 |

### 5.2 删除的组件

| 原组件 | 原因 |
|---|---|
| TabBar | 移动端底部导航，纯 Web 不需要 |
| ShareCard | 分享功能删除 |
| SharePopup | 分享功能删除 |

### 5.3 新增的组件

| 新组件 | 说明 |
|---|---|
| Layout | 从 router/index.tsx 提取，包含顶部导航和内容区 |

### 5.4 封装原则

- 只提取**全局公共**组件（Layout、ProtectedRoute）
- 页面内的小组件直接写在页面里，不单独抽文件
- 一个组件只做一件事

## 6. 页面迁移详情

### Home（首页）
- 去掉所有 Tailwind class，换成普通 CSS
- 城市下拉搜索保留
- 天气展示区域保留
- 热门城市保留
- Ant Design 替换：Toast → `message`，图标换成 `@ant-design/icons`

### Detail（详情页）
- 行程展示、AgentSteps、BudgetTable 等保留
- 去掉移动端特有的布局调整

### Weather（天气页）
- 天气查询和展示保留

### Chat（AI 咨询）
- SSE 流式对话保留
- ChatBubble、ChatAgentSteps 组件保留

### Login（登录页）
- 登录/注册表单保留
- Ant Design 替换：Toast → `message`

### 共性变化
- 所有 `import type` 语句删除
- 所有 TypeScript 类型注解删除
- `useState<string>` → `useState`
- 函数参数类型删除

## 7. Store、Hooks 和工具函数

### 7.1 Store

| Store | 状态 | 说明 |
|---|---|---|
| auth.js | 保留，转 JS | 登录状态 + persist 水合 |
| chat.js | 保留，转 JS | 聊天消息管理 |
| itinerary.js | 保留，转 JS | Detail 页行程数据 |
| history.ts | 删除 | History 页面已删除 |

### 7.2 Hooks

| Hook | 状态 | 说明 |
|---|---|---|
| useSSE.js | 保留，转 JS | SSE 流式请求 |
| useWeather.js | 保留，转 JS | 天气查询 |

### 7.3 工具函数

| 文件 | 状态 |
|---|---|
| utils/storage.js | 保留，转 JS |
| constants/cities.js | 保留，转 JS |

### 7.4 新增 `api/index.js`

统一管理所有 API 请求，替代散落在 store 和页面中的 `fetch` 调用。

## 8. Ant Design 集成

### 8.1 替换对照

| 原 antd-mobile | 替换为 Ant Design |
|---|---|
| `Toast.show()` | `message.success()` / `message.error()` |
| `Dialog.show()` | `Modal.confirm()` |
| `antd-mobile-icons` | `@ant-design/icons` |
| 无 | `Button`、`Input`、`InputNumber`、`Select`、`Card`、`Table`、`Spin` |

### 8.2 主题定制

```jsx
// App.jsx
import { ConfigProvider } from 'antd'

export default function App() {
  return (
    <ConfigProvider
      theme={{
        token: {
          colorPrimary: '#6366f1',
          borderRadius: 8,
          fontFamily: "'Noto Sans SC', sans-serif",
        },
      }}
    >
      <RouterProvider router={router} />
    </ConfigProvider>
  )
}
```

## 9. 迁移执行顺序

### 阶段 1：基础设施

1. 清理 package.json — 移除 TS/tailwind/antd-mobile/vitest 相关依赖，加入 antd + @ant-design/icons
2. 删除 tsconfig*.json、vitest.config.ts
3. vite.config.ts → vite.config.js，去掉 tailwind 插件
4. main.tsx → main.jsx
5. App.tsx → App.jsx，加入 ConfigProvider
6. global.css — 去掉 `@import "tailwindcss"` 和 `@theme` 块

### 阶段 2：基础层

1. stores/*.ts → stores/*.js，去掉类型注解
2. hooks/*.ts → hooks/*.js
3. utils/storage.ts → utils/storage.js
4. constants/cities.ts → constants/cities.js
5. 新建 src/api/index.js
6. 删除 history.ts、types/index.ts、test/ 目录

### 阶段 3：组件层

1. 新建 Layout 组件（从 router 提取）
2. 逐个重写组件：去掉 TS、去掉 Tailwind、写对应 CSS
3. 删除 TabBar、ShareCard、SharePopup

### 阶段 4：页面层

1. 逐个重写页面：Home → Login → Weather → Chat → Detail
2. router/index.tsx → router/index.jsx
3. 删除 History、Profile、Share 页面
4. 最终全量检查

每个阶段完成后项目应该能正常运行（`pnpm dev`）。

## 10. 不做的事情

- 不改后端代码（server/ 目录）
- 不做移动端适配
- 不引入 CSS Modules 或 CSS-in-JS
- 不引入状态管理替代方案（继续用 Zustand）
- 不引入新的测试框架
- 不做过度封装（不搞 HOC、render props 等模式）
