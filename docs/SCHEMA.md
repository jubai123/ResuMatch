# SCHEMA.md — 数据库设计文档

> **用途**：写 SQL / ORM 查询前必读。包含所有表结构、索引、枚举值和关系。

---

## 1. ER 关系总览

```
user (1) ─────< resume (N)          (一个用户可有多个简历版本)
user (1) ─────< preference (1)      (一个用户一套偏好)
user (1) ─────< match_result (N)
user (1) ─────< push_batch (N)
user (1) ─────< application (N)
user (1) ─────< order (N)
user (1) ─────< device_token (N)

job (1) ─────< match_result (N)
job (1) ─────< application (N)

push_batch (1) ──< match_result (N)
```

---

## 2. 表定义

### 2.1 `users` — 用户账户

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK, DEFAULT gen_random_uuid() | |
| `phone_hash` | VARCHAR(64) | NOT NULL, UNIQUE | 手机号 SHA-256 哈希（不存明文） |
| `phone_masked` | VARCHAR(16) | NOT NULL | 脱敏显示 138****8000 |
| `nickname` | VARCHAR(64) | NOT NULL, DEFAULT '' | |
| `avatar_url` | VARCHAR(512) | | MinIO/S3 路径 |
| `school_verified` | BOOLEAN | NOT NULL, DEFAULT FALSE | 学生认证状态 |
| `school_tier` | school_tier_enum | | 院校层次 |
| `major` | VARCHAR(128) | | |
| `degree` | degree_enum | | |
| `graduation_year` | SMALLINT | | 如 2026 |
| `subscribed` | BOOLEAN | NOT NULL, DEFAULT FALSE | 当前是否有有效订阅 |
| `subscribe_plan` | plan_enum | | |
| `subscribe_expire_at` | TIMESTAMPTZ | | |
| `student_discount` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否享受学生价 |
| `last_login_at` | TIMESTAMPTZ | | |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `deleted_at` | TIMESTAMPTZ | | 软删除 |

**索引**：
- `idx_users_phone_hash` UNIQUE BTREE (phone_hash) WHERE deleted_at IS NULL
- `idx_users_subscribed` BTREE (subscribed) WHERE subscribed = TRUE
- `idx_users_school_tier` BTREE (school_tier) WHERE school_tier IS NOT NULL

---

### 2.2 `resumes` — 简历（支持历史版本）

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | NOT NULL, FK → users.id | |
| `version` | INTEGER | NOT NULL, DEFAULT 1 | 同一用户版本递增 |
| `is_current` | BOOLEAN | NOT NULL, DEFAULT TRUE | 当前在用版本 |
| `source` | resume_source_enum | NOT NULL | 上传渠道 |
| `raw_text` | TEXT | | 解析出的纯文本全文 |
| `structure` | JSONB | NOT NULL, DEFAULT '{}' | 结构化字段（见 ResumeStructure schema） |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：
- `idx_resumes_user_current` BTREE (user_id) WHERE is_current = TRUE
- `idx_resumes_structure` GIN (structure) — JSONB 全文检索

**约束**：
- UNIQUE (user_id, version)
- 每个 user_id 最多保留 10 个版本（应用层限制）

---

### 2.3 `resume_rewrites` — AI 改写记录

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK | |
| `resume_id` | UUID | NOT NULL, FK → resumes.id | |
| `field` | VARCHAR(128) | NOT NULL | JSON path: `projects.0.description` |
| `original` | TEXT | NOT NULL | |
| `suggested` | TEXT | NOT NULL | |
| `reason` | TEXT | NOT NULL | AI 解释 |
| `status` | rewrite_status_enum | NOT NULL, DEFAULT 'pending' | |
| `user_final` | TEXT | | 微调后版本 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：
- `idx_rewrites_resume` BTREE (resume_id, field)

---

