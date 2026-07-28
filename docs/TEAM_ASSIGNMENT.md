# TEAM_ASSIGNMENT.md — 三人协作开发任务分配

> **项目**：ResuMatch 应届生个人简历市场匹配助手 MVP
> **团队规模**：3 人
> **周期**：2 周（14 天，实际开发 10 个工作日 + 2 天缓冲）
> **总工时**：23.5 人·天（并行后约 8 个工作日）
> **仓库**：https://github.com/jubai123/ResuMatch

---

## 一、分工总览

```
           Person A                 Person B                 Person C
        "后端基础设施             "后端核心业务               "前端全栈
          + 用户简历线"            + 匹配投递线"              + 联调集成"

Day1-2   ██ M0 脚手架+DB+中间件   ██ 读文档+搭环境           ██ uni-app壳工程+设计系统
Day3-4   ██ M1 认证+用户+简历     ██ 岗位模块+爬虫(mock)     ██ 登录引导+简历上传页
Day5-6   ██ 偏好设置+AI服务       ██ 匹配引擎(核心)          ██ 首页推送+岗位详情页
Day7-8   ██ 接口联调支持A         ██ 推送+文案+投递+支付     ██ 投递台+招呼语+漏斗页
Day9-10  ██ Code Review+补单测   ██ Code Review+补单测      ██ 支付页+搜索+我的+联调
Day11-12 ██ 集成联调+Bug修复      ██ 集成联调+Bug修复        ██ 集成联调+Bug修复
Day13-14 ██ AI真实接入+Demo准备   ██ AI真实接入+Demo准备     ██ AI真实接入+Demo准备
```

**关键**：Person A 必须在 Day3 结束前完成认证+用户模块，这是 B 和 C 的前置依赖。

---

## 二、Person A：后端基础设施 + 用户简历线

### 负责范围

| 里程碑 | 任务 | 产出文件 |
|---|---|---|
| M0 | T0.1 项目脚手架 | `app-backend/`, `docker-compose.yml`, `.env.example` |
| M0 | T0.2 数据库迁移+种子数据 | `app-backend/migrations/`, `scripts/seed.py` |
| M0 | T0.3 中间件+工具函数 | `app/utils/jwt.py`, `app/exceptions.py`, pagination, CORS |
| M1 | T1.1 认证模块 | `app/api/auth.py`, `app/services/auth_service.py` |
| M1 | T1.2 用户模块 | `app/api/user.py`, `app/services/user_service.py` |
| M1 | T1.3 简历模块 | `app/api/resume.py`, `app/services/resume_service.py` |
| M1 | T1.4 偏好设置 | `app/api/preference.py`, `app/services/preference_service.py` |
| M5 | T5.2 AI 服务真实接入 | `app/services/ai_service.py`, `app/prompts/*.txt` |

### 依赖关系

- **被 Person B 依赖**：auth（JWT 验证中间件）、resume（匹配引擎输入）、preference（匹配参数）
- **被 Person C 依赖**：auth API（登录注册页）、resume API（上传/改写页）、preference API（偏好设置页）
- **阻塞点**：T1.1 认证模块必须在 Day3 完成，Person B 和 C 的所有接口调用都依赖 JWT 认证

### 完整工作流程

#### 第 1 步：Fork 仓库并创建分支（Day 1 上午，30 分钟）

```bash
# 1. 克隆仓库
git clone https://github.com/jubai123/ResuMatch.git
cd ResuMatch

# 2. 切换到 main 分支（远程默认分支）
git checkout main

# 3. 创建个人开发分支
git checkout -b feat/infra-auth-resume

# 4. 确认分支状态
git branch
# 应显示：* feat/infra-auth-resume
#          main
#          master
```

#### 第 2 步：搭建项目脚手架（Day 1，约 4h）

```bash
# 在项目根目录下创建后端工程
mkdir -p app-backend/app/{api,services,models,schemas,tasks,utils,prompts}
mkdir -p app-backend/tests/factories
mkdir -p app-backend/migrations/versions
mkdir -p app-backend/scripts
```

