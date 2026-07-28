# TEST_GUIDE.md — 编码与测试规范

> **用途**：AI 在生成业务代码的同时，自动补全符合团队规范的测试用例。

---

## 1. 测试框架

| 层 | 框架 | 说明 |
|---|---|---|
| 后端单元测试 | pytest + pytest-asyncio | async 测试原生支持 |
| 后端 API 测试 | httpx (ASGI transport) | 不启动真实服务器 |
| 数据库测试 | PostgreSQL test DB / SQLite in-memory | 开发用 SQLite，CI 用 PostgreSQL |
| Mock | pytest-mock + respx | 外部 API (Claude/SMS/支付) 一律 mock |
| 前端单元测试 | Vitest + @vue/test-utils | uni-app 条件编译代码需单独处理 |
| 覆盖率 | coverage.py (后端) / c8 (前端) | CI 门禁 |

---

## 2. 测试命名规则

```
测试文件：test_{module_name}.py          # 如 test_match_service.py
测试类：  Test{ClassName}                 # 如 TestMatchScoreCalculator
测试函数：test_{method}_{scenario}        # 如 test_calculate_score_with_full_match
```

**命名必须描述场景**，禁止 `test_1`、`test_foo` 等无意义名称。

---

## 3. 测试目录结构

```
app-backend/
├── tests/
│   ├── conftest.py              # 全局 fixture（DB会话、client、mock用户）
│   ├── test_auth.py             # 认证模块
│   ├── test_resume.py           # 简历模块
│   ├── test_match_engine.py     # 匹配引擎（核心，覆盖率>90%）
│   ├── test_job.py
│   ├── test_application.py
│   ├── test_order.py
│   ├── test_ai_service.py       # AI 服务（含 mock）
│   └── factories/               # 测试数据工厂
│       ├── user_factory.py
│       ├── resume_factory.py
│       ├── job_factory.py
│       └── match_factory.py

app-frontend/
├── src/
│   └── __tests__/
│       ├── components/
│       └── composables/
```

---

## 4. 覆盖率要求

| 层级 | 最低覆盖率 | 说明 |
|---|---|---|
| `services/match_service.py` | ≥ 90% | 核心业务逻辑 |
| `services/ai_service.py` | ≥ 85% | 含 mock 降级路径 |
| 其他 `services/*.py` | ≥ 80% | |
| `api/*.py` (路由层) | ≥ 70% | 参数校验 + 状态码 |
| `models/*.py` | ≥ 60% | ORM 模型无需高覆盖 |
| 前端 `composables/` | ≥ 80% | 可复用逻辑 |
| 前端 `components/` | ≥ 60% | UI 组件 |

**CI 门禁**：整体覆盖率 < 75% 时禁止合并。

---

## 5. 测试样板

### 5.1 后端 Service 测试

```python
import pytest
from app.services.match_service import calculate_match_score

@pytest.mark.asyncio
async def test_calculate_score_with_full_match(resume_factory, job_factory, preference_factory):
    """当简历技能与岗位要求完全匹配时，应返回 90 分以上"""
    resume = resume_factory(skills=["Python", "PyTorch", "推荐系统"])
    job = job_factory(title="算法工程师", skills=["Python", "PyTorch"])
    preference = preference_factory(cities=["北京"])

    score = await calculate_match_score(resume, job, preference)

    assert score >= 90

@pytest.mark.asyncio
async def test_calculate_score_blacklist_company_returns_zero(resume_factory, job_factory, preference_factory):
    """黑名单公司的岗位匹配分应为 0"""
    preference = preference_factory(exclude_companies=["某公司"])
    job = job_factory(company_name="某公司")

    score = await calculate_match_score(resume_factory(), job, preference)

    assert score == 0
```

### 5.2 后端 API 测试（不启动服务器）

```python
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.mark.asyncio
async def test_login_with_valid_code(mock_sms):
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.post("/api/auth/login", json={
            "phone": "13800138000",
            "code": "123456"
        })
    assert resp.status_code == 200
    assert "accessToken" in resp.json()
```

### 5.3 AI Mock

```python
# conftest.py 全局 fixture
@pytest.fixture
def mock_claude(respx_mock):
    """Mock Claude API 返回固定响应"""
    respx_mock.post("https://api.anthropic.com/v1/messages").mock(
        return_value=httpx.Response(200, json={
            "content": [{"text": '{"suggested": "...", "reason": "..."}' }]
        })
    )
```

### 5.4 前端组件测试

```typescript
// __tests__/components/JobCard.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import JobCard from '@/components/JobCard.vue'

describe('JobCard', () => {
  it('renders match score with color coding', () => {
    const wrapper = mount(JobCard, {
      props: { job: { title: '算法工程师', score: 92 } }
    })
    expect(wrapper.text()).toContain('92')
    expect(wrapper.find('.score-high').exists()).toBe(true)
  })
})
```

---

## 6. Factories（测试数据工厂）

```python
# tests/factories/job_factory.py
import factory
from app.models.job import Job

class JobFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = Job

    id = factory.Faker("uuid4")
    title = factory.Faker("job")
    company_name = factory.Faker("company")
    city = factory.Iterator(["北京", "上海", "深圳", "杭州"])
    salary_min = factory.Faker("random_int", min=10, max=40)
    salary_max = factory.LazyAttribute(lambda o: o.salary_min + factory.Faker("random_int", min=5, max=15).generate({}))
    degree = "master"
    status = "active"
```

---

## 7. 禁止事项

- 禁止在测试中使用 `time.sleep()` 等待异步结果
- 禁止测试依赖执行顺序（每个测试独立）
- 禁止访问真实的第三方 API（CI 无网络仍通过）
- 禁止 0 断言的测试（必须有显式 assert）
- 禁止提交使覆盖率下降的代码

---

## 8. 运行命令

```bash
# 后端
cd app-backend
pytest -s -v --cov=app --cov-report=term-missing

# 前端
cd app-frontend
npx vitest run --coverage
```
