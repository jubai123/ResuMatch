# resume — 简历模块

## 职责边界
- **负责**：简历上传（PDF/Word/图片）、解析、在线填写、AI 改写、版本管理
- **不负责**：简历与岗位的匹配计算（→ match_engine）、投递记录（→ application）

## 主要入口

| 函数/端点 | 位置 | 说明 |
|---|---|---|
| `GET /api/resume` | `api/resume.py:get_current_resume()` | 获取当前用户的当前版本简历 |
| `POST /api/resume/upload` | `api/resume.py:upload_resume()` | 上传文件 → 触发异步解析 |
| `POST /api/resume/online` | `api/resume.py:create_online_resume()` | 在线填写（传入完整 structure） |
| `POST /api/resume/{id}/rewrite` | `api/resume.py:trigger_rewrite()` | 触发 AI 改写（异步），返回 taskId |
| `GET /api/resume/{id}/rewrite/{taskId}` | `api/resume.py:poll_rewrite()` | 轮询改写结果 |
| `POST /api/resume/{id}/rewrite/accept` | `api/resume.py:accept_rewrite()` | 接受/拒绝/微调一条改写建议 |
| `GET /api/resume/history` | `api/resume.py:get_history()` | 获取版本列表 |

## 核心服务

```python
# services/resume_service.py
async def parse_resume_file(file_bytes: bytes, filename: str) -> ResumeStructure:
    """解析 PDF/Word/图片 → 结构化字段。先用本地解析库，再用 Claude API 提取"""

async def generate_rewrite_suggestions(resume: Resume) -> list[AiRewriteRecord]:
    """调用 ai_service.rewrite_resume() 逐字段生成改写建议"""

async def apply_rewrite(resume_id: UUID, field: str, action: str, modified_text: str | None) -> None:
    """应用改写 → 如果全部接受，生成新版本（version+1），旧版本 is_current=FALSE"""

async def create_new_version(user_id: UUID) -> Resume:
    """基于当前版本创建新版本副本"""
```

## 简历解析流程（异步）

```
[upload] → 存储 MinIO → 根据文件类型:
  ├─ PDF: PyMuPDF 提取文本 → Claude API 结构化提取
  ├─ Word: python-docx 提取文本 → Claude API 结构化提取
  └─ 图片: OCR (Tesseract/PaddleOCR) → Claude API 结构化提取
→ 返回 structure → 前端展示预览 → 用户确认 → 保存
```

## 异常策略
| 场景 | 处理 |
|---|---|
| 文件格式不支持 | 422 `UNSUPPORTED_FORMAT`，支持 PDF/DOCX/JPG/PNG |
| 文件过大 | 413 `FILE_TOO_LARGE`，限制 10MB |
| AI 改写超时 | Celery 任务超时 30s，重试 1 次，失败返回 `failed` |
| 解析结果置信度低 | 标记 `low_confidence_fields` 提示用户手动确认 |
| 达到版本上限（10版） | 400 `VERSION_LIMIT`，提示删除旧版本 |

## 依赖模块
- **依赖**：auth（认证）、user（用户信息）、ai_service（解析+改写）
- **被依赖**：match_engine（读取简历计算匹配分）
