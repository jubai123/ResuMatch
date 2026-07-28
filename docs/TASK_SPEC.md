# TASK_SPEC.md — MVP 原子级任务规格

> **用途**：将 PRD 拆解为 AI 可独立完成的任务。每个任务包含输入条件、输出标准、验收清单、关联文件。
> **排期基准**：3-5 人团队，2 周交付 MVP Demo。工时单位：人·天（1 人·天 = 8h）。

---

## 里程碑 M0：项目基础设施（Day 1-2）

### T0.1 项目脚手架搭建
- **输入**：[CLAUDE.md](../CLAUDE.md)、[ARCHITECTURE.md](ARCHITECTURE.md)
- **输出**：
  - `app-backend/`：FastAPI 项目骨架（main.py, config.py, 路由注册）
  - `app-frontend/`：uni-app 项目骨架（Vite + Vue 3 + Pinia）
  - `docker-compose.yml`：PostgreSQL + Redis + MinIO + API
  - `.env.example`：环境变量模板
- **验收**：`docker-compose up` 后 API 返回 200；uni-app H5 `npm run dev` 可访问
- **关联文件**：项目根目录全部配置
- **工时**：1 人·天

### T0.2 数据库迁移与种子数据
- **输入**：[SCHEMA.md](SCHEMA.md)
- **输出**：
  - Alembic 初始迁移脚本（全部 11 张表）
  - 种子脚本：50 条模拟岗位（覆盖 5 大平台、5 个城市、3 个行业）
  - 种子脚本：3 份模拟简历（CS硕、EE硕、商科硕）
- **验收**：`alembic upgrade head` 无报错；种子脚本运行后数据库有数据
- **关联文件**：`app-backend/migrations/`、`app-backend/scripts/seed.py`
- **工时**：1 人·天

### T0.3 通用中间件与工具函数
- **输入**：[CLAUDE.md](../CLAUDE.md)、[ARCHITECTURE.md](ARCHITECTURE.md)
- **输出**：
  - JWT 签发/验证（`utils/jwt.py`）
  - 全局异常处理（`app/exceptions.py`）
  - structlog 日志配置（JSON 格式）
  - 分页工具（`utils/pagination.py`）
  - CORS 中间件（允许 H5 跨域）
- **验收**：单测覆盖 JWT 签发→验证→过期；异常处理返回统一 JSON 格式
- **关联文件**：`app/utils/`、`app/exceptions.py`、`app/main.py`
- **工时**：0.5 人·天

---

## 里程碑 M1：用户 + 简历闭环（Day 2-4）

