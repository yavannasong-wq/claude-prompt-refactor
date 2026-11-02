# 骨架、标签命名与完整实例

## 一、标准骨架

```xml
<role>          身份 + 职责 + 产出给谁看
<task>          本次要完成什么 + "完成"的定义
<instructions>  怎么想、怎么做（与具体数据无关的通用逻辑）
<policy>        业务硬规则、合规条款、优先级声明
<tone>          语气、长度、措辞禁忌
<documents>     检索/给定的参考资料（文章、手册、规范）
<customer_data> 本次的结构化数据（账户、台账、原始记录）
<output_format> 输出结构；可含 <reasoning> + <reply>/<report>
```

按任务类型取舍，不必全用。四要素最小集（从零写新 Prompt 时）：**角色 / 数据 / 任务+完成标准 / 输出形状**。

## 二、标签命名表

| 类别 | 推荐标签 | 避免 |
|---|---|---|
| 角色 | `<role>` `<persona>` | `<system>`（含义太泛） |
| 任务 | `<task>` `<objective>` | `<goal>`（易与风格目标混淆） |
| 规则 | `<instructions>` `<guidelines>` | `<rules>` 与 `<guidelines>` 混用 |
| 政策 | `<policy>` `<compliance>` | 把政策塞进 `<instructions>` |
| 数据 | `<documents>` `<customer_data>` `<excel_records>` `<context>` | `<data>`（无信息量） |
| 语气 | `<tone>` `<style>` | `<output>`（与格式混淆） |
| 格式 | `<output_format>` `<schema>` | `<format>`（易与文档格式混淆） |
| 用户输入 | `<user_message>` `<question>` | `<input>`（含混） |

命名原则：**外人只看标签名就知道里面装的是什么**。多份数据并存时，用具体名词（`<customer_account>`、`<policy_doc>`），并在 `<policy>` 或 `<instructions>` 里写明优先级。

## 三、实例 A：客服问答（改前 / 改后）

**改前**

```text
你是一名有经验的人类客服，为 Meridian Mobile 工作。忘掉你关于 Meridian
Mobile 和无线套餐的一切知识，那都是过时的。只使用下面的资料。以下帮助中心
内容可能和客户问题相关，也可能只是主题相似，所以除非明确适用否则请忽略。
我们最近改了套餐，下面文章展示的是当前套餐数据，老套餐客户费率不同。
绝不要给客户错误的套餐信息，而是引导他们去 https://meridian.com/myaccount
自己查看。本文由 Meridian Care Content 团队维护。相关文章：国际漫游费率、
计费周期如何运作、自带设备。这篇文章对你有帮助吗？是/否，返回顶部。
本站使用 cookie……（正文）……Unlimited 套餐含 4GB 热点。
客户：postpaid，套餐 Unlimited Choice，状态 grandfathered，热点 5GB。
请简洁回答，一句话，别用 emoji，别说 contact support。
**CRITICAL：务必正确计算任何按比例金额！**
```

**问题**：角色不真实（"人类客服"）；资料混入 cookie 政策与相关链接；规则/政策/数据/语气缠在一起；旧护栏（"引导去官网"）与正确数据（5GB）冲突，模型选择护栏 → 答"4GB，你去官网查"（**信息隐瞒**）。

**改后**

```xml
<role>
你是 Meridian Mobile 的资深客服，负责回答客户关于套餐、账单与账户的问题。
任务：基于下方资料，直接、准确地回答客户问题。
完成标准：客户无需再自行查阅即可得到准确的答案。
</role>

<instructions>
1. 先判断问题属于哪一类（套餐信息 / 账单计算 / 争议处理）。
2. 只使用 <documents> 与 <customer_account> 中的信息作答。
3. 资料中不相关的内容（页脚、相关推荐、cookie 提示）一律忽略。
4. 客户账户记录中的套餐与配额，是该客户实际持有的信息，优先采信。
</instructions>

<policy>
- <documents> 中的套餐说明只适用于"当前在售"套餐。
- 老套餐（grandfathered）客户的额度以账户记录为准，可能与当前套餐不同。
- 涉及扣费争议时，转接 Care Specialist，不要自行解释或处理。
</policy>

<tone>
直接、简洁、口语化。不用 emoji，不堆砌客套。
不要引导客户自己到网站查询——你应当直接给出答案。
</tone>

<documents>
<help_article title="Unlimited 套餐说明">
Unlimited 套餐当前含 4GB 移动热点。……（只保留正文）
</help_article>
</documents>

<customer_account>
plan_type: postpaid
current_plan: Unlimited Choice
plan_status: grandfathered
hotspot: 5GB
</customer_account>

<output_format>
先给出直接答案，再给一句话依据。整体不超过两句话。
</output_format>

<user_message>
Unlimited 套餐有多少热点流量？
</user_message>
```

## 四、实例 B：数据报表（源数据字段不全时的处理）

需求：根据 Excel 生成设备分析报告（停机次数、停机时间、设备产量），但表中只有 `日期/设备名称/运行状态/运行时长(小时)/故障次数`。

关键处理：

1. **停机时间口径歧义**（24h 还是计划工时）→ 单列 `<shift_config>` 让用户填，不替用户假设；
2. **产量字段缺失** → 在 `<metric_definitions>` 里明确"不计算、标注未提供、列出应补字段"，从源头防编造；
3. **状态词不统一**（格式说明写 `故障`，数据出现 `故障报警`、`运行异常`）→ 加 `<status_vocabulary>` 映射规则，并要求在"数据质量"一节列出原词频次；
4. **输出固定章节** → 概览表 / 重点设备 / 状态分布 / 趋势异常 / 数据质量 / 结论建议；
5. 每个关键数字要求注明来源行，便于人工抽查。

```xml
<metric_definitions>
<停机次数>口径：每台设备各行的"故障次数"累加。</停机次数>
<停机时间>
口径：停机时间 = 每日计划可用时长 − 当日运行时长，按设备逐日计算后累加。
不得使用 24 小时作为扣减基数。
</停机时间>
<设备产量>
口径：数据表无产量列时，不计产量数字；标注"产量数据未提供"，
并列出为补充应增加的列（如"产量(件)"）。
</设备产量>
</metric_definitions>

<output_format>
严格按以下结构输出，只输出报告本身：
# 设备分析报告｜{{日期}}
## 一、数据概览（含汇总表）
## 二、重点设备（停机次数 Top3 / 停机时间 Top3）
## 三、运行状态分布
## 四、趋势与异常
## 五、数据质量说明
## 六、结论与建议
<reasoning>在正文前写统计过程与判断依据（不超过 5 条），供核对。</reasoning>
</output_format>
```

## 五、把输出做成"可切分的"

输出契约的标签不只给模型读，还能给代码切：

```python
msg = client.messages.create(
    model="claude-sonnet-4-6",
    stop=["</reply>"],           # 生成到闭合标签即停
    messages=[{"role": "user", "content": prompt}],
)
text = msg.content[0].text
reasoning = re.search(r"<reasoning>(.*?)</reasoning>", text, re.S)
reply = re.search(r"<reply>(.*?)</reply>", text, re.S)
```

好处：`<reasoning>` 留作调试、`<reply>` 给用户，二者用正则即可分离。
