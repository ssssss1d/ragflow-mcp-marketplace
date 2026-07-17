# Enterprise Knowledge Base MCP Plugin

企业知识库问答 MCP 插件，基于 RAGFlow，通过 Claude Code 一键安装使用。

---

## 目录

1. [架构概述](#架构概述)
2. [内部测试（开发环境）](#内部测试开发环境)
3. [部署到云服务器](#部署到云服务器)
4. [公网测试](#公网测试)
5. [客户使用说明](#客户使用说明)
6. [故障排查](#故障排查)

---

## 架构概述

```
┌─────────────────┐      SSE over HTTPS      ┌─────────────────┐
│  客户端         │ ───────────────────────► │  云服务器        │
│  Claude Code    │   Bearer Token 鉴权      │  MCP Server     │
│                 │                          │  (Port 8765)    │
└─────────────────┘                          └────────┬────────┘
                                                      │
                                                      │ API 调用
                                                      ▼
                                             ┌─────────────────┐
                                             │  RAGFlow        │
                                             │  (Port 9380)    │
                                             │  + 知识库数据    │
                                             └─────────────────┘
```

**组件说明：**

| 组件 | 作用 | 端口 |
|------|------|------|
| MCP Server | 提供 SSE 接口，Bearer Token 鉴权 | 8765 |
| RAGFlow | 知识库管理，文档解析，问答 API | 9380 |
| RagFlow Web | 知识库管理界面 | 80 |

---

## 内部测试（开发环境）

适用于：开发者在内网环境测试 MCP 功能。

### 前置条件

- WSL2 / Linux 环境
- Docker 已安装
- Python 3.10+
- Git

### 步骤 1：启动 RAGFlow

```bash
# 进入 RAGFlow 目录
cd ~/ragflow_gitee/docker

# 启动容器（CPU 模式）
sudo docker compose -p ragflow up -d

# 等待服务就绪（约 2 分钟）
sudo docker logs -f ragflow-ragflow-cpu-1
# 看到 "ready for connections" 表示成功
```

### 步骤 2：配置 RagFlow

1. 浏览器访问 http://localhost
2. 注册/登录账号
3. 创建知识库，上传测试文档
4. 获取 API Token：点击右上角头像 → 设置 → API Token → 复制

### 步骤 3：配置 MCP Server

```bash
cd ~/ByoMCP/ragflow-mcp-server

# 编辑 config.json，填入 RagFlow API Token
cat > config.json << 'EOF'
{
  "ragflow_api_url": "http://localhost:9380/api/v1",
  "ragflow_api_token": "ragflow-你的API_TOKEN",
  "mcp_port": 8765,
  "mcp_access_token": "testtoken-123456"
}
EOF
```

### 步骤 4：启动 MCP Server

```bash
# 方式 A：使用测试脚本
bash run-wsl-test.sh

# 方式 B：直接启动
pip install -r requirements.txt
python mcp_server.py --sse --port 8765
```

### 步骤 5：本地验证

```bash
# 测试 SSE 端点（应返回 200）
curl -I -H "Authorization: Bearer testtoken-123456" http://localhost:8765/sse

# 测试无 token（应返回 401）
curl -I http://localhost:8765/sse
```

### 步骤 6：内网跨机测试

让其他内网电脑连接测试：

**在 Windows 管理员 PowerShell 执行：**

```powershell
# 获取 WSL IP（在 WSL 中执行）
# ip addr show eth0 | grep -oP 'inet \K[\d.]+'
# 假设是 172.24.87.180

# 获取 Windows 内网 IP
ipconfig
# 假设是 192.168.113.74

# 添加端口转发
netsh interface portproxy add v4tov4 listenport=8765 listenaddress=0.0.0.0 connectport=8765 connectaddress=172.24.87.180

# 添加防火墙规则
netsh advfirewall firewall add rule name="MCP Server 8765" dir=in action=allow protocol=tcp localport=8765
```

**在另一台电脑测试：**

```bash
curl -I -H "Authorization: Bearer testtoken-123456" http://192.168.113.74:8765/sse
# 应返回 HTTP/1.1 200 OK
```

---

## 部署到云服务器

### 服务器要求

- Ubuntu 22.04+ / CentOS 8+
- 4GB+ 内存
- 50GB+ 磁盘
- 公网 IP + 域名（推荐）

### 步骤 1：安装依赖

```bash
# 安装 Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 安装 Python
sudo apt install python3 python3-pip python3-venv -y
```

### 步骤 2：部署 RAGFlow

```bash
# 克隆 RAGFlow
git clone https://github.com/infiniflow/ragflow.git
cd ragflow/docker

# 修改 .env（使用国内镜像）
sed -i 's|^RAGFLOW_IMAGE=.*|RAGFLOW_IMAGE=swr.cn-north-4.myhuaweicloud.com/infiniflow/ragflow:v0.26.4|' .env

# 启动
sudo docker compose up -d
```

### 步骤 3：部署 MCP Server

```bash
# 克隆 marketplace（包含 server）
git clone https://github.com/ssssss1d/ragflow-mcp-marketplace.git
cd ragflow-mcp-marketplace

# 创建配置
cat > config.json << 'EOF'
{
  "ragflow_api_url": "http://localhost:9380/api/v1",
  "ragflow_api_token": "ragflow-你的API_TOKEN",
  "mcp_port": 8765,
  "mcp_access_token": "生成一个强密码"
}
EOF

# 生成强密码
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

### 步骤 4：用 systemd 守护进程

```bash
sudo tee /etc/systemd/system/ragflow-mcp.service << 'EOF'
[Unit]
Description=RAGFlow MCP Server
After=network.target docker.service

[Service]
Type=simple
WorkingDirectory=/opt/ragflow-mcp-server
ExecStart=/opt/ragflow-mcp-server/.venv/bin/python mcp_server.py --sse --port 8765
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now ragflow-mcp
sudo systemctl status ragflow-mcp
```

### 步骤 5：配置公网访问

**方式 A：Cloudflare Tunnel（推荐，无需开端口）**

```bash
# 安装 cloudflared
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
sudo mv cloudflared /usr/local/bin/
sudo chmod +x /usr/local/bin/cloudflared

# 登录
cloudflared tunnel login

# 创建隧道
cloudflared tunnel create ragflow-mcp

# 配置路由
cloudflared tunnel route dns ragflow-mcp kb.yourdomain.com

# 启动隧道
cloudflared tunnel run ragflow-mcp --url http://localhost:8765
```

**方式 B：Nginx + Let's Encrypt**

```bash
# 安装 Nginx
sudo apt install nginx certbot python3-certbot-nginx -y

# 配置
sudo tee /etc/nginx/sites-available/mcp << 'EOF'
server {
    listen 80;
    server_name kb.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:8765;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/mcp /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx

# 获取证书
sudo certbot --nginx -d kb.yourdomain.com
```

---

## 公网测试

### 验证服务运行

```bash
# 测试 MCP 端点
curl -I -H "Authorization: Bearer 你的TOKEN" https://kb.yourdomain.com/sse

# 应返回 HTTP/2 200
```

### 更新 Marketplace 配置

编辑 `enterprise-kb/.mcp.json`：

```json
{
  "mcpServers": {
    "enterprise-kb-qa": {
      "type": "sse",
      "url": "https://kb.yourdomain.com/sse",
      "headers": {
        "Authorization": "Bearer ${ENTERPRISE_KB_TOKEN}"
      }
    }
  }
}
```

提交并推送：

```bash
git add . && git commit -m "update: 公网地址" && git push
```

### 模拟客户测试

在任意能访问公网的电脑上：

```bash
# 1. 设置环境变量
export ENTERPRISE_KB_TOKEN=你的TOKEN

# 2. 安装插件
/plugin marketplace add https://github.com/ssssss1d/ragflow-mcp-marketplace.git
/plugin install enterprise-kb@byo-plugins

# 3. 验证
/mcp
# 应显示 plugin:byo-plugins:enterprise-kb-qa · connected

# 4. 测试问答
从知识库查询 xxx 是什么？
```

---

## 客户使用说明

### 安装步骤

#### 1. 设置访问令牌

将管理员发给你的令牌设为环境变量：

**Windows（PowerShell）：**
```powershell
# 临时（当前终端）
$env:ENTERPRISE_KB_TOKEN = "管理员发给你的令牌"

# 永久（需重启 Claude Code）
[Environment]::SetEnvironmentVariable("ENTERPRISE_KB_TOKEN", "管理员发给你的令牌", "User")
```

**macOS / Linux：**
```bash
# 临时
export ENTERPRISE_KB_TOKEN="管理员发给你的令牌"

# 永久（加入 shell 配置）
echo 'export ENTERPRISE_KB_TOKEN="管理员发给你的令牌"' >> ~/.bashrc
source ~/.bashrc
```

> 设置后需**重启 Claude Code** 才能生效。

#### 2. 添加插件市场

在 Claude Code 中执行：

```
/plugin marketplace add https://github.com/ssssss1d/ragflow-mcp-marketplace.git
```

#### 3. 安装插件

```
/plugin install enterprise-kb@byo-plugins
```

#### 4. 重新加载

```
/reload-plugins
```

#### 5. 验证安装

```
/mcp
```

应显示：
```
plugin:byo-plugins:enterprise-kb-qa · connected · 1 tool
```

### 使用方式

直接向 Claude 提问，插件会自动调用知识库：

```
从知识库查询：BDS 启动流程是什么？
```

```
帮我查一下产品手册中关于安装的步骤
```

---

## 故障排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| unauthorized / 401 | Token 未设置或错误 | 检查 ENTERPRISE_KB_TOKEN 环境变量 |
| 连接超时 | 网络不通 | 检查防火墙、代理设置 |
| 工具未显示 | 插件未加载 | /reload-plugins 后重试 |
| 知识库暂无内容 | RagFlow 无数据 | 登录 RagFlow 上传文档 |
| SSE 连接断开 | 网络不稳定 | 检查网络，重连即可 |

### 查看日志

**MCP Server 日志：**
```bash
sudo journalctl -u ragflow-mcp -f
```

**RAGFlow 日志：**
```bash
sudo docker logs ragflow-ragflow-cpu-1 -f
```

---

## 安全建议

1. **生产环境必须设置强 Token**：`python -c "import secrets; print(secrets.token_urlsafe(32))"`
2. **使用 HTTPS**：防止 Token 被窃听
3. **定期轮换 Token**：建议每季度更换
4. **限制 RagFlow API Token 权限**：只授予必要权限
5. **Cloudflare Access**：增加邮箱验证层（可选）

---

## 联系支持

- GitHub Issues: https://github.com/ssssss1d/ragflow-mcp-marketplace/issues