**需要创建的文件**（按顺序）：

1. `.env.example` — 环境变量模板
2. `docker-compose.yml` — PostgreSQL 15 + Redis 7 + MinIO + API 服务
3. `app-backend/requirements.txt` — fastapi, uvicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic, redis, celery, structlog, python-jose, httpx, pytest 等
4. `app-backend/app/main.py` — FastAPI 应用入口，注册路由，配置 CORS
5. `app-backend/app/config.py` — pydantic-settings 配置管理

**验收**：`docker-compose up` 后 `curl http://localhost:8000/api/health` 返回 200

**提交**：
```bash
git add .
git commit -m "feat: 项目脚手架搭建 — FastAPI入口 + Docker Compose + 配置管理"
```

#### 第 3 步：数据库迁移（Day 1-2，约 3h）

1. 参考 `docs/SCHEMA.md` 的 11 张表定义
2. 创建 SQLAlchemy ORM 模型（`app/models/*.py`）
3. 初始化 Alembic 并生成迁移脚本
4. 编写种子脚本 `scripts/seed.py`：50 条模拟岗位 + 3 份模拟简历

```bash
# 初始化 Alembic
cd app-backend
alembic init migrations

# 生成初始迁移
alembic revision --autogenerate -m "init_all_tables"

# 执行迁移
alembic upgrade head

# 运行种子数据
python scripts/seed.py
```

**提交**：
```bash
git add .
git commit -m "feat: 数据库迁移 — 11张表 + Alembic初始化 + 种子数据50岗位3简历"
```

#### 第 4 步：中间件+工具函数（Day 2，约 2h）

1. `app/utils/jwt.py` — `create_access_token()` / `create_refresh_token()` / `decode_token()`
2. `app/exceptions.py` — 全局异常处理器，统一 JSON 错误格式
3. `app/utils/pagination.py` — 分页工具
4. `app/main.py` 中注册 structlog 日志 + CORS + exception_handler

**验收**：单测覆盖 JWT 签发→验证→过期

```bash
# 运行测试
cd app-backend && pytest tests/test_jwt.py -v
```

**提交**：
```bash
git add .
git commit -m "feat: 通用中间件 — JWT签发验证 + 全局异常处理 + 分页 + structlog"
```

#### 第 5 步：认证模块（Day 2-3，约 4h）⚠️ 阻塞点

1. `app/api/auth.py` — 3 个端点（sms/login/refresh）
2. `app/services/auth_service.py` — 验证码生成/验证 + 用户查找/创建
3. `get_current_user()` 依赖注入（`app/utils/jwt.py`）

```bash
# 写完运行全量单测
pytest tests/test_auth.py -v
```

**提交**：
```bash
git add .
git commit -m "feat: 认证模块 — 短信登录 + JWT签发 + Token刷新"
```

**⚠️ 此时将分支推送到远程，通知 Person B 和 C 可以开始对接**：
```bash
git push -u origin feat/infra-auth-resume
```

#### 第 6 步：用户 + 简历 + 偏好模块（Day 3-5，约 6h）

按顺序完成：
1. **用户模块**（T1.2）：profile CRUD + 学生认证
2. **简历模块**（T1.3）：上传/解析/改写/版本管理（mock Claude API）
3. **偏好设置**（T1.4）：`GET/PUT /api/preference`

每个模块完成后的提交节奏：
```bash
# 完成用户模块
git add app/api/user.py app/services/user_service.py app/models/user.py
git commit -m "feat: 用户模块 — 个人信息CRUD + 学生认证"
# 推送到远程
git push

# 完成简历模块
git add app/api/resume.py app/services/resume_service.py
git commit -m "feat: 简历模块 — 上传解析 + AI改写 + 版本管理"
git push

# 完成偏好设置
git add app/api/preference.py app/services/preference_service.py
git commit -m "feat: 偏好设置 — 求职偏好CRUD + 参数校验"
git push
```

