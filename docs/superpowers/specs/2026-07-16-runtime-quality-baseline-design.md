# 运行时质量基线与最小 E2E 设计

## 背景

上一轮项目规范化已经完成统一 API 层、SSE 稳定性、后端校验、错误处理和阿里云 ECS 部署文档。运行时验收发现两个前端控制台问题：

1. Ant Design v5 在 React 19 下提示兼容警告。
2. 静态 `message.*` 无法消费 `ConfigProvider` 动态主题上下文。

项目即将部署到阿里云 ECS，下一轮应优先保证关键路径可重复验证，并建立轻量质量基线，而不是进行大规模架构重构。

## 目标

1. 消除 Ant Design v5 + React 19 的主要兼容警告。
2. 将静态 `message.*` 迁移为能消费上下文的用法。
3. 新增最小 E2E 验证，覆盖首页、登录/注册、天气查询和 AI 咨询。
4. 为 API/SSE 建立轻量类型基线，便于后续逐步 TypeScript 化。
5. 增加包体积观察文档，记录当前构建产物，为下一轮拆包优化提供依据。

## 非目标

1. 不做全量 TypeScript 迁移。
2. 不把 E2E 强制纳入 `pnpm check`，避免本地日常检查变慢。
3. 不在本轮做大规模拆包、虚拟列表或 UI 重构。
4. 不迁移到 Ant Design v6。
5. 不修改后端核心 Agent/RAG 逻辑。

## 外部依据

Ant Design 官方文档建议：

- Ant Design v5 配合 React 19 时，在入口引入 `@ant-design/v5-patch-for-react-19`。
- 静态 `message.*` 无法消费 React Context，建议使用 `App.useApp()` 或 `message.useMessage()`。
- `App` 组件应放在应用外层，使消息、弹窗等反馈 API 能读取上下文主题。

## 方案

采用“运行时兼容 + 最小 E2E + 轻量类型基线 + 包体积观察”的方案。

### 阶段一：运行时兼容修复

改动范围：

- 安装 `@ant-design/v5-patch-for-react-19`。
- 在前端入口 `src/main.jsx` 引入 patch。
- 在 `src/App.jsx` 中引入 AntD `App` 容器，并保留现有 `ConfigProvider` 主题。
- 将以下静态 message 使用改为上下文 API：
  - `src/components/ProtectedRoute/index.jsx`
  - `src/pages/Home/index.jsx`
  - `src/pages/Login/index.jsx`
- 评估 `src/pages/Chat/index.jsx` 中的 `Modal.confirm`，优先迁移为 `Modal.useModal`，避免未来上下文问题。

验收：

- 浏览器控制台不再出现 AntD React 19 兼容警告。
- 浏览器控制台不再出现静态 message 上下文警告。
- 登录、未登录跳转提示、聊天清空确认仍正常工作。

### 阶段二：最小 E2E 验证

改动范围：

- 新增开发依赖 `@playwright/test`。
- 新增 `playwright.config.js`。
- 新增 `e2e/app.spec.js`。
- 新增脚本：
  - `pnpm e2e`
  - 可选 `pnpm e2e:headed`
- E2E 自动启动后端和前端，使用真实浏览器验证关键路径。

覆盖场景：

1. 首页能打开并显示“AI 旅行规划师”。
2. 注册本地测试账号后返回首页。
3. 天气页查询“北京”后显示天气概览。
4. 登录后进入 AI 咨询页，点击“北京有哪些必去的景点？”，能看到景点结果或参考来源。

验收：

- `pnpm e2e` 可独立运行通过。
- E2E 不依赖真实 LLM key；后端可走 Mock/知识库降级。
- 测试账号使用时间戳或随机后缀，避免重复注册冲突。

### 阶段三：API/SSE 类型基线

改动范围：

- 新增 `src/types/api.js` 或 `src/types/api.d.ts`，描述 API 错误、天气响应、SSE 事件和认证响应。
- 在 `src/api/client.js`、`src/api/auth.js`、`src/api/weather.js`、`src/hooks/useSSE.js` 中补充 JSDoc 类型。
- 不强制把文件改成 `.ts`，避免一次性扩大改动范围。

验收：

- `pnpm typecheck` 继续通过。
- API/SSE 的输入输出结构能被编辑器理解。
- 不引入大面积隐式 any 清理任务。

### 阶段四：包体积观察文档

改动范围：

- 新增 `docs/performance/bundle-baseline.md`。
- 记录当前 `pnpm build` 产物中的主要 chunk：
  - `index` 主包
  - `Chat` 页面包
  - `Home` 页面包
  - 关键 CSS 包
- 给出下一轮优化方向：
  - Markdown 渲染延迟加载或独立拆分。
  - 检查 AntD icons 引入方式。
  - 保持路由级懒加载。
  - 必要时再引入 bundle analyzer。

验收：

- 文档记录当前可观察基线。
- 本轮不为了追求数字而做高风险拆包。

## 验证策略

每个阶段至少执行：

- `pnpm check`
- `pnpm --dir server check`

E2E 阶段额外执行：

- `pnpm e2e`

最终运行时验证：

- 启动后端和前端。
- 浏览器访问首页、登录页、天气页和聊天页。
- 查看控制台是否仍有 AntD 兼容或静态 message 警告。

## 风险与控制

1. `@ant-design/v5-patch-for-react-19` 是兼容补丁，后续升级 AntD v6 时需要移除：在文档中记录。
2. E2E 需要浏览器依赖，若本地缺少 Playwright 浏览器，需要执行 `pnpm exec playwright install`。
3. 聊天 E2E 依赖后端响应时间，断言应等待关键文本，不使用过短超时。
4. 类型基线只做边界约束，不追求全量严格，避免阻塞部署。
5. 包体积文档只记录现状和方向，不做没有验证的大规模优化。

## 最终交付

1. AntD React 19 兼容修复。
2. 上下文安全的 message/Modal 用法。
3. 可重复执行的最小 E2E。
4. API/SSE 类型基线。
5. 包体积观察文档。
6. 最终评估：验证结果、剩余警告、下一轮优化建议。
