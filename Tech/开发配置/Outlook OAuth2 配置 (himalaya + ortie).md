---
tags:
  - Linux
---

# Outlook OAuth2 配置 (himalaya + ortie)

> 配置日期：2026-03-28
> 环境：ECS 服务器 (Linux)

## 概述

在 ECS 服务器上配置 himalaya 邮件客户端，通过 OAuth2 认证访问 Outlook 邮箱。使用 ortie 作为 OAuth2 token 管理工具。

## 组件说明

- **himalaya**: CLI 邮件客户端，支持 IMAP/SMTP
- **ortie**: OAuth2 token 管理工具，负责 token 获取和刷新
- **mimosa 替代**: 自定义脚本存储 token

## Azure 应用注册

1. 访问 https://entra.microsoft.com → 应用注册 → 新建注册
2. 名称：`himalaya-mail`（或任意名称）
3. 支持的账户类型：**任何组织目录中的账户和个人 Microsoft 账户**
4. 重定向 URI：添加平台 → 移动和桌面应用程序 → `http://localhost`
5. 记录：应用程序 (客户端) ID

### API 权限

添加以下 Microsoft Graph 权限：
- `IMAP.AccessAsUser.All`
- `SMTP.Send`

## ortie 配置

配置文件：`~/.config/ortie/config.toml`

```toml
[accounts.outlook]
default = true

# 客户端 ID（从 Azure 获取）
client-id = "client-id"

# 注意：PKCE 模式不需要 client-secret
# client-secret.raw = "xxx"

# OAuth2 端点（个人账户用 /consumers/）
endpoints.authorization = "https://login.microsoftonline.com/consumers/oauth2/v2.0/authorize"
endpoints.token = "https://login.microsoftonline.com/consumers/oauth2/v2.0/token"
endpoints.redirection = "http://localhost:18080"

# 必需的 scope
scopes = ["https://outlook.office.com/IMAP.AccessAsUser.All", "https://outlook.office.com/SMTP.Send"]

# PKCE 启用（推荐，更安全）
pkce = true

# 自动刷新 token
auto-refresh = true

# Token 存储（使用自定义脚本替代 mimosa）
storage.read.command = ["/home/ecs-user/.local/bin/ortie-read"]
storage.write.command = ["/home/ecs-user/.local/bin/ortie-write"]
```

## Token 存储脚本

由于 mimosa 未安装，创建了简单的替代脚本：

**~/.local/bin/ortie-read:**
```bash
#!/bin/bash
cat ~/.local/share/ortie/tokens.json 2>/dev/null || echo '{}'
```

**~/.local/bin/ortie-write:**
```bash
#!/bin/bash
cat > ~/.local/share/ortie/tokens.json
```

## 首次授权流程

### 1. 生成授权链接

```bash
ortie auth get --account outlook
```

输出类似：
```
Created authorization request with:
 - state: xxx
 - pkce: xxx

Click on the link to start the authorization process: https://login.microsoftonline.com/consumers/oauth2/v2.0/authorize?...
```

### 2. 浏览器授权

在浏览器中打开输出的链接，登录 Microsoft 账户并授权。

### 3. 完成授权

授权后浏览器会跳转到 `http://localhost:18080/?code=xxx&state=xxx`，复制完整 URL。

### 4. 换取 Token

```bash
ortie auth resume --account outlook \
  --state "<state值>" \
  --pkce "<pkce值>" \
  --redirect-uri "http://localhost:18080/" \
  "<完整的redirect URL>"
```

成功输出：
```
Access token successfully issued (expires in 1h)
```

## 常见问题

### AADSTS900144: The request body must contain 'scope'

**原因**: ortie 配置中 `scopes = []` 为空

**解决**: 添加必需的 scopes

### AADSTS90023: Public clients can't send a client secret

**原因**: PKCE 模式与 client-secret 不兼容

**解决**: 注释掉 `client-secret` 配置项

### userAudience 配置错误

**原因**: Azure 应用配置为 " 仅个人账户 "，但使用了 `/common/` 端点

**解决**: 改用 `/consumers/` 端点，或修改 Azure 应用为 " 任何组织目录中的账户和个人 Microsoft 账户 "

## himalaya 配置

配置文件：`~/.config/himalaya/config.toml`

```toml
[accounts.outlook]
email = "listener1381@outlook.com"
default = true
display-name = "listener1381"
downloads-dir = "/home/ecs-user/Downloads"

backend.type = "imap"
backend.host = "outlook.office365.com"
backend.port = 993
backend.login = "listener1381@outlook.com"
backend.auth.type = "oauth2"
backend.auth.method = "xoauth2"
backend.auth.access-token.cmd = "ortie --account outlook token show --auto-refresh"
backend.auth.pkce = true
backend.auth.scopes = ["https://outlook.office.com/IMAP.AccessAsUser.All", "https://outlook.office.com/SMTP.Send"]

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.mail.outlook.com"
message.send.backend.port = 587
message.send.backend.starttls = true
message.send.backend.login = "listener1381@outlook.com"
message.send.backend.auth.type = "oauth2"
message.send.backend.auth.method = "xoauth2"
message.send.backend.auth.access-token.cmd = "ortie --account outlook token show --auto-refresh"
message.send.backend.auth.pkce = true
message.send.backend.auth.scopes = ["https://outlook.office.com/IMAP.AccessAsUser.All", "https://outlook.office.com/SMTP.Send"]
```

## 日常使用

```bash
# 列出邮件
himalaya envelope list --account outlook

# 读取邮件
himalaya message read <ID> --account outlook

# 写邮件
himalaya message write --account outlook

# 查看文件夹
himalaya folder list --account outlook

# 搜索邮件
himalaya envelope list --account outlook from:xxx subject:xxx
```

## Token 刷新

- Access token 有效期：1 小时
- `auto-refresh = true` 时，ortie 会在 token 过期时自动刷新
- 刷新使用 refresh token，无需用户再次授权
- 如果长时间不用（refresh token 过期），需要重新授权

## 参考链接

- [himalaya GitHub](https://github.com/pimalaya/himalaya)
- [ortie GitHub](https://github.com/pimalaya/ortie)
- [Microsoft OAuth2 文档](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow)
