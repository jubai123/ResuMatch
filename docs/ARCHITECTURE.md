# ARCHITECTURE.md — ResuMatch 系统架构概览

> **用途**：跨模块修改前必读。明确每个模块的职责边界、上下游依赖、数据流向。

---

## 1. 系统分层架构

```
┌──────────────────────────────────────────────────────────┐
│                    客户端层 (Client Tier)                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │
│  │ iOS App  │  │Android App│  │  H5 (Mobile Web)     │   │
│  │ (uni-app)│  │ (uni-app) │  │  (uni-app H5 build)  │   │
│  └────┬─────┘  └────┬─────┘  └──────────┬───────────┘   │
│       └──────────────┼──────────────────┘               │
│                      │ HTTPS + JWT                       │
└──────────────────────┼──────────────────────────────────┘
                       │
┌──────────────────────┼──────────────────────────────────┐
│                 API 网关层 (Gateway Tier)                  │
│  ┌───────────────────┴──────────────────────────────┐   │
│  │  Nginx (rate-limit, TLS termination, static)      │   │
│  └───────────────────┬──────────────────────────────┘   │
│                      │                                   │
│  ┌───────────────────┴──────────────────────────────┐   │
│  │  FastAPI 应用服务器 (Uvicorn, 多 worker)          │   │
│  │  ├─ /api/auth/*       认证模块                    │   │
│  │  ├─ /api/resume/*     简历模块                    │   │
│  │  ├─ /api/match/*      匹配模块                    │   │
│  │  ├─ /api/job/*        岗位模块                    │   │
│  │  ├─ /api/application/* 投递模块                   │   │
│  │  ├─ /api/preference/* 偏好模块                    │   │
│  │  ├─ /api/order/*      支付模块                    │   │
│  │  └─ /api/ai/*         AI 服务代理                  │   │
│  └───────────────────┬──────────────────────────────┘   │
└──────────────────────┼──────────────────────────────────┘
                       │
┌──────────────────────┼──────────────────────────────────┐
│                 服务层 (Service Tier)                     │
│  ┌───────────────────┴──────────────────────────────┐   │
│  │              Celery Worker 集群                     │   │
│  │  ├─ job_crawler      岗位数据采集（每日定时）       │   │
│  │  ├─ match_engine     匹配计算 + 周推送生成          │   │
│  │  ├─ ai_service       AI 调用（简历改写/文案生成）   │   │
│  │  └─ notification     推送通知发送                   │   │
│  └───────────────────┬──────────────────────────────┘   │
└──────────────────────┼──────────────────────────────────┘
                       │
┌──────────────────────┼──────────────────────────────────┐
│                 数据层 (Data Tier)                        │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐    │
│  │PostgreSQL│  │  Redis   │  │  MinIO / S3        │    │
│  │ (主数据) │  │(缓存/队列)│  │  (简历文件/头像)   │    │
│  └──────────┘  └──────────┘  └────────────────────┘    │
└──────────────────────────────────────────────────────────┘
                       │
┌──────────────────────┼──────────────────────────────────┐
│                 外部依赖 (External)                       │
│  ├─ Claude API (简历改写/匹配解释/文案生成)              │
│  ├─ 第三方招聘数据聚合 API (薪乐宝/职涯宝类)             │
│  ├─ 微信支付 / 支付宝 回调                              │
│  ├─ 学信网 OCR 认证接口                                  │
│  └─ APNs / FCM 推送服务                                 │
└──────────────────────────────────────────────────────────┘
```

---

## 2. 模块划分与依赖

```
                    ┌─────────────┐
                    │   auth      │  认证与授权（JWT 签发/验证）
                    └──────┬──────┘
                           │ 被所有模块依赖
        ┌──────────────────┼──────────────────┐
        │                  │                  │
  ┌─────┴─────┐   ┌───────┴───────┐   ┌──────┴──────┐
  │  resume   │   │  preference   │   │    user     │
  │ 简历管理   │   │  偏好设置     │   │  用户账户   │
  └─────┬─────┘   └───────┬───────┘   └──────┬──────┘
        │                 │                  │
        └────────┬────────┘                  │
                 │                           │
          ┌──────┴──────┐                    │
          │ match_engine│  匹配计算引擎       │
          └──────┬──────┘                    │
                 │                           │
        ┌────────┼────────┐                  │
        │        │        │                  │
  ┌─────┴──┐ ┌───┴───┐ ┌─┴──────┐   ┌──────┴──────┐
  │  job   │ │ cover │ │  push  │   │ application │
  │ 岗位库 │ │ 文案  │ │ 推送   │   │  投递管理   │
  └────────┘ └───────┘ └────────┘   └──────┬──────┘
                                           │
                                     ┌─────┴──────┐
                                     │   order    │
                                     │  支付订阅  │
                                     └────────────┘
```

### 依赖关系矩阵

