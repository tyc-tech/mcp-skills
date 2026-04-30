---
name: tyc-exec-bg
description: 投资决策前的关键人员背调工具 V2.0，对创始人/董事/核心高管执行 18 维现状 + 14 维历史穿透
category: invest
version: 2.0
---

# 高管背景核查 V2.0（个人穿透升级版）

## 触发条件

投前对法定代表人、实际控制人、CEO/COO/CFO 等核心高管做个人风险背调；并购前关键人员尽调。

关键词：高管背调、关键人员、个人风险、董监高画像、个人涉诉

## 输入要求

本技能需要两个独立输入：**企业标识**和**人员姓名**。

| 字段 | 必填 | 说明 | L0 处理 |
|---|---|---|---|
| `searchKey` | ✅ | 企业名 / USCC / 简称 / 曾用名 / 品牌名 | **见 Step 0**（按形态分流；简称类必走 L0） |
| `humanName` | ✅ | 目标人员姓名 | **不走 L0**（L0 是企业实体锚定，对人员无效；姓名直接透传） |

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

### Step 1: 人员画像基础
- `get_person_profile` (`searchKey`, `humanName`) — 简介 + 商业角色 + 相关公司
- `get_person_partners` (`searchKey`, `humanName`) — 合作伙伴网络

### Step 2: 个人现状风险（6 维）
- `get_personnel_dishonest` — 失信
- `get_personnel_judgment_debtor` — 被执行人
- `get_personnel_high_consumption_ban` — 限制高消费
- `get_personnel_terminated_cases` — 终本案件
- `get_person_judicial_assistance` (`searchKey`, `humanName`) — 司法协助（含股权冻结）
- `get_person_risk_overview` — 个人风险总览

### Step 3: 控制企业网络
- `get_personnel_controlled_companies` — 实际控制的全部企业
- `get_personnel_related_companies` — 全部关联企业（任何角色）
- `get_personnel_legal_rep_roles` — 担任法定代表人企业
- `get_personnel_positions` — 在外任职

### Step 4: 个人历史穿透（V2.0 新增）
- `get_personnel_historical_dishonest` — 历史失信
- `get_personnel_historical_judgment_debtor` — 历史被执行
- `get_personnel_historical_high_consumption_ban` — 历史限高

## 输出格式

```markdown
# 高管个人风险档案 — {humanName} @ {searchKey}

## 一、个人画像
- 姓名: {humanName}
- 当前任职: {位置（董事长/CEO/...）}
- 主企业: {searchKey}
- 个人简介: ...

## 二、商业角色全景
- 法定代表人企业: {n} 家
- 实际控制企业: {n} 家
- 全部关联企业（任意角色）: {n} 家

## 三、个人现状风险（6 维）
| 风险类型 | 当前条数 | 状态 |
|---------|---------|------|
| 失信被执行 | ... | ... |
| 被执行人 | ... | ... |
| 限制高消费 | ... | ... |
| 终本案件 | ... | ... |
| 司法协助（冻结） | ... | ... |
| 综合风险评级 | — | 低/中/高 |

## 四、个人历史风险（V2.0 14 维）
| 历史维度 | 已结案数 | 结清状态 |
|---------|---------|---------|
| 历史失信 | ... | ... |
| 历史被执行 | ... | ... |
| 历史限高 | ... | ... |

## 五、合作伙伴网络
（关键合作伙伴 + 共同公司列表）

## 六、控制企业风险评估
- 控制企业中的高风险企业: {n} 家
- 已注销 / 吊销: {n} 家
- 出现失信/被执行的关联企业: {n} 家

## 七、综合背调结论
- 个人合规等级: A/B/C/D
- 是否建议作为投资关键人: 是 / 否 / 加强尽调
- 关键风险点: ...
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）
- `humanName` 在该企业无对应记录 → 提示"建议确认人员姓名"
- 个人维度数据缺失（部分维度 TYC 不提供）→ 标注"该维度暂无数据"

## 示例

输入: `searchKey = "字节跳动"`, `humanName = "张一鸣"`


---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
## V2.0 升级说明

V2.0 引入 T1.1 的 `executive` 分类 11 个个人维度工具 + 4 个人员画像扩展工具（人员画像/合作伙伴/风险总览/司法协助），实现"双参数实体强锚定"，告别旧版基于企业数据反推个人的模糊核查。
