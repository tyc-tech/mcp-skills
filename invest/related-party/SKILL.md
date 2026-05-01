---
name: tyc-related-party
description: 关联方穿透识别 — 股权穿透 + 实控人控制企业 + 共同董监高识别，IPO 关联交易披露核心
category: invest
version: 1.0
---

# 关联方穿透识别

## 触发条件

IPO 项目关联交易披露、上市公司审计独立性核查、反洗钱 PEP 识别、实控人调查时触发。

关键词：关联方、关联交易、IPO 披露、一致行动人、实控人控制企业、共同董监高

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

### Step 1: 股权侧关联方
- `get_shareholder_info` — 直接股东
- `get_beneficial_owners` — 受益所有人（UBO）
- `get_actual_controller` — 实际控制人
- `get_equity_tree` — 股权树（多层）
- `get_parent_company` — 母公司
- `get_external_investments` — 对外投资（识别子公司）

### Step 2: 实控人控制面（同一实控人下的兄弟公司）
若 Step 1 识别到实控人 A：
- `get_controlled_companies`（with searchKey=A.name）— A 控制的所有企业
- 得出「同一实控人」的兄弟公司清单

### Step 3: 董监高交叉关联
对目标公司核心董监高（法代、董事、监事）逐个调用：
- `get_personnel_controlled_companies` — 该人控制的其他企业
- `get_personnel_related_companies` — 该人关联的其他企业
- `get_person_partners` — 该人的合作伙伴

> 识别「共同董监高」关联方

### Step 4: 图谱验证
- `get_relation_graph` — 目标企业与已识别关联方的关系图（验证）
- `get_relation_path` — 必要时计算两点之间的最短关联路径

## 输出格式

```markdown
# 关联方穿透报告 — {name}

> 出具: {ISO8601} · 企业代号: {creditCode}

## 一、关联方清单（共 {N} 个）

### A. 股权关联（{n} 个）
| # | 关联方 | 关联类型 | 股权比例 / 层级 | 控制性 |
|---|-------|---------|----------------|-------|
| 1 | {实控人} | 实际控制人 | 穿透后 {ratio} | 控制 |
| 2 | {母公司} | 直接控股股东 | {ratio} | 控制 |
| 3 | {子公司 A} | 直接投资 | {ratio} | 控制 |
| ... | | | | |

### B. 同一实控人关联（{n} 个）
| # | 关联方 | 共同实控人 | 该方持股 | 备注 |
|---|-------|----------|---------|------|
| 1 | {兄弟公司} | {A} | {ratio} | 同业 / 上下游 / 其他 |

### C. 共同董监高关联（{n} 个）
| # | 关联方 | 共同人员 | 本企业角色 | 对方角色 |
|---|-------|---------|----------|---------|
| 1 | ... | {张三} | 董事 | 法定代表人 |

## 二、风险提示

### 关联交易风险
- 🔴 高: 与 {n} 家同一实控人企业存在业务重叠
- 🟡 中: 共同董监高 {n} 人需披露
- 🔵 低: 股权关联方已知清晰

### IPO / 审计披露建议
- 需披露关联方: {完整清单}
- 重点关注: {TOP 3}
- 建议尽调: 近 3 年关联交易金额、定价公允性

## 三、关系图（Mermaid）

```mermaid
graph TB
  TARGET[{name}]
  CTRL[{实控人}]
  PARENT[{母公司}]
  BRO1[{兄弟公司1}]
  SUB1[{子公司1}]

  CTRL -->|100%| PARENT
  PARENT -->|{ratio}| TARGET
  CTRL -->|{ratio}| BRO1
  TARGET -->|{ratio}| SUB1
```
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）
- `get_controlled_companies` 的实控人姓名为空 → 跳过 Step 2
- 董监高 > 10 人 → 仅对 TOP 5 核心（法代 + 董事长 + 总经理 + 财务总监 + 董秘）做 Step 3
- `_empty: true` 场景 → 报告标注"无公开关联方"，不报错

## 示例

输入: `searchKey = "某拟 IPO 公司"`


---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
## 与其他 Skill 的关系

- 纯股权图谱可视化 → `/tyc-equity-graph`
- 集团族谱（母子公司全图） → `/tyc-group-analysis`
- 高管个人背调 → `/tyc-exec-bg`
