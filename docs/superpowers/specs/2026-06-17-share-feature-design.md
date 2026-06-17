# 行程分享裂变功能设计文档

**日期：** 2026-06-17
**状态：** 待审阅

## 目标

用户规划完行程后，可以一键生成精美行程卡片图并获取分享链接，用于微信朋友圈、小红书等社交平台传播。朋友看到后点击链接进入落地页，引导新用户转化。

**核心流程：** 用户规划行程 → 生成分享卡片图/链接 → 分享到社交平台 → 朋友点击 → 落地页展示行程 → CTA 引导新用户规划

## 技术方案

客户端生成分享卡片图（`html-to-image`）+ 后端公开分享链接存储。

选择此方案的原因：
- 微信/小红书是图片驱动平台，必须有卡片图
- 客户端生成零服务器成本，`html-to-image` 库成熟
- 短链接 + OG 标签覆盖「点击打开」场景
- 后续可平滑升级为服务端 Puppeteer 渲染

---

## 一、分享卡片设计

### 卡片内容（从上到下）

1. **顶部装饰区域** — 渐变背景（复用现有 `#0f172a → #334155`），带模糊圆形装饰元素
2. **主标题** — 城市名 + 天数 + 预算范围，Noto Serif SC 大字
3. **每日行程亮点** — 每天一行，展示景点名称列表，带图标分隔
4. **底部品牌区** — 水印文字「用 AI 旅行助手规划你的旅程」+ 产品 logo

### 技术实现

- **库：** `html-to-image`（基于 SVG foreignObject，比 html2canvas 更轻量、字体渲染更好）
- **组件：** `ShareCard` 组件，`position: fixed; left: -9999px` 隐藏渲染，生成图片后移除 DOM
- **尺寸：** 750×1334px（手机竖屏 2x 清晰度）
- **格式：** PNG，`quality: 0.92`
- **操作：** 点击「分享」→ 弹出预览弹窗 → 「保存图片」/「复制链接」

### 样式规范

- 复用现有设计系统：渐变色、Noto Serif SC 标题字体、`rounded-2xl` 圆角、阴影系统
- 行程亮点文字超出时截断并显示省略号（每天最多展示 4 个景点）
- 卡片背景色：`#ffffff`，与深色 Hero 区域形成对比

---

## 二、后端分享链接系统

### 数据存储

新增文件 `server/data/shared_itineraries.json`，结构为数组：

```json
[
  {
    "id": "a1b2c3d4",
    "city": "杭州",
    "days": 3,
    "budget": 3000,
    "itinerary": { /* 完整行程数据，复用 Itinerary 类型 */ },
    "createdAt": "2026-06-17T10:30:00.000Z",
    "viewCount": 0
  }
]
```

选择 JSON 文件而非 SQLite 的理由：分享数据只增不改，读多写少，无需迁移数据库。

### 短 ID 生成

使用 `nanoid` 的 `customAlphabet`（数字 + 小写字母，8 位），支撑千万级分享量。

### 新增 API

#### `POST /api/travel/share`

创建分享链接。

- **认证：** 需要 JWT（复用现有认证流程）
- **请求体：** `{ city, days, budget, itinerary }`
- **响应：** `{ shareId: "a1b2c3d4", shareUrl: "/share/a1b2c3d4" }`
  - 前端拼接完整 URL：`window.location.origin + shareUrl`
- **幂等性：** 同一用户对同一行程（city + days + budget 组合）重复分享时返回已有的 shareId
- **并发安全：** JSON 文件写入使用 `fs.writeFile` 原子写（先写临时文件再 rename），避免并发请求导致数据损坏

#### `GET /api/travel/share/:id`

公开接口，无需认证。

- **响应：** `{ city, days, budget, itinerary, createdAt, viewCount }`
- **副作用：** 递增 viewCount 并写回 JSON 文件
- **404：** 分享不存在时返回 `{ error: '分享不存在或已过期' }`

### 数据清理

暂不实现自动过期。JSON 文件定期手动清理或后续增加定时任务。

---

## 三、分享落地页 `/share/:id`

### 页面设计

复用 Detail 页面的行程展示组件，移除所有操作按钮，改为只读展示：

