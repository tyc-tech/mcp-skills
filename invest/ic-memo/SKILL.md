---
name: tyc-ic-memo
description: PE/VC 投资决策的核心尽调工具，并行完成工商/股权/司法/知产全维度扫描，30 秒输出 IC 备忘录
category: invest
version: 1.0
featured: true
---

# IC Memo 投资备忘录 ⭐ 精选

## 触发条件

PE/VC 项目立项、投委会会前 IC Memo 出具、并购前期尽调时触发。

关键词：IC Memo、投委会、PE 尽调、VC 备忘录、PreMoney 评估

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

### Step 1: 主体与股权
- `get_company_registration_info` — 工商基础
- `get_actual_controller` — 实控人
- `get_shareholder_info` — 股东构成
- `get_equity_ratio` — 股权控制路径

### Step 2: 融资历史
- `get_financing_records` — 融资轮次、金额、估值、投资方

### Step 3: 司法风险
- `get_judicial_documents`
- `get_dishonest_info`
- `get_judgment_debtor_info`

### Step 4: 知识产权
- `get_patent_info` — 专利
- `get_trademark_info` — 商标
- `get_software_copyright_info` — 软件著作权
- `get_ipr_score` — 创新力评分

### Step 5: 经营画像
- `get_recruitment_info` — 招聘动态
- `get_news_sentiment` — 新闻舆情
- `get_team_members` — 核心团队
- `get_competitors` — 竞品

## 输出格式

```markdown
# IC Memo — {name}

> 项目代号: ... · 撰写人: AI · 出具时间: {ISO8601}

## 一、项目概览
- 公司名称: {name}
- 法定代表人: {legalPersonName}
- 实际控制人: {actualController.name} ({totalRatio})
- 成立日期 / 注册资本: {estiblishTime} / {regCapital}
- 业务定位: ...
- 行业: {industryAll.category}

## 二、股权结构
（含股权图，列出 TOP 5 股东）

## 三、融资历史
| 轮次 | 时间 | 金额 | 估值 | 投资方 |
|------|------|------|------|--------|

## 四、司法风险摘要
风险评级: 低 / 中 / 高

## 五、知识产权资产
- 专利: {n} 件（含发明授权 {x} 件）
- 商标: {n} 件
- 软件著作权: {n} 件
- 创新力评分: {score}

## 六、核心团队
（关键高管 + 简介）

## 七、竞品对照
（同行竞品列表 + 关键对照指标）

## 八、经营活跃度
- 招聘热度: ↑/↓
- 媒体声量: ↑/↓
- 中标记录: {n} 项

## 九、IC 决议建议
- 投资建议: 推进 / 暂缓 / 否决
- 估值区间: ...
- 主要风险: ...
- 后续 DD 重点: ...
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）

## 示例

输入: `searchKey = "Moonshot AI 月之暗面 (北京月之暗面科技有限公司)"`


---

**新输入形态参考**（Step 0 引入后的三种典型）：

**例 1（USCC 直通，跳过 L0）**：

输入: `userInput = "91110108551385082Q"` → Step 0 判定 USCC，`searchKey = userInput`，直接进 Step 1。

**例 2（完整企业名直通，跳过 L0）**：

输入: `userInput = "北京字节跳动科技有限公司"` → Step 0 判定含 `有限公司` 后缀且长度 ≥ 6，`searchKey = userInput`，直接进 Step 1。

**例 3（简称，必走 L0）**：

输入: `userInput = "字节"` → Step 0 调 `search_companies (searchKey: "字节")`，过滤存续后取 Top 5，向用户列出候选请求确认；用户回复"1"（北京字节跳动科技有限公司）→ `searchKey = items[0].creditCode`，再进 Step 1。
## V2.0 升级（向高管背调延伸）

可与 `tyc-exec-bg` 串联：完成 IC Memo 后自动对创始人/CEO 触发 30+ 维度个人风险扫描。
