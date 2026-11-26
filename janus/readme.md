<!--
 * @Author: jackning 270580156@qq.com
 * @Date: 2025-08-05 07:53:04
 * @LastEditors: jackning 270580156@qq.com
 * @LastEditTime: 2025-08-05 13:52:39
 * @Description: bytedesk.com https://github.com/Bytedesk/bytedesk
 *   Please be aware of the BSL license restrictions before installing Bytedesk IM – 
 *  selling, reselling, or hosting Bytedesk IM as a service is a breach of the terms and automatically terminates your rights under the license. 
 *  Business Source License 1.1: https://github.com/Bytedesk/bytedesk/blob/main/LICENSE 
 *  contact: 270580156@qq.com 
 * 
 * Copyright (c) 2025 by bytedesk.com, All Rights Reserved. 
-->
# Janus WebRTC Server

本项目推荐使用 Docker Compose 方式启动 Janus，并与 coturn 配套使用。传统裸机启动方式仍保留在文末。

## 快速开始（Docker Compose）

使用以下任意一个 Compose 文件：

- starter/src/main/resources/compose.yaml
- starter/src/main/resources/compose-rabbitmq.yaml

示例：

```bash
cd /Users/ningjinpeng/Desktop/git/private/github/bytedesk-private
docker compose -f starter/src/main/resources/compose-rabbitmq.yaml up -d
```

## 端口与访问地址

Janus 容器端口在 Compose 中映射如下：

- 18089 -> 8088 (HTTP API)
- 18090 -> 8089 (HTTPS API)
- 18091 -> 7088 (Admin/Monitor HTTP API，路径 /admin，需 admin_secret)
- 18180 -> 80 (Web 入口/可用于 WS 代理)
- 18181 -> 443 (HTTPS Web)
- 18188 -> 8188 (WebSocket API)
- 17188 -> 7188 (Admin/Monitor WebSocket API)
- 18178 -> 3478/udp (STUN/ICE)
- 18179 -> 3478/tcp (STUN/ICE TCP)
- 11000-11200/udp -> 10000-10200/udp (媒体端口)

常用访问链接：

- Janus HTTP API: http://localhost:18089/janus
- Janus Admin: http://localhost:18089/admin

## API 认证说明

- 主 Janus API（`janus.jcfg` 中 `token_auth = false`）：微语采用「浏览器直连 Janus WS」架构（前端 `janus-gateway` SDK 直连 ws-url，不经后端信令中继），存储型 token 需经 Admin API `add_token` 动态发放且重启即失效，不适用该架构，故本部署包默认关闭主 API 认证；主 API 安全依赖网络层隔离（内网部署/反向代理限流）。
- Admin/Monitor API（`admin_secret = "janusoverlord"`）：始终开启，需在所有请求中携带 `admin_secret`（与后端 `webrtc.properties` 的 `bytedesk.webrtc.janus.admin.secret` 保持一致）；错误 secret 返回 403。
- 验证命令：
  - `curl -sS -X POST -H 'Content-Type: application/json' --data '{"janus":"create","transaction":"t"}' http://localhost:18089/janus` → 应返回 `janus: success`（此前 token_auth=true 时代码路径会返回 403 "Unauthorized request (wrong or missing secret/token)"）
- Janus 演示页面 (如果启用了 web): http://localhost:18180

## STUN/TURN 说明（coturn）

Compose 同时包含 coturn，默认账号：

- user: bytedesk
- pass: bytedesk123
- STUN/TURN 端口: 13478 (映射到容器内 3478)

前端推荐配置：

- STUN: stun:localhost:13478
- TURN: turn:localhost:13478?transport=udp (user/pass 如上)

## 前端 Janus 配置（web）

已在前端默认使用本地 Compose 地址，并支持 VITE_* 覆盖：

- VITE_JANUS_HOST
- VITE_JANUS_HTTP_PORT
- VITE_JANUS_WS_PORT
- VITE_JANUS_SCHEME
- VITE_JANUS_WS_SCHEME
- VITE_JANUS_SERVER
- VITE_JANUS_WS
- VITE_JANUS_ADMIN

默认值示例：

- server: http://localhost:18089/janus
- websocket: ws://localhost:18180/janus
- admin: http://localhost:18089/admin

## 裸机运行（可选）

### 配置文件

```bash
# 修改 /opt/janus/etc/janus/janus.transport.http.jcfg
general: {
    ip = "0.0.0.0"
}
admin: {
    admin_http = true
    admin_ip = "0.0.0.0"
}
```

### 前台运行

```bash
# 安装路径
cd /opt/janus
# 配置文件
./etc/janus/janus.jcfg
# 查看帮助
./bin/janus --help
# 前台启动
./bin/janus
# 查看端口占用情况
netstat -ntlp | grep janus
```

### 后台运行（脚本/服务）

```bash
chmod +x deploy/janus/start.sh
chmod +x deploy/janus/stop.sh
chmod +x deploy/janus/status.sh

./deploy/janus/start.sh
./deploy/janus/stop.sh
./deploy/janus/status.sh
```

```bash
sudo cp deploy/janus/janus.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable janus
sudo systemctl start janus
sudo systemctl status janus
sudo journalctl -u janus -f
```

```bash
nohup /opt/janus/bin/janus > /opt/janus/logs/janus.log 2>&1 &
ps aux | grep janus
pkill -f janus
```
