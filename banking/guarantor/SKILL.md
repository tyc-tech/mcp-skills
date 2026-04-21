---
name: tyc-guarantor
description: 贷款担保审批前的担保方资信与偿付能力评估
category: banking
version: 1.0
---

# 担保方资信核查

## 触发条件

用户需要在贷款担保审批前对担保企业进行资信核查、评估实质偿付能力时触发。

关键词: 担保核查、担保方、资信、偿付能力、抵押、出质

## 输入要求

- **keyword** (必填): 担保企业名称、公司ID、注册号或统一社会信用代码

## 执行流程

### Step 1: 担保方基本信息

- 调用 `t1_ic_baseinfoV3` (参数: keyword) — 获取企业基本信息、注册资本、经营状态

### Step 2: 股权出质情况

- 调用 `t1_mr_equityInfo` (参数: keyword, pageNum, pageSize) — 查询股权出质信息

### Step 3: 动产抵押情况

- 调用 `t1_mr_mortgageInfo` (参数: keyword, pageNum, pageSize) — 查询动产抵押信息

### Step 4: 土地抵押情况

- 调用 `t1_mr_landMortgage` (参数: keyword, pageNum, pageSize) — 查询土地抵押信息

### Step 5: 执行与失信

- 调用 `t1_jr_zhixinginfo` (参数: keyword, pageNum, pageSize) — 被执行人信息
- 调用 `t1_jr_dishonest` (参数: keyword, pageNum, pageSize) — 失信被执行人
- 调用 `t1_jr_endCase` (参数: keyword, pageNum, pageSize) — 终本案件

### Step 6: 股权质押比例（如为上市公司）

- 调用 `t1_mr_stockPledge_ratio` (参数: keyword) — 股权质押比例

### Step 7: 综合风险

- 调用 `t1_risk_riskInfo` (参数: keyword) — 天眼风险综合评估

## 输出格式

```markdown
# 担保方资信核查报告 — {企业名称}

## 一、担保方基本信息
| 字段 | 值 |

## 二、资产占用情况
| 类型 | 笔数 | 涉及金额/数额 | 状态 |
| 股权出质 | | | |
| 动产抵押 | | | |
| 土地抵押 | | | |
| 股权质押 | | | |

## 三、执行与失信记录
| 类型 | 数量 | 关键案件 |
| 被执行人 | | |
| 失信被执行 | | |
| 终本案件 | | |

## 四、天眼风险评估

## 五、偿付能力评估结论
- 资产占用率: 高/中/低
- 执行风险: 有/无
- 担保有效性判断: ...
- 建议: ...
```

## 错误处理

- 股权质押比例接口如返回空，说明非上市公司，跳过该项
- 土地抵押数据如不可用，标注"土地抵押数据暂缺"

## 示例

输入: `keyword = "某担保集团有限公司"`

输出: 完整担保方资信核查报告。
