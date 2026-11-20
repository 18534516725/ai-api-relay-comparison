# 中转站怎么选：用四段教程字幕验收三家 Qwen 文本接口

给视频教程修字幕，最省事的演示是粘入一段转写稿，让模型润色。真正交付的时候还要把每句话贴回时间轴。能否正确回填，数字有没有被改动，决定了候选服务能不能进入剪辑流程。这里给出可重建的离线样本、实际跑过的代码，以及按同一任务口径整理的采购表。

### 给每家留一张空白试用单

输入媒体版本、字幕版本、片段数、允许的编辑类型和预计预算先填好；返回后再填实际型号、输入输出用量、丢行数、错改参数数、需要回听的行数。缺项留空。付费试用时三家使用同一份合成材料，先遮住平台名字审核文字，再合并到采购记录里。这样某个答案更像编辑的个人文风，也不会左右参数正确性的判断。

### 做一段八秒四的测试片

下面是一份合成的教程字幕。时间统一用从片头起算的整数毫秒表示；把文本和时钟分别存起来，之后才知道是谁改了哪一项。实操时可以自己读这四句话录一小段音频，也可以直接使用数据做离线测试。本文实际运行的是后者。

| 片段 ID | 起点 | 终点 | 原句 |
|---|---:|---:|---|
| a | 0 | 1800 | 先保留原文件。 |
| b | 1800 | 4200 | 把等待时间设成 15 秒。 |
| c | 4600 | 6900 | 不要覆盖输出目录。 |
| d | 6900 | 8400 | 现在可以预览。 |

为什么在 b 和 c 之间故意空出 400 毫秒？编辑器可能希望把几行接得更顺，自动让上一句的终点贴到下一句的起点。如果业务只是改错字，这种“顺手修复”已经改变了成片。把停顿放进样本，就能立刻察觉。另一个小设计是 b 有一个操作参数，c 有一个否定词；两句话都可以被改得非常流畅，同时彻底偏离讲者的意思。

### 一段能离线运行的 Python 检查

把下面内容存为脚本，用 Python 3 执行。它接收的只是解析后的纯文本片段。复杂字幕文件里的样式、位置、说话人和轨道设置仍要由支持具体格式的读写器保留。

```python
from collections import Counter

cues = [
    ("a", 0, 1800, "先保留原文件。"),
    ("b", 1800, 4200, "把等待时间设成 15 秒。"),
    ("c", 4600, 6900, "不要覆盖输出目录。"),
    ("d", 6900, 8400, "现在可以预览。"),
]

def rebuild(cues, edits):
    ids = [row[0] for row in cues]
    if Counter(ids) != Counter(k for k, _ in edits):
        raise ValueError("id mismatch")
    if len(set(ids)) != len(ids):
        raise ValueError("duplicate source")
    text = dict(edits)
    if any(not isinstance(s, str) or not s.strip() for s in text.values()):
        raise ValueError("empty text")
    return [(k, start, end, text[k]) for k, start, end, _ in cues]

edits = [(k, s) for k, _, _, s in reversed(cues)]
assert rebuild(cues, edits) == cues
try:
    rebuild(cues, edits[:-1] + edits[:1])
except ValueError as e:
    print(str(e))
else:
    raise AssertionError("duplicate accepted")
changed = [(k, s.replace("15", "50")) for k, s in edits]
print(rebuild(cues, changed)[1])
```

第一行输出 id mismatch：把末尾结果替换成首条，列表长度照旧，重复和遗漏同时发生。第二行仍会输出带“50 秒”的 b。这个结果说明结构检查只能把答案安放到正确位置，语义上的参数篡改还要靠另一道复核。第一条断言还验证了一件小事：倒序返回四个正确片段可以无损恢复。

### 三家公开报价放在一张表里

2026 年 11 月 20 日查询 Qwen3.8 Flash 普通文本按量报价。统一假设每批 10,000 普通输入 Token、2,000 输出 Token，按公开币种算一遍。作者参与 NexoToken 维护。这里只计算已经拿到文字后的编辑请求。

| 平台与目录型号 | 每百万普通输入 | 每百万输出 | 这一批的模型费 |
|---|---:|---:|---:|
| NexoToken 国产优选，qwen3.8-flash | ¥0.24 | ¥0.81 | ¥0.00402 |
| OpenRouter，qwen/qwen3.8-flash | US$0.15 | US$0.47 | US$0.00244 |
| Together AI，Qwen3.8 Flash | US$0.15 | US$0.47 | US$0.00244 |

把输入价乘 0.01、输出价乘 0.002 后相加即可复算。没有折算汇率、加入税费或假设缓存命中。按人民币记项目预算时，NexoToken 的这行可以直接使用。通过 OpenRouter 动态选模型时，保留实际请求型号来对账；已经在 Together AI 管理账户和用量时，核对现有条件下新增该任务的开销。两行美元报价相同，账户权限、支付条件和实际响应仍得分别验证。

公开页面可用来更新这张表。[NexoToken 公开目录](https://www.nexotoken.net/api/pricing) · [OpenRouter 模型目录](https://openrouter.ai/api/v1/models) · [Together AI 定价](https://www.together.ai/pricing)

### 什么时候值得切换

把可交付的一期教程作为分母。如果换一家后要多花十分钟人工重新对轴，即使账单上的单次模型费更低，项目整体也可能多花钱。采购人可以拿现有供应商做基线，保留一个替代选项；等目标型号、账单和回填三项都能核对，再决定迁移范围。
