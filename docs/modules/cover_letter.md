# cover_letter — 招呼语文案模块

## 职责边界
- **负责**：调用 AI 服务生成打招呼语（3 个版本）、文案模板管理
- **不负责**：投递记录管理（→ application）、匹配打分（→ match_engine）

## 主要入口

| 函数/端点 | 位置 | 说明 |
|---|---|---|
| `POST /api/cover-letter/generate` | `api/cover_letter.py:generate()` | 生成 3 个版本招呼语 |

## 核心服务

```python
# services/cover_letter_service.py
async def generate_cover_letters(job_id: UUID, user_id: UUID, style: str | None) -> CoverLetterResult:
    """
    1. 获取岗位信息 + 用户简历摘要
    2. 调用 ai_service.generate_cover_letters()
    3. 返回 3 个版本
    """
```

## 三种风格定义

| style | 字数 | 特点 | 适用场景 |
|---|---|---|---|
| `professional` | 80-120字 | 正式称呼、完整背景、专业表达 | 大厂/国企 |
| `concise` | 30-60字 | 直奔主题、核心亮点 | 互联网/创业公司 |
| `portfolio` | 50-80字 | 附带项目链接、强调成果 | 技术岗/设计岗 |

## 输入数据映射

```python
# 传给 AI 的数据结构
context = {
    "job_title": "算法工程师（校招）",
    "company_name": "字节跳动",
    "hr_name": "张经理",       # 可选
    "my_school": "清华大学",
    "my_major": "计算机科学与技术",
    "my_degree": "硕士",
    "my_highlights": [         # 从简历提取的 Top 3 亮点
        "Kaggle 竞赛排名前 5%",
        "推荐系统项目经验",
        "Python/PyTorch 熟练"
    ]
}
```

## 异常策略
| 场景 | 处理 |
|---|---|
| AI 服务不可用 | 返回 3 个模板化文案（`_fallback_cover_letters()`） |
| 简历信息不完整 | 用"应届毕业生"替代缺失字段 |
| HR 信息缺失 | 省略称呼，直接"您好" |

## 依赖模块
- **依赖**：auth（认证）、job（岗位信息）、resume（简历摘要）、ai_service（文案生成）
- **被依赖**：application（投递时可选择已生成的文案）
