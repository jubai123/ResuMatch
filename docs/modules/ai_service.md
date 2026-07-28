# ai_service — AI 服务模块

## 职责边界
- **负责**：封装 Claude API 调用，提供简历改写、JD 解析、匹配解释、招呼语生成能力
- **不负责**：业务逻辑编排（由各调用方 service 负责）、API 限流（由 Nginx 层负责）

## 主要入口

```python
# services/ai_service.py

async def rewrite_resume_section(field: str, original: str, context: dict) -> AiRewriteResult:
    """
    改写简历某个字段（项目经历/实习/自我评价）。
    返回: {suggested, reason}
    """

async def parse_jd(jd_text: str) -> JdStructured:
    """
    结构化解析 JD 文本 → {responsibilities, requirements, skills, softSkills}
    """

async def explain_match(resume_summary: str, job_summary: str) -> list[str]:
    """
    生成匹配解释（3 条以内，每条 1-2 句话）
    """

async def generate_cover_letters(job_title: str, company_name: str, resume_brief: str, style: str) -> list[str]:
    """
    生成 3 个版本的打招呼语: professional / concise / portfolio
    """

async def extract_resume_fields(raw_text: str) -> ResumeStructure:
    """
    从简历原始文本提取结构化字段（首次解析用，非改写）
    """
```

## 调用规范

```python
# 所有 AI 调用统一走此封装
from app.services.ai_service import call_claude

response = await call_claude(
    system_prompt="你是一个专业简历顾问...",
    user_message=f"用户的简历段落：{original}",
    max_tokens=1024,
    temperature=0.7,
)
```

**降级策略**：
- 单次调用超时 30s
- 失败后重试 1 次（间隔 1s）
- 仍失败 → 返回模板化兜底结果（`ai_service.py:_fallback_rewrite()`）
- 兜底结果带标记 `source: "fallback"`，前端展示"AI 暂不可用，已提供通用模板"

## Prompt 管理

所有 system prompt 集中管理在 `app/prompts/` 目录：

```
app/prompts/
├── rewrite_resume.txt      # 简历改写 prompt
├── parse_jd.txt            # JD 解析 prompt
├── explain_match.txt       # 匹配解释 prompt
├── cover_letter.txt        # 招呼语 prompt
└── extract_resume.txt      # 简历提取 prompt
```

每个 prompt 文件包含：
1. 角色定义
2. 输出格式（JSON schema 约束）
3. 示例（few-shot，3 个示例）
4. 禁止事项

## 成本控制（MVP 阶段）

| 接口 | 模型 | max_tokens | 预估调用量（月） |
|---|---|---|---|
| rewrite_resume | claude-sonnet-4-6 | 1024 | 5000 |
| parse_jd | claude-haiku-4-5 | 512 | 50000（批量） |
| explain_match | claude-haiku-4-5 | 256 | 50000 |
| cover_letter | claude-sonnet-4-6 | 768 | 10000 |
| extract_resume | claude-sonnet-4-6 | 2048 | 3000 |

## 异常策略
| 场景 | 处理 |
|---|---|
| API 超时（>30s） | 重试 1 次，仍失败走降级 |
| API 返回格式不符 JSON schema | 记录错误日志，走降级 |
| API key 耗尽/过期 | 抛 `AI_SERVICE_UNAVAILABLE`，所有 AI 功能走降级 |
| 输入过长超 token 限制 | 截断输入（保留前 4000 字符） |
| 敏感内容检测 | 返回 `CONTENT_FILTERED`，提示用户修改 |

## 依赖模块
- **依赖**：无（仅依赖 anthropic SDK）
- **被依赖**：resume、match_engine、cover_letter、job
