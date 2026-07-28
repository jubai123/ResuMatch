# push — 推送通知模块

## 职责边界
- **负责**：设备 Token 注册、推送通知调度、触达率统计
- **不负责**：推送内容生成（→ match_engine）、消息模板管理（各调用方自行拼接）

## 主要入口

| 函数/端点 | 位置 | 说明 |
|---|---|---|
| `POST /api/notification/register-device` | `api/.../notification.py:register_device()` | 注册设备推送 Token |

## 核心服务

```python
# tasks/notification.py (Celery 任务)
async def send_weekly_push(user_id: UUID, batch_id: UUID):
    """发送每周一推送通知。由 match_engine 生成批次后触发"""

async def send_single_push(user_id: UUID, title: str, body: str, data: dict):
    """发送单条推送通知（关键岗位实时推送等）"""
```

## 推送消息格式

```json
{
  "title": "本周为你匹配 18 条岗位",
  "body": "字节跳动·算法工程师 92分 | 腾讯·产品经理 89分 | 更多→",
  "data": {
    "type": "weekly_push",
    "batchId": "uuid",
    "url": "/pages/match/weekly"
  },
  "apns": {
    "sound": "default",
    "badge": 18
  },
  "android": {
    "channel_id": "weekly_push",
    "priority": "high"
  }
}
```

## 推送策略

| 场景 | 触发时机 | 通道 | 频率限制 |
|---|---|---|---|
| 周推送 | 每周一 09:00 | FCM/APNs | 1次/周 |
| 关键岗位加推 | 每日 10:00 | FCM/APNs | 最多 1次/天 |
| 投递提醒 | "想投"14天未动 | 本地通知 | — |
| 订阅到期提醒 | 到期前 3 天 | FCM/APNs + 短信 | 1次 |

## 推送文案规则

1. 必须含具体公司名 + 匹配分（提高打开率——用户研究结论）
2. iOS 通知正文 ≤ 110 字符
3. 点击跳转 deep link 到对应页面

## 异常策略
| 场景 | 处理 |
|---|---|
| Token 失效（APNs 410 / FCM UNREGISTERED） | 标记失效，下次推送跳过 |
| 推送平台服务不可用 | 记录告警，降级为仅本地通知 |
| 用户关闭通知权限 | 静默跳过，不影响其他功能 |

## 依赖模块
- **依赖**：user（用户设备关联）
- **被依赖**：match_engine（推送触发）、order（到期提醒）