#### 第 7 步：创建 Pull Request（Day 5）

```bash
# 确保所有改动已推送
git push

# 去 GitHub 页面创建 PR：feat/infra-auth-resume → main
# PR 标题："后端基础设施 + 用户简历闭环（M0+M1）"
# PR 描述：列出完成的任务 + 验收证据
```

#### 第 8 步：AI 服务真实接入（Day 9-10，约 3h）

在等待 PR review 期间，完成 T5.2：
1. `app/services/ai_service.py` — 封装 Claude API 调用
2. `app/prompts/*.txt` — 5 个 prompt 文件

```bash
git checkout feat/infra-auth-resume  # 切回自己的分支
# ... 开发 AI 服务 ...
git add app/services/ai_service.py app/prompts/
git commit -m "feat: AI服务真实接入 — Claude API封装 + 5个prompt + 降级策略"
git push
```

#### 第 9 步：集成联调（Day 11-12，全员协作）

切换到 `main` 分支，三人一起联调：
```bash
git checkout main
git pull origin main
# 解决任何合并冲突
# 运行全量测试
pytest -v --cov=app --cov-report=term-missing
```

### Person A 需要阅读的文档

| 文档 | 何时读 |
|---|---|
| `CLAUDE.md` | 开始前 |
| `docs/ARCHITECTURE.md` | 开始前 |
| `docs/SCHEMA.md` | 做数据库迁移前 |
| `docs/modules/auth.md` | 做认证模块前 |
| `docs/modules/resume.md` | 做简历模块前 |
| `docs/modules/ai_service.md` | 做 AI 服务前 |
| `docs/API_SCHEMA.yaml` | 写每个端点前确认字段名 |
| `docs/TEST_GUIDE.md` | 写单测前 |
| `AI产品经理培训作业/应届生个人简历市场匹配助手--方案-v1.md` | 理解产品边界 |

---

## 三、Person B：后端核心业务线

### 负责范围

| 里程碑 | 任务 | 产出文件 |
|---|---|---|
| M2 | T2.1 岗位模块 + 爬虫 | `app/api/job.py`, `app/services/job_service.py`, `app/tasks/job_crawler.py` |
| M2 | T2.2 匹配引擎 ★核心 | `app/api/match.py`, `app/services/match_service.py`, `app/tasks/match.py` |
| M2 | T2.3 推送通知 | `app/api/notification.py`, `app/tasks/notification.py` |
| M3 | T3.1 招呼语文案 | `app/api/cover_letter.py`, `app/services/cover_letter_service.py` |
| M3 | T3.2 投递管理 | `app/api/application.py`, `app/services/application_service.py` |
| M3 | T3.3 支付订阅 | `app/api/order.py`, `app/services/order_service.py` |

### 依赖关系

- **依赖 Person A**：auth（JWT 验证）、resume（匹配输入）、preference（匹配参数）
- **被 Person C 依赖**：match API（首页推送）、job API（搜索/详情）、application API（投递台）、cover_letter API（文案页）、order API（支付页）
- **自包含**：job_crawler（mock 第三方 API，不依赖 A 的其他模块）

### 完整工作流程

#### 第 1 步：创建分支 + 先做不依赖 A 的工作（Day 1-2）

```bash
git clone https://github.com/jubai123/ResuMatch.git
cd ResuMatch
git checkout main
git checkout -b feat/match-apply
```

**Day 1**（A 还没完成基础设施）：
- 精读所有技术文档（CLAUDE.md / ARCHITECTURE.md / SCHEMA.md / API_SCHEMA.yaml）
- 精读模块文档（job.md / match_engine.md / application.md / cover_letter.md / order.md / push.md）
- 在本地搭建 Docker 环境（等 A 完成 docker-compose.yml 后拉取）
- **先写不依赖外部 API 的纯逻辑代码**：
  - `app/services/match_service.py:calculate_match_score()` — 6 维度评分算法（纯函数，不需要数据库）
  - 匹配引擎的单元测试

