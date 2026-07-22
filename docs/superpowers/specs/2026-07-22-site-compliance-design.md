# 站点备案与基础合规完善设计

## 背景

项目已经新增公安备案展示，备案号为 `浙公网安备33019202003146号`。现在需要继续补齐个人备案网站上线后常见的基础展示与安全配置：ICP备案号、用户协议、隐私政策、登录页备案展示统一，以及 Nginx 安全响应头示例。

项目是 React 19 + TypeScript SPA，路由集中在 `src/router/index.tsx`，全站布局在 `src/components/Layout/index.tsx`，登录页在 `src/pages/Login/index.tsx`，部署示例在 `deploy/nginx/react-travel.conf.example`。

## 目标

1. 在全站 Footer 展示 ICP 备案号：`浙ICP备2026054747号-1`。
2. 保留并统一展示公安备案号：`浙公网安备33019202003146号`。
3. 新增用户协议和隐私政策页面，并在 Footer、登录页提供入口。
4. 协议文案按个人备案低暴露方案编写，不公开个人姓名。
5. 协议联系方式使用公开邮箱：`joygyzhi@outlook.com`。
6. 更新 Nginx 示例，补充 HTTPS 跳转和基础安全响应头。
7. 不影响现有首页、登录、AI 咨询、天气、景点、个人中心等功能。

## 非目标

1. 不提供正式法律意见；协议文案是个人项目上线基础版本，后续可由专业人士审阅。
2. 不新增后台 CMS 或数据库配置项，备案与协议内容先使用静态代码管理。
3. 不公开个人姓名、身份证号、手机号等敏感信息。
4. 不强制修改真实服务器上的 Nginx 配置；本次只更新仓库中的示例配置。
5. 不新增 Cookie 弹窗或复杂授权管理，因为当前项目未使用广告追踪、第三方营销 Cookie 或复杂个人信息处理流程。

## 推荐方案

采用“轻量完整方案”：静态协议页面 + 统一备案页脚 + Nginx 示例安全增强。

备选方案对比：

1. **最小备案方案**
   - 只补 ICP 备案号和登录页展示。
   - 优点：改动最少。
   - 缺点：登录注册和 AI 对话产品缺少用户协议、隐私政策入口。

2. **轻量完整方案（采用）**
   - Footer 展示 ICP、公安备案、用户协议、隐私政策。
   - 新增 `/terms` 和 `/privacy` 两个静态页面。
   - 登录页复用同一套备案/协议链接。
   - Nginx 示例补安全头与 HTTPS 配置。
   - 优点：覆盖当前个人项目上线最常见合规需求，成本可控。
   - 缺点：协议内容仍是通用版本，不等同于正式法务审查。

3. **正式法务方案**
   - 使用真实主体名称、完整联系方式、数据处理清单和法律条款。
   - 优点：更正式。
   - 缺点：会公开更多主体信息，并且需要额外法务确认。

## 信息展示设计

### 全站 Footer

在 `src/components/Layout/index.tsx` 中扩展现有备案 Footer：

- `© 2026 Travel AI`
- `浙ICP备2026054747号-1`
  - 链接：`https://beian.miit.gov.cn/`
  - 新窗口打开，使用 `rel="noreferrer"`
- 公安备案：`浙公网安备33019202003146号`
  - 链接：`https://beian.mps.gov.cn/#/query/webSearch?code=33019202003146`
  - 保留公安图标 `public/images/beian-gongan.png`
- `用户协议`
  - 链接：`/terms`
- `隐私政策`
  - 链接：`/privacy`

样式继续保持低干扰：居中、12px、低对比、可换行。备案和协议链接在键盘聚焦时有可见 outline，保证基础可访问性。

### 登录页统一展示

`src/pages/Login/index.tsx` 当前左侧底部只有英文版权文案。改为复用布局里的备案/协议链接组件，或抽出独立 `ComplianceLinks` 组件供 Layout 与 Login 共同使用。

推荐抽出组件，避免重复维护备案号、链接和邮箱相关入口。

## 组件结构设计

新增共享组件：`src/components/ComplianceFooter/index.tsx` 和 `src/components/ComplianceFooter/style.css`。

组件职责：

1. 集中维护 ICP 备案号、公安备案号、协议路由、外部备案查询链接。
2. 支持不同场景：
   - 全站 Footer：展示版权 + ICP + 公安备案 + 协议入口。
   - 登录页紧凑展示：同样展示备案与协议入口，但适配深色图片背景。
3. 不依赖业务 store，不触发网络请求。

建议 API：

```ts
interface ComplianceFooterProps {
  variant?: 'default' | 'overlay'
  showCopyright?: boolean
}
```

- `default`：用于普通页面 Footer，浅色背景。
- `overlay`：用于登录页图片背景，文字颜色更适合深色背景。
- `showCopyright`：默认 `true`。

`Layout` 使用默认样式；`Login` 使用 `overlay` 样式。

## 路由与页面设计

### 路由

在 `src/router/index.tsx` 新增公开路由：

- `/terms` → 用户协议页面
- `/privacy` → 隐私政策页面

两个页面不需要登录保护。

### 页面组织

新增页面：

- `src/pages/Terms/index.tsx`
- `src/pages/Terms/style.css`
- `src/pages/Privacy/index.tsx`
- `src/pages/Privacy/style.css`

如果两者样式高度一致，可以新增共享样式：

- `src/pages/Legal/style.css`

推荐使用共享样式，避免重复。

### 用户协议内容

