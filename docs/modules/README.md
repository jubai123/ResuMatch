# Module READMEs — 模块索引

每个核心模块一个 README，说明职责边界、入口函数、异常策略。AI 修改代码时**必须**先读对应模块的 README。

| 模块 | 文件 | 一句话职责 | 关键依赖 |
|---|---|---|---|
| 认证 | [auth.md](auth.md) | JWT 签发/验证、短信验证码 | — |
| 用户 | [user.md](user.md) | 账户管理、学生认证 | auth |
| 简历 | [resume.md](resume.md) | 上传解析、AI改写、版本管理 | auth, user, ai_service |
| 偏好 | [preference.md](preference.md) | 求职偏好增删改查 | auth, user |
| 岗位 | [job.md](job.md) | 岗位库搜索、详情、数据采集 | job_crawler |
| 匹配引擎 | [match_engine.md](match_engine.md) | 匹配计算、周推送生成、反馈回收 | resume, preference, job |
| 文案 | [cover_letter.md](cover_letter.md) | AI招呼语生成 | match_engine, ai_service |
| 投递 | [application.md](application.md) | 投递CRUD、状态机、漏斗统计 | match_engine, cover_letter |
| 支付 | [order.md](order.md) | 订阅下单、支付回调、续费管理 | auth, user |
| AI服务 | [ai_service.md](ai_service.md) | Claude API封装、prompt管理 | — |
| 推送 | [push.md](push.md) | 推送通知调度、设备注册 | push_batch, user |

---

## 目录映射

```
app-backend/
├── app/
│   ├── main.py              # FastAPI 应用入口
│   ├── config.py            # 配置管理（pydantic-settings）
│   ├── api/                 # 路由层（薄层，参数校验 + 调用 service）
│   │   ├── auth.py
│   │   ├── user.py
│   │   ├── resume.py
│   │   ├── preference.py
│   │   ├── job.py
│   │   ├── match.py
│   │   ├── cover_letter.py
│   │   ├── application.py
│   │   └── order.py
│   ├── services/            # 业务逻辑层
│   │   ├── auth_service.py
│   │   ├── user_service.py
│   │   ├── resume_service.py
│   │   ├── preference_service.py
│   │   ├── job_service.py
│   │   ├── match_service.py
│   │   ├── cover_letter_service.py
│   │   ├── application_service.py
│   │   ├── order_service.py
│   │   └── ai_service.py
│   ├── models/              # SQLAlchemy ORM 模型
│   │   ├── user.py
│   │   ├── resume.py
│   │   ├── job.py
│   │   └── ...
│   ├── schemas/             # Pydantic 请求/响应模型
│   │   ├── auth.py
│   │   ├── resume.py
│   │   └── ...
│   ├── tasks/               # Celery 异步任务
│   │   ├── job_crawler.py
│   │   ├── match.py
│   │   ├── ai.py
│   │   └── notification.py
│   └── utils/               # 工具函数
│       ├── jwt.py
│       ├── phone.py
│       └── pagination.py
```