```bash
# Day 1-2 的提交
git add app/services/match_service.py tests/test_match_engine.py
git commit -m "feat: 匹配引擎核心算法 — 6维度加权评分 + 单元测试"
```

#### 第 2 步：等待 A 的分支就绪后拉取（Day 2-3）

Person A 推送 `feat/infra-auth-resume` 后：

```bash
# 拉取 A 的最新代码到本地
git fetch origin feat/infra-auth-resume

# 将 A 的分支合并到自己的分支
git merge origin/feat/infra-auth-resume

# 解决冲突（如有）
# ... 解决后 ...
git add .
git commit -m "merge: 合并feat/infra-auth-resume — 获取认证+用户+简历+偏好API"
```

#### 第 3 步：岗位模块 + 爬虫（Day 3-4，约 5h）

1. `app/api/job.py` — `GET /api/job/{id}` + `GET /api/job/search`
2. `app/services/job_service.py` — 搜索/详情/批量入库
3. `app/tasks/job_crawler.py` — Celery 定时任务，mock 第三方聚合 API

```bash
git add app/api/job.py app/services/job_service.py app/tasks/job_crawler.py
git commit -m "feat: 岗位模块 — 搜索筛选 + 详情 + Celery爬虫(mock)"
git push
```

#### 第 4 步：匹配引擎完整实现（Day 4-6，约 8h）★ 最核心

1. `app/services/match_service.py` — 补全：
   - `calculate_match_score()` — 已写核心算法
   - `generate_initial_matches()` — 新用户首次匹配 Top 5
   - `generate_weekly_batch()` — Celery 周推送生成
2. `app/api/match.py` — 3 个端点
3. `app/tasks/match.py` — Celery Beat 定时任务

```bash
# 分两次提交
# 第一次：首次匹配
git add app/api/match.py app/services/match_service.py
git commit -m "feat: 匹配引擎 — 首次匹配Top5 + 周推送生成 + 分层查询"
git push

# 第二次：反馈+定时任务
git add app/tasks/match.py
git commit -m "feat: 匹配引擎 — 用户反馈反哺 + Celery Beat周推送调度"
git push
```

#### 第 5 步：推送通知 + 招呼语 + 投递 + 支付（Day 6-8，约 8h）

按依赖顺序开发（每个模块一个 commit）：

```bash
# 推送通知（不依赖其他模块）
git commit -m "feat: 推送通知 — 设备注册 + 周推送通知发送(mock)"

# 招呼语文案（依赖 match_engine + AI service）
git commit -m "feat: 招呼语文案 — 3风格版本生成 + 降级模板"

# 投递管理（依赖 match_engine + cover_letter）
git commit -m "feat: 投递管理 — 状态机 + 漏斗统计 + 14天提醒"

# 支付订阅（依赖 auth + user）
git commit -m "feat: 支付订阅 — 3套餐下单 + 支付回调(mock) + 订阅激活"
```

每次 commit 后 `git push` 保持远程同步。

#### 第 6 步：创建 Pull Request（Day 8）

```bash
git push  # 确保所有提交已推送

# 去 GitHub 创建 PR：feat/match-apply → main
# PR 标题："后端核心业务线 — 匹配+投递+支付闭环（M2+M3）"
```

#### 第 7 步：补充单测 + Code Review 修改（Day 9-10）

根据 PR review 意见修改代码，补充单测至覆盖率 ≥80%（匹配引擎 ≥90%）。

#### 第 8 步：集成联调（Day 11-12，全员）

```bash
git checkout main
git pull origin main
# 三人一起联调，修复跨模块 Bug
```

### Person B 需要阅读的文档

