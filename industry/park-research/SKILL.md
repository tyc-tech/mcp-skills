---
name: tyc-park-research
description: 园区企业研究（TYC 独有），园区画像/入园企业搜索/经纬度雷达/附近公司一站式
category: industry
version: 1.0
---

# 园区企业研究（TYC 独有扩展）

## 触发条件

园区招商画像、产业集群分析、商业地理选址、银行支行获客、政府区域经济研究。

关键词：产业园、招商、产业集群、商业地理、选址、获客

## 输入要求

本技能支持两种互斥的输入模式：

| 模式 | 输入字段 | 是否需要 Step 0 |
|---|---|---|
| **企业 / 园区模式** | `searchKey`：园区名称 或 企业名称 | ✅ 走 Step 0（同 A 类三形态分流） |
| **地理坐标模式** | `longitude` + `latitude` + `radius` | ❌ **跳过 Step 0**（坐标无需实体锚定） |

企业 / 园区模式下的三形态分流：

| 输入形式 | 示例 | 是否需要 L0 |
|---|---|---|
| **完整企业名 / 完整园区名**（含 `有限公司` / `产业园` / `开发区` 等后缀） | `北京字节跳动科技有限公司` / `中关村软件园` | ❌ 跳过 L0 |
| **USCC** 18 位 | `91110108551385082Q` | ❌ 跳过 L0 |
| **简称 / 曾用名 / 品牌名** | `字节` / `中关村` | ✅ **必须先走 L0**（园区简称同样支持） |

> 用户若给出经纬度，直接进入坐标模式，不要做任何企业实体锚定。

## 执行流程

### Step 0: 实体锚定（条件性 · 仅企业 / 园区模式 · L0 工具：`search_companies`）

**目的**：仅在用户走 `searchKey`（企业 / 园区名）模式时执行；坐标模式直接进 Step 1。

**模式判定**：
- 若 `longitude` + `latitude` + `radius` 全部就位 → **跳过 Step 0**，直接进 Step 1（坐标模式）
- 否则按 `searchKey` 走以下分流

**`searchKey` 三形态分流**：

1. **匹配 USCC 正则** `^[0-9A-Z]{18}$` → 跳过 L0
2. **含组织形式后缀**（`有限公司` / `股份` / `集团` 等）或园区后缀（`产业园` / `开发区` / `软件园`）且长度 ≥ 6 → 跳过 L0
3. **否则**（企业简称 / 园区简称 / 品牌名）→ **走 L0**：

   - 调用 `search_companies` (`searchKey: userInput`)
   - 候选 = 1 → 自动锚定为 `searchKey = items[0].creditCode` 或 `items[0].name`
   - 候选 ≥ 2 → 暂停并向用户列候选清单请求确认
   - 候选 = 0 → 终止流程并建议改用坐标模式

### Mode A: 园区画像（输入园区名称）
- `get_park_info` (`searchKey`) — 园区基本信息（面积/主管单位/所属地区）
- `search_park_companies` (`searchKey`) — 入园企业列表

### Mode B: 企业园区归属（输入企业名称）
- `get_park_info` (`searchKey`) — 反查企业所在园区
- `get_company_location` — 企业经纬度

### Mode C: 经纬度雷达（输入经纬度）
- `get_nearby_companies` (`longitude`, `latitude`, `radius`) — 半径内附近公司

### Step 后续：选定关键企业再深入
- `get_company_registration_info`
- `get_company_logo` — 用于报告封面

## 输出格式

```markdown
# 园区企业研究 — {searchKey or 经纬度}

## Mode A: 园区画像
- 园区名称: {parkName}
- 面积: ...
- 主管单位: ...
- 所属地区: ...

### 入园企业（TOP 20）
| 企业 | 类型 | 成立日期 | 状态 | 法定代表人 |
|------|------|---------|------|----------|

### 园区产业集群分析
- 主导行业: {industry list}
- 上市公司数: {n}
- 高新技术企业数: {n}

## Mode B: 企业园区归属
- 企业: {name}
- 所属园区: {park}
- 经纬度: {longitude}, {latitude}

## Mode C: 经纬度雷达
查询坐标: {longitude}, {latitude} · 半径: {radius} 米

附近公司（按距离排序）:
| 企业 | 距离 | 经营状态 | 经纬度 |
|------|------|---------|--------|

## 商业洞察
- 行业聚集度: ...
- 区域经济活跃度: ...
- 招商画像 / 选址建议: ...
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）

## 示例

输入 (Mode A): `searchKey = "中关村软件园"`
输入 (Mode C): `longitude = 116.391, latitude = 39.907, radius = 1000`

---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