### T1.1 认证模块（auth）
- **输入**：[auth.md](modules/auth.md)、[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `POST /api/auth/sms` — 发送验证码（mock SMS）
  - `POST /api/auth/login` — 登录→JWT
  - `POST /api/auth/refresh` — 刷新 Token
  - `get_current_user` 依赖注入
- **验收**：
  - [ ] 登录成功返回 accessToken + refreshToken
  - [ ] 未登录访问受保护端点返回 401
  - [ ] Token 过期后 refresh 成功
  - [ ] 验证码 60s 内不可重发
- **关联文件**：`app/api/auth.py`、`app/services/auth_service.py`、`app/utils/jwt.py`
- **工时**：1 人·天

### T1.2 用户模块（user）
- **输入**：[SCHEMA.md](SCHEMA.md)、[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `GET/PATCH /api/user/profile`
  - `POST /api/user/verify-student`
  - 学生认证 OCR（mock 学信网）
- **验收**：
  - [ ] 获取/更新个人信息
  - [ ] 上传认证图片→返回审核状态
- **关联文件**：`app/api/user.py`、`app/services/user_service.py`、`app/models/user.py`
- **工时**：0.5 人·天

### T1.3 简历模块（resume）
- **输入**：[resume.md](modules/resume.md)、[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `POST /api/resume/upload` — 文件上传 + 解析（mock Claude 提取）
  - `POST /api/resume/online` — 在线填写
  - `POST /api/resume/{id}/rewrite` — AI 改写（异步）
  - `GET /api/resume/{id}/rewrite/{taskId}` — 轮询结果
  - `POST /api/resume/{id}/rewrite/accept` — 接受/拒绝
  - `GET /api/resume` / `GET /api/resume/history`
- **验收**：
  - [ ] PDF 上传→返回结构化预览字段
  - [ ] AI 改写 30s 内返回
  - [ ] 逐条接受/拒绝/微调
  - [ ] 接受全部后生成新版本
- **关联文件**：`app/api/resume.py`、`app/services/resume_service.py`、`app/models/resume.py`
- **工时**：1.5 人·天

### T1.4 偏好设置（preference）
- **输入**：[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `GET/PUT /api/preference`
  - 偏好校验（最多 3 城市、5 关键词）
- **验收**：
  - [ ] 获取/全量更新偏好
  - [ ] 非法输入返回 422
- **关联文件**：`app/api/preference.py`、`app/services/preference_service.py`
- **工时**：0.5 人·天

---

## 里程碑 M2：匹配 + 推送闭环（Day 4-7）

### T2.1 岗位模块（job + job_crawler）
- **输入**：[job.md](modules/job.md)、[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `GET /api/job/{id}` — 岗位详情
  - `GET /api/job/search` — 搜索筛选
  - Celery 任务：每日拉取第三方数据（mock 聚合 API）
  - JD 结构化解析（mock Claude）
- **验收**：
  - [ ] 搜索返回分页结果
  - [ ] 详情含匹配分（如有）
  - [ ] 定时任务可手动触发
  - [ ] 种子数据 50 条可搜索
- **关联文件**：`app/api/job.py`、`app/services/job_service.py`、`app/tasks/job_crawler.py`
- **工时**：1.5 人·天

### T2.2 匹配引擎（match_engine）★ 核心
- **输入**：[match_engine.md](modules/match_engine.md)、[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `services/match_service.py:calculate_match_score()` — 6 维度加权评分
  - `POST /api/match/initial` — 首次匹配 Top 5
  - Celery 任务：`generate_weekly_batch()` — 周推送生成
  - `GET /api/match/weekly` — 分层分页获取本周推送
  - `POST /api/match/{id}/feedback` — 用户反馈
  - 反馈权重调优（简化版）
- **验收**：
  - [ ] 匹配分在 0-100 范围
  - [ ] 技能匹配的分数 > 无技能匹配
  - [ ] 黑名单公司 score=0
  - [ ] 同城市岗位分数 > 异地
  - [ ] 首次匹配 5s 内返回
  - [ ] 周推送每人 15-25 条
  - [ ] 用户反馈后状态更新
- **关联文件**：`app/api/match.py`、`app/services/match_service.py`、`app/tasks/match.py`
- **工时**：2 人·天（最核心模块）

### T2.3 推送通知（push）
- **输入**：[push.md](modules/push.md)、[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `POST /api/notification/register-device`
  - Celery 任务：周推送通知发送（mock FCM/APNs）
- **验收**：
  - [ ] 设备注册成功
  - [ ] 周推送触发通知（控制台日志可验证）
  - [ ] Token 失效自动跳过
- **关联文件**：`app/api/notification.py`、`app/tasks/notification.py`
- **工时**：0.5 人·天

---

## 里程碑 M3：投递 + 支付闭环（Day 7-10）

### T3.1 招呼语文案（cover_letter）
- **输入**：[cover_letter.md](modules/cover_letter.md)、[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `POST /api/cover-letter/generate` — 生成 3 版招呼语（mock Claude）
- **验收**：
  - [ ] 返回 3 种风格文案
  - [ ] AI 不可用时走降级模板
- **关联文件**：`app/api/cover_letter.py`、`app/services/cover_letter_service.py`
- **工时**：0.5 人·天

### T3.2 投递管理（application）
- **输入**：[application.md](modules/application.md)、[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `GET/POST /api/application` — 投递列表 + 创建
  - `PATCH /api/application/{id}` — 状态更新
  - `GET /api/application/funnel` — 漏斗统计
- **验收**：
  - [ ] 状态机流转正确
  - [ ] 不合法转移返回 422
  - [ ] 漏斗数据计算正确
  - [ ] 14 天未动返回 remindAt
- **关联文件**：`app/api/application.py`、`app/services/application_service.py`
- **工时**：1 人·天

### T3.3 支付订阅（order）
- **输入**：[order.md](modules/order.md)、[API_SCHEMA.yaml](API_SCHEMA.yaml)
- **输出**：
  - `POST /api/order` — 创建订单（mock 支付网关）
  - `GET /api/order/{id}/status` — 查询状态
  - `POST /api/order/{id}/pay-callback` — 支付回调（mock）
  - `GET /api/subscription` — 订阅状态
- **验收**：
  - [ ] 下单→回调→订阅激活流程
  - [ ] 过期订阅自动降级
  - [ ] 学生折扣价计算正确
- **关联文件**：`app/api/order.py`、`app/services/order_service.py`
- **工时**：1 人·天

---

## 里程碑 M4：前端 + 联调（Day 10-13）

### T4.1 前端壳工程 + 设计系统
- **输入**：功能设计文档（线框图）、[CLAUDE.md](../CLAUDE.md)
- **输出**：
  - uni-app 项目初始化（Vue 3 + Pinia + TypeScript）
  - 设计令牌（颜色/字号/间距 CSS 变量）
  - 通用组件库（Button, Card, Tag, Input, Modal, Toast, ProgressBar, Empty, Loading, Skeleton）
  - 路由配置（4 Tab + 子页面）
  - HTTP 封装（JWT 自动附加 + 401 跳转登录）
- **验收**：H5 可访问，4 Tab 切换正常，设计令牌与线框图一致
- **关联文件**：`app-frontend/src/` 全部初始化文件
- **工时**：1 人·天

### T4.2 注册登录 + 引导页
- **输入**：线框图 7.2-7.4
- **输出**：
  - 启动页（3s 跳过）
  - 3 屏引导页
  - 手机号登录页
  - 学生认证页（可选跳过）
- **验收**：登录→getCurrentUser→跳转首页
- **关联文件**：`app-frontend/src/pages/splash/`, `guide/`, `login/`, `verify/`
- **工时**：1 人·天

### T4.3 简历上传 + AI 改写
- **输入**：线框图 7.5-7.6
- **输出**：
  - 简历上传四选一（PDF/在线/拍照/微信）
  - PDF 上传→解析预览→确认
  - AI 改写对比页（原文/AI版/理由/接受/拒绝/微调）
  - 偏好设置 3 步引导
  - 首次匹配结果展示
- **验收**：上传→改写→偏好→5 条匹配 全链路
- **关联文件**：`app-frontend/src/pages/resume/`, `preference/`, `match/initial/`
- **工时**：2 人·天

### T4.4 首页推送 + 岗位详情
- **输入**：线框图 7.8
- **输出**：
  - 首页（Top 3 卡片 + 进度条 + 全部抽屉）
  - 岗位列表分层加载（90+/80-89/70-79/70-）
  - 岗位详情页（匹配点高亮 + 优势/差距 + AI 建议）
  - 反馈操作（想投/不合适/稍后看）
- **验收**：首页展示匹配卡片、分层加载、反馈操作
- **关联文件**：`app-frontend/src/pages/home/`, `job/detail/`
- **工时**：2 人·天

### T4.5 投递台 + 文案 + 漏斗
- **输入**：线框图 7.10-7.11
- **输出**：
  - 招呼语生成页（3 选 1 + 微调 + 复制）
  - 投递台（状态分类 + 时间线）
  - 漏斗统计页
- **验收**：招呼语生成→复制→标记已投递→漏斗更新
- **关联文件**：`app-frontend/src/pages/application/`, `cover-letter/`
- **工时**：1.5 人·天

### T4.6 岗位库搜索 + 我的 + 支付
- **输入**：线框图 7.12-7.14
- **输出**：
  - 岗位库搜索页（关键词 + 多维筛选 + 分页）
  - 我的页面（简历预览/偏好/订阅/推送设置）
  - 支付页（3 套餐 + 支付方式 + 支付状态）
- **验收**：搜索→筛选→查看；套餐选择→下单→支付→激活
- **关联文件**：`app-frontend/src/pages/search/`, `profile/`, `subscribe/`
- **工时**：1.5 人·天

---

## 里程碑 M5：联调 + Demo 准备（Day 13-14）

### T5.1 前后端联调 + Bug 修复
- **输入**：所有已完成的模块
- **输出**：全链路通过（注册→简历→匹配→投递→反馈）
- **验收**：[验收标准清单](#验收标准清单) 全部通过
- **工时**：1 人·天

### T5.2 AI 服务真实接入
- **输入**：[ai_service.md](modules/ai_service.md)
- **输出**：
  - 替换 mock Claude 为真实 API 调用
  - Prompt 调优（简历改写/匹配解释/招呼语）
  - 降级策略验证
- **验收**：AI 改写返回符合预期的质量
- **工时**：0.5 人·天

### T5.3 Demo 准备
- **输入**：全部功能
- **输出**：
  - 3 份种子简历 + 50 条种子岗位
  - 演示脚本（注册→匹配→投递 完整链路）
  - H5 部署（可公网访问）
- **工时**：0.5 人·天

---

## 验收标准清单

### 功能完整性（必须 100%）
- [ ] 手机号注册/登录 100% 通过
- [ ] ≥2 种简历上传方式可用
- [ ] AI 改写 30s 内返回
- [ ] 注册后 30s 内出首批 5 条匹配
- [ ] 每周推送 15-25 条，匹配分排序
- [ ] 岗位详情含匹配点+差距分析
- [ ] 投递状态机完整流转
- [ ] AI 招呼语 3 版本可切换
- [ ] 投递漏斗数据正确
- [ ] 订阅支付流程完整（mock）

### 性能（Demo 级别）
- [ ] 首页加载 ≤ 1.5s
- [ ] 搜索响应 ≤ 1s
- [ ] AI 改写 ≤ 30s（含降级）
- [ ] 推送生成 ≤ 5min（100 用户）

### 代码质量
- [ ] 后端整体测试覆盖率 ≥ 75%
- [ ] match_service 覆盖率 ≥ 90%
- [ ] 无 `print()` / `console.log()` 残留
- [ ] 无硬编码密钥/密码
- [ ] ESLint / Ruff 零告警

---

## 工时汇总

| 里程碑 | 内容 | 工时（人·天） |
|---|---|---|
| M0 | 基础设施 | 2.5 |
| M1 | 用户+简历闭环 | 3.5 |
| M2 | 匹配+推送闭环 | 4.0 |
| M3 | 投递+支付闭环 | 2.5 |
| M4 | 前端+联调 | 9.0 |
| M5 | 联调+Demo | 2.0 |
| **合计** | | **23.5** |

3-5 人并行，预计 6-8 个工作日完成（按瓶颈路径：M2 匹配引擎最耗时，可提前启动）。
