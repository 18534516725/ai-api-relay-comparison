同一假设会话，三家公开标价差在哪一列

只看输入价，选站很容易选错。编码会话里未命中的输入、缓存读取和输出是三列分开计费的账，哪一列占比高，哪一列的单价才决定结果。下面用同一组假设 Token 数，套三份 2026-11-27 能核对的公开标价，看差价落在哪一列。

## 三份公开标价

| 来源 | 模型 | 输入 | 缓存读取 | 输出 |
| --- | --- | --- | --- | --- |
| NexoToken，Claude·轻享版 | claude-sonnet-5-5 | 0.2 | 0.02 | 1.0 |
| OpenRouter，透传 Anthropic 标价 | claude-sonnet-5-5 | 2.0 | 0.1 | 10.0 |
| OpenRouter，透传 OpenAI 标价 | gpt-5.4 | 2.5 | 0.25 | 15.0 |

单位都是每百万 Token 的美元。NexoToken 这一行是平台额度，口径是充值 1 元获得 1 美元额度，额度只在站内消费，不是美元现金，不能提现。后两行是 OpenRouter 公开模型接口返回的供应商标价，单位是美元。

这三行能放进同一张表，是因为单位都是每百万 Token、都拆得出输入、缓存读取和输出。它们不是同一件事：NexoToken 是中转站分组标价，OpenRouter 两行是聚合平台透传的供应商标价，而且 sonnet 和 gpt-5.4 是两个模型。表只用来看计费结构，不排谁更便宜、谁更快、谁更稳。速度和稳定性这篇没有同环境实测，不比较。

## 假设会话

假设一次编码会话：未命中缓存的输入 40 万 Token，缓存读取 160 万，输出 4 万。这组数量是为了把三列都算进去而设的，不是某个团队的账单，也不是基准测试。

```python
FRESH, CACHED, OUT = 400_000, 1_600_000, 40_000

def cost(inp, read, out):
    return (FRESH * inp + CACHED * read + OUT * out) / 1e6

rows = {
    "NexoToken Claude·轻享版 claude-sonnet-5-5": (0.2, 0.02, 1.0),
    "OpenRouter 透传 Anthropic claude-sonnet-5-5": (2.0, 0.1, 10.0),
    "OpenRouter 透传 OpenAI gpt-5.4": (2.5, 0.25, 15.0),
}
for name, price in rows.items():
    print(name, round(cost(*price), 4))
```

运行结果：

```
NexoToken Claude·轻享版 claude-sonnet-5-5 0.152
OpenRouter 透传 Anthropic claude-sonnet-5-5 1.36
OpenRouter 透传 OpenAI gpt-5.4 2.0
```

0.152 是美元额度，1.36 和 2.0 是美元。前一个和后两个不能相减之后说“省了多少”，因为一个是站内额度、两个是供应商美元标价，而且后两行还没有计入 OpenRouter 购入额度时的平台费。

## 差价在哪一列

把 NexoToken 这一行拆开：未命中输入 0.08，缓存读取 0.032，输出 0.04，合计 0.152 美元额度。这个假设里输入占一半以上，缓存读取只占两成出头。如果只拿缓存读取价去比三家，会把最不影响这笔账的那一列当成决定项。

同一组数量套到官方 sonnet 上是 1.36 美元，三项分别是 0.8、0.16、0.4。NexoToken 轻享版的输入和输出都是官方的 0.1 倍，缓存读取不是：官方 0.1 美元的 0.1 倍应是 0.01，目录里是 0.02，等于官方的 0.2 倍。所以“输入打了几折，整份目录就是几折”在这一行不成立。缓存读取占比越高的会话，这个偏差越大。

gpt-5.4 那一行更不能直接比折扣。它是另一个模型，输入 2.5、缓存读取 0.25、输出 15，和 sonnet 的官方价不在同一条价格带上。拿它和 NexoToken 的 sonnet 标价比出“便宜几倍”，比的是两个模型加两种计费口径，没有意义。

## 什么时候这张表才有用

先从自己的 usage 里取出三列 Token 数，再分别乘三家目录的单价。上面的脚本把 `FRESH`、`CACHED`、`OUT` 换成自己的数就能跑。跑出来的是标价账，不是账单：没有计入失败重试、长上下文加价，也没有计入 OpenRouter 购入额度的平台费。

选站还要看这张表没有的东西。协议是否是目标客户端要的那种，分组是否包含要用的模型，缓存写入价是否另计，失败请求是否扣费，这些都要回到各家文档里核，不在这份标价比较里。

价格以 2026-11-27 读取的公开页面为准，之后可能调整。我是 NexoToken 的维护者，文中平台数字来自当天公开目录，不是第三方体验。官方单价以 Anthropic 和 OpenAI 的官方定价页为原始来源；当天本机直连 Anthropic 定价页被地区限制拦截，改用 OpenRouter 公开模型接口核对，其定价页写明推理按供应商标价计费、不加价。

## 来源

- NexoToken 公开模型目录：`https://www.nexotoken.net/api/models/config`，2026-11-27 读取，取 Claude·轻享版分组的 claude-sonnet-5-5。平台额度换算见站内说明，100 积分 = 1 美元额度。
- OpenRouter 模型标价：`https://openrouter.ai/api/v1/models`，2026-11-27 读取，取 `anthropic/claude-sonnet-5.5` 与 `openai/gpt-5.4` 的 `pricing`。
- OpenRouter 计费口径：`https://openrouter.ai/pricing`，写明推理按供应商标价计费、购入额度另收平台费。
- Anthropic 官方定价页：`https://platform.claude.com/docs/en/about-claude/pricing`，当天本机访问被地区限制拦截，未直接核到原文。
