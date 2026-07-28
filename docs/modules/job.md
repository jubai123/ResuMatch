# job — 岗位模块

## 职责边界
- **负责**：岗位库搜索/筛选、岗位详情、岗位数据入库（接收 job_crawler 的输出）
- **不负责**：数据采集（→ Celery `tasks/job_crawler.py`）、匹配打分（→ match_engine）

## 主要入口

| 函数/端点 | 位置 | 说明 |
|---|---|---|
| `GET /api/job/{id}` | `api/job.py:get_job_detail()` | 岗位详情（含当前用户匹配分） |
| `GET /api/job/search` | `api/job.py:search_jobs()` | 关键词+多维筛选+分页 |

## 核心服务

```python
# services/job_service.py
async def get_job_with_match(job_id: UUID, user_id: UUID) -> JobDetail:
    """获取岗位详情 + 当前用户匹配分（未匹配则为 None）"""

async def search_jobs(keyword, city, industry, salary_min, degree, page, page_size) -> PaginatedResult:
    """全文搜索 + 筛选，按发布时间倒序"""

async def bulk_upsert_jobs(jobs: list[dict]) -> int:
    """批量 upsert 岗位（job_crawler 调用），按 (source, external_id) 去重"""
```

## 搜索实现

```sql
-- 关键词搜索使用 PostgreSQL 全文搜索
SELECT * FROM jobs
WHERE status = 'active'
  AND (to_tsvector('simple', title) @@ plainto_tsquery('simple', :keyword)
       OR jd_text ILIKE '%' || :keyword || '%')
  AND (:city IS NULL OR city = :city)
  AND (:industry IS NULL OR industry_tag = :industry)
  AND (:salary_min IS NULL OR salary_max >= :salary_min)
  AND (:degree IS NULL OR degree = :degree OR degree = 'any')
ORDER BY published_at DESC
LIMIT :page_size OFFSET :offset;
```

## 数据采集（job_crawler）

```
Celery Beat：每日 02:00 触发
  │
  ├── 调用第三方聚合 API（薪乐宝/职涯宝类）
  │   └── 拉取最近 24h 增量岗位
  │
  ├── 数据清洗
  │   ├── 去重（source + external_id）
  │   ├── 字段标准化（城市映射、学历枚举）
  │   └── JD 结构化（调用 ai_service.parse_jd）
  │
  └── 批量写入 jobs 表 (bulk_upsert)
```

## 异常策略
| 场景 | 处理 |
|---|---|
| 搜索无结果 | 返回空列表 + total:0 |
| 岗位已过期/下线 | status='offline'，详情页显示"已下架" |
| 第三方 API 调用失败 | Celery 任务重试 3 次（指数退避），仍失败告警 |
| JD 解析失败 | 仅存 `jd_text` 原文，`jd_structured` 留空 |

## 依赖模块
- **依赖**：ai_service（JD 结构化解析）
- **被依赖**：match_engine（读取岗位计算匹配分）、application（投递关联岗位）
