---
name: tyc-counterparty
description: 贸易融资业务中交易对手的多维风险评估与评分
category: banking
version: 1.0
---

# 交易对手风险评估

## 触发条件

用户需要评估贸易融资、供应链金融中交易对手方的信用风险时触发。

关键词: 交易对手、对手方风险、信用评估、违约风险

## 输入要求

- **keyword** (必填): 企业名称、公司ID、注册号或统一社会信用代码

## 执行流程

### Step 1: 基本信息

- 调用 `t1_ic_baseinfoV3` (参数: keyword) — 获取企业基本信息

### Step 2: 司法风险

- 调用 `t1_jr_lawSuit` (参数: keyword, pageNum, pageSize) — 法律诉讼
- 调用 `t1_jr_zhixinginfo` (参数: keyword, pageNum, pageSize) — 被执行人
- 调用 `t1_jr_dishonest` (参数: keyword, pageNum, pageSize) — 失信被执行人

### Step 3: 经营风险

- 调用 `t1_mr_punishmentInfo` (参数: keyword, pageNum, pageSize) — 行政处罚
- 调用 `t1_mr_abnormal` (参数: keyword, pageNum, pageSize) — 经营异常
- 调用 `t1_mr_ownTax` (参数: keyword, pageNum, pageSize) — 欠税信息

### Step 4: 进出口信用

- 调用 `t1_m_importAndExport` (参数: keyword) — 海关注册与进出口信用

### Step 5: 信用评级

- 调用 `t1_m_creditRating` (参数: keyword, pageNum, pageSize) — 信用评级信息

### Step 6: 综合风险

- 调用 `t1_risk_riskInfo` (参数: keyword) — 天眼风险综合评估

## 输出格式

```markdown
# 交易对手风险评估报告 — {企业名称}

## 一、对手方基本信息
| 字段 | 值 |

## 二、风险评分矩阵
| 维度 | 指标 | 得分 | 风险等级 |
|------|------|------|---------|
| 司法风险 | 诉讼/被执行/失信 | | |
| 经营风险 | 处罚/异常/欠税 | | |
| 信用资质 | 进出口信用/信用评级 | | |
| 综合风险 | 天眼风险评分 | | |

## 三、司法风险详情
| 类型 | 数量 | 关键案件 |

## 四、经营风险详情
| 类型 | 数量 | 详情 |

## 五、信用资质
（进出口信用等级、外部评级）

## 六、综合评估
- 交易对手风险等级: 低/中/高
- 违约概率评估: ...
- 交易建议: ...
```

## 错误处理

- 进出口信用和信用评级接口若返回空，标注"无相关记录"
- 风险评分矩阵中缺失维度标注"N/A"

## 示例

输入: `keyword = "某国际贸易有限公司"`

输出: 完整交易对手风险评估报告。
