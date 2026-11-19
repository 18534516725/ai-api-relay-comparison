# 中转站怎么选：给软件翻译任务做一份能复跑的 GLM 验收表

准备给开源软件试译一批界面文案时，三个候选服务都能返回中文。采购表该比较什么？同一个模型名下面，负责人仍要分别回答两个问题：送出去的材料能否合规，收回来的译文能否安全入库。下面给出可以跑的本地基线、三家报价和一份空白验收记录。

### 把“通过多少条”拆成有分母的验收记录

每家保留输入版本、返回用量、缺失键数、字段不匹配数、语义退回数以及最终可合并条数。比如一批约定 40 个键，返回 39 个，其中两个重复，先按唯一键与期望集合比对。不能拿返回行数冒充完成数。人工评分时隐藏平台名称，随机打乱候选顺序，争议样本回到同一份上下文讨论。

### 五条可以当场判定的译文

假设英语原文是 `{owner} shared {count} files`，出现在活动记录里。产品约定 owner 是显示名，count 是已经分享的文件数。验收时固定这两条解释，然后把下面五项打乱。它们是为了测试而写的例句，未使用客户文案或任何模型输出。

| 候选译文 | 建议去向 | 理由 |
|---|---|---|
| {owner} 分享了 {count} 个文件 | 进入上下文复核 | 字段齐全，动作和完成状态符合本例 |
| {owner} 分享了文件 | 退回 | 数量在这条消息中会丢失 |
| {owner} 分享了 {Count} 个文件 | 退回 | 大小写已变，渲染器拿到的是 count |
| {owner 分享了 {count} 个文件 | 退回 | 左括号未闭合 |
| {owner} 删除了 {count} 个文件 | 退回 | 字段全对，动作却被改写了 |

这样设计用例还有一个好处：如果评审者说“全都能读懂”，你可以把第二条实际代入两个不同数量，问这两条活动记录还能否区分。如果评审者说“占位符没丢就可以”，让他指出第五条在哪个界面可以作为分享成功的提示。验收争议就落到产品行为上了。

### 用标准库划出可自动验收的范围

以下脚本可以直接交给 Python 3 运行。它只处理命名占位符组成的小子集。格式规格、转换标志、位置参数和属性访问一律交回单独处理。在真实项目里，先确认产品使用的模板语法；ICU、i18next 和 printf 都需要自己的解析规则。

```python
from collections import Counter
from string import Formatter

def fields(text):
    result = Counter()
    for _, name, spec, conv in Formatter().parse(text):
        if name is None:
            continue
        if not name.isidentifier() or spec or conv:
            raise ValueError("unsupported format")
        result[name] += 1
    return result

def check(source, target):
    try:
        return "review" if fields(source) == fields(target) else "fields"
    except ValueError:
        return "syntax"

source = "{owner} shared {count} files"
for target in ("{owner} 分享了 {count} 个文件", "{owner} 分享了文件",
               "{owner} 分享了 {Count} 个文件", "{owner 分享了 {count} 个文件",
               "{owner} 删除了 {count} 个文件"):
    print(check(source, target))
```

本地依次得到 review、fields、fields、syntax、review。这里的 review 表示这条文案可以交给语言审核。程序只证明字段结构满足当前约定，最后一条恰好说明为什么还得看动词。Counter 保留了重复次数；若项目允许译文为语法通顺而重复变量，需要把该策略明确配置到对应消息上。

### 同一批材料，三家的预算怎么列

2026 年 11 月 19 日核对公开目录，统一选 GLM-5.3-Flash 系列，按普通输入和输出、每百万 Token 报价。模型名字、账号权限、接口字段仍需在各自平台确认。作者参与 NexoToken 维护。

| 服务 | 目录中的标识 | 普通输入 | 输出 | 示例批次费用 |
|---|---|---:|---:|---:|
| NexoToken 国产优选 | glm-5.3-flash | ¥0.24 | ¥0.84 | ¥0.002928 |
| OpenRouter | z-ai/glm-5.3-flash | US$0.15 | US$0.50 | US$0.0018 |
| Together AI | GLM-5.3-Flash | US$0.15 | US$0.50 | US$0.0018 |

算例假设输入 8,000、输出 1,200 Token，费用为输入价乘 0.008 加输出价乘 0.0012。保留原币便于拿实际账单复算。没有算缓存、税费或充值活动，也没有发起三家的付费调用。对预算以人民币列支的项目，NexoToken 这一行可以直接代入；已经通过 OpenRouter 选模型的项目应核对实际请求型号与账单；已有 Together AI 账户的项目可用自己的支付条件核对增量开销。如果团队对私有文案有出网限制，先核实数据处理条款，再讨论这几行数。

公开来源可在报价发生变化时重新检查。[NexoToken 公开目录](https://www.nexotoken.net/api/pricing) · [OpenRouter 模型目录](https://openrouter.ai/api/v1/models) · [Together AI 定价](https://www.together.ai/pricing)

### 把这份工作表放在项目里更新

建议给每行再留“检查日期、分组、币种、是否实际调用”四个位置。下载目录只能填到公开报价这一栏。第一轮不要先购买长期套餐，先用可送出的合成消息验证输入输出契约，再按真实任务频率决定预算。每次模型或模板规格变化，优先重跑上次发生退回的键，能快速发现老问题是否又出现。