| 文档 | 何时读 |
|---|---|
| `CLAUDE.md` | 开始前 |
| `docs/ARCHITECTURE.md` | 开始前（重点看模块依赖关系图） |
| `docs/SCHEMA.md` | 写 ORM 查询前（重点看 jobs/match_results/applications/orders 表） |
| `docs/modules/job.md` | 做岗位模块前 |
| `docs/modules/match_engine.md` | 做匹配引擎前 ★ |
| `docs/modules/cover_letter.md` | 做文案模块前 |
| `docs/modules/application.md` | 做投递模块前（重点看状态机） |
| `docs/modules/order.md` | 做支付模块前 |
| `docs/modules/push.md` | 做推送模块前 |
| `docs/API_SCHEMA.yaml` | 每个端点写之前对照 |
| `docs/TEST_GUIDE.md` | 写单测前 |

---

## 四、Person C：前端全栈 + 联调集成

### 负责范围

| 里程碑 | 任务 | 产出文件 |
|---|---|---|
| M4 | T4.1 前端壳工程+设计系统 | `app-frontend/` 全部初始化 |
| M4 | T4.2 注册登录+引导页 | pages: splash, guide, login, verify |
| M4 | T4.3 简历上传+AI改写 | pages: resume, preference, match/initial |
| M4 | T4.4 首页推送+岗位详情 | pages: home, job/detail |
| M4 | T4.5 投递台+文案+漏斗 | pages: application, cover-letter |
| M4 | T4.6 岗位库搜索+我的+支付 | pages: search, profile, subscribe |
| M5 | T5.1 前后端联调 | 集成测试 |
| M5 | T5.3 Demo 准备 | 演示脚本 + H5 部署 |

### 依赖关系

- **依赖 Person A**：所有后端 API（登录前可先用 mock 数据开发 UI）
- **依赖 Person B**：匹配/投递/支付 API（同样可先 mock）
- **关键策略**：先用 mock 数据开发全部页面，后端 API 就绪后切换为真实接口

### 完整工作流程

#### 第 1 步：创建前端工程（Day 1-2）

```bash
git clone https://github.com/jubai123/ResuMatch.git
cd ResuMatch
git checkout main
git checkout -b feat/frontend-app
```

**Day 1**：uni-app 项目初始化
```bash
# 使用 uni-app CLI 创建项目
npx degit dcloudio/uni-preset-vue#vite-ts app-frontend
cd app-frontend
npm install

# 安装依赖
npm install pinia axios
```

创建目录结构：
```
app-frontend/src/
├── pages/          # 页面组件
│   ├── splash/
│   ├── guide/
│   ├── login/
│   ├── verify/
│   ├── resume/
│   ├── preference/
│   ├── home/
│   ├── job/
│   ├── application/
│   ├── cover-letter/
│   ├── search/
│   ├── profile/
│   └── subscribe/
├── components/     # 通用组件
├── composables/    # 逻辑复用
├── stores/         # Pinia stores
├── utils/          # 工具函数（HTTP封装等）
└── styles/         # 设计令牌 CSS 变量
```

**Day 2**：设计系统 + 通用组件 + HTTP 封装
- `styles/tokens.css` — 设计令牌（品牌色/字号/间距/圆角）
- 通用组件：Button, Card, Tag, Input, Modal, Toast, ProgressBar, Empty, Loading, Skeleton
- `utils/request.ts` — axios 封装（JWT 自动附加 + 401 跳转登录 + 响应拦截）
- Pinia stores：`useAuthStore`, `useUserStore`
- 路由配置：4 Tab + 子页面

```bash
git add .
git commit -m "feat: 前端壳工程 — uni-app初始化 + 设计系统 + 通用组件 + HTTP封装"
git push -u origin feat/frontend-app
```

#### 第 2 步：注册登录 + 引导页（Day 3-4，用 mock 数据）

**每个页面开发流程**（以登录页为例）：

1. 读线框图（`AI产品经理培训作业/应届生个人简历市场匹配助手-MVP移动端App功能设计文档+线框图.md` 第 7.3 节）
2. 读 API 契约（`docs/API_SCHEMA.yaml` 中 `/auth/sms` 和 `/auth/login` 的定义）
3. 创建 mock handler（在 `utils/mock.ts` 中，返回符合 API 契约的假数据）
4. 编写页面组件（Vue 3 Composition API + TypeScript）
5. 在 H5 浏览器中验证交互

