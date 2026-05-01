---
name: tyc-credit-rating
description: 企业信用评级 — 多维度信用评分 + 授信额度与利率建议，适用银行信贷 / 供应链金融 / 保理
category: invest
version: 1.0
---

# 企业信用评级

## 触发条件

银行对公信贷评审、供应链金融白名单准入、保理公司授信、ToB 赊销准入时触发。

关键词：信用评级、授信额度、贷后预警、AA 评级、保理、赊销

## 输入要求

用户输入的企业标识可能是以下三种形式之一，**Step 0 会按形式分流**，不要急着把用户原话当成 `searchKey` 喂给后续步骤：

| 输入形式 | 示例 | 是否需要 L0 实体锚定 |
|---|---|---|
| **完整企业名**（含组织形式后缀：`有限公司` / `股份有限公司` / `集团` / `合伙企业` / `个体工商户` / `事务所` / `中心` 等） | `北京字节跳动科技有限公司` / `腾讯科技（深圳）有限公司` | ❌ 跳过 L0，直接当 `searchKey` |
| **统一社会信用代码（USCC）** 18 位大写字母+数字 | `91110108551385082Q` | ❌ 跳过 L0，直接当 `searchKey` |
| **企业简称 / 曾用名 / 品牌名 / 模糊指代** | `字节` / `抖音` / `今日头条` / `乐视` / `阿里` | ✅ **必须先走 L0**，由用户在候选中确认唯一企业，再拿 `creditCode` 作为 `searchKey` |

> 经过 Step 0 锚定后，下文 Step 1+ 的 `searchKey` 都指**精确企业名或 USCC**，调用方式不变。

## 执行流程

### Step 0: 实体锚定（条件性 · L0 工具：`search_companies`）

**目的**：用户给的常常是简称、曾用名、模糊指代，下游所有步骤都依赖精确企业；先用 L0 把它消歧成唯一 `creditCode`，避免后续步骤在错主体上反复烧 token。

**判定与分流**：

1. **若 userInput 匹配 USCC 正则** `^[0-9A-Z]{18}$` → 跳过 L0，`searchKey = userInput`
2. **若 userInput 含组织形式后缀**（`有限公司` / `股份` / `集团` / `合伙企业` / `事务所` / `个体工商户` / `分公司` 等）**且长度 ≥ 6** → 跳过 L0，`searchKey = userInput`
3. **否则**（简称 / 曾用名 / 品牌名 / 任何看起来不完整的字符串）→ **必须走 L0**：

   - 调用 `search_companies` (`searchKey: userInput`)
   - 在返回 `items[]` 中默认按 `regStatus ∈ {存续, 在业, 在营, 开业}` 过滤，按 `regCapital` 倒序取 Top 5 作为候选展示
   - **候选 = 1** → 自动锚定，`searchKey = items[0].creditCode`
   - **候选 ≥ 2** → **暂停执行**，向用户输出候选清单（`name` / `creditCode` / `regStatus` / `regLocation` / `legalPersonName`）请求确认，待回复后取选定企业的 `creditCode` 作为 `searchKey`
   - **候选 = 0** → 终止流程，提示"未找到匹配企业，请提供更完整的名称或换关键词"

**对用户的话术（候选 ≥ 2 时）**：

> 你说的「{userInput}」匹配到 N 家企业，请确认是哪一家：
>
> | # | 企业名称 | USCC | 状态 | 法定代表人 | 注册地 |
> |---|---|---|---|---|---|
> | 1 | … | … | 存续 | … | … |
> | 2 | … | … | 存续 | … | … |
>
> 回复编号（1-N）以继续，或回复"都不是"重新输入。

### Step 1: 基础资信（正向维度）
- `get_company_registration_info` — 工商基础
- `get_financial_data` — 财务表现
- `get_company_scale` — 企业规模

### Step 2: 信用加分项
- `get_credit_evaluation` — 信用评价
- `get_honor_info` — 荣誉资质
- `get_import_export_credit` — 进出口信用

### Step 3: 风险扣分项（司法）
- `get_dishonest_info` — 失信
- `get_judgment_debtor_info` — 被执行人
- `get_high_consumption_restriction` — 限高

### Step 4: 风险扣分项（财产）
- `get_equity_pledge_info` — 股权出质
- `get_equity_freeze` — 股权冻结
- `get_chattel_mortgage_info` — 动产抵押
- `get_stock_pledge_info` — 股权质押（上市）

### Step 5: 风险扣分项（税务与经营）
- `get_tax_arrears_notice` — 欠税
- `get_tax_violation` — 税收违法
- `get_business_exception` — 经营异常

## 输出格式

```markdown
# 企业信用评级报告 — {name}

> 评级出具: {ISO8601} · 评级模型: TYC-CR-v1.0

## 一、综合评级
- **评级结果：AA / A / BBB / BB / B / CCC / D**
- 综合得分：{score} / 100
- 建议授信额度：¥{min} - ¥{max}
- 建议利率区间：{minRate}% - {maxRate}%

## 二、评分明细

| 维度 | 得分 | 权重 | 加权 | 说明 |
|------|------|------|------|------|
| 基础资信 | 85 | 25% | 21.3 | 注册资本实缴 {ratio} |
| 财务能力 | 78 | 20% | 15.6 | 最近营收 {amount} |
| 经营稳定性 | 72 | 15% | 10.8 | 成立 {years} 年 |
| 信用正向 | 90 | 10% | 9.0 | 荣誉 {n} 项 |
| 司法风险 | -15 | 15% | -2.25 | 失信 {n} 条 |
| 财产风险 | -8 | 10% | -0.8 | 股权冻结 {n} 条 |
| 税务风险 | -5 | 5% | -0.25 | 欠税 {amount} |
| **合计** | **—** | **100%** | **{total}** | |

## 三、风险预警
- 🔴 高危: {风险清单}
- 🟡 关注: {风险清单}
- 🔵 信息: {风险清单}

## 四、授信建议
- **建议通过 / 人工复核 / 谢绝**
- 还款能力: 强 / 中 / 弱
- 担保要求: 无 / 保证 / 抵押 / 混合
- 复评周期: 1 / 3 / 6 / 12 个月

## 五、历史趋势（如适用）
（若有历史评级记录，展示评级变化）
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）
- 财务数据不可得（非上市+未披露） → 该维度降权至 5%
- 风险维度全部 `_empty` → 记为"无公开风险"，不扣分
- 税务违法 tyc `error_code: 300000` → 该维度得 0（不扣分）

## 示例

输入: `searchKey = "宁德时代新能源科技股份有限公司"`


---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
## 与其他 Skill 的关系

- 银行客群专用入口 → `/tyc-kyb` (banking)
- 贷后监控 → `/tyc-credit-monitor` (banking)
- 供应商准入 → `/tyc-new-supplier` (supply)