### 2.4 `preferences` — 用户偏好设置

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `user_id` | UUID | PK, FK → users.id | 一对一 |
| `cities` | VARCHAR(64)[] | NOT NULL, DEFAULT '{}' | 最多 3 个 |
| `industries` | VARCHAR(64)[] | | 最多 3 个 |
| `salary_min` | INTEGER | | 月薪下限（k） |
| `degree` | degree_enum | | |
| `keywords` | VARCHAR(32)[] | | 最多 5 个 |
| `accept_relocate` | BOOLEAN | NOT NULL, DEFAULT FALSE | 接受异地 |
| `exclude_companies` | VARCHAR(128)[] | DEFAULT '{}' | 公司黑名单 |
| `exclude_keywords` | VARCHAR(32)[] | DEFAULT '{}' | 关键词黑名单 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

---

### 2.5 `jobs` — 岗位（来自外部聚合数据）

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK | |
| `external_id` | VARCHAR(128) | NOT NULL | 原平台 ID |
| `source` | job_source_enum | NOT NULL | 来源平台 |
| `title` | VARCHAR(256) | NOT NULL | 岗位名称 |
| `company_name` | VARCHAR(256) | NOT NULL | |
| `company_logo` | VARCHAR(512) | | |
| `company_industry` | VARCHAR(64) | | |
| `company_size` | VARCHAR(32) | | |
| `city` | VARCHAR(64) | NOT NULL | |
| `district` | VARCHAR(64) | | |
| `salary_min` | INTEGER | | 月薪下限（k） |
| `salary_max` | INTEGER | | 月薪上限（k） |
| `salary_months` | SMALLINT | DEFAULT 12 | 年薪月数（12/14/16） |
| `degree` | degree_enum | | |
| `experience` | VARCHAR(16) | | fresh / intern / 1-3 / 3-5 |
| `industry_tag` | VARCHAR(64) | | 行业赛道标签 |
| `jd_text` | TEXT | NOT NULL | 原始 JD 文本 |
| `jd_structured` | JSONB | | 结构化 JD（职责/要求/技能） |
| `hr_name` | VARCHAR(64) | | |
| `hr_title` | VARCHAR(64) | | |
| `hr_active_days` | INTEGER | | HR 最近活跃天数 |
| `published_at` | TIMESTAMPTZ | NOT NULL | 原始发布时间 |
| `expired_at` | TIMESTAMPTZ | | |
| `status` | job_status_enum | NOT NULL, DEFAULT 'active' | |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：
- `idx_jobs_external` UNIQUE BTREE (source, external_id) — 去重
- `idx_jobs_city` BTREE (city) WHERE status = 'active'
- `idx_jobs_industry` BTREE (industry_tag) WHERE status = 'active'
- `idx_jobs_published` BTREE (published_at DESC) WHERE status = 'active'
- `idx_jobs_title` GIN (to_tsvector('simple', title)) — 全文搜索岗位名
- `idx_jobs_salary` BTREE (salary_min) WHERE status = 'active'

---

### 2.6 `match_results` — 匹配结果

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | NOT NULL, FK → users.id | |
| `job_id` | UUID | NOT NULL, FK → jobs.id | |
| `score` | SMALLINT | NOT NULL, CHECK (score >= 0 AND score <= 100) | 匹配分 |
| `reasons` | TEXT[] | NOT NULL | AI 解释，3 条以内 |
| `matched_skills` | VARCHAR(64)[] | | 匹配上的技能 |
| `gaps` | VARCHAR(128)[] | | 差距分析 |
| `push_batch_id` | UUID | FK → push_batches.id | 哪周推送的 |
| `user_status` | match_status_enum | NOT NULL, DEFAULT 'pending' | |
| `feedback_at` | TIMESTAMPTZ | | 用户反馈时间 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：
- `idx_match_user_week` BTREE (user_id, push_batch_id) — 获取用户本周推送
- `idx_match_user_score` BTREE (user_id, score DESC) — 按分数排序
- `idx_match_user_status` BTREE (user_id, user_status) — 按状态筛选
- `idx_match_feedback_time` BTREE (user_id, feedback_at) WHERE feedback_at IS NOT NULL