Mock 数据示例：
```typescript
// utils/mock.ts — 后端未就绪时的临时 mock
export const mockHandlers = {
  '/api/auth/login': () => ({
    accessToken: 'mock-jwt-token',
    refreshToken: 'mock-refresh-token',
    user: { id: 'mock-uuid', phone: '138****8000', nickname: '测试用户' }
  }),
  '/api/match/weekly': () => ({
    batchInfo: { weekOf: '2026-08-03', totalCount: 18, processedCount: 5 },
    results: [ /* 3条模拟匹配结果 */ ]
  })
}
```

**此阶段完成的页面**：
- 启动页（3s 跳过）
- 3 屏引导页（价值主张 / 信任建立 / CTA）
- 手机号登录页（验证码输入 + 60s 倒计时）
- 学生认证页（可选跳过）

```bash
git add app-frontend/src/pages/splash/ app-frontend/src/pages/guide/ app-frontend/src/pages/login/
git commit -m "feat: 启动引导登录页 — 4页面 + mock数据驱动"
git push
```

#### 第 3 步：简历 + 偏好 + 首次匹配（Day 4-6，约 10h）

**此阶段完成的页面**：
- 简历上传四选一（PDF/在线/拍照/微信）
- 简历解析预览 + 确认
- AI 改写对比页（原文/AI版/理由/接受/拒绝/微调）
- 偏好设置 3 步引导（城市/行业/薪资+关键词）
- 首次匹配结果展示（5 条卡片）

```bash
git add app-frontend/src/pages/resume/ app-frontend/src/pages/preference/ app-frontend/src/pages/match/
git commit -m "feat: 简历+偏好+首次匹配页面 — 上传解析 + AI改写对比 + 首次5条匹配"
git push
```

#### 第 4 步：首页推送 + 岗位详情 + 岗位库（Day 6-8）

**此阶段完成的页面**：
- 首页（Top 3 高匹配卡片 + 进度条 + 全部抽屉分层加载）
- 岗位详情页（匹配点高亮 + 优势/差距 + AI 建议）
- 岗位库搜索（关键词 + 城市/行业/薪资筛选 + 分页）

```bash
git add app-frontend/src/pages/home/ app-frontend/src/pages/job/ app-frontend/src/pages/search/
git commit -m "feat: 首页+岗位详情+搜索 — 分层加载 + 匹配解释 + 多维筛选"
git push
```

#### 第 5 步：投递台 + 招呼语 + 支付 + 我的（Day 8-10）

**此阶段完成的页面**：
- 招呼语生成页（3 选 1 + 微调 + 复制）
- 投递台（状态分类 + 时间线 + 提醒）
- 漏斗统计页（数据可视化）
- 支付页（3 套餐 + 微信/支付宝 + 支付状态）
- 我的页面（简历预览/偏好/订阅/推送设置）

```bash
git add app-frontend/src/pages/application/ app-frontend/src/pages/cover-letter/ app-frontend/src/pages/subscribe/ app-frontend/src/pages/profile/
git commit -m "feat: 投递台+文案+支付+我的 — 投递状态机 + 3版招呼语 + 订阅支付"
git push
```

#### 第 6 步：切换到真实 API（Day 10-11）

当 Person A 和 B 的后端 API 就绪后：

```bash
# 1. 拉取最新的 main
git checkout main && git pull origin main

# 2. 切回自己的分支，合并 main
git checkout feat/frontend-app
git merge main

# 3. 移除 mock，切换到真实 API
# 修改 utils/request.ts，将 mock 拦截器移除
# 逐个页面验证真实 API 调用

# 4. 修复前端与真实 API 的不一致（字段名、格式等）
```

**切换策略**：逐个页面切换（而非一次性全部切换），降低风险：
1. 先切换登录注册（验证 auth API）
2. 再切换简历上传（验证 resume API）
3. 再切换首页推送（验证 match API）
4. 依次切换剩余页面

