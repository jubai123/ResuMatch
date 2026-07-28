# match_engine — 匹配引擎模块

## 职责边界
- **负责**：计算简历-岗位匹配分、生成每周推送批次、处理用户反馈反哺模型
- **不负责**：简历管理、岗位数据采集（→ job_crawler）、推送通知发送（→ push）

## 主要入口

| 函数/端点 | 位置 | 说明 |
|---|---|---|
| `POST /api/match/initial` | `api/match.py:initial_match()` | 新用户首次匹配，立即返回 Top 5 |
| `GET /api/match/weekly` | `api/match.py:get_weekly()` | 获取本周推送，支持分层分页 |
| `POST /api/match/{id}/feedback` | `api/match.py:submit_feedback()` | 用户反馈（想投/不合适/已投等） |

## 核心服务

```python
# services/match_service.py
async def calculate_match_score(resume: Resume, job: Job, preference: Preference) -> int:
    """计算单个岗位的匹配分（0-100）"""

async def generate_initial_matches(user_id: UUID) -> list[MatchResult]:
    """新用户首次匹配，返回 Top 5"""

async def generate_weekly_batch(user_id: UUID) -> PushBatch:
    """生成用户本周推送（15-25条），Celery Beat 定时调用"""
```

## 匹配算法（MVP 阶段：规则+关键词加权）

```
总分 = Σ (维度得分 × 权重)

| 维度            | 权重  | 评分方式                                |
|-----------------|-------|-----------------------------------------|
| 行业赛道标签    | 30%   | preference.industries ∩ job.industry_tag |
| 薪资下限        | 25%   | job.salary_min ≥ preference.salary_min   |
| 户口/城市匹配   | 15%   | job.city in preference.cities            |
| 岗位名称        | 15%   | resume 关键词 × job.title 余弦相似度     |
| 技能要求        | 10%   | resume.structure.skills ∩ jd_structured.skills |
| 公司（黑/白名单）| 5%   | 白名单加分，黑名单直接过滤              |

说明：
- 每个维度归一化到 0-100 后加权求和
- 黑名单公司直接 score=0（不推送）
- 同公司同岗位去重（取最高分）
- 最终每人取 Top 15-25 条（根据用户活跃度）
```

## 周推送生成流程（Celery Beat: 每周一 08:00）

```
1. 查询所有 subscribed=TRUE 且 subscribe_expire_at > NOW() 的用户
2. 遍历用户（分批，每批 100 人）:
   a. 获取用户最新简历 + 偏好
   b. 查询本周新增岗位（jobs.published_at >= 上周一）
   c. 预过滤：城市/薪资/公司黑名单
   d. 逐岗位计算 match_score
   e. 取 Top N（15-25），生成 match_results
   f. 创建 push_batch 记录
3. 触发推送通知（调用 push 模块）
```

## 反馈反哺

用户每次标记 `interested` / `not_interested` 后：
1. 更新 `match_results.user_status` + `feedback_at`
2. 记录到 `training_log` 表（简化版：{user_id, job_id, status, feature_vector}）
3. 每周日凌晨，Celery 任务汇总本周反馈，调整用户画像权重：
   - `interested` → 相似岗位的维度特征权重 +5%
   - `not_interested` → 标记的 reason 对应维度权重 -5%

## 异常策略
| 场景 | 处理 |
|---|---|
| 用户简历为空 | 返回 400 `RESUME_REQUIRED` |
| 用户偏好未设置 | 使用默认偏好（城市=北上广深，不限行业） |
| 本周无新岗位 | 降级为最近 14 天岗位池 |
| 匹配结果 < 5 条 | 放宽城市限制（acceptRelocate 逻辑） |
| 批量生成超时 | Celery 任务 10 分钟超时，已计算的先入库 |

## 依赖模块
- **依赖**：resume（读取简历）、preference（读取偏好设置）、job（读取岗位库）
- **被依赖**：push（读取 push_batch 发通知）、application（创建投递记录）
