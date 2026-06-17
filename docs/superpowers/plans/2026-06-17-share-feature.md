# 行程分享裂变功能实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 用户规划完行程后可生成精美卡片图并获取分享链接，用于社交平台传播裂变。

**Architecture:** 客户端用 `html-to-image` 生成分享卡片 PNG，后端用 JSON 文件存储公开分享链接，新增 `/share/:id` 落地页展示只读行程。

**Tech Stack:** html-to-image, nanoid, React, Express, Zustand

## Global Constraints

- 包管理器：pnpm
- 前端路径别名：`@/*` → `src/*`
- 后端 ESM 模块（`"type": "module"`）
- 样式复用现有设计系统 CSS 变量（`--c-terracotta`、`--c-ink` 等）
- UI 组件库：antd-mobile + antd-mobile-icons

---

### Task 1: 安装依赖

**Files:**
- Modify: `package.json`
- Modify: `server/package.json`

- [ ] 前端安装 `html-to-image`：`pnpm add html-to-image`
- [ ] 后端安装 `nanoid`：`cd server && pnpm add nanoid`
- [ ] 提交

---

### Task 2: 类型定义

**Files:**
- Modify: `src/types/index.ts`

**Produces:** `ShareData` 接口，供后续 Task 3、5、6 使用

- [ ] 在 `src/types/index.ts` 末尾新增：

```ts
/** 分享数据 - 存储在后端的公开行程 */
export interface ShareData {
  id: string           // 短 ID（8 位）
  city: string
  days: number
  budget: number
  itinerary: ItineraryResult  // 完整行程数据
  createdAt: string    // ISO 时间戳
  viewCount: number
}
```

- [ ] 在 `HistoryRecord` 接口中新增可选字段：`shareId?: string`
- [ ] 提交

---

### Task 3: 后端分享服务

**Files:**
- Create: `server/services/share.js`
- Create: `server/data/shared_itineraries.json`（初始化为 `[]`）

**Produces:** `createShare(data)` 和 `getShare(id)` 函数，供 Task 4 使用

- [ ] 创建 `server/data/shared_itineraries.json`，内容为 `[]`
- [ ] 创建 `server/services/share.js`，实现：
  - `createShare({ city, days, budget, itinerary })` — 生成 nanoid，写入 JSON，返回 shareId
  - `getShare(id)` — 读取并递增 viewCount，返回数据或 null
  - 使用 `fs.writeFileSync` + 临时文件 rename 实现原子写入
  - 幂等：同一 city+days+budget 组合返回已有 shareId
- [ ] 提交

---

### Task 4: 后端分享路由

**Files:**
- Create: `server/routes/share.js`
- Modify: `server/index.js`

**Consumes:** `createShare()`, `getShare()` from Task 3

- [ ] 创建 `server/routes/share.js`：
  - `POST /share` — 需 JWT 认证，调用 `createShare`，返回 `{ shareId, shareUrl: '/share/' + shareId }`
  - `GET /share/:id` — 公开，调用 `getShare`，返回行程数据或 404
- [ ] 修改 `server/index.js`，在路由挂载区新增：`app.use('/api/travel', shareRoutes)`
- [ ] 启动后端验证两个接口可用
- [ ] 提交

---

### Task 5: 前端状态管理变更

**Files:**
- Modify: `src/stores/itinerary.ts`
- Modify: `src/stores/history.ts`

**Produces:** `shareId` 字段和 `setShareId` 方法，供 Task 7 使用

- [ ] `itinerary.ts` 新增 `shareId: string | null` 初始值 `null`，新增 `setShareId` 方法
- [ ] `history.ts` 的 `addRecord` 支持 `shareId` 字段（类型已在 Task 2 添加）
- [ ] 提交

---

### Task 6: ShareCard 分享卡片组件

**Files:**
- Create: `src/components/ShareCard/index.tsx`

**Consumes:** `ItineraryResult` from `@/types`
**Produces:** `ShareCard` 组件，供 Task 7 使用

