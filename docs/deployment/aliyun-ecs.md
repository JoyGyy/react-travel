# 阿里云 ECS 部署说明

本文档适用于个人项目首版上线：前端由 Nginx 托管静态资源，后端由 PM2 守护 Node.js 服务，Nginx 将 `/api` 反向代理到本机 `3030` 端口。

## 1. 部署前准备

### ECS 基础要求

- 系统：Ubuntu、Debian、Alibaba Cloud Linux 或其他常见 Linux 发行版均可。
- Node.js：建议 20+。
- 包管理器：pnpm。
- 进程管理：PM2。
- Web 服务：Nginx。
- 阿里云安全组：至少开放 SSH、80、443 端口。

备案完成前，可以先用 ECS 公网 IP 验证服务；备案完成后，再把域名 A 记录解析到 ECS 公网 IP。

## 2. 安装运行环境

以下命令以常见 Linux 环境为例，具体包管理器命令可按系统调整。

```bash
node -v
corepack enable
corepack prepare pnpm@latest --activate
pnpm -v
pnpm add -g pm2
pm2 -v
```

安装 Nginx：

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

如果使用 Alibaba Cloud Linux，可用系统对应的软件源安装 Nginx。

## 3. 上传或拉取代码

推荐目录：

```text
/var/www/react-travel
```

示例：

```bash
sudo mkdir -p /var/www
sudo chown -R $USER:$USER /var/www
cd /var/www
git clone <your-repo-url> react-travel
cd react-travel
```

## 4. 配置环境变量

后端会从仓库根目录读取 `.env`，所以 `.env` 必须放在项目根目录，而不是 `server/` 目录。

```bash
cp .env.example .env
```

生产环境至少需要修改：

```env
NODE_ENV=production
PORT=3030
JWT_SECRET=replace-with-a-random-secret-at-least-32-chars
CORS_ORIGIN=https://your-domain.com,https://www.your-domain.com
```

注意：

- `JWT_SECRET` 生产环境至少 32 个字符。
- `SILICONFLOW_API_KEY` 和 `DEEPSEEK_API_KEY` 可以先留空；留空时会进入 Mock/知识库降级模式，便于先完成部署演示。
- 如果备案未完成、暂时用公网 IP 验证，`CORS_ORIGIN` 可以先配置为对应 IP 访问地址，例如 `http://公网IP`。

## 5. 安装依赖并构建前端

前端依赖：

```bash
pnpm install
```

后端依赖：

```bash
cd server
pnpm install
cd ..
```

构建前端：

```bash
pnpm build
```

构建产物位于：

```text
/var/www/react-travel/dist
```

## 6. 启动后端服务

项目根目录已经提供 PM2 示例配置：`ecosystem.config.cjs`。

```bash
pm2 start ecosystem.config.cjs
pm2 save
pm2 status
```

验证后端健康检查：

```bash
curl http://127.0.0.1:3030/api/health
```

预期返回类似：

```json
{"status":"ok","timestamp":"2026-07-15T00:00:00.000Z"}
```

查看日志：

```bash
pm2 logs react-travel-server
```

## 7. 配置 Nginx

仓库提供示例文件：

```text
deploy/nginx/react-travel.conf.example
```

复制到 Nginx 配置目录：

```bash
sudo cp deploy/nginx/react-travel.conf.example /etc/nginx/sites-available/react-travel.conf
```

编辑域名：

```bash
sudo vim /etc/nginx/sites-available/react-travel.conf
```

把：

```nginx
server_name your-domain.com www.your-domain.com;
```

改为你的真实域名。备案完成前也可以临时写 ECS 公网 IP，或使用默认 server 配置测试。

启用站点：

```bash
sudo ln -s /etc/nginx/sites-available/react-travel.conf /etc/nginx/sites-enabled/react-travel.conf
sudo nginx -t
sudo systemctl reload nginx
```

如果系统没有 `sites-available` / `sites-enabled` 结构，可以把配置放到 `/etc/nginx/conf.d/react-travel.conf`。

## 8. 验证访问

备案完成前：

```text
http://ECS公网IP
http://ECS公网IP/api/health
```

备案完成后：

```text
http://your-domain.com
http://your-domain.com/api/health
```

应验证：

- 首页能正常打开。
- 登录页能打开。
- 天气查询能返回数据或明确错误。
- AI 咨询和行程规划不会永久 loading。
- `/api/health` 正常返回 JSON。

## 9. 常见问题

### 访问首页 404

检查 Nginx 是否配置了 SPA fallback：

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

### `/api` 请求失败

检查后端是否运行：

```bash
pm2 status
curl http://127.0.0.1:3030/api/health
```

检查 Nginx 反代：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3030/api/;
}
```

### 生产环境启动失败

检查 `.env`：

- `JWT_SECRET` 是否存在且至少 32 个字符。
- `CORS_ORIGIN` 是否配置为真实访问来源。
- `.env` 是否位于仓库根目录。

### SSE 流式响应中断

确认 Nginx 中关闭代理缓冲并延长读取超时：

```nginx
proxy_buffering off;
proxy_read_timeout 180s;
```

## 10. HTTPS 后续建议

备案完成、域名解析生效后，建议配置 HTTPS。可以使用阿里云免费证书或 Let's Encrypt，并将 80 自动跳转到 443。
