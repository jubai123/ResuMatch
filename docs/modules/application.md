# application — 投递管理模块

## 职责边界
- **负责**：投递状态机（想投→已投→沟通→面试→Offer）、投递漏斗统计、投递记录增删改查
- **不负责**：匹配结果生成（→ match_engine）、招呼语文案（→ cover_letter）、岗位详情（→ job）

## 主要入口

| 函数/端点 | 位置 | 说明 |
|---|---|---|
| `GET /api/application` | `api/application.py:list_applications()` | 投递列表，支持按状态筛选 |
| `POST /api/application` | `api/application.py:create_application()` | 创建投递记录 |
| `PATCH /api/application/{id}` | `api/application.py:update_status()` | 更新投递状态，写入 timeline |
| `GET /api/application/funnel` | `api/application.py:get_funnel()` | 近 N 天漏斗统计 |

## 状态机

```
interested (想投)
    │
    ├──▶ applied (已投递) ──▶ communicating (沟通中)
    │                              │
    │                        ┌─────┴──────┐
    │                        ▼            ▼
    │                  interviewing    rejected
    │                   (面试中)       (不合适)
    │                        │
    │                        ▼
    │                     offered (已Offer)
    │
    └──▶ rejected (不合适)
```

14 天未动的"想投"→ 前端弹窗提醒"是否已投递"（`GET /api/application` 时返回 `remindAt` 字段）

## 核心服务

```python
# services/application_service.py
async def create_application(user_id, job_id, match_result_id, cover_letter) -> Application:
    """创建投递记录，status=interested"""

async def update_application_status(app_id, new_status, note) -> Application:
    """更新状态 + 追加 timeline 条目"""

async def get_funnel(user_id, days) -> FunnelData:
    """聚合查询：receivedPush → interested → applied → communicating → interviewing → offered"""
```

## 漏斗算法

```python
# 近 N 天的漏斗（SQL 层面聚合）
funnel = {
    "receivedPush":  COUNT(match_results WHERE created_at >= N days ago),
    "interested":     COUNT(match_results WHERE user_status IN ('interested','applied','communicating','interviewing','offered')),
    "applied":        COUNT(applications WHERE applied_at >= N days ago),
    "communicating":  COUNT(applications WHERE status = 'communicating'),
    "interviewing":   COUNT(applications WHERE status = 'interviewing'),
    "offered":        COUNT(applications WHERE status = 'offered'),
    "conversionRate": applied / interested * 100  # 投递转化率
}
```

## 异常策略
| 场景 | 处理 |
|---|---|
| 同一岗位重复创建投递记录 | 返回现有记录（幂等），不重复创建 |
| 状态跃迁不合法 | 422 `INVALID_TRANSITION`，如 offered→interested |
| matchResultId 不存在 | 404 |
| 漏斗查询无数据 | 返回全 0（不报错） |

## 依赖模块
- **依赖**：auth（认证）、match_engine（matchResultId 关联）、cover_letter（招呼语文案）
- **被依赖**：无
