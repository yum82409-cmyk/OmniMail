# OmniMail 部署过程记录

日期：2026-09-13

本记录对应 Cloudflare 账户中的首次生产部署。敏感信息（设置令牌、密码、API
密钥和 Cookie）不写入仓库。

## 1. 本地发布前检查

在 `F:\Projects\OmniMail` 执行：

```powershell
npm run build
npm test
npm run test:worker
npm run deploy -- --dry-run
```

结果：

- 生产构建成功，前端静态资源生成到 `dist/`。
- 单元测试通过：157 个测试文件，676 个测试。
- Worker 测试通过：3 个测试文件，10 个测试。
- 部署预检成功，识别 D1、R2、Queue、Workflow、AI 和 Static Assets 绑定。
- `npm run test:extension` 未完成，失败发生在 Playwright 启动浏览器进程阶段，
  Windows 返回 `spawn UNKNOWN`，没有进入扩展断言。该问题属于本机浏览器运行库，
  不代表扩展功能测试通过。

## 2. Cloudflare 授权

使用 Wrangler 设备授权完成登录：

```powershell
npx wrangler login --device --browser=false --use-keyring
npx wrangler whoami
```

授权需要在外部浏览器打开 Cloudflare 设备授权页面并手动批准。登录后使用
`npx wrangler whoami` 验证账户和令牌权限，不在仓库保存 OAuth 凭据。

## 3. 首次部署

执行：

```powershell
npm run deploy
```

由于是首次部署，部署脚本先发布 Worker 以触发资源自动创建，再执行远程 D1
迁移。Cloudflare 自动创建了：

- D1：`omni-mail-db`
- R2：`omni-mail-mail-bucket`
- R2：`omni-mail-backups`
- Queue：`omnimail-mail`
- 死信队列：`omnimail-mail-dead`
- Workflow：`omni-mail-backup`、`omni-mail-cleanup`

随后完成 34 个 D1 迁移，迁移记录校验通过。Worker 地址为：

```text
https://omni-mail.yum82409.workers.dev
```

## 4. 首次应用初始化

在 Worker 配置中设置管理员邮箱和一次性 `SETUP_TOKEN`，通过部署地址的首次运行
页面创建主管理员账户。初始化完成后再次读取 `/api/config`，确认：

- `setupComplete: true`
- `databaseReady: true`
- `storageReady: true`
- `queueReady: true`

确认管理员账户创建后，删除一次性令牌：

```powershell
npx wrangler secret delete SETUP_TOKEN --name omni-mail
```

## 5. 线上验证

```powershell
Invoke-WebRequest `
  -Uri "https://omni-mail.yum82409.workers.dev/api/health" `
  -Method Get
```

实测返回 `200` 和 `{"ok":true}`。初始化完成后公开配置显示
`setupComplete: true`，删除令牌后 `setupTokenReady: false`。

## 6. 后续配置

当前实例已经可以登录使用，但主动发信、Email Routing、iCloud、Gmail、Microsoft、
QQ、NAVER 和 Yandex 等能力仍需按需配置对应的域名、Secret、OAuth 或邮箱授权信息。
配置真实邮箱前，应使用专用测试账号完成收发信、同步、附件、队列重试和权限隔离验收。
