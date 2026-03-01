# OpenClaw + Rocket.Chat 部署指南

## 概述

Rocket.Chat 是一个开源的团队协作平台，支持自托管、国内直连、端到端加密。本文档介绍如何自建 Rocket.Chat 并与 OpenClaw 对接。

## 方案对比

| 方案 | 难度 | 国内直连 | E2EE | OpenClaw 支持 |
|------|------|----------|------|---------------|
| Rocket.Chat (自建) | 中 | ✅ | ✅ | ✅ (openclaw-rocketchat) |
| Matrix (自建) | 高 | ✅ | ✅ | ✅ (已有) |
| Telegram | 低 | ✅ | ✅ | ✅ (原生) |

## 前置要求

- 一台境内服务器（VPS）
- Docker 和 Docker Compose
- 域名（可选，用于 HTTPS）

---

## 一、安装 Rocket.Chat

### 方法1: Docker 一键部署

```bash
# 创建目录
mkdir -p ~/rocket.chat
cd ~/rocket.chat

# 创建 docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  rocketchat:
    image: rocketchat/rocket.chat:6.0
    container_name: rocketchat
    ports:
      - "3000:3000"
    environment:
      - PORT=3000
      - ROOT_URL=http://localhost:3000
      - MONGO_URL=mongodb://mongo:27017/rocketchat
      - MONGO_OPLOG_URL=mongodb://mongo:27017/local
      - DEPLOY_METHOD=docker
    depends_on:
      - mongo

  mongo:
    image: mongo:6.0
    container_name: mongo
    volumes:
      - mongo_data:/data/db
    command: mongod --replSet rs0 --oplogSize 128

volumes:
  mongo_data:
EOF

# 启动
docker-compose up -d

# 等待启动完成（约2-3分钟）
docker logs -f rocketchat
# 看到 "Rocket.Chat server started" 即启动完成
```

### 方法2: 懒人安装脚本

```bash
curl -sL https://raw.githubusercontent.com/rocketchat/Rocket.Chat/develop/docker-compose.yml > docker-compose.yml

# 编辑 ROOT_URL 为你的域名
vim docker-compose.yml

docker-compose up -d
```

---

## 二、初始化配置

### 1. 访问 Web 界面

```
http://你的服务器IP:3000
```

### 2. 创建管理员账户

按照向导填写：
- 管理员用户名
- 管理员邮箱
- 密码

### 3. 创建机器人

1. 登录后，点击右上角头像 → **Administration**
2. → **Users** → **New User**
3. 填写：
   - Name: `openclaw-bot`
   - Username: `openclawbot`
   - Email: `bot@yourdomain.com`
   - Password: 设置一个强密码
   - Roles: `bot`
4. 点击 **Save**

### 4. 获取 API Token

1. 以机器人登录
2. 点击头像 → **Profile**
3. 滚动到 **API** → **Generate Personal Access Token**
4. 填写名称（如 `openclaw`）→ **Generate**
5. 复制生成的 Token

---

## 三、配置 OpenClaw

### 1. 查看 OpenClaw 配置

```bash
cat ~/.openclaw/openclaw.json | jq '.channels'
```

### 2. 添加 Rocket.Chat 信道

```bash
openclaw configure set channels.rocketchat.enabled true
openclaw configure set channels.rocketchat.url http://你的服务器IP:3000
openclaw configure set channels.rocketchat.user openclawbot
openclaw configure set channels.rocketchat.password 机器人密码
```

或者直接编辑：

```bash
cat > /tmp/rocketchat_config.json << 'EOF'
{
  "enabled": true,
  "url": "http://你的服务器IP:3000",
  "user": "openclawbot",
  "password": "机器人密码"
}
EOF

# 合并配置
jq '.channels.rocketchat = load("/tmp/rocketchat_config.json")' ~/.openclaw/openclaw.json > /tmp/openclaw.json
mv /tmp/openclaw.json ~/.openclaw/openclaw.json
```

### 3. 重启 OpenClaw

```bash
openclaw gateway restart
```

---

## 四、测试

### 1. 在 Rocket.Chat 创建一个测试房间

1. 点击 **+** → **Create Channel**
2. 命名为 `test`
3. 添加机器人 `openclawbot` 到房间

### 2. 发送测试消息

在测试房间里发送：
```
hello
```

### 3. 检查 OpenClaw 响应

机器人应该自动回复。

---

## 五、可选配置

### 1. 配置 HTTPS（推荐）

使用 Nginx 反向代理：

```nginx
server {
    listen 443 ssl;
    server_name rocketchat.yourdomain.com;

    ssl_certificate /path/to/ssl.crt;
    ssl_certificate_key /path/to/ssl.key;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

### 2. 启用端到端加密

Rocket.Chat 支持 E2EE：
1. 用户设置 → **Security** → **Enable E2E Encryption**
2. 需要保存恢复密钥

### 3. 配置推送通知

安装 Rocket.Chat 的移动 App，登录后会自动推送。

---

## 六、常见问题

### Q: 连接超时
A: 检查防火墙是否开放 3000 端口：
```bash
sudo ufw allow 3000
```

### Q: 机器人不回复
A: 检查 OpenClaw 日志：
```bash
journalctl --user -u openclaw-gateway -f
```

### Q: 如何更新版本？
```bash
docker-compose pull
docker-compose up -d
```

---

## 七、一键卸载

```bash
cd ~/rocket.chat
docker-compose down -v
rm -rf ~/rocket.chat
```

然后移除 OpenClaw 配置：
```bash
jq 'del(.channels.rocketchat)' ~/.openclaw/openclaw.json > /tmp/openclaw.json
mv /tmp/openclaw.json ~/.openclaw/openclaw.json
```

---

## 总结

| 项目 | 值 |
|------|-----|
| Web 界面 | http://服务器IP:3000 |
| 机器人用户 | openclawbot |
| 协议 | HTTP/WebSocket |
| 加密 | 可选 E2EE |

这样就完成了 Rocket.Chat 的自建和 OpenClaw 对接！

*文档版本: 1.0*
*最后更新: 2026-03-02*