- [ ] 创建 `ShareCard` 组件，接收 `city, days, budget, itinerary` props
- [ ] 卡片布局：
  - 顶部渐变 Hero（复用 `#0f172a → #1e293b`）
  - 城市名 + 天数 + 预算标题
  - 每日行程亮点（每天一行，最多 4 个景点名，超出省略）
  - 底部品牌水印
- [ ] 尺寸 750×1334px，白色背景，圆角 `rounded-2xl`
- [ ] 组件默认隐藏（`position: fixed; left: -9999px`），由外部控制显示
- [ ] 提交

---

### Task 7: SharePopup 分享弹窗组件

**Files:**
- Create: `src/components/SharePopup/index.tsx`

**Consumes:** `ShareCard` from Task 6, `ShareData` from Task 2
**Produces:** `SharePopup` 组件，供 Task 8、9 使用

- [ ] 创建 `SharePopup` 组件，props：`visible, onClose, city, days, budget, itinerary`
- [ ] 使用 antd-mobile `Popup` 底部弹窗
- [ ] 内容：
  - `ShareCard` 预览（可见区域，非隐藏渲染）
  - 「保存图片」按钮 — 调用 `html-to-image` 的 `toPng` 生成图片并下载
  - 「复制链接」按钮 — POST `/api/travel/share`，拿到 shareUrl 后 `navigator.clipboard.writeText`，Toast 提示
- [ ] 错误处理：图片失败 Toast 提示，网络失败 Toast 提示，剪贴板不可用降级显示链接文本
- [ ] 提交

---

### Task 8: Detail 页面集成分享按钮

**Files:**
- Modify: `src/pages/Detail/index.tsx`

**Consumes:** `SharePopup` from Task 7

- [ ] 在 Detail 页面底部「咨询 AI」按钮上方新增「分享行程」按钮
- [ ] 按钮样式：与「保存到历史」风格一致
- [ ] 点击后弹出 `SharePopup`
- [ ] 仅在行程数据加载完成后显示（`itinerary.length > 0 && !showLoading`）
- [ ] 提交

---

### Task 9: History 页面集成分享按钮

**Files:**
- Modify: `src/pages/History/index.tsx`

**Consumes:** `SharePopup` from Task 7

- [ ] 在 History 卡片头部删除按钮旁新增分享图标按钮
- [ ] 点击后弹出 `SharePopup`，传入该条记录的行程数据
- [ ] 提交

---

### Task 10: 分享落地页

**Files:**
- Create: `src/pages/Share/index.tsx`

**Consumes:** `ShareData` from Task 2

- [ ] 创建 Share 页面，从 URL 取 `:id`，调用 `GET /api/travel/share/:id` 获取数据
- [ ] 复用 Detail 页面的展示组件（SpotItem、BudgetTable、WeatherCard 等）
- [ ] Hero 区域显示「XX 的 X 天旅行计划」
- [ ] 底部固定 CTA 栏：「我也要规划一次旅行 →」按钮，点击跳转 `/`
- [ ] 显示「已有 XX 人查看过此行程」社交证明
- [ ] 动态设置 `document.title` 和 OG meta 标签
- [ ] 加载中和错误状态处理
- [ ] 提交

---

### Task 11: 路由配置

**Files:**
- Modify: `src/router/index.tsx`

**Consumes:** `Share` page from Task 10

- [ ] 新增 `const Share = React.lazy(() => import('@/pages/Share'))`
- [ ] 在路由 children 中新增 `{ path: 'share/:id', element: <Share /> }`（不需要 ProtectedRoute）
- [ ] `showTabbar` 逻辑中排除 `/share` 路径（分享页不显示底部导航）
- [ ] 提交

---

### Task 12: 端到端验证

- [ ] 启动前后端，规划一次行程
- [ ] 在 Detail 页面点击「分享行程」，验证弹窗显示
- [ ] 点击「保存图片」，验证 PNG 下载成功
- [ ] 点击「复制链接」，验证链接已复制且 Toast 提示
- [ ] 在新标签页打开分享链接，验证落地页正常显示
- [ ] 在 History 页面点击分享图标，验证功能一致
- [ ] 提交
