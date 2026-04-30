---
name: tyc-kyb
description: 银行开户和授信审批前的企业主体核验与全维风险扫描，30 秒输出符合 FATF/PBOC/CBIRC 标准的合规报告
category: banking
version: 1.0
---

# KYB 企业核验

## 触发条件

银行/金融机构在客户开户、授信审批、续贷复审等关键节点对企业进行尽调时触发。

关键词：KYB、企业核验、开户审查、主体核验、合规审查、授信前置

## 输入要求

用户输入的企业标识可能是以下三种形式之一，**Step 0 会按形式分流**，不要急着把用户原话当成 `searchKey` 喂给后续工具：

| 输入形式 | 示例 | 是否需要 L0 实体锚定 |
|---|---|---|
| **完整企业名**（含组织形式后缀：`有限公司` / `股份有限公司` / `集团` / `合伙企业` / `个体工商户` / `事务所` / `中心` 等） | `北京字节跳动科技有限公司` / `腾讯科技（深圳）有限公司` | ❌ 跳过 L0，直接当 `searchKey` |
| **统一社会信用代码（USCC）** 18 位大写字母+数字 | `91110108551385082Q` | ❌ 跳过 L0，直接当 `searchKey` |
| **企业简称 / 曾用名 / 品牌名 / 模糊指代** | `字节` / `抖音` / `今日头条` / `乐视` / `阿里` | ✅ **必须先走 L0**，由用户在候选中确认唯一企业，再拿 `creditCode` 作为 `searchKey` |

> 经过 Step 0 锚定后，下文 Step 1+ 的 `searchKey` 都指**精确企业名或 USCC**，工具调用方式不变。

## 执行流程

### Step 0: 实体锚定（条件性 · L0 工具：`search_companies`）

**目的**：用户给的常常是简称、曾用名、模糊指代，下游所有工具都依赖精确企业；先用 L0 把它消歧成唯一 `creditCode`，避免后 5 步在错主体上反复烧 token。

**判定与分流**：

1. **若 userInput 匹配 USCC 正则** `^[0-9A-Z]{18}$` → 跳过 L0，`searchKey = userInput`
2. **若 userInput 含上文表格中的组织形式后缀**（`有限公司` / `股份` / `集团` / `合伙企业` / `事务所` / `个体工商户` / `分公司` 等）**且长度 ≥ 6** → 跳过 L0，`searchKey = userInput`
3. **否则**（简称 / 曾用名 / 品牌名 / 任何看起来不完整的字符串）→ **必须走 L0**：

   - 调用 `search_companies` (`searchKey: userInput`)
   - 在返回 `items[]` 中默认按 `regStatus ∈ {存续, 在业, 在营, 开业}` 过滤，按 `regCapital` 倒序取 Top 5 作为候选展示
   - **候选 = 1** → 自动锚定，`searchKey = items[0].creditCode`，在报告"零、企业锚定"中注明"已自动锚定 → `{items[0].name}`"
   - **候选 ≥ 2** → **暂停执行**，向用户输出候选清单（`name` / `creditCode` / `regStatus` / `regLocation` / `legalPersonName`）并请求确认，待用户回复后取选定企业的 `creditCode` 作为 `searchKey`
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

### Step 1: 主体真实性核验
- 调用 `get_company_registration_info` (`searchKey`) — 工商基础（法人/资本/状态/经营范围/参保人数）
- 调用 `verify_company_accuracy` (`searchKey`, `legalPersonName?`) — 三要素一致性校验

### Step 2: 股权与控制结构
- 调用 `get_shareholder_info` (`searchKey`) — 股东构成与出资明细
- 调用 `get_actual_controller` (`searchKey`) — 实际控制人
- 调用 `get_beneficial_owners` (`searchKey`) — 受益所有人 (UBO)

### Step 3: 司法风险扫描
- 调用 `get_judicial_documents` (`searchKey`) — 裁判文书
- 调用 `get_judgment_debtor_info` (`searchKey`) — 被执行人
- 调用 `get_dishonest_info` (`searchKey`) — 失信被执行人
- 调用 `get_high_consumption_restriction` (`searchKey`) — 限制高消费

### Step 4: 经营风险扫描
- 调用 `get_business_exception` (`searchKey`) — 经营异常
- 调用 `get_serious_violation` (`searchKey`) — 严重违法
- 调用 `get_administrative_penalty` (`searchKey`) — 行政处罚
- 调用 `get_tax_arrears_notice` (`searchKey`) — 欠税公告

### Step 5: 综合风险评估
- 调用 `get_risk_overview` (`searchKey`) — 自身/周边/预警风险

## 输出格式

```markdown
# KYB 企业核验报告 — {name}

> 核验时间: {ISO8601}
> 数据来源: 天眼查 T1.1 业务语义聚合层

## 零、企业锚定（如经过 L0）

> 用户原始输入：「{userInput}」
> 锚定方式：USCC 直通 / 完整名直通 / L0 search_companies 候选确认（N 选 1）
> 最终锚定：{name} · {creditCode}

## 一、主体信息
| 字段 | 值 |
|------|-----|
| 企业名称 | {name} |
| 统一社会信用代码 | {creditCode} |
| 法定代表人 | {legalPersonName} |
| 企业类型 | {companyOrgType} |
| 经营状态 | {regStatus} |
| 成立日期 | {estiblishTime} |
| 注册资本 | {regCapital} |
| 参保人数 | {socialStaffNum} |
| 三要素核验 | 一致/不一致 |

## 二、股权穿透
**主要股东**: 表格列出 top 5
**实际控制人**: {actualController.name} (持股比例: {totalRatio})
**最终受益人 (UBO)**: 表格列出全部

## 三、司法风险（4 维）
| 维度 | 数量 | 风险等级 |
|------|------|---------|
| 裁判文书 | {n} | 低/中/高 |
| 被执行 | {n} | ... |
| 失信 | {n} | ... |
| 限高 | {n} | ... |

## 四、经营风险（4 维）
| 维度 | 数量 | 风险等级 |
|------|------|---------|

## 五、综合风险
{risk_overview._summary}

## 六、核验结论
- 主体真实性: 通过 / 未通过
- 风险等级: 低 / 中 / 高
- 是否准入: 建议准入 / 加强审核 / 拒绝
- 关键风险点: ...
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）
- 若 `verify_company_accuracy` 中 match=false → 标注"主体真实性未通过"，但其他维度仍执行
- 若任一风险扫描工具返回 `_empty: true` → 该维度记 0，使用 `_summary` 文案
- 若 `get_risk_overview` 失败 → 用 4 个司法 + 4 个经营维度合计推导

## 示例

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。

输出: 6 段完整核验报告，含主体锚定记录、主体信息、股权穿透、风险扫描与准入结论。
