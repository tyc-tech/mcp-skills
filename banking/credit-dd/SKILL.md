---
name: tyc-credit-dd
description: 授信审批前的企业全维度尽调底稿
category: banking
version: 1.0
---

# 授信尽调报告

## 触发条件

用户需要在信贷审批流程中对目标企业进行全维度风险尽调时触发。

关键词: 授信尽调、信贷审批、贷前调查、授信底稿

## 输入要求

- **keyword** (必填): 企业名称、公司ID、注册号或统一社会信用代码

## 执行流程

### Step 1: 工商信息全景

- 调用 `t1_cb_ic` (参数: keyword) — 组合接口一次性获取企业基本信息、主要人员、股东、对外投资、分支机构

### Step 2: 司法风险全景

- 调用 `t1_cb_judicial` (参数: keyword) — 组合接口获取法律诉讼、法院公告、开庭公告、失信人、被执行人、立案信息、送达公告

### Step 3: 经营风险

- 调用 `t1_mr_abnormal` (参数: keyword, pageNum, pageSize) — 经营异常
- 调用 `t1_mr_ownTax` (参数: keyword, pageNum, pageSize) — 欠税信息
- 调用 `t1_mr_taxContravention` (参数: keyword, pageNum, pageSize) — 税收违法

### Step 4: 担保抵押情况

- 调用 `t1_mr_equityInfo` (参数: keyword, pageNum, pageSize) — 股权出质
- 调用 `t1_mr_mortgageInfo` (参数: keyword, pageNum, pageSize) — 动产抵押
- 调用 `t1_mr_landMortgage` (参数: keyword, pageNum, pageSize) — 土地抵押

### Step 5: 上市信息（如适用）

- 调用 `t1_stock_companyInfo` (参数: keyword) — 上市公司简介
- 调用 `t1_stock_mainIndex` (参数: keyword) — 年度主要指标

### Step 6: 综合风险评估

- 调用 `t1_risk_riskInfo` (参数: keyword) — 天眼风险

## 输出格式

```markdown
# 授信尽调报告 — {企业名称}

## 一、企业概况
（基本信息、主要人员、注册资本、经营状态等）

## 二、股东与股权结构
（股东列表、对外投资、分支机构）

## 三、司法风险分析
| 维度 | 数量 | 风险等级 | 关键案件摘要 |

## 四、经营风险分析
| 维度 | 数量 | 风险等级 |
| 经营异常 | | |
| 欠税信息 | | |
| 税收违法 | | |

## 五、担保抵押情况
| 类型 | 笔数 | 涉及金额 |
| 股权出质 | | |
| 动产抵押 | | |
| 土地抵押 | | |

## 六、上市信息（如适用）
（主要财务指标、年度数据）

## 七、天眼风险综合评估

## 八、授信建议
- 风险等级: ...
- 授信建议: ...
- 需关注事项: ...
```

## 错误处理

- 上市信息接口若返回空数据，说明企业非上市公司，跳过第六章节
- 组合接口失败时，降级为逐个调用对应单接口

## 示例

输入: `keyword = "某大型制造集团有限公司"`

输出: 按上述模板生成完整授信尽调底稿。
