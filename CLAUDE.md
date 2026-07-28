# CLAUDE.md — ResuMatch 项目宪法

> 本文件是 AI 编程助手的最高行为准则。所有代码生成、修改、建议必须遵守本文件。

---

## 1. 项目概述

**ResuMatch** — 应届生个人简历市场匹配助手。AI 驱动的简历-岗位匹配与投递管理平台。

**目标用户**：985/211 + 双一流院校在校研究生（硕/博），CS/EE/AI/半导体/生物医药/金融
**产品形态**：App（iOS + Android）+ H5 轻量版，uni-app 跨端框架
**定价**：基础版 49元/月 | 进阶版 99元/月 | 尊享版 199元/月

**参考文档**（位于 `AI产品经理培训作业/`）：
- `应届生个人简历市场匹配助手--方案-v1.md` — **最高决策依据**（12 刀质询后定稿）
- `应届生个人简历市场匹配助手-MVP移动端App功能设计文档+线框图.md` — UI/UX 规格
- `应届生个人简历市场匹配助手-用户深度研究报告.md` — 市场与用户数据

---

## 2. 技术栈（不可自行变更）

| 层 | 选型 | 版本约束 |
|---|---|---|
| 前端跨端框架 | uni-app (Vue 3) | ^3.0 |
| 前端构建 | Vite | ^5.0 |
| 后端框架 | Python FastAPI | ^0.115 |
| 数据库 | PostgreSQL | ≥15 |
| 缓存/队列 | Redis | ≥7 |
| 异步任务 | Celery + Redis broker | ^5.4 |
| AI 服务 | Claude API (anthropic) | latest |
| 对象存储 | MinIO (dev) / S3 (prod) | — |
| 容器化 | Docker + docker-compose | — |

---

## 3. 编码规范

### 前端 (Vue 3 / uni-app)
- Composition API + `<script setup>` 语法，禁止 Options API
- TypeScript 严格模式，禁止 `any`（除非第三方库无类型定义）
- 组件命名：PascalCase，文件名与组件名一致
- 目录：`src/pages/`（页面）、`src/components/`（组件）、`src/composables/`（逻辑复用）、`src/stores/`（Pinia）
- 状态管理：Pinia，禁止组件内跨页面共享可变状态
- CSS：8pt 网格（8/16/24/32/48），设计令牌变量化，不写硬编码色值

### 后端 (Python FastAPI)
- Python 3.12+，PEP 8 风格
- 所有函数必须标注类型（type hints），返回类型不可省略
- Pydantic v2 模型定义请求/响应，禁止裸 dict 传参
- 路由文件放在 `app/api/` 下，服务逻辑放在 `app/services/` 下
- 异步端点默认 `async def`，CPU 密集型操作用 `run_in_executor`

### 通用
- 日志：`structlog`，JSON 格式，级别分级：DEBUG(dev) / INFO(prod) / WARNING / ERROR
- 错误处理：FastAPI 全局 exception_handler，前端全局 errorHandler；禁止静默吞异常
- 禁止使用 `print()`、`console.log()` 作为日志；禁止 `sleep()` 等待异步结果
- 注释：只写 WHY，不写 WHAT；禁止 TODO 注释（用 Issue 跟踪）

---

## 4. 禁止事项

- 禁止裸爬招聘平台（BOSS/智联/猎聘等）— 仅使用第三方聚合 API + 官方 API
- 禁止 AI 自动代投简历或自动发消息 — MVP 阶段仅生成文案供用户手动复制
- 禁止在代码中硬编码密钥、Token、密码 — 使用环境变量 + `.env` 文件
- 禁止引入重量级状态管理方案替代 Pinia
- 禁止在 uni-app 项目中使用 Web 专有 API（如 `window`、`document`）
- 禁止使用 `var`、`==`（用 `===`）、回调风格异步（用 async/await）

---

## 5. 文件组织

```
ResuMatch/
├── CLAUDE.md                  # 本文件
├── docs/                      # 技术文档
│   ├── ARCHITECTURE.md        # 系统架构
│   ├── API_SCHEMA.yaml        # OpenAPI 3.0 接口契约
│   ├── SCHEMA.md              # 数据库设计
│   ├── TEST_GUIDE.md          # 测试规范
│   ├── TASK_SPEC.md           # 原子级任务规格
│   └── modules/               # 模块级 README
├── AI产品经理培训作业/         # PRD/方案/用户研究（只读参考）
├── app-frontend/              # uni-app 前端工程
├── app-backend/               # FastAPI 后端工程
├── docker-compose.yml
└── .env.example
```

---

## 6. Git 工作流（元规则）

> AI 不主动执行 git 命令。满足以下条件时，**向用户请示**是否执行：

| 触发条件 | 请示内容 |
|---|---|
| 完成一个 TASK_SPEC 中定义的任务 | "任务 X 已完成，是否提交 commit？" |
| 单次修改 ≥3 个文件 | "本次修改涉及 N 个文件，是否提交？" |
| 用户说"保存"/"提交"/"commit" | 执行 `git add` + `git commit` |
| 完成一个 milestone | "里程碑完成，是否创建 tag？" |

**AI 永远不执行**：`git push --force`、`git reset --hard`、`git rebase`（除非用户明确输入完整命令）。

---

## 7. 文档索引

| 文档 | 用途 | 何时查阅 |
|---|---|---|
| `docs/ARCHITECTURE.md` | 模块划分与依赖关系 | 跨模块修改前 |
| `docs/API_SCHEMA.yaml` | 接口契约（OpenAPI 3.0） | 前后端联调、Mock 生成 |
| `docs/SCHEMA.md` | 数据库表结构与索引 | 写 SQL/ORM 查询前 |
| `docs/TASK_SPEC.md` | 可执行任务清单 | 接到新需求时 |
| `docs/TEST_GUIDE.md` | 测试框架与覆盖率要求 | 写测试用例前 |
| `docs/modules/*.md` | 各模块职责边界 | 定位修改目标文件前 |

---

## 8. AI 行为准则

1. **修改前先读**：修改任何文件前，先读完相关 `docs/modules/` 文档
2. **跨模块修改**：先查 `ARCHITECTURE.md` 确认影响范围，再动手
3. **新接口/新表**：先更新 `API_SCHEMA.yaml` 或 `SCHEMA.md`，再写代码
4. **不确定的字段含义**：查 `SCHEMA.md` 的字段注释，不猜测
5. **任务完成标准**：代码通过 + 测试通过 + 对应文档已更新
6. **技术方案冲突时**：以本文件第 2 节技术栈为准，如有更优方案向用户说明理由后请示
