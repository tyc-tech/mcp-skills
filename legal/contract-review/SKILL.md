---
name: tyc-contract-review
description: 合同审核与主体核验一体化工具，AI 审合同同步核验双方主体工商状态与 34 类司法风险
category: legal
version: 1.0
---

# 合同核验 (contract-review)

## 触发条件

合同签署前对相对方主体真实性 + 司法风险一并核验，与 AI 合同审阅工具集成。

关键词：合同审核、主体核验、签约前核查、抗 AI 幻觉

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

### Step 1: 主体核验
- `get_company_registration_info` — 工商登记 + 经营状态
- `verify_company_accuracy` — 三要素核验

### Step 2: 经营异常红线
- `get_business_exception`
- `get_serious_violation`
- `get_administrative_penalty`

### Step 3: 司法红线
- `get_dishonest_info`
- `get_judgment_debtor_info`
- `get_judicial_documents`
- `get_high_consumption_restriction`

### Step 4: 资产风险
- `get_equity_freeze`
- `get_chattel_mortgage_info`
- `get_judicial_auction`

### Step 5: 综合
- `get_risk_overview`

## 输出格式

```markdown
# 合同主体核验批注 — {name}

## 高风险标注
🚨 营业执照已注销 / 吊销
🚨 失信被执行 ({n} 条)
🚨 大额被执行 (累计金额: ...)
🚨 股权被冻结 / 司法拍卖中

## 中风险标注
⚠️ 经营异常 ({n} 条)
⚠️ 行政处罚 ({n} 条)
⚠️ 主要人员变动频繁

## 一致性标注
✓ 三要素核验一致
✓ 工商状态: {regStatus}

## 签约建议
- 是否建议签约: ✓ 是 / ❌ 否 / ⚠️ 加保函
- 风险条款建议: ...
- 履约监控建议: ...
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）

## 示例

输入: `searchKey = "合同甲方企业名称"`

---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