| 模块 | 依赖 | 被依赖 |
|---|---|---|
| `auth` | — | 所有模块 |
| `user` | `auth` | `resume`, `preference`, `order` |
| `resume` | `auth`, `user` | `match_engine`, `ai_service` |
| `preference` | `auth`, `user` | `match_engine` |
| `job_crawler` | — | `job`, `match_engine` |
| `job` | `job_crawler` | `match_engine`, `application` |
| `match_engine` | `resume`, `preference`, `job` | `push`, `application` |
| `push` | `match_engine`, `user` | — |
| `cover_letter` | `match_engine`, `ai_service` | `application` |
| `application` | `match_engine`, `cover_letter`, `user` | — |
| `ai_service` | — | `resume`, `match_engine`, `cover_letter` |
| `order` | `auth`, `user` | — |
| `notification` | `push`, `user` | — |

---

## 3. 核心数据流

### 3.1 用户注册 → 首次匹配（关键路径）

```
[App]──POST /api/auth/sms────────────▶ [auth] ──短信──▶ 第三方SMS
[App]──POST /api/auth/login──────────▶ [auth] ──验证──▶ PostgreSQL ──返回JWT──▶ [App]
[App]──POST /api/resume/upload───────▶ [resume] ──存储文件──▶ MinIO
                                     [resume] ──触发AI解析──▶ Celery ──▶ Claude API
[App]──PUT /api/preference───────────▶ [preference] ──▶ PostgreSQL
[App]──POST /api/match/initial───────▶ [match_engine] ──读取简历+偏好──▶ PostgreSQL
                                     [match_engine] ──向量匹配──▶ 岗位库
                                     [match_engine] ──生成Top 5──▶ [App]
```

### 3.2 每周一推送流程（定时任务）

```
Celery Beat (每周一 08:00) ──触发──▶ match_engine.generate_weekly_batch()
                                     │
                                     ├── 1. 读取所有活跃用户的 简历 + 偏好
                                     ├── 2. 读取本周增量岗位（过去7天）
                                     ├── 3. 逐用户计算匹配分（6 维度加权）
                                     ├── 4. 每人取 Top 15-25，生成 PushBatch
                                     ├── 5. 写入 match_results 表
                                     └── 6. 触发推送通知
                                           ├── FCM (Android)
                                           └── APNs (iOS)
```

### 3.3 投递反馈闭环

```
[App]──POST /api/match/{id}/feedback────▶ [match_engine]
       { status: "interested" }           │
                                          ├── 更新 match_results.user_status
                                          ├── 若 interested: 触发文案生成
                                          │     └──▶ Claude API ──▶ 返回 3 版招呼语
                                          └── 反馈数据写入 training_log
                                               └── 每周聚合更新用户画像权重
```

---

## 4. AI 服务集成

```
app/services/ai_service.py
├── rewrite_resume(section: str, content: str) → AiRewriteResult
├── explain_match(resume: Resume, job: Job) → str
├── generate_cover_letter(job: Job, resume: Resume, style: str) → list[str]
├── parse_jd(jd_text: str) → JdStructured
└── extract_resume_fields(raw_text: str) → ResumeStructure

调用方式：Celery 异步任务（非阻塞）
模型：Claude Sonnet 4 (平衡质量与成本)
降级：单次调用超时 30s，失败重试 1 次，仍失败则返回模板化兜底结果
```

---

## 5. 技术选型理由

| 决策 | 理由 |
|---|---|
| **uni-app** 而非 Flutter/RN | 方案文档指定；H5 + App + 小程序三端一次开发 |
| **FastAPI** 而非 NestJS/Spring | AI 服务（Python 生态）；async 原生支持；Pydantic 类型安全；原型开发快 |
| **PostgreSQL** 而非 MySQL | JSONB 支持（简历结构化字段）；全文检索；窗口函数（匹配排序） |
| **Celery** 而非 BullMQ | Python 技术栈统一；成熟的定时任务（Celery Beat 做周推送） |
| **Redis** 而非 RabbitMQ | 已用于缓存 + 队列，减少运维组件 |
| **Claude API** 而非本地模型 | 2 周 MVP 无时间微调；Claude 文本生成质量高；按量付费成本可控 |

---

## 6. 变更影响检查清单

修改某个模块前，检查以下影响：

| 修改模块 | 需同步检查 |
|---|---|
| `auth` 的 JWT payload | 所有模块的 `get_current_user` 依赖 |
| `resume` 的 structure 字段 | `match_engine` 的评分算法输入 |
| `preference` 的匹配维度 | `match_engine` 权重计算 + `push` 推送筛选 |
| `job` 的数据源/字段 | `match_engine` + `job_crawler` 采集逻辑 |
| `match_engine` 评分公式 | `push` 推送质量 + `application` 转化漏斗 |
| `ai_service` 的 prompt | `resume` 改写输出 + `cover_letter` 文案质量 |
| API 响应字段 | 前端 uni-app 对应页面组件 |
| 数据库表结构 | 所有 ORM 模型 + 相关 API 的 Pydantic schema |

---

## 7. 部署架构（Docker Compose）

```
services:
  nginx:       反向代理 + 静态资源
  api:         FastAPI (2 replica)
  celery-worker: 异步任务 (2 replica)
  celery-beat: 定时任务调度 (1 replica)
  postgres:    PostgreSQL 15
  redis:       Redis 7
  minio:       对象存储 (dev)
```

开发环境：`docker-compose up` 一键启动全部服务。
生产环境：api/celery 独立扩缩容，postgres/redis 用托管服务。
