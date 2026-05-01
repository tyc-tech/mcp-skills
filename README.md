# 天眼查 T1.1 MCP Skills

> 面向行业场景的 AI Skill 集合，基于 T1.1 共 **162 个业务语义聚合 MCP Tool** 编排。
>
> 底层工具：[`conf/api-registry.yaml`](../../conf/api-registry.yaml)  ·  分层总表：[`docs/t1_1/tool-layers.md`](../../docs/t1_1/tool-layers.md)

## 工具分层（L0 / L1 / L2 / L3 + L2 优先 ★）

每个 Skill 内部工具引用都带 `[LX]` 徽章（含 `[L2★]`），表明其在 tyc-cli 4 层架构中的位置：

- **[L0] Resolve · 1**：实体锚定 — 把用户输入(简称 / 曾用名 / 模糊名)解析为精确企业（USCC + 全名）。Agent 第 0 跳，下游全靠它锚定主体
- **[L1] Overview · 6**：**6 facet × 1 队长** — company-base（`get_company_registration_info`）/ 风险 / 知产 / 经营信用 / 历史 / 董监高 各 1 个总览队长，拿到 USCC 后按 facet 选对应队长
- **[L2] Drill-down · 57**：一级数据维度展开（股权 / 诉讼 / 招投标 / 年报 …）；其中 **6 个为 ★ L2 优先（"L2★ 队副"，每 facet 1 个）**：company→`get_shareholder_info` / risk→`get_judicial_case` / ip→`get_trademark_info` / operation→`get_bidding_info` / history→`get_historical_registration` / executive→`get_person_risk_overview`，Agent 拿到 L1 `_summary` 不确定走哪个 L2 时默认推这 6 个之一
- **[L3] Specialized · 98**：ID 详情 / `search_*` / 上市专项 / 私募 / 建筑 / 人员微查询

Skill 执行流程内部的 Step 顺序一般遵循 L0 → L1 → L2 → L3 的下钻梯度。AI Agent 读 SKILL.md 时可以直接按徽章判断工具的信息密度与调用代价；Step 写"L1 之后该走哪个 L2"时，缺省可用对应 facet 的 ★ L2 优先作为安全默认。

CLI 侧工具发现入口：`tyc layers`（含 L2 优先 ★ 映射）/ `tyc L0 list` / `tyc L1 list` / `tyc L2 list`（★ 置顶）/ `tyc L3 list`。

### 分层设计亮点

- **LLM 选择率抗塌方**：LLM 工具选择准确率 >95%（≤10 工具）→ ~70%（30）→ <50%（100+）；本项目默认给 LLM 暴露 = L0 + L1 = 7 个，远低于"≤ 15"的工具暴露阈值
- **L0 只做实体锚定**：用户输入的常是简称/曾用名/模糊指代，把消歧从概览里剥离，Skill 第一步统一走 search_companies 拿到 USCC，避免下游在错主体上重复烧 token
- **L2 优先（★）= 6 facet × 1 队副**：57 个 L2 里挑 6 个高优先工具与 L1 facet 一一对齐，Skill 在"L1 → L2"那一跳缺省可推 ★；它**不进默认 LLM 暴露面**，只在渐进披露的下一跳显式标注
- **渐进披露 vs 扁平暴露**：竞品 QA 把 146 工具一次推给 LLM（其中 38 对相似度 > 0.8），本项目按 `_summary + drill_down` 按需加载，相同任务 token ≤ QA × 0.8
- **信息密度递减**：层号越小，单位 token 信息密度越高；Skill 写 Step 0 用 L0、Step 1 用 L1、Step 2-3 用 L2（不确定→★）、Step 4+ 用 L3 是默认模板
- **徽章即契约**：徽章同时出现在 Skill 文档、apis md（含 `[L2★]`）、CLI help、catalog.json 四处，事前透明；v0.4 改版期间 SSOT 暂以 `cli/t1_1/src/catalog.json` 为准（含 6 个 L2 工具的 `priority: true` 字段；详见项目根 CLAUDE.md "SSOT 漂移"小节）

## 核心特点

- **业务语义层**：每个 Skill 编排多个 T1.1 聚合工具（不直接接触 tyc 原子 API），屏蔽多源合并、时间戳格式化等底层细节
- **覆盖全面**：49 个 Skill 覆盖银行 / 投资 / 法务 / 律师 / 供应链 / 单供应商深度评估 6 大业务角色
- **TYC 强项数据**：8 个 Skill 利用 T1.1 在集团关联、股权图谱、上市财务、私募基金、建筑资质、园区地理等 TYC 强项数据源上设计
- **分层可观测**：每 Skill 工具清单按 L0/L1/L2/L3 徽章标注，AI Agent 能按信息密度挑选调用顺序

## 分类总览

| 分类 | 目标用户 | Skill 数量 | 目录 |
|------|---------|-----------|------|
| 银行合规 (banking) | 银行·合规风控 | 8 | [banking/](banking/) |
| 投资尽调 (invest) | 投资人 / FA | 19 | [invest/](invest/) |
| 法律法务 (legal) | 律师·法务 | 11 | [legal/](legal/) |
| 供应链 (supply) | 采购·供应链 | 6 | [supply/](supply/) |
| 集团与图谱 (group) | 风控·战略 | 2 | [group/](group/) |
| 行业专题 (industry) | 垂直行业 | 3 | [industry/](industry/) |
| **合计** | — | **50** | — |

