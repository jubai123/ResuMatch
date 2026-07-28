# order — 支付与订阅模块

## 职责边界
- **负责**：创建支付订单、处理支付回调、查询订阅状态、学生折扣管理
- **不负责**：用户账户管理（→ user）、支付渠道 SDK 对接（通过第三方支付网关）

## 主要入口

| 函数/端点 | 位置 | 说明 |
|---|---|---|
| `POST /api/order` | `api/order.py:create_order()` | 创建订单，返回支付链接/SDK参数 |
| `GET /api/order/{id}/status` | `api/order.py:get_order_status()` | 轮询订单支付状态 |
| `POST /api/order/{id}/pay-callback` | `api/order.py:pay_callback()` | 微信/支付宝回调（验签） |
| `GET /api/subscription` | `api/order.py:get_subscription()` | 获取当前订阅状态 |

## 套餐定义

| plan | 价格 | 月推送 | AI改写 | 其他 |
|---|---|---|---|---|
| `basic` | 49元/月 | 15条/周 | 10次/月 | AI招呼语 |
| `pro` | 99元/月 | 20条/周 | 30次/月 | +简历润色+薪资报告 |
| `premium` | 199元/月 | 25条/周 | 无限 | +1v1简历修改+导师答疑 |

学生认证后（`users.student_discount=TRUE`）价格打八折，`amount` 字段存实付金额。

## 支付流程

```
[App]──POST /api/order {plan, payMethod}
         │
         ├── 生成 order 记录（status=pending）
         ├── 调用支付网关创建预支付单
         └── 返回 {orderId, paymentUrl/payParams}

[用户完成支付]
         │
[支付网关]──POST /api/order/{id}/pay-callback {trade_no, sign}
         │
         ├── 验签（RSA/SM2）
         ├── 更新 order.status=paid, paid_at=NOW()
         ├── 更新 users.subscribed=TRUE, subscribe_expire_at=NOW()+30d
         └── 返回 success

[App 轮询]──GET /api/order/{id}/status → {status: "paid"}
```

## 核心服务

```python
# services/order_service.py
async def create_order(user_id, plan, pay_method) -> OrderCreateResult:
    """创建订单 + 调用支付网关"""

async def handle_pay_callback(order_id, trade_no, raw_data, signature) -> bool:
    """验签 + 更新订单 + 激活订阅"""

async def get_subscription(user_id) -> Subscription:
    """查询当前订阅状态，返回 plan + expireAt + autoRenew"""

async def check_and_expire_subscriptions() -> int:
    """Celery 定时任务：过期订阅自动降级"""
```

## 异常策略
| 场景 | 处理 |
|---|---|
| 支付回调验签失败 | 400，记录告警日志，不更新订单状态 |
| 重复支付回调 | 幂等处理（order.status 已为 paid 则直接返回成功） |
| 续费时旧订阅未过期 | 新订阅从旧 expire_at+1s 开始计算 |
| 支付网关超时 | 订单保持 pending，5 分钟后自动过期 |
| 订阅过期后使用付费功能 | API 层返回 402 `SUBSCRIPTION_REQUIRED` |

## 依赖模块
- **依赖**：auth（认证）、user（更新订阅字段）
- **被依赖**：无
