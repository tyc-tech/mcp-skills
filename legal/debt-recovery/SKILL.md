---
name: tyc-debt-recovery
description: 诉前财产保全决策的核心评估工具 V2.0，财务硬数据 + 资产清单 + 历史信用 + 实控人画像
category: legal
version: 2.0
---

# 债务清偿能力评估 V2.0（财务底盘升级版）

## 触发条件

诉前财产保全决策、债权追偿可行性评估、申请破产前的资产盘查。

关键词：债务清偿、财产保全、追偿评估、资产盘查、实控人兜底

## 输入要求

本技能需要两个独立输入：**债务人企业标识**和**人员姓名**。

| 字段 | 必填 | 说明 | L0 处理 |
|---|---|---|---|
| `searchKey` | ✅ | 债务人企业名 / USCC / 简称 / 曾用名 / 品牌名 | **见 Step 0**（按形态分流；简称类必走 L0） |
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

### Step 1: 财务硬数据（V2.0 核心）
- `get_financial_data` (`searchKey`) — 直接获取 3 年完整财报（资产负债率/速动比率/所有者权益）
- `get_company_scale` — 企业规模（参保 / 分支 / 投资）

### Step 2: 可执行资产清单
- `get_chattel_mortgage_info` — 动产抵押
- `get_land_mortgage_info` — 土地抵押
- `get_equity_pledge_info` — 股权出质
- `get_external_investments` — 对外投资（潜在可执行股权）
- `get_branches` — 分支机构（潜在标的）

### Step 3: 历史偿债模式
- `get_historical_dishonest` — 历史失信
- `get_historical_judgment_debtor` — 历史被执行
- `get_historical_judicial_docs` — 历史裁判文书

### Step 4: 资产被冻结状态
- `get_equity_freeze` — 股权冻结
- `get_judicial_auction` — 司法拍卖

### Step 5: 终本警示
- `get_terminated_cases` — 终本案件（关键警示信号）

### Step 6: 实控人个人兜底（V2.0，若提供 humanName）
- `get_personnel_dishonest`
- `get_personnel_judgment_debtor`
- `get_personnel_high_consumption_ban`
- `get_personnel_controlled_companies` — 实控人控制的其他企业（潜在追偿对象）

## 输出格式

```markdown
# 债务清偿能力评估 V2.0 — {name}

## 一、财务硬指标（基于 3 年完整财报）
- 资产负债率: {x}%
- 速动比率: {x}
- 流动比率: {x}
- 所有者权益: {x} 元
- 经营性现金流: {x}

财务健康度评级: A / B / C / D

## 二、可执行资产清单（按优先级）
### 高优先级（流动性强）
- 银行存款 / 应收账款（推断）
- 现有股权资产（含对外投资）

### 中优先级
- 不动产 / 厂房（含分支机构所在地）
- 大额应收

### 低优先级
- 知产资产（专利 / 商标）

## 三、资产受限情况
| 资产类型 | 已抵押 / 冻结 | 处置可行性 |
|---------|-------------|----------|
| 股权 | ... | ... |
| 动产 | ... | ... |
| 土地 | ... | ... |

## 四、历史偿债模式
- 历史失信: {n} 条
- 历史被执行总金额: ...
- 历史败诉案件: {n}
- 偿债倾向: 主动履行 / 被动执行 / 抗拒履行

## 五、终本警示（重要）
- 终本案件数: {n}
- 终本累计未履行金额: ...
- 警示等级: 🚨 严重 / ⚠️ 警示 / ✓ 安全

## 六、实控人个人兜底（V2.0，若提供 humanName）
- 个人偿付能力评估: ...
- 实控人控制的其他企业（备选追偿主体）: {n} 家

## 七、追偿成功率精算
- 主体追偿可行性: {percentage}%
- 实控人兜底可行性: {percentage}%
- 综合追偿成功率: {percentage}%

## 八、保全决策建议
- 财产保全建议: 推进 / 暂缓
- 优先保全标的（按优先级排序）: ...
- 保全金额建议: ...
- 替代措施: 调解 / 和解 / 申请破产
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）

## 示例

输入: `searchKey = "ABC 债务企业"`, `humanName = "李四"`


---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
## V2.0 升级说明

V2.0 首次引入 T1.1 的 `get_financial_data` 直接返回完整财报，告别旧版靠失信/被执行事件信号反推偿债的模糊判断。同步叠加 `executive` 分类的 4 个个人维度工具，实现"企业 + 实控人"双兜底分析。
