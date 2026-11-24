# 中转站怎么选：用四份讲义验收三家 Qwen 文件命名接口

假设你为课程资料库接一个 AI 命名助手。第一次演示里四个名字都很顺，维护者却得回答更具体的问题：这四条建议能不能对应回四份原件，会不会和目录里的另一份讲义撞名，有没有凭空编出日期？下面给出可重建的离线真值和采购工作表。

### 先把接口需要答的题和程序要查的事写清

模型试题包括保留 ID、按材料起名、没有版本依据时主动留空；程序试题包括检测字符表示差异、大小写冲突以及已有路径占用。把同一份合成描述发给每家，不要求模型猜原始文件字节。若评测人发现一个“重复”结论，要追问它是在说文字语义、目标名还是内容。三者的证据完全不同。

### 一份专门让名字和内容“唱反调”的样本

想象有四份讲义附件，都准备归到同一目录。F01 等编号只是本轮工作清单里的稳定标识。为方便复核，文件内容缩短为几个 ASCII 字节，候选名也是人为设定。所有输入都可以在内存里创建，无需在真实目录里做重命名。

| ID | 文件字节 | 建议的新名字 |
|---|---|---|
| F01 | rev=7 | Café.txt，é 为单个字符 U+00E9 |
| F02 | rev=8 | Café.txt，末尾重音字母由 e 和 U+0301 组合 |
| F03 | rev=7 | Archive.txt |
| F04 | rev=9 | archive.TXT |

仅看第一列名字，F01 和 F02 在许多界面里几乎分辨不出，但打开内容看到的是两个版本。F03、F04 的区别更明显，只是大小写；在本例定义的命名策略下也会碰撞。真正字节相同的是 F01、F03，它们的建议名反而差得很远。审稿时可以把最后一栏盖住，先按内容分组，再盖住中间一栏按命名策略分组。两张清单之间的错位，正是产品需要处理的情况。

### 离线复现，两条断言分别管一件事

下面使用 Python 3 标准库。此处约定同一目标目录内，候选名经过 NFC 规范化、再做大小写折叠后不允许重复。这是演练项目主动选择的规则；具体文件系统、网络盘和同步服务的比较规则需要分别验证。

```python
from collections import defaultdict
from hashlib import sha256
from unicodedata import normalize

# ID, original bytes, proposed display name
samples = [
    ("F01", b"rev=7", "Caf\u00e9.txt"),
    ("F02", b"rev=8", "Cafe\u0301.txt"),
    ("F03", b"rev=7", "Archive.txt"),
    ("F04", b"rev=9", "archive.TXT"),
]
names, digests = defaultdict(list), defaultdict(list)
for ident, payload, name in samples:
    names[normalize("NFC", name).casefold()].append(ident)
    digests[sha256(payload).hexdigest()].append(ident)
collisions = sorted(v for v in names.values() if len(v) > 1)
duplicates = sorted(v for v in digests.values() if len(v) > 1)
assert collisions == [["F01", "F02"], ["F03", "F04"]]
assert duplicates == [["F01", "F03"]]
print("name:", collisions)
print("bytes:", duplicates)
```

本地执行后，name 两组是 F01/F02 和 F03/F04，bytes 一组是 F01/F03。摘要只覆盖本例给定的字节。真实文件很大时可流式读取；涉及删除等高影响动作，还要复核内容、权限和用途。即使两个文件逐字节相同，归档人也可能因为它们分别属于不同课程或合同而需要保留两份。脚本只打印两份清单，没有写入文件。

### 采购表先固定任务量和计费单位

2026 年 11 月 24 日核对三家 Qwen3.8 Flash 文本目录。统一按普通输入 8,000 Token、输出 600 Token 估算一批候选名。作者参与 NexoToken 维护。字符数无法直接当 Token 数，后面要换成自己的真实请求用量。

| 平台与目录标识 | 每百万普通输入 | 每百万输出 | 示例批次费用 |
|---|---:|---:|---:|
| NexoToken 国产优选，qwen3.8-flash | ¥0.24 | ¥0.81 | ¥0.002406 |
| OpenRouter，qwen/qwen3.8-flash | US$0.15 | US$0.47 | US$0.001482 |
| Together AI，Qwen3.8 Flash | US$0.15 | US$0.47 | US$0.001482 |

末列用输入单价乘 0.008，加上输出单价乘 0.0006。人民币和美元分别保留，排除缓存、手续费和税费。人民币预算可直接引用 NexoToken 这组数；已经用 OpenRouter 管理多模型的团队，应按实际选用的型号与路由对账；已有 Together AI 组织账户的团队，先核对已有数据处理约定和增量费用。后两行公开标价相同，也不能由此推断两家会给出同样的建议。这里没有做付费调用或质量排名。

更新报价可复查三个公开来源。[NexoToken 模型目录](https://www.nexotoken.net/api/pricing) · [OpenRouter 模型目录](https://openrouter.ai/api/v1/models) · [Together AI 定价](https://www.together.ai/pricing)

### 采购记录如何落笔

每家填上实际模型字段、请求用量、缺失 ID 数、凭空添加信息的条数、需要人工改名的条数。另记工程投入：能否在现有账号下运行，故障时能否回读结果，团队是否允许把对应描述交给该服务。待核实的项目留空。适合自己的接口，应当能兑现这份任务契约；不能把离线脚本跑通当成三家模型都已通过试用。
