---
name: tyc-contract-party
description: 合同签署前的主体快速核验工具，自动查验对方工商状态/法定代表人/经营异常
category: legal
version: 1.0
---

# 合同相对方主体核验

## 触发条件

合同签字盖章前的快速主体核验、防范无效合同 / 履约失败风险。

关键词：合同主体、签约前核查、主体真实性、注销吊销

## 输入要求

本技能需要两个独立输入：**合同对方企业标识**和**人员姓名**。

| 字段 | 必填 | 说明 | L0 处理 |
|---|---|---|---|
| `searchKey` | ✅ | 合同对方企业名 / USCC / 简称 / 曾用名 / 品牌名 | **见 Step 0**（按形态分流；简称类必走 L0） |
| `legalPersonName` | ❌（可选） | 目标人员姓名 | **不走 L0**（L0 是企业实体锚定，对人员无效；姓名直接透传） |

`searchKey` 三种形态分流：

| 输入形式 | 示例 | 是否需要 L0 |
|---|---|---|
| **完整企业名**（含组织形式后缀：`有限公司` / `股份` / `集团` 等） | `北京字节跳动科技有限公司` | ❌ 跳过 L0 |
| **统一社会信用代码（USCC）** 18 位大写字母+数字 | `91110108551385082Q` | ❌ 跳过 L0 |
| **企业简称 / 曾用名 / 品牌名 / 模糊指代** | `字节` / `抖音` / `阿里` | ✅ **必须先走 L0** |

> Step 0 仅作用于 `searchKey`；`legalPersonName` 始终透传至 Step 1+。

## 执行流程

### Step 0: 实体锚定（条件性 · L0 工具：`search_companies`）

**目的**：先把 `searchKey`（企业输入）消歧到唯一企业；`legalPersonName` 不参与 L0，原样透传。

**判定与分流（仅作用于 `searchKey`）**：

1. **若 `searchKey` 匹配 USCC 正则** `^[0-9A-Z]{18}$` → 跳过 L0
2. **若 `searchKey` 含组织形式后缀**（`有限公司` / `股份` / `集团` 等）且长度 ≥ 6 → 跳过 L0
3. **否则**（简称 / 曾用名 / 品牌名）→ **必须走 L0**：

   - 调用 `search_companies` (`searchKey: userInput`)
   - 候选 = 1 → 自动锚定为 `searchKey = items[0].creditCode`
   - 候选 ≥ 2 → 暂停并向用户列候选清单（与 A 类同），待用户回复后取选定 `creditCode`
   - 候选 = 0 → 终止流程

**注意**：`legalPersonName` 不做 L0 处理（L0 是企业锚定）。若同一姓名跨多个企业出现，仍由 Step 1+ 的"企业名 + 人员姓名"双锚定保证精度。

### Step 1: 主体存续
- `get_company_registration_info` — 工商状态
- `get_history_names` — 曾用名（避免老合同名称失效）

### Step 2: 三要素一致性
- `verify_company_accuracy` — 名称/法人/信用代码

### Step 3: 红线检查
- `get_business_exception` — 经营异常
- `get_serious_violation` — 严重违法
- `get_cancellation_record_info` — 注销备案

### Step 4: 联系信息
- `get_contact_info` — 联系电话/邮箱（用于交付）

## 输出格式

```markdown
# 合同主体快速核验 — {name}

## 一、主体存续状态
- ✓ / ❌ 工商登记: {regStatus}
- ✓ / ❌ 三要素核验: 一致 / 不一致

## 二、企业真实性
- 当前名称: {name}
- 曾用名: {historyNames}（如有）
- 注册时间: {estiblishTime}
- 注册资本 / 实缴: ...

## 三、红线警告
| 红线 | 触发 | 建议 |
|------|------|------|
| 营业执照注销 | 是/否 | 终止签约 |
| 营业执照吊销 | 是/否 | 终止签约 |
| 经营异常 | {n} 条 | 加强审核 |
| 严重违法 | {n} 条 | 加保函 |

## 四、联系方式核验
- 电话: ...
- 邮箱: ...
- 官网: ...

## 五、签约决策
- 是否可以签约: ✓ 是 / ❌ 否
- 加固条款建议: ...
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）

## 示例

输入: `searchKey = "ABC 公司"`, `legalPersonName = "张三"`

---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
