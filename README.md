# byo-plugins · 企业知识库 MCP 插件市场

通过 Claude Code 插件机制，一键安装企业知识库问答 MCP（基于 RAGFlow）。
安装后在 `Manage MCP servers` 中显示为 `plugin:byo-plugins:enterprise-kb-qa`。

## 客户安装步骤

### 1. 设置访问令牌（环境变量）

服务端开启了 Bearer Token 鉴权。将管理员发给你的令牌设为环境变量：

**Windows（PowerShell，永久）：**
```powershell
[Environment]::SetEnvironmentVariable("ENTERPRISE_KB_TOKEN", "管理员发给你的令牌", "User")
```

**macOS / Linux：**
```bash
echo 'export ENTERPRISE_KB_TOKEN="管理员发给你的令牌"' >> ~/.zshrc   # 或 ~/.bashrc
source ~/.zshrc
```

> 设置后**重启 Claude Code**（或重开终端），确保其读到新变量。

### 2. 添加本插件市场

在 Claude Code 中执行：

```
/plugin marketplace add https://github.com/chenwenchao/ragflow-mcp-marketplace.git
```

### 3. 安装插件

```
/plugin install enterprise-kb@byo-plugins
```

### 4. 重新加载并验证

```
/reload-plugins
/mcp
```

应能看到 `plugin:byo-plugins:enterprise-kb-qa · ✔ connected · 1 tool`。

### 5. 使用

直接向 Claude 提问即可，例如：

```
从知识库查询 BDS 启动流程是什么？
```

---

## 首次连接的 Cloudflare Access 登录

服务端公网入口由 Cloudflare Access 保护。首次使用时，Claude Code 连接会触发一次浏览器登录
（邮箱 OTP 或你被授权的登录方式）。登录通过后该会话即可正常使用。
若访问被拒，请联系管理员把你加入 Access 授权名单。

---

## 故障排查

| 现象 | 排查 |
|------|------|
| `unauthorized` / 401 | 检查 `ENTERPRISE_KB_TOKEN` 是否设置且与管理员下发一致；重启 Claude Code |
| 连接超时 | 检查网络能否访问服务端域名；公司代理是否放行 |
| 工具未显示 | `/reload-plugins` 后再 `/mcp` 查看；确认 Claude Code 版本较新 |
| Cloudflare Access 拦截 | 联系管理员将你加入授权名单 |

---

## 给管理员

服务端实现、鉴权改造、部署说明见：`ragflow-mcp-server/` 目录。
