---
name: tyc-legal-risk
description: 立案前的对手方风险排查工具 V2.0，三层穿透扫描（企业当前 × 企业历史 × 核心人员）
category: legal
version: 2.0
---

# 诉讼风险评估 V2.0（三层诉讼全景）

## 触发条件

律师立案前评估对手方诉讼泥潭、企业胜诉概率研判、应诉策略制定。

关键词：诉讼风险、对手方分析、立案评估、应诉策略

## 输入要求

本技能需要两个独立输入：**对手方企业标识**和**人员姓名**。

| 字段 | 必填 | 说明 | L0 处理 |
|---|---|---|---|
| `searchKey` | ✅ | 对手方企业名 / USCC / 简称 / 曾用名 / 品牌名 | **见 Step 0**（按形态分流；简称类必走 L0） |
| `humanName` | ❌（可选） | 目标人员姓名 | **不走 L0**（L0 是企业实体锚定，对人员无效；姓名直接透传） |

`searchKey` 三种形态分流：

| 输入形式 | 示例 | 是否需要 L0 |
|---|---|---|
| **完整企业名**（含组织形式后缀：`有限公司` / `股份` / `集团` 等） | `北京字节跳动科技有限公司` | ❌ 跳过 L0 |
| **统一社会信用代码（USCC）** 18 位大写字母+数字 | `91110108551385082Q` | ❌ 跳过 L0 |
| **企业简称 / 曾用名 / 品牌名 / 模糊指代** | `字节` / `抖音` / `阿里` | ✅ **必须先走 L0** |

> Step 0 仅作用于 `searchKey`；`humanName` 始终透传至 Step 1+。

## 执行流程

### Step 0: 实体锚定（条件性 · L0 工具：`search_companies`）

**目的**：先把 `searchKey`（企业输入）消歧到唯一企业；`humanName` 不参与 L0，原样透传。

**判定与分流（仅作用于 `searchKey`）**：

1. **若 `searchKey` 匹配 USCC 正则** `^[0-9A-Z]{18}$` → 跳过 L0
2. **若 `searchKey` 含组织形式后缀**（`有限公司` / `股份` / `集团` 等）且长度 ≥ 6 → 跳过 L0
3. **否则**（简称 / 曾用名 / 品牌名）→ **必须走 L0**：

   - 调用 `search_companies` (`searchKey: userInput`)
   - 候选 = 1 → 自动锚定为 `searchKey = items[0].creditCode`
   - 候选 ≥ 2 → 暂停并向用户列候选清单（与 A 类同），待用户回复后取选定 `creditCode`
   - 候选 = 0 → 终止流程

**注意**：`humanName` 不做 L0 处理（L0 是企业锚定）。若同一姓名跨多个企业出现，仍由 Step 1+ 的"企业名 + 人员姓名"双锚定保证精度。

### Step 1: 企业当前层（5 维司法 + 经营）
- `get_judicial_documents` — 裁判文书
- `get_dishonest_info` — 失信
- `get_judgment_debtor_info` — 被执行
- `get_high_consumption_restriction` — 限高
- `get_terminated_cases` — 终本案件

### Step 2: 企业历史层（V2.0 新增 - 时间序列）
- `get_historical_judicial_docs` — 历史裁判文书
- `get_historical_dishonest` — 历史失信
- `get_historical_judgment_debtor` — 历史被执行
- `get_historical_court_notice` — 历史法院公告
- `get_historical_hearing_notice` — 历史开庭公告

### Step 3: 核心人员层（若提供 humanName）
- `get_personnel_dishonest` (`searchKey`, `humanName`)
- `get_personnel_judgment_debtor` (`searchKey`, `humanName`)
- `get_personnel_high_consumption_ban` (`searchKey`, `humanName`)
- `get_personnel_terminated_cases` (`searchKey`, `humanName`)

### Step 4: 案件深度（如需详情）
- `get_lawsuit_detail` (`id`) — 关键案件文书详情
- `get_judicial_case` — 司法解析

### Step 5: 综合
- `get_risk_overview`

## 输出格式

```markdown
# 诉讼风险评估 V2.0 — {name}

## 一、企业当前诉讼地位
- 累计裁判文书: {n} 件
- 当前被执行: {n} 件
- 失信记录: {n} 条
- 终本案件: {n} 件（无可执行财产）

## 二、企业历史诉讼轨迹（10 年）
| 年度 | 裁判文书 | 失信 | 被执行 | 趋势 |
|------|---------|------|--------|------|

时间序列分析:
- 诉讼频率: 高/中/低
- 历史败诉率: {rate}
- 偿付能力趋势: 改善/稳定/恶化

## 三、核心人员涉诉档案（若提供 humanName）
- 个人失信记录: {n} 条
- 个人被执行: {n} 条
- 个人限高: {n} 条
- 综合判断: ...

## 四、关键案件文书摘要
（重点案件的裁判结果与赔偿金额）

## 五、立案建议
- 胜诉概率推演: ...
- 执行风险: 高 / 中 / 低（含终本警示）
- 建议措施: 立案 / 调解 / 财产保全 / 暂缓
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）

## 示例

输入: `searchKey = "ABC 被告企业"`, `humanName = "张三"`


---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
## V2.0 升级说明

V2.0 新增"企业历史层"（5 个 history 工具）+ "核心人员层"（4 个 executive 工具），实现真实时间序列趋势分析与个人涉诉穿透。
