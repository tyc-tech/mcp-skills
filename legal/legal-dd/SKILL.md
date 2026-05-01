---
name: tyc-legal-dd
description: 法律尽职调查（10 大板块底稿 + 报告）— 主体 / 股权 / 治理 / 资产 / 经营 / 财税 / 劳动 / 债权 / 诉讼 / 其他
category: legal
version: 1.0
---

# 法律尽职调查

## 触发条件

律师事务所承接法律尽调项目、拟 IPO 法律意见书撰写、并购交易买方法律尽调、合规审查项目启动时触发。

关键词：法律尽调、DD 底稿、尽调报告、法律意见书、LDD

## 输入要求

用户输入的企业标识可能是以下三种形式之一，**Step 0 会按形式分流**，不要急着把用户原话当成 `searchKey` 喂给后续步骤：

| 输入形式 | 示例 | 是否需要 L0 实体锚定 |
|---|---|---|
| **完整企业名**（含组织形式后缀：`有限公司` / `股份有限公司` / `集团` / `合伙企业` / `个体工商户` / `事务所` / `中心` 等） | `北京字节跳动科技有限公司` / `腾讯科技（深圳）有限公司` | ❌ 跳过 L0，直接当 `searchKey` |
| **统一社会信用代码（USCC）** 18 位大写字母+数字 | `91110108551385082Q` | ❌ 跳过 L0，直接当 `searchKey` |
| **企业简称 / 曾用名 / 品牌名 / 模糊指代** | `字节` / `抖音` / `今日头条` / `乐视` / `阿里` | ✅ **必须先走 L0**，由用户在候选中确认唯一企业，再拿 `creditCode` 作为 `searchKey` |

> 经过 Step 0 锚定后，下文 Step 1+ 的 `searchKey` 都指**精确企业名或 USCC**，调用方式不变。

## 执行流程（10 大板块并发调用）

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

### 板块 1: 主体资格
- `get_company_registration_info` — 工商登记
- `get_actual_controller` — 实控人核验

### 板块 2: 股权结构
- `get_shareholder_info` — 股东构成
- `get_equity_tree` — 多层股权树
- `get_beneficial_owners` — UBO

### 板块 3: 公司治理
- `get_key_personnel` — 董事 / 监事 / 高管
- `get_branches` — 分支机构

### 板块 4: 核心资产
- `get_external_investments` — 对外投资
- `get_patent_info` / `get_trademark_info` / `get_software_copyright_info` — 知产资产
- `get_land_mortgage_info` — 土地不动产

### 板块 5: 业务经营
- `get_products_info` — 主营业务
- `get_qualifications` — 资质
- `get_administrative_license` — 行政许可

### 板块 6: 财税
- `get_financial_data` — 财务
- `get_tax_arrears_notice` — 欠税
- `get_tax_violation` — 税收违法

### 板块 7: 劳动人事
- `get_company_scale` — 参保人数 / 规模
- `get_recruitment_info` — 招聘动态

### 板块 8: 债权债务
- `get_equity_pledge_info` — 股权出质
- `get_chattel_mortgage_info` — 动产抵押
- `get_judgment_debtor_info` — 被执行人

### 板块 9: 诉讼仲裁
- `get_judicial_documents` — 裁判文书
- `get_judicial_case` — 司法案件
- `get_lawsuit_detail` — 重大案件详情
- `get_dishonest_info` — 失信
- `get_administrative_penalty` — 行政处罚

### 板块 10: 其他事项
- `get_risk_overview` — 风险总览
- `get_historical_overview` — 历史信息

## 输出格式

```markdown
# 法律尽职调查底稿 — {name}

> 律所: ... · 项目代号: ... · 撰写律师: AI 辅助 · 出具: {ISO8601}
> 数据来源: 天眼查 OpenAPI（公开商业数据）+ 客户提供资料
> 底稿状态: {draft / report}

## 第一章 主体资格
1. 企业名称: {name}
2. 统一社会信用代码: {creditCode}
3. 注册资本 / 实缴资本: {regCapital} / {actualCapital}
4. 法定代表人: {legalPersonName}
5. 实际控制人: {actualController.name}（累计持股 {totalRatio}）
6. 成立日期: {estiblishTime}
7. 存续状态: {regStatus}

**风险提示**:
- 🔴 高: ...
- 🟡 中: ...

## 第二章 股权结构
（TOP 10 股东表 + 股权穿透 Mermaid 图 + UBO 清单）

## 第三章 公司治理
（三会一层 / 董监高清单 / 分支机构 / 授权链条）

## 第四章 核心资产
4.1 知识产权（专利 {n} / 商标 {n} / 软著 {n}）
4.2 不动产与土地
4.3 对外投资（子公司 {n} 家）

## 第五章 业务经营
5.1 主营业务与产品
5.2 资质许可清单（需说明有效期 / 延续要求）
5.3 行政许可

## 第六章 财税
6.1 财务数据（营收 / 利润 / 资产负债）
6.2 欠税 / 税收违法
**风险评级**: 低 / 中 / 高

## 第七章 劳动人事
7.1 员工规模与参保
7.2 劳动监察黑名单

## 第八章 债权债务
8.1 担保与抵押（股权出质 / 动产抵押）
8.2 被执行人记录

## 第九章 诉讼仲裁
9.1 案件总数与类型分布（原 / 被告位）
9.2 重大案件 TOP 5 深度
9.3 失信与限高

## 第十章 其他事项
10.1 历史信息（工商变更 / 曾用名）
10.2 综合风险

---

## 尽调结论
- **法律风险等级**: 低 / 中 / 高 / 重大
- **重大不利事项**: ...
- **投资 / 交易影响**: 可推进 / 需修订交易结构 / 建议终止
- **后续律师关注点**: 现场访谈、合同调阅、工商档案调档、政府走访
```

## 错误处理

- 若 Step 0 候选 = 0 → 终止流程，提示"未找到匹配企业，请提供更完整的名称或 USCC"，不要带着错主体往下跑
- 若 Step 0 候选 ≥ 2 而用户在合理时间内未回复 → 暂存上下文，**不要自行选择**第一条作为锚定（错锚定的代价远大于等待）
- 单板块数据缺失 → 对应章节标注 `[!] 数据暂不可用，建议现场调档`
- `_empty` → 标注"经查无公开记录"（不影响整体结论）
- tyc `error_code: 300000` → 归一化为"经查无结果"（非报错）

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

- **投资向快速尽调**（30 秒）→ `/tyc-ic-memo` (invest)
- **关联方穿透** → `/tyc-related-party` (invest)
- **法律研究 + 案例检索** → `/tyc-legal-research`
- **合同审查** → `/tyc-contract-review`（vibe 向，带批注 docx）
