# 中转站怎么选：给会议时间提取做一套三家 Qwen 接口验收题

日程助手最容易演示的情况是“明天下午三点开会”。它能返回一个规整的时间戳，往往给人已经可以上线的印象。换个样本：组织者连日期和城市都写好了，可那个地方的钟会把同一个时间显示两次。下面提供本地真值、试用记录方法和公开报价。

### 采购试用先约定哪一层负责什么

给三个候选相同的脱敏文本，模型只提取事实。用固定时区数据解释候选，把提取错日期、补造时区、吞掉追问、误发邀请分别计数。一个服务答出正确 UTC 却没有保留原文证据，如果碰巧猜中，同样很难投入生产。评审时先看证据，再看结果，最后把供应商名字揭开。

### 三封只差日期的邀请

假设三个组织者都在纽约，都明确用了当地时间，日期也都写全。下面用的是合成文本，适合拿来测日程助手。它们特意排除了“下周二”“我这里”等语言歧义，以便把日历本身的问题单独测出来。

| 编号 | 消息里的开始时间 | 应当交付什么 |
|---|---|---|
| S1 | 2026 年 3 月 8 日纽约 02:30 | 提醒该墙钟时间不存在，请组织者重选 |
| S2 | 2026 年 11 月 1 日纽约 01:30 | 展示两种可能，由组织者选择 |
| S3 | 2026 年 11 月 2 日纽约 01:30 | 返回唯一候选，再按产品流程确认 |

纽约在春季调快时钟，S1 跳过了；秋季拨回时钟，S2 经过两次。S2 对应的 UTC 分别是 05:30 和 06:30，相差整整一小时。到 S3，唯一结果是 11 月 2 日 06:30 UTC。把 S2 发给两个参会人，如果各自的客户端碰巧采用了不同默认值，双方本地都显示“01:30”，却可能根本没有约在同一时刻。

日期、时间、地区三个字段都提取正确，只代表模型读懂了文字。直到查完时区规则，应用才知道自己有几个可选时刻。这是可以离线核对的边界。

### 用标准库做一次往返检查

示例适用于 Python 3.9 及以上，环境要能找到 IANA 时区数据。它只接收无偏移的本地日期时间以及完整地区名。Python 的 fold 用来区分重复时段，给本地时间附加 tzinfo 本身不会替你拒绝春季跳过的那一小时。

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

def candidates(local_text, zone_name):
    local = datetime.fromisoformat(local_text)
    if local.tzinfo is not None:
        raise ValueError("expected local time without offset")
    zone = ZoneInfo(zone_name)
    result = set()
    for fold in (0, 1):
        attached = local.replace(tzinfo=zone, fold=fold)
        utc = attached.astimezone(timezone.utc)
        back = utc.astimezone(zone).replace(tzinfo=None)
        if back == local:
            result.add(utc.isoformat())
    return sorted(result)

for local in ("2026-03-08T02:30", "2026-11-01T01:30",
              "2026-11-02T01:30"):
    hits = candidates(local, "America/New_York")
    print(local, len(hits), *hits)
```

本地执行得到的候选数量依次为 0、2、1。关键是先换到 UTC，再从 UTC 换回纽约，检查墙钟读数是否和输入一致。普通时刻尝试两个 fold 会落到同一个 UTC，集合会去重；重复时刻会留下两个；不存在的时间两个都不能原样返回。数据源缺失或时区名拼错应当单独报错，这与“确实查到了零个时刻”是两种不同的问题。

### 三家公开接口在同一批输入下的报价

以下为 2026 年 11 月 22 日核对的 Qwen3.8 Flash 普通文本报价。作者参与 NexoToken 维护。各家统一按 6,000 普通输入 Token、1,000 输出 Token 计算。时区库、日历写入和人工追问另行核算。

| 平台与目录型号 | 每百万输入 | 每百万输出 | 本批模型费 |
|---|---:|---:|---:|
| NexoToken 国产优选，qwen3.8-flash | ¥0.24 | ¥0.81 | ¥0.00225 |
| OpenRouter，qwen/qwen3.8-flash | US$0.15 | US$0.47 | US$0.00137 |
| Together AI，Qwen3.8 Flash | US$0.15 | US$0.47 | US$0.00137 |

用输入单价乘 0.006，再加输出单价乘 0.001。金额保留原币，不含缓存、汇兑和手续费。用人民币管理预算的团队可以直接代入 NexoToken 这行；已有 OpenRouter 项目时，保留实际请求型号与账户账单核对；已经在 Together AI 做数据审批的团队，可以先用原有结算条件估增量成本。两行美元标价一致，仍需分别确认账号可用型号、日志条款及返回结构。本次未做付费速度或质量横评。

重新核对时可看各家公开来源。[NexoToken 公开目录](https://www.nexotoken.net/api/pricing) · [OpenRouter 模型目录](https://openrouter.ai/api/v1/models) · [Together AI 定价表](https://www.together.ai/pricing)

### 留一张能在下次调价时继续用的表

每家各记当前型号、普通输入输出用量、候选字段通过数、追问是否必要、人工修订分钟数及付费币种。产品只承担建议功能时，重点衡量提取质量；能实际发邀请时，再加单次写入和撤销演练。把公开价格和实跑字段分开，未来即使只更新了目录，也不会把旧的测试结论误写成新验证。