- **顶部 Hero：** 「XX 的 X 天 XX 旅行计划」（不暴露具体预算数字）
- **行程展示：** 复用 Detail 页面的每日行程、天气卡片、贴士等组件
- **底部固定 CTA 栏：**
  - 「我也要规划一次旅行 →」按钮，醒目渐变色，带动画
  - 显示「已有 XX 人通过分享链接规划了旅行」社交证明（基于 viewCount 聚合）

### 路由配置

- 新增 `/share/:id` 路由，不需要认证
- 使用 `React.lazy` 懒加载
- 在 `ProtectedRoute` 之外

### OG 标签方案

SPA 无法直接服务端渲染 OG 标签，采用以下方案：

1. 后端 `GET /api/travel/share/:id` 响应中附带 `ogTitle`、`ogDescription` 字段
2. 前端在页面加载后通过 DOM 操作动态注入 `<meta>` 标签
3. `og:image` 使用模板化的服务端预览图 URL（后续迭代，首版可暂不实现）

**动态 meta 注入：**
```ts
document.title = `${city} ${days}天旅行计划`
document.querySelector('meta[property="og:title"]')?.setAttribute('content', title)
```

---

## 四、前端集成与用户流程

### 分享入口位置

1. **Detail 页面** — 行程生成完成后，底部操作栏新增「分享行程」按钮，与「保存到历史」并列
2. **History 页面** — 行程卡片上新增分享图标按钮，复用已保存的行程数据

### 分享流程

1. 用户点击「分享行程」
2. 弹出底部弹窗（antd-mobile `Popup` 组件），内容：
   - 行程卡片预览（`ShareCard` 组件，可见区域展示）
   - 「保存图片」按钮 — 触发 `html-to-image` 生成 PNG 并下载
   - 「复制链接」按钮 — 调用 `POST /api/travel/share`，拿到链接后复制到剪贴板
3. 使用 `navigator.clipboard.writeText` + antd-mobile `Toast` 提示「链接已复制」
4. 首次分享时上传行程数据到后端，后续分享同一行程复用已有 shareId

### 状态管理变更

- **itinerary store** — 新增 `shareId: string | null` 字段，行程生成后关联分享 ID
- **history store** — 每条记录新增 `shareId` 字段，避免重复创建分享

### 错误处理

- 图片生成失败 → Toast 提示「图片生成失败，请重试」
- 网络请求失败 → Toast 提示「分享链接创建失败」
- 剪贴板 API 不可用 → 降级为显示链接文本框，用户手动复制

---

## 五、涉及文件变更

### 新增文件

| 文件 | 说明 |
|------|------|
| `src/components/ShareCard/index.tsx` | 分享卡片组件 |
| `src/components/SharePopup/index.tsx` | 分享弹窗组件（预览 + 操作按钮） |
| `src/pages/Share/index.tsx` | 分享落地页 |
| `server/routes/share.js` | 分享 API 路由 |
| `server/data/shared_itineraries.json` | 分享数据存储 |

### 修改文件

| 文件 | 变更 |
|------|------|
| `src/pages/Detail/index.tsx` | 新增分享按钮入口 |
| `src/pages/History/index.tsx` | 新增分享图标按钮 |
| `src/stores/itinerary.ts` | 新增 shareId 字段 |
| `src/stores/history.ts` | 记录新增 shareId 字段 |
| `src/types/index.ts` | 新增分享相关类型定义 |
| `src/router/index.tsx` | 新增 /share/:id 路由 |
| `server/index.js` | 注册分享路由 |
| `package.json` | 新增 `html-to-image`、`nanoid` 依赖 |
| `server/package.json` | 新增 `nanoid` 依赖 |

---

## 六、依赖项

- `html-to-image` — 客户端 DOM 转图片（~50KB gzipped）
- `nanoid` — 短 ID 生成（前后端共用）

## 七、不做的事情（YAGNI）

- 不做服务端 Puppeteer 渲染（首版客户端够用）
- 不做分享链接过期清理（手动管理）
- 不做分享数据统计面板（viewCount 够用）
- 不做图片上传 CDN（首版直接客户端下载）
- 不做微信 JS-SDK 深度集成（直接保存图片 + 复制链接覆盖大部分场景）