**约束**：
- UNIQUE (user_id, job_id, push_batch_id) — 同一岗位上同一批次不重复推送

---

### 2.7 `push_batches` — 每周推送批次

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | NOT NULL, FK → users.id | |
| `week_of` | DATE | NOT NULL | 周一日期 |
| `generated_at` | TIMESTAMPTZ | NOT NULL | 生成时间 |
| `job_count` | INTEGER | NOT NULL | 本批次岗位数 |
| `viewed_count` | INTEGER | NOT NULL, DEFAULT 0 | 已浏览数 |
| `interested_count` | INTEGER | NOT NULL, DEFAULT 0 | "想投"数 |
| `applied_count` | INTEGER | NOT NULL, DEFAULT 0 | 已投递数 |
| `notified_at` | TIMESTAMPTZ | | 推送通知发送时间 |

**索引**：
- `idx_batch_user_week` UNIQUE BTREE (user_id, week_of) — 每周一批

---

### 2.8 `applications` — 投递记录

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | NOT NULL, FK → users.id | |
| `job_id` | UUID | NOT NULL, FK → jobs.id | |
| `match_result_id` | UUID | FK → match_results.id | |
| `status` | application_status_enum | NOT NULL, DEFAULT 'interested' | |
| `cover_letter` | TEXT | | AI 生成的招呼语 |
| `applied_at` | TIMESTAMPTZ | | 投递时间 |
| `timeline` | JSONB | NOT NULL, DEFAULT '[]' | [{status, at, note}] |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：
- `idx_applications_user` BTREE (user_id, status)
- `idx_applications_job` BTREE (job_id)
- `idx_applications_applied` BTREE (user_id, applied_at) WHERE applied_at IS NOT NULL

---

### 2.9 `orders` — 支付订单

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | NOT NULL, FK → users.id | |
| `plan` | plan_enum | NOT NULL | |
| `amount` | INTEGER | NOT NULL | 实付金额（分） |
| `pay_method` | pay_method_enum | NOT NULL | |
| `status` | order_status_enum | NOT NULL, DEFAULT 'pending' | |
| `third_party_trade_no` | VARCHAR(128) | | 微信/支付宝交易号 |
| `paid_at` | TIMESTAMPTZ | | |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：
- `idx_orders_user` BTREE (user_id, created_at DESC)
- `idx_orders_trade` BTREE (third_party_trade_no) WHERE third_party_trade_no IS NOT NULL

---

### 2.10 `device_tokens` — 推送设备注册

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | NOT NULL, FK → users.id | |
| `token` | VARCHAR(512) | NOT NULL | APNs / FCM token |
| `platform` | device_platform_enum | NOT NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：
- `idx_devices_user` BTREE (user_id)
- `idx_devices_token` UNIQUE BTREE (token)

---

### 2.11 `student_verifications` — 学生认证记录

| 列 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | NOT NULL, FK → users.id | |
| `image_url` | VARCHAR(512) | NOT NULL | 上传的学信网截图 |
| `ocr_result` | JSONB | | OCR 识别结果 |
| `status` | verification_status_enum | NOT NULL, DEFAULT 'pending' | |
| `reviewed_by` | UUID | | 审核人（管理员） |
| `reviewed_at` | TIMESTAMPTZ | | |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

---

## 3. 枚举值定义

