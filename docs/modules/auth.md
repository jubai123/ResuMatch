# auth — 认证模块

## 职责边界
- **负责**：短信验证码发送、手机号登录、JWT 签发与刷新、Token 验证中间件
- **不负责**：用户信息管理（→ user 模块）、权限控制（MVP 仅区分登录/未登录）

## 主要入口

| 函数/端点 | 位置 | 说明 |
|---|---|---|
| `POST /api/auth/sms` | `api/auth.py:send_sms()` | 发送验证码，60s 频率限制 |
| `POST /api/auth/login` | `api/auth.py:login()` | 手机号+验证码→JWT |
| `POST /api/auth/refresh` | `api/auth.py:refresh_token()` | 用 refreshToken 换新 accessToken |
| `get_current_user()` | `utils/jwt.py` | FastAPI Depends，所有需登录端点的依赖 |

## JWT 设计
- **accessToken**：有效期 24h，payload: `{sub: user_id, phone_masked, iat, exp}`
- **refreshToken**：有效期 30d，存 Redis `refresh:{user_id}`，一次性（刷新后旧 token 失效）
- 签发：`utils/jwt.py:create_access_token()` / `create_refresh_token()`

## 异常策略
| 场景 | HTTP 状态 | 错误码 | 处理方式 |
|---|---|---|---|
| 验证码过期/错误 | 401 | `INVALID_CODE` | 返回 retryAfter |
| Token 过期 | 401 | `TOKEN_EXPIRED` | 前端自动调 refresh |
| Token 无效 | 401 | `INVALID_TOKEN` | 前端跳转登录页 |
| 验证码发送频率超限 | 429 | `SMS_RATE_LIMITED` | 返回 retryAfter |
| 手机号格式错误 | 422 | — | Pydantic 校验 |

## 依赖模块
- **被依赖**：所有需要认证的模块通过 `get_current_user` 依赖注入

## 安全注意
- 验证码 5 分钟内有效，验证后立即删除
- 手机号 SHA-256 + salt 存储，日志中不打印明文
- 登录失败连续 5 次，锁定 IP 30 分钟