用户协议使用“本站 / Travel AI / ICP 备案主体”作为运营表述，不公开个人姓名。

核心章节：

1. 协议适用范围。
2. 服务内容：AI 旅行规划、天气查询、景点推荐、对话咨询等。
3. 账号使用：用户应妥善保管账号信息，不得冒用他人身份。
4. 用户行为规范：不得提交违法、有害、侵权、攻击性内容，不得滥用接口或破坏服务。
5. AI 内容说明：AI 生成结果仅供旅行参考，实际出行应以官方渠道、现场信息和个人判断为准。
6. 知识产权：页面、代码、文案、设计和数据内容的权利边界。
7. 服务变更与中止：个人项目可能根据维护情况调整服务。
8. 责任限制：因第三方服务、网络、政策、天气变化等造成的信息变化，本站不承担超出法律规定的责任。
9. 联系方式：`joygyzhi@outlook.com`。

### 隐私政策内容

隐私政策同样采用低暴露主体表述。

核心章节：

1. 收集的信息：用户名、登录状态、旅行目的地、预算、天数、偏好、聊天内容、天气查询城市、浏览器基础信息等。
2. 使用目的：账号登录、生成行程、AI 咨询、天气查询、故障排查和安全防护。
3. 本地存储：前端使用 `travel_auth` 持久化登录状态；说明用户可通过退出登录或浏览器清理数据。
4. 第三方服务：AI 服务、天气服务、托管/服务器基础设施可能处理必要请求数据。
5. 信息保护：采取合理技术措施，但互联网服务无法保证绝对安全。
6. 用户权利：用户可通过邮箱请求查询、更正、删除相关信息。
7. 未成年人提示：未成年人应在监护人指导下使用。
8. 政策更新：页面更新后生效。
9. 联系方式：`joygyzhi@outlook.com`。

## Nginx 示例设计

更新 `deploy/nginx/react-travel.conf.example`，保留示例性质。

推荐结构：

1. HTTP server：
   - `listen 80;`
   - `server_name joygytrip.cn www.joygytrip.cn;`
   - `return 301 https://$host$request_uri;`

2. HTTPS server：
   - `listen 443 ssl http2;`
   - `server_name joygytrip.cn www.joygytrip.cn;`
   - 证书路径沿用仓库当前证书文件名示例：`/etc/nginx/ssl/www.joygytrip.cn.pem` 和 `/etc/nginx/ssl/www.joygytrip.cn.key`。
   - `root /var/www/react-travel/dist;`
   - `index index.html;`

3. 安全响应头：
   - `Strict-Transport-Security "max-age=31536000; includeSubDomains" always;`
   - `X-Content-Type-Options "nosniff" always;`
   - `X-Frame-Options "SAMEORIGIN" always;`
   - `Referrer-Policy "strict-origin-when-cross-origin" always;`
   - `Permissions-Policy "geolocation=(), microphone=(), camera=()" always;`

暂不加入严格 `Content-Security-Policy`，因为项目使用 Vite、Ant Design、Sentry 和外部 AI/API 场景，CSP 需要结合真实域名和资源域名调试，避免示例误导导致线上资源被拦截。

4. 现有能力保留：
   - `/api/` 反向代理到 `127.0.0.1:3030`。
   - `proxy_buffering off` 保持 SSE 流式响应。
   - SPA fallback：`try_files $uri $uri/ /index.html;`
   - 静态资源缓存：保留 7 天和 `Cache-Control`。

## 测试与验证

实现后需要执行：

1. `pnpm lint`
2. `pnpm typecheck`
3. `pnpm build`
4. 如新增测试，执行 `pnpm test:run`

手动检查点：

1. 首页、登录页、受保护页面底部均能看到 ICP 和公安备案信息。
2. 点击 ICP 跳转到工信部备案站。
3. 点击公安备案跳转到公安备案查询页，并携带备案数字码。
4. `/terms` 和 `/privacy` 可直接访问，不需要登录。
5. 登录页视觉上不出现两套冲突的版权/备案区域。
6. 移动端 Footer 可换行且不遮挡主要内容。

## 风险与取舍

1. 协议文案是个人项目基础版本，不构成正式法律意见。
2. 不公开个人姓名可以降低隐私暴露，但文案需要使用“ICP备案主体 / 本站运营者”这类泛称。
3. Nginx 示例中的 HSTS 只适合确认 HTTPS 稳定后启用；如果证书或 HTTPS 配置未稳定，应谨慎上线。
4. CSP 暂不配置，避免破坏现有资源加载；后续可以在确认所有资源域名后单独设计。

## 实施边界

本次实现应控制在以下文件范围内：

- `src/components/ComplianceFooter/index.tsx`
- `src/components/ComplianceFooter/style.css`
- `src/components/Layout/index.tsx`
- `src/components/Layout/style.css`（移除旧备案 Footer 样式或改为引用新组件样式）
- `src/pages/Login/index.tsx`
- `src/pages/Login/style.css`
- `src/pages/Terms/index.tsx`
- `src/pages/Terms/style.css` 或 `src/pages/Legal/style.css`
- `src/pages/Privacy/index.tsx`
- `src/pages/Privacy/style.css` 或 `src/pages/Legal/style.css`
- `src/router/index.tsx`
- `deploy/nginx/react-travel.conf.example`
- 必要时更新 `src/components/Layout/__tests__/Layout.test.tsx`

不改动后端接口、数据库结构、认证逻辑和 AI 服务逻辑。
