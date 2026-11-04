<!--
 * @Author: jackning 270580156@qq.com
 * @Date: 2024-03-12 10:21:18
 * @LastEditors: jackning 270580156@qq.com
 * @LastEditTime: 2025-09-19 10:27:57
 * @Description: bytedesk.com https://github.com/Bytedesk/bytedesk
 *   Please be aware of the BSL license restrictions before installing Bytedesk IM – 
 *  selling, reselling, or hosting Bytedesk IM as a service is a breach of the terms and automatically terminates your rights under the license.
 *  Business Source License 1.1: https://github.com/Bytedesk/bytedesk/blob/main/LICENSE 
 *  contact: 270580156@qq.com 
 *  联系：270580156@qq.com
 * Copyright (c) 2024 by bytedesk.com, All Rights Reserved. 
-->
# docker

**语言 / Language:** [中文](README.zh.md) | [English](README.md) | [Demo](https://www.weiyuai.cn/reactdemo)

## 微语启动步骤

```bash
# https://www.weiyuai.cn/docs/zh-CN/docs/deploy/docker
# 克隆项目
git clone https://github.com/Bytedesk/bytedesk-docker-compose
# 进入docker目录
cd bytedesk-docker-compose/docker
# 配置环境变量，根据需要修改
cp .env.example .env
# 重要：密码/API Key/JWT 等敏感信息已统一迁移到 .env
# start.sh/stop.sh 会通过 --env-file 自动加载 deploy/docker/.env
# 如需调整本地 ASR 模型或设备，可在 .env 中修改 FUNASR_DEVICE、FUNASR_MODEL、FUNASR_VAD_MODEL。
# 默认启动全栈（mysql + artemis + redis + elasticsearch + bytedesk 应用）
./start.sh

# 访问 http://127.0.0.1:9003，默认账号 admin@email.com / admin
```

## 常用命令

```bash
# 默认全栈（等价 ./start.sh all）
./start.sh
./stop.sh              # 停止
./stop.sh down         # 删除容器（保留数据卷）

# 仅中间件（源码本地开发，不启动 bytedesk 应用镜像）
./start.sh middleware
./stop.sh middleware down

# 切换数据库 / 消息队列（四选一 db，二选一 mq）
./start.sh postgresql rabbitmq middleware
./start.sh oracle middleware
./start.sh kingbase all

# 呼叫中心（call = freeswitch + mrcp；仅支持 mysql/postgresql）
./start.sh call all
./start.sh call middleware
./stop.sh call middleware down

# WebRTC 音视频（webrtc = coturn + janus）
./start.sh call webrtc all
./start.sh call webrtc middleware minio searxng neo4j  # 任意组合
./stop.sh call webrtc middleware minio searxng neo4j down
## 使用 postgresql
./start.sh postgresql call webrtc middleware minio searxng neo4j  # 任意组合
./stop.sh postgresql call webrtc middleware minio searxng neo4j down
# 之后反复切库，几秒完成
./switch-db.sh mysql
./switch-db.sh postgresql

# 可选组件（任意组合）
./start.sh all minio searxng james
./start.sh middleware obs        # obs = prometheus + grafana + zipkin + otelcol
./stop.sh middleware obs
./start.sh middleware logstash kibana
./stop.sh middleware logstash kibana down

# 查看启动log
docker logs -f bytedesk -n 100
# 查看log目录
sudo ls -lh /var/lib/docker/volumes/bytedesk_log_data/_data/
```

## 关键字速查

| 类别 | 关键字（别名） | 说明 |
| --- | --- | --- |
| 数据库（四选一，默认 mysql） | `mysql` `postgresql`(pg) `oracle` `kingbase`(kingbase9) | 自动建库 |
| 消息队列（二选一，默认 artemis） | `artemis` `rabbitmq` | |
| 核心中间件 | `redis` `elasticsearch`(es) | 任何栈自动包含 |
| 呼叫中心 | `freeswitch` `mrcp`，组合 `call` | call 仅支持 mysql/postgresql |
| WebRTC | `coturn` `janus`，组合 `webrtc` | |
| 搜索/存储 | `searxng`(search) `minio` `rustfs` `neo4j` | rustfs = S3 兼容对象存储（Apache 2.0，MinIO 开源替代）；企业版功能 |
| 邮件服务器 | `james` | Apache James 私有邮件服务器：定位企业内部员工邮件沟通（协同）；通知/验证码等轻量邮件建议继续用第三方邮箱，详见 [James 文档](./readme/readme.james.md) |
| 日志 | `logstash` `kibana` | 依赖 elasticsearch |
| 可观测 | `prometheus` `grafana` `zipkin` `otelcol`(opentelemetry)，组合 `obs` | 追踪后端二选一：zipkin（Brave）或 otelcol（OTLP），详见 [可观测性文档](./readme/readme.observability.md) |
| 目标 | `middleware` / `all`(bytedesk)，默认 all | 展开见下表 |

### all / middleware 对应镜像

| 目标关键字 | 展开包含的镜像（compose 文件） | 说明 |
| --- | --- | --- |
| `all`（默认） | `middleware` 全部 + **bytedesk 应用**（compose-redis + compose-elasticsearch + compose-<db> + compose-<mq> + compose-bytedesk） | 全栈：中间件 + 应用镜像 |
| `middleware` | compose-redis + compose-elasticsearch + compose-<所选db> + compose-<所选mq> | 仅中间件，**不含** bytedesk 应用，用于源码本地开发 |

说明：redis 与 elasticsearch 为核心中间件，任何栈都自动包含；`<所选db>` 默认 mysql（可选 postgresql/oracle/kingbase），`<所选mq>` 默认 artemis（可选 rabbitmq）；其余组件（freeswitch、obs、minio 等）按关键字叠加。

停止动作：`./stop.sh [stop|down] [关键字...]`，动作词可在任意位置，关键字与启动时一致；`down` 删除容器保留数据卷。

## 文件清单

所有 compose 文件及各组件配套配置（`searxng/`、`grafana/`、`logstash/`、`prometheus.yml`、`ik-plugin-cache/`、`james/`）均位于 [`compose/`](./compose/) 目录：

| compose 文件（位于 compose/） | 镜像 | 默认端口 | 详细说明 |
| --- | --- | --- | --- |
| compose/compose-redis.yaml | redis | 16379 | — |
| compose/compose-elasticsearch.yaml | elasticsearch:9.4.2 | 19200/19300 | — |
| compose/compose-mysql.yaml | mysql | 13306 | [数据库](./readme/readme.database.md) |
| compose/compose-postgresql.yaml | postgres:17 | 15432 | [数据库](./readme/readme.database.md) |
| compose/compose-oracle.yaml | gvenzl/oracle-free:23-slim | 11521 | [数据库](./readme/readme.database.md) |
| compose/compose-kingbase.yaml | kingbase V9 | 54321 | [数据库](./readme/readme.database.md) |
| compose/compose-artemis.yaml | activemq-artemis:2.40.0 | 16161/18161 | [消息队列](./readme/readme.mq.md) |
| compose/compose-rabbitmq.yaml | rabbitmq:4.2.3 | 5673/15673 | [消息队列](./readme/readme.mq.md) |
| compose/compose-bytedesk.yaml | bytedesk 应用 | 9003/9885 | — |
| compose/compose-freeswitch.yaml | bytedesk/freeswitch | 15060/18021 | [FreeSWITCH](./readme/readme.freeswitch.md) |
| compose/compose-mrcp.yaml | bytedesk/mrcp | 11544 | [呼叫中心](./readme/readme.call.md) |
| compose/compose-coturn.yaml | coturn:4.6.3 | 13478 | [WebRTC](./readme/readme.webrtc.md) |
| compose/compose-janus.yaml | bytedesk/janus | 18089/18188 | [WebRTC](./readme/readme.webrtc.md) |
| compose/compose-searxng.yaml | searxng | 18888 | [SearXNG](./readme/readme.searxng.md) |
| compose/compose-neo4j.yaml | neo4j:5.26.30 | 17474/17687 | [Neo4j 知识图谱](./readme/readme.neo4j.md) |
| compose/compose-logstash.yaml | logstash:9.4.2 | 19600 | [Logstash](./readme/readme.logstash.md) |
| compose/compose-kibana.yaml | kibana:9.4.2 | 15601 | [Kibana](./readme/readme.kibana.md) |
| compose/compose-minio.yaml | minio | 19000/19001 | [MinIO](./readme/readme.minio.md) |
| compose/compose-rustfs.yaml | rustfs | 19000/19001 | [RustFS](./readme/readme.rustfs.md) |
| compose/compose-prometheus.yaml | prometheus | 19090 | [可观测性](./readme/readme.observability.md) |
| compose/compose-grafana.yaml | grafana | 13000 | [可观测性](./readme/readme.observability.md) |
| compose/compose-zipkin.yaml | zipkin | 19411 | [可观测性](./readme/readme.observability.md) |
| compose/compose-otelcol.yaml | otel/opentelemetry-collector-contrib | 14317/14318 | [可观测性](./readme/readme.observability.md)——OTLP 接收，trace 转发 Zipkin |
| compose/compose-james.yaml | apache/james:jpa-latest | 10025/10465/10587/10143/10993/10110/10995/18000 | [James 邮件服务器](./readme/readme.james.md) |

其他文件：`start.sh`/`stop.sh`（组合启停）、`watchdog.sh`（应用看门狗，详见 [watchdog 使用说明](./readme/readme.watchdog.md)）、`.env`（敏感配置）、`one/`（all-in-one 单文件部署）。
