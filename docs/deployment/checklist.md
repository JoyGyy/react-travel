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
