---
name: tyc-esg-assessment
description: ESG 风险评估 — 环境 / 社会 / 治理三维评分，ESG 基金 / 绿色金融 / 可持续投资筛选
category: invest
version: 1.0
---

# ESG 风险评估

## 触发条件

ESG 基金投前筛选、绿色信贷审批、可持续投资 due diligence、ESG 报告信息采集时触发。

关键词：ESG、环境、社会、治理、绿色金融、可持续投资、双碳

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

### Step 1: E（环境）维度
- `get_environmental_penalty` — 环境行政处罚
- `get_land_mortgage_info` — 土地使用（含 EIA 间接信号）
- `get_administrative_license` — 环评许可（若有 EIA 类许可）

### Step 2: S（社会）维度
- `get_recruitment_info` — 招聘（用工规模与合规信号）
- `get_administrative_penalty` — 行政处罚（筛选涉劳动 / 涉消费者项）
- `get_company_scale` — 员工规模与参保人数
- `get_import_export_credit` — 海关信用（供应链 S 维度）

### Step 3: G（治理）维度
- `get_actual_controller` — 实控人结构清晰度
- `get_shareholder_info` — 股东透明度
- `get_equity_freeze` — 股权冻结（治理风险信号）
- `get_key_personnel` — 董事会构成（独立董事占比）
- `get_judicial_documents` — 涉诉（治理规范性）
- `get_dishonest_info` — 失信（治理合规性）

### Step 4: 正向加分项
- `get_honor_info` — 荣誉（含 ESG / CSR 荣誉）
- `get_credit_evaluation` — 信用评价
- `get_ranking_list_info` — 榜单（ESG 专项榜单）

## 输出格式

```markdown
# ESG 风险评估报告 — {name}

> 出具: {ISO8601} · 评估框架: TYC-ESG-v1.0

## 一、综合评级
- **综合评级: AAA / AA / A / BBB / BB / B / CCC**
- 综合得分: {score} / 100
- 同行业分位: TOP {percentile}%
- 投资建议: 可投 / 观察 / 负面清单

## 二、三维评分

### E（环境）: {escore} / 100
| 指标 | 评分 | 说明 |
|------|------|------|
| 环保处罚 | -{n} | 近 3 年 {n} 次 |
| 历史环保处罚 | -{n} | 累计 {n} 次 |
| 环评许可 | +{n} | 持有 EIA 类许可 {n} 项 |
| 土地合规 | {n} | 土地抵押状态 |

### S（社会）: {sscore} / 100
| 指标 | 评分 | 说明 |
|------|------|------|
| 劳动监察 | -{n} | 黑名单 {n} 次 |
| 涉劳动处罚 | -{n} | 近 3 年 {n} 次 |
| 用工规模 | +{n} | 参保 {n} 人 |
| 海关信用 | +{n} | {等级} |

### G（治理）: {gscore} / 100
| 指标 | 评分 | 说明 |
|------|------|------|
| 股权透明度 | {n} | 穿透 {层} / 实控人清晰度 |
| 股权冻结 | -{n} | 冻结 {n} 次 |
| 董事会结构 | {n} | 独立董事 {n} 人 |
| 涉诉规模 | -{n} | 裁判文书 {n} 篇 |
| 失信 | -{n} | 失信 {n} 次 |

## 三、正向加分项
- 荣誉: ESG / CSR 类荣誉 {n} 项
- 信用评价: {A / AA / AAA 等级}
- 榜单: 入选 {榜单名称}

## 四、主要 ESG 风险
- 🔴 高: {风险点}
- 🟡 中: {风险点}
- 🔵 低: {风险点}

## 五、改进建议
- E: ...
- S: ...
- G: ...

## 六、ESG 信息披露建议
- 需披露事项: 环保处罚、劳动监察、治理结构
- 建议 ESG 报告章节: ...
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）
- 环保 / 劳动数据 `_empty` → 该子维度评为"无已知风险"（满分）
- 全部风险维度 `_empty` → 输出"ESG 公开数据不足，建议结合企业自披露"
- 部分 tyc `error_code: 300000` → 归一化 0 count，不报错

## 示例

输入: `searchKey = "某 ESG 筛池候选企业"`


---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
## 与其他 Skill 的关系

- 广义健康扫描 → `/tyc-health-scan`
- 法律风险可视化 → `/tyc-legal-risk` (legal)
- 供应链风险 → `/tyc-supply-chain-risk-fs`