| 枚举类型 | 值 | 说明 |
|---|---|---|
| `school_tier_enum` | `'985_211'`, `'double_first_class'`, `'regular'`, `'junior_college'` | 院校层次 |
| `degree_enum` | `'junior_college'`, `'bachelor'`, `'master'`, `'phd'`, `'any'` | 学历 |
| `plan_enum` | `'basic'`, `'pro'`, `'premium'` | 订阅套餐（49/99/199） |
| `resume_source_enum` | `'pdf'`, `'word'`, `'photo'`, `'online'`, `'wechat'` | 简历来源 |
| `rewrite_status_enum` | `'pending'`, `'accepted'`, `'rejected'`, `'modified'` | 改写状态 |
| `job_source_enum` | `'zhipin'`, `'zhilian'`, `'51job'`, `'liepin'`, `'lagou'` | 岗位来源 |
| `job_status_enum` | `'active'`, `'offline'` | 岗位状态 |
| `match_status_enum` | `'pending'`, `'interested'`, `'not_interested'`, `'deferred'`, `'applied'`, `'communicating'`, `'interviewing'`, `'offered'`, `'rejected'` | 匹配状态 |
| `application_status_enum` | `'interested'`, `'applied'`, `'communicating'`, `'interviewing'`, `'offered'`, `'rejected'` | 投递状态（子集） |
| `order_status_enum` | `'pending'`, `'paid'`, `'expired'`, `'refunded'` | 支付状态 |
| `pay_method_enum` | `'wechat'`, `'alipay'` | 支付方式 |
| `device_platform_enum` | `'ios'`, `'android'` | 设备平台 |
| `verification_status_enum` | `'pending'`, `'approved'`, `'rejected'` | 认证审核状态 |

---

## 4. JSONB 结构参考

### `resumes.structure`（对应 API_SCHEMA 的 ResumeStructure）

```json
{
  "basics": { "name": "", "phone": "", "email": "", "school": "", "major": "", "degree": "", "graduationYear": 2026 },
  "education": [{ "school": "", "major": "", "degree": "", "startDate": "", "endDate": "", "gpa": "" }],
  "projects": [{ "name": "", "role": "", "startDate": "", "endDate": "", "description": "", "technologies": [] }],
  "internships": [{ "company": "", "position": "", "startDate": "", "endDate": "", "description": "" }],
  "competitions": [{ "name": "", "level": "", "date": "", "award": "", "description": "" }],
  "skills": [],
  "selfEvaluation": ""
}
```

### `jobs.jd_structured`

```json
{
  "responsibilities": ["参与推荐系统迭代", "维护用户增长模型"],
  "requirements": ["熟练 Python/PyTorch", "有推荐/广告/NLP 经验优先"],
  "skills": ["Python", "PyTorch", "推荐系统"],
  "softSkills": ["沟通能力"]
}
```

### `applications.timeline`

```json
[
  { "status": "interested", "at": "2026-08-01T09:00:00Z", "note": null },
  { "status": "applied",   "at": "2026-08-02T14:30:00Z", "note": "使用专业版文案" },
  { "status": "communicating", "at": "2026-08-05T10:00:00Z", "note": "HR 回复消息" }
]
```

---

## 5. 外键关系图

```
users.id ──< resumes.user_id
users.id ──< preferences.user_id
users.id ──< match_results.user_id
users.id ──< push_batches.user_id
users.id ──< applications.user_id
users.id ──< orders.user_id
users.id ──< device_tokens.user_id
users.id ──< student_verifications.user_id

resumes.id ──< resume_rewrites.resume_id

jobs.id ──< match_results.job_id
jobs.id ──< applications.job_id

push_batches.id ──< match_results.push_batch_id
match_results.id ──< applications.match_result_id
```

---

## 6. 迁移注意事项

1. **手机号不存明文**：`phone_hash` 用 SHA-256 哈希 + salt 存储，`phone_masked` 存脱敏显示版
2. **JSONB 字段用 GIN 索引**：`resumes.structure` 和 `jobs.jd_structured` 需全文检索能力
3. **软删除**：`users.deleted_at` 非 NULL 表示已删除；GDPR 合规：用户可请求永久删除
4. **时序数据分区建议**（生产环境）：`match_results` 按月分区，保留 12 个月后归档
5. **连接池**：FastAPI 使用 asyncpg，连接池大小 20（dev）/ 50（prod）