### invest 分类细目（19 个）

| 类别 | Skill |
|-----|-------|
| 核心（10）| ic-memo / profile / equity-structure / competitor / exec-bg / funding-history / health-scan / bankruptcy-monitor / vc-research / financial-deep |
| 尽调与评估（9）| dd-checklist / credit-rating / merger-screening / industry-analysis / related-party / ip-due-diligence / litigation-analysis / esg-assessment / supply-chain-risk-fs |

### legal 分类细目（11 个）

| 类别 | Skill | 去向开源仓 |
|-----|-------|-----------|
| 法务向（8）| ad-compliance / contract-party / debt-recovery / ip-asset / ip-infringement / labor-compliance / legal-risk / license-validation | tyc-legal-assistant |
| 律师向（3）| contract-review / legal-dd / legal-research | tyc-vibe-lawyering |

### supply 分类细目（6 个）

| 类别 | Skill | 去向开源仓 |
|-----|-------|-----------|
| 采购流程（3）| new-supplier / supplier-annual / vendor | tyc-supply-chain（+ vendor 独立仓 tyc-vendor-assessment）|
| 风险与分析（3）| supply-risk / supply-chain-brief / spend-analysis | tyc-supply-chain |

## Skill 命名规范

- 命令前缀：`/tyc-`
- 格式：`/tyc-<场景英文简称>`
- 示例：`/tyc-kyb`、`/tyc-ic-memo`、`/tyc-equity-graph`

## Skill 文件结构

每个 Skill 一个目录，含一个 `SKILL.md`：

```
mcp_skills/t1_1/<category>/<skill-name>/SKILL.md
```

`SKILL.md` 标准结构（frontmatter + 6 段）：

```markdown
---
name: tyc-xxx
description: <一句话定位>
category: <分类英文>
version: 1.0
---

# <Skill 中文名>

## 触发条件
## 输入要求
## 执行流程        ← 调用 N 个 T1.1 工具的有序步骤
## 输出格式        ← Markdown 报告模板
## 错误处理
## 示例
```

## T1.1 底层工具映射

| Go 包 | 中文分类 | 工具数 | 常见用途 |
|-------|---------|--------|---------|
| `company` | 企业基础信息（合并） | 50 | 工商登记/股东/主要人员/财务数据 + 股权图谱（7）+ 集团（4）+ 企业搜索（4）+ 财务分析（11）+ 地理园区（5） |
| `risk` | 风险合规 | 35 | 司法/失信/被执行/破产/行政处罚 |
| `intellectual_property` | 知识产权（合并） | 14 | 专利/商标/著作权/知产出质 + 建筑资质（4） |
| `operation` | 经营与公示（合并） | 32 | 招投标/资质/许可/舆情/招聘 + 投资机构（5）+ 私募基金（4） |
| `history` | 历史信息 | 18 | 历史工商/历史司法/历史投资 |
| `executive` | 董监高 | 15 | 高管个人画像/控制企业/合作伙伴 |

完整工具清单见 [`docs/t1_1/apis/README.md`](../../docs/t1_1/apis/README.md)。

## 输出格式说明（CLI 端）

Skill 编排既可以通过 AI Agent 直接调 MCP Server `tools/call`，也可以通过 `tyc-cli` 在命令行调用（CLI 同样走 MCP 协议）。CLI 提供 3 种输出模式（互斥优先级：`--md` > `--pretty` > 默认）：

| 模式 | 适合场景 |
|------|---------|
| 默认（紧凑 JSON 单行） | 脚本管道、`jq` 解析、批处理 |
| `--pretty`（缩进 JSON） | 调试、日志归档 |
| `--md`（Markdown 表格） | 人类阅读、AI Agent 直接上屏（自动渲染 `_summary` / 元数据 / `items` 表格） |

> **AI Agent 编排建议**：当 Skill 输出最终面向用户展示时，建议在 CLI 调用层加 `--md`，让 Agent 直接获得格式化好的 Markdown 报告，省去再做模板套壳的成本。

## Skill 与底层工具

T1.1 Skill 基于业务语义聚合层的 162 个工具编排。每个工具内部自动并发调用多个 tyc 原子 API，按业务语义合并输出，且自动获得 `_summary`/`_empty` 元数据用于决策分支，编排步骤短（典型场景 5-6 步）。

## 与开源仓的关系

本目录（`mcp_skills/t1_1/`）是 **apimcp 主仓真源**，对应的开源规划见 [`docs/opensource-plan.md`](../../docs/opensource-plan.md)：

| 本目录 | 对应开源仓 | SKILL 数 |
|-------|-----------|---------|
| `banking/` | `tyc-banking-plugin` | 8 |
| `invest/` + `group/` + `industry/pf-compliance/` + `industry/park-research/` | `tyc-financial-services` | 23（19 + 2 + 1 + 1）|
| `legal/`（除 contract-review / legal-dd / legal-research）| `tyc-legal-assistant` | 8 |
| `legal/contract-review/` + `legal/legal-dd/` + `legal/legal-research/` | `tyc-vibe-lawyering` | 3 |
| `supply/`（除 vendor）+ `industry/construction-bid/` | `tyc-supply-chain` | 6（3 + 2 + 1） |
| `supply/vendor/` | `tyc-vendor-assessment` | 1 |