```bash
git commit -m "refactor: 移除mock — 切换到真实API + 字段适配 + 错误处理"
git push
```

#### 第 7 步：创建 Pull Request（Day 11）

```bash
# 去 GitHub 创建 PR：feat/frontend-app → main
# PR 标题："前端全栈 — 全部页面 + 设计系统 + 真实API对接（M4+M5）"
```

#### 第 8 步：集成联调 + Demo 准备（Day 11-14，全员）

```bash
git checkout main
git pull origin main

# 端到端测试：注册 → 上传简历 → 偏好 → 匹配 → 投递 → 反馈
# 录屏演示脚本
# H5 部署
```

### Person C 需要阅读的文档

| 文档 | 何时读 |
|---|---|
| `CLAUDE.md` | 开始前（重点看前端编码规范） |
| `docs/ARCHITECTURE.md` | 开始前（理解系统全貌） |
| `docs/API_SCHEMA.yaml` | **每个页面开发前对照**（字段名/类型/枚举值） |
| `docs/TEST_GUIDE.md` | 写组件测试前 |
| `docs/TASK_SPEC.md` | 规划每日工作时 |
| `AI产品经理培训作业/应届生个人简历市场匹配助手-MVP移动端App功能设计文档+线框图.md` | **每个页面开发前读对应线框图** |

---

## 五、Git 协作规范（三人通用）

### 5.1 分支策略

```
main (默认分支，只接受 PR 合并，禁止直接 push)
  ├── feat/infra-auth-resume  (Person A)
  ├── feat/match-apply        (Person B)
  └── feat/frontend-app       (Person C)
```

### 5.2 每日协作节奏

```
上午 09:30 — 站会 15 分钟（同步进度 + 阻塞点）
上午 10:00 — 各自开发
中午 12:00 — （可选）check-in
下午 14:00 — 各自开发
下午 17:30 — 提交代码并 push
下午 18:00 — 检查 CI/测试是否通过
```

### 5.3 每日 Git 操作清单

```bash
# === 每天开始工作前 ===
git checkout main
git pull origin main                  # 拉取最新代码

git checkout <你的分支>
git merge main                        # 将 main 的更新合并到你的分支
# 解决冲突（如有）

# === 开发过程中 ===
# 每完成一个子任务就 commit（小步提交，方便回滚）
git add <改动的文件>
git commit -m "<type>: <简短描述>"

# === 每天结束工作前 ===
git push origin <你的分支>            # 推送到远程备份
```

### 5.4 Commit 规范

```
<type>: <简短描述>

type 类型：
  feat:     新功能
  fix:      Bug 修复
  refactor: 重构
  test:     测试
  docs:     文档
  chore:    杂项（依赖更新、配置修改）

示例：
  feat: 匹配引擎 — 6维度加权评分算法
  fix: 修复登录页验证码倒计时不重置
  test: 补全match_service单元测试覆盖率至91%
  refactor: 简历解析改用异步Celery任务
```

### 5.5 冲突解决流程

```bash
# 1. 更新本地 main
git checkout main && git pull origin main

# 2. 切回自己的分支，合并 main
git checkout <你的分支>
git merge main

# 3. 如果出现冲突
# CONFLICT (content): Merge conflict in xxx.py

# 4. 打开冲突文件，找到 <<<<<<< HEAD 和 >>>>>>> main
# 手动选择保留哪部分代码

# 5. 标记已解决
git add <冲突文件>

# 6. 完成合并
git commit -m "merge: 解决与main的冲突 — <简述冲突原因>"

# 7. 推送
git push origin <你的分支>
```

**冲突时不要慌**：
1. 先看是哪几行冲突
2. 理解两边的改动意图
3. 如果拿不准，在团队群里问对方
4. 不要直接 `git checkout --theirs` 或 `git checkout --ours` 全量覆盖

### 5.6 Pull Request 规范

