---
name: tyc-supplier-annual
description: 供应商年度评审的标准化核查工具，输出经营/资质/信用变更结构化报告
category: supply
version: 1.0
---

# 供应商年度健康体检

## 触发条件

年度供应商评审会议、年度合作续签决策、采购年度回顾。

关键词：供应商年度、合作续签、年度体检、采购回顾

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

### Step 1: 经营变化
- `get_company_registration_info` — 当前 vs 去年
- `get_change_records` — 工商变更（近 12 月）
- `get_company_scale` — 规模变化

### Step 2: 资质有效性
- `get_qualifications` — 资质证书与到期
- `get_administrative_license` — 行政许可

### Step 3: 信用记录变化
- `get_credit_evaluation` — 信用评级
- `get_administrative_penalty` — 处罚（近 12 月）
- `get_business_exception` — 经营异常

### Step 4: 司法记录
- `get_judicial_documents`
- `get_dishonest_info`

### Step 5: 业务能力
- `get_bidding_info` — 中标记录
- `get_team_members` — 核心团队

## 输出格式

```markdown
# 供应商年度体检 — {name}

> 体检周期: {year-1} → {year}

## 一、经营变化（同比）
| 指标 | 上年 | 本年 | 变化 |
|------|------|------|------|
| 注册资本 | | | |
| 参保人数 | | | |
| 分支机构 | | | |
| 法定代表人 | | | 变更 / 未变 |

## 二、资质有效性
- 总资质数: {n}（同比 {±x}）
- 临期资质: {n}
- 过期资质: {n}
- 新增资质: {n}

## 三、信用记录变化
- 信用评级: {上年} → {本年}
- 新增处罚: {n} 条
- 经营异常: {n} 次

## 四、司法记录
- 新增诉讼: {n}
- 新增失信: {n}

## 五、业务能力
- 中标项次（本年）: {n}
- 核心团队稳定性: ...

## 六、续签建议
- 综合评级变化: A → A / A → B / ...
- 续签建议: 续签 / 调整合作模式 / 终止
- 关注事项: ...
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）

## 示例

输入: `searchKey = "某长期合作供应商"`

---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
