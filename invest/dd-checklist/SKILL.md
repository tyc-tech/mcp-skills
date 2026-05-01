---
name: tyc-dd-checklist
description: 尽调清单自动化 — 标准化尽调清单自动填充 + 进度追踪，PE/VC/投行项目组首日 DD 套件
category: invest
version: 1.0
---

# 尽调清单自动化

## 触发条件

PE/VC、投行项目组启动尽调首日，需要标准化尽调清单 + 企业数据自动填充时触发。

关键词：尽调清单、DD Checklist、due diligence、尽调进度、VDR、尽调分工

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

### Step 1: 主体与股权（工商维度）
- `get_company_registration_info` — 工商登记基本信息
- `get_shareholder_info` — 股东构成
- `get_actual_controller` — 实控人
- `get_key_personnel` — 主要人员（法代 / 高管）
- `get_branches` — 分支机构
- `get_external_investments` — 对外投资

### Step 2: 财务（财务维度）
- `get_financial_data` — 财务数据
- `get_company_scale` — 企业规模
- `get_annual_reports` — 年报

### Step 3: 风险扫描（合规维度）
- `get_risk_overview` — 风险总览
- `get_dishonest_info` — 失信
- `get_judgment_debtor_info` — 被执行人
- `get_administrative_penalty` — 行政处罚
- `get_bankruptcy_reorganization` — 破产

### Step 4: 知识产权（技术维度）
- `get_patent_info` — 专利
- `get_trademark_info` — 商标
- `get_software_copyright_info` — 软件著作权
- `get_ipr_score` — 创新力评分

### Step 5: 经营与资质（商业维度）
- `get_bidding_info` — 中标记录
- `get_qualifications` — 企业资质
- `get_administrative_license` — 行政许可
- `get_financing_records` — 融资历史

## 输出格式

```markdown
# 尽调清单 — {name}

> 出具时间: {ISO8601} · 撰写人: AI · 企业代号: {creditCode}

## A. 主体与股权（已填充 6/6）
- [x] A1. 工商登记核验（{name} / {creditCode} / {legalPersonName}）
- [x] A2. 股东构成（TOP 5：...）
- [x] A3. 实控人（{actualController}，累计持股 {totalRatio}）
- [x] A4. 主要人员（法代、董事、监事清单）
- [x] A5. 分支机构（{总数} 家，{涉及省份}）
- [x] A6. 对外投资（{总数} 家）

## B. 财务（已填充 3/3）
- [x] B1. 财务数据（最近 3 年营收、利润）
- [x] B2. 企业规模（参保人数、人员规模）
- [x] B3. 年报（最近 {n} 年年报摘要）

## C. 合规风险（已填充 5/5）
- [x] C1. 风险总览（总数 {n}）
- [x] C2. 失信（{n} 条）
- [x] C3. 被执行人（{n} 条）
- [x] C4. 行政处罚（{n} 条）
- [x] C5. 破产清算（{状态}）

## D. 知识产权（已填充 4/4）
- [x] D1. 专利（{总数}，发明授权 {n}）
- [x] D2. 商标（{n} 件）
- [x] D3. 软件著作权（{n} 件）
- [x] D4. 创新力评分（{score}）

## E. 经营资质（已填充 4/4）
- [x] E1. 招投标（{n} 次中标）
- [x] E2. 企业资质（{高资质清单}）
- [x] E3. 行政许可（{n} 项）
- [x] E4. 融资历史（最近轮次 {round}）

---

## 二、待 PM 补充（人工环节）
- [ ] F. 现场访谈（创始人 / CFO / CTO）
- [ ] G. 合同调阅（VDR）
- [ ] H. 审计底稿对照

## 三、进度汇总
- 自动填充: **22/22 条**
- 人工环节: **待 0/3 条**
- 总体进度: 22/25 (88%)
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）
- 单条命令失败 → 对应 checklist 项标注 `[!]` + 失败原因
- `_empty: true` → 标注 `[~] 暂无数据`（非失败）

## 示例

输入: `searchKey = "北京百度网讯科技有限公司"`


---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
## 与其他 Skill 的关系

- 完整版备忘录 → `/tyc-ic-memo`
- 深度股权穿透 → `/tyc-equity-graph`
- 财务深度分析（上市） → `/tyc-financial-deep`
