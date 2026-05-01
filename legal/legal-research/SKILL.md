---
name: tyc-legal-research
description: 法律研究（案例检索 / 法律意见书 / 合规研究）— 关联企业案例自动检索 + 涉诉风险分析
category: legal
version: 1.0
---

# 法律研究

## 触发条件

律师案例检索 / 法律意见书撰写 / 监管合规研究 / 涉诉企业深度分析时触发。

关键词：法律研究、案例检索、法律意见书、判例分析、企业合规研究

## 输入要求

本技能输入是 **研究对象（法律问题关键词 / 案号 / 企业名 等）** 而非具体企业，**Step 0 不会发起 L0 实体锚定**：

| 字段 | 必填 | 说明 |
|---|---|---|
| `target` | ✅ | 研究对象（法律问题关键词 / 案号 / 企业名 等）（如"储能电池"、"工业机器人"） |

> ⚠ **判定钩子**：若用户给出的 `target` 实际像一个**企业名 / 简称 / USCC**（含组织形式后缀、18 位代码、典型品牌名），不要在本技能内继续——这是用户问错了 skill。建议先调 L0（`search_companies` / `tyc company companies`）锚定到具体企业，再切换到对应单企业类 skill（如 `tyc-profile` / `tyc-credit-dd` / `tyc-ic-memo`）。

## 执行流程

### Step 0: 输入合理性确认（不调 L0）

本技能的输入是行业 / 地域 / 关键词，**不需要做企业实体锚定**。但需做一次"用户是否问错 skill"的钩子检查：

- 若用户给的 `industry` / `target` 实际看起来像一个**企业名 / 简称 / USCC**（含组织形式后缀、18 位代码、典型品牌名），**先暂停本技能**，提示用户：
  - 你似乎在问一家具体公司的情况，本技能是行业 / 关键词分析。建议改用 `tyc-profile` / `tyc-credit-dd` / `tyc-ic-memo` 等单企业类 skill。
  - 若坚持走本技能，请明确给出**行业关键词**（如"储能电池"、"工业机器人"）。

钩子放行后即可进入 Step 1。

### 模式 A: 企业案例检索（case）

1. 识别主体 → `get_company_registration_info <target>`
2. 涉诉全景 → `get_judicial_documents` / `get_judicial_case`
3. 重大案件 → `get_lawsuit_detail <caseNo>` × TOP 10
4. 执行端 → `get_dishonest_info` / `get_judgment_debtor_info`
5. 案件拓展（同一实控人下关联主体的判例）
   - `get_actual_controller` 获取实控人
   - `get_controlled_companies` 获取兄弟公司
   - 对兄弟公司并发调用 `get_judicial_documents`，识别共性案由
6. 历史趋势 → `get_historical_judicial_docs`

### 模式 B: 法律意见书（legal-opinion）

1. 主体资格核验（同板块 1）
2. 重大合规事项扫描：
   - `get_administrative_penalty` — 行政处罚
   - `get_business_exception` — 经营异常
   - `get_environmental_penalty` — 环保
   - `get_tax_violation` — 税务
3. 资质有效性：`get_qualifications` / `get_administrative_license`
4. 形成意见书章节化模板

### 模式 C: 合规研究（compliance）

1. 行业参照企业（同类资质 / 同品类）：
   - `search_companies_by_industry_region`
   - `search_companies_by_tag`
2. 合规基准：收集同行近 3 年行政处罚、涉诉、经营异常典型样本
3. 对比研究输出

## 输出格式

```markdown
# 法律研究报告 — {target}

> 律所: ... · 研究员: ... · 出具: {ISO8601}
> 研究模式: {case / legal-opinion / compliance}

## 一、研究背景
## 二、法律问题
## 三、证据与数据
### 3.1 主体情况
### 3.2 案件 / 处罚 / 合规事项清单（含证据来源）
### 3.3 关联企业扩展（同一实控人下类似案件）

## 四、判例分析
| 案号 | 案由 | 标的 | 判决要点 | 对本案启示 |
|-----|------|------|---------|-----------|

## 五、法律意见 / 研究结论
- 法律依据: ...
- 论证过程: ...
- 结论与建议: ...
- 风险提示: ...

## 六、证据索引（可追溯来源）
- {证据 1}: 天眼查 - 裁判文书 - {caseNo}
- {证据 2}: 天眼查 - 行政处罚 - {docId}
...
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）
- `_empty` → 对应章节标注"经查未发现相关记录"
- 案件数量 > 100 → 按年度 + 案由做聚合摘要，TOP 20 取详情
- 单案件详情失败 → 标注 `[!]` 跳过，不影响其他

## 示例

- 输入: `target = "某涉诉企业"`, `mode = "case"`
- 输入: `target = "食品经营许可合规性研究"`, `mode = "compliance"`


---

**新输入形态参考**（Step 0 钩子判定的三种典型）：

**例 1（合法关键词，钩子放行）**：

输入: `industry = "储能电池"` → Step 0 判定为行业关键词，直接进 Step 1。

**例 2（用户问错 skill — 给了完整企业名）**：

输入: `industry = "宁德时代新能源科技股份有限公司"` → Step 0 钩子判定形似企业名（含组织形式后缀），暂停本技能，建议改用单企业类 skill（如 `tyc-profile` / `tyc-credit-dd` / `tyc-ic-memo`）。

**例 3（用户问错 skill — 给了企业简称）**：

输入: `industry = "宁德时代"` → Step 0 钩子判定形似品牌/简称，暂停本技能，提示先调 L0（`search_companies` / `tyc company companies`）锚定到精确企业再切换 skill。

## 与其他 Skill 的关系

- **法律尽调 10 章底稿** → `/tyc-legal-dd`
- **纯诉讼风险分析**（投资视角）→ `/tyc-litigation-analysis` (invest)
- **关联方穿透** → `/tyc-related-party` (invest)