**创建 PR 时**：
- 标题：`<模块名> — <完成的任务>`
- 描述：
  ```
  ## 完成的任务
  - [x] T1.1 认证模块
  - [x] T1.2 用户模块

  ## 验收证据
  - 单元测试通过：pytest tests/ -v (42 passed)
  - 覆盖率：85%

  ## 依赖
  - 依赖 PR #2（Person B 的岗位模块）
  ```

**Review PR 时**（每天至少 review 一个队友的 PR）：
1. 先看 CI 是否通过
2. 看代码逻辑是否正确
3. 看是否符合编码规范（参考 CLAUDE.md）
4. 看单测是否覆盖核心路径
5. 提出具体修改意见（不要只说"这里有问题"）

### 5.7 紧急情况处理

| 情况 | 操作 |
|---|---|
| 不小心 commit 了错误代码 | `git revert <commit-hash>` 然后 push（不要用 reset） |
| 需要临时保存未完成的代码 | `git stash` → 切分支 → 回来后 `git stash pop` |
| 开发方向错了想回退 | `git log --oneline` 找到目标commit → `git reset --hard <hash>` 仅限本地未push的分支 |
| 不确定当前状态 | `git status` + `git log --oneline -5` |

### 5.8 禁止操作（三人共同遵守）

- ❌ **永远不要** `git push --force` 到 main 分支
- ❌ **永远不要** 直接 push 到 main（必须走 PR）
- ❌ **永远不要** `git reset --hard` 已经 push 的 commit
- ❌ **永远不要** `git commit --amend` 已经 push 的 commit
- ❌ **永远不要** 在 merge 冲突时 `git merge --abort` 后不告知队友

---

## 六、关键节点与交付物

| 日期 | 节点 | Person A | Person B | Person C |
|---|---|---|---|---|
| Day 1 晚 | 环境就绪 | docker-compose up 成功 | 本地环境跑通 | H5 dev server 跑通 |
| Day 3 晚 | **阻塞点解除** | 认证 API 可用 + 推送到远程 | — | 登录页 mock 完成 |
| Day 5 晚 | M1 完成 | 用户+简历+偏好 API 完成 | 岗位模块完成 | 简历+偏好页面完成 |
| Day 8 晚 | M2+M3 完成 | — | 匹配+投递+支付 API 完成 | 投递+支付页面完成 |
| Day 11 晚 | 联调完成 | 后端 bug 清零 | 后端 bug 清零 | 真实 API 全部对接 |
| Day 14 晚 | **Demo** | AI 服务上线 | 匹配质量达标 | H5 公网可访问 |

---

## 七、沟通约定

1. **每天站会**：每人说 3 件事——昨天做了什么、今天计划做什么、有没有阻塞
2. **阻塞立即说**：遇到依赖队友的阻塞，立即在群里 @对方，不要自己闷头等
3. **PR 24h 内 Review**：队友提交 PR 后，24h 内必须给出 review 意见
4. **文档即真相**：对字段名/接口行为有疑问时，先查 `API_SCHEMA.yaml` 和 `SCHEMA.md`，文档没说再问人
5. **提交即通知**：push 代码后在群里发一句"我 push 了，commit xxx"

---

## 八、参考文档速查

| 想查什么 | 打开哪个文件 |
|---|---|
| 这个产品到底做什么 | `AI产品经理培训作业/应届生个人简历市场匹配助手--方案-v1.md` |
| 页面长什么样 | `AI产品经理培训作业/应届生个人简历市场匹配助手-MVP移动端App功能设计文档+线框图.md` |
| 不能做什么 | `CLAUDE.md` 第 4 节 |
| 怎么写代码 | `CLAUDE.md` 第 3 节 |
| 改这里会影响哪里 | `docs/ARCHITECTURE.md` 第 6 节 |
| 接口字段叫什么 | `docs/API_SCHEMA.yaml` |
| 数据库表结构 | `docs/SCHEMA.md` |
| 这个模块做什么 | `docs/modules/<模块名>.md` |
| 测试怎么写 | `docs/TEST_GUIDE.md` |
| 今天做什么 | `docs/TASK_SPEC.md` |
