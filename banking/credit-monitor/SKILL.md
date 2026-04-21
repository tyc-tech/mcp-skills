---
name: tyc-credit-monitor
description: 贷后存量借款客户的持续风险监控与变化预警
category: banking
version: 1.0
---

# 信贷风险定期监控

## 触发条件

用户需要对存量贷款客户进行定期风险监控、贷后管理检查时触发。

关键词: 贷后管理、风险监控、信贷监控、风险预警、定期体检

## 输入要求

- **keyword** (必填): 企业名称、公司ID、注册号或统一社会信用代码

## 执行流程

### Step 1: 综合风险快照

- 调用 `t1_risk_riskInfo` (参数: keyword) — 获取当前天眼风险综合评估

### Step 2: 司法风险变化

- 调用 `t1_jr_lawSuit` (参数: keyword, pageNum, pageSize) — 最新法律诉讼
- 调用 `t1_jr_zhixinginfo` (参数: keyword, pageNum, pageSize) — 最新被执行人
- 调用 `t1_jr_dishonest` (参数: keyword, pageNum, pageSize) — 最新失信信息

### Step 3: 经营异常变化

- 调用 `t1_mr_abnormal` (参数: keyword, pageNum, pageSize) — 经营异常
- 调用 `t1_mr_illegalinfo` (参数: keyword, pageNum, pageSize) — 严重违法

### Step 4: 工商变更记录

- 调用 `t1_ic_changeinfo` (参数: keyword, pageNum, pageSize) — 变更记录（法人变更、注册资本变更、股东变更等）

### Step 5: 历史股东变更

- 调用 `t1_hi_holder` (参数: keyword, pageNum, pageSize) — 历史股东变动

## 输出格式

```markdown
# 信贷风险监控报告 — {企业名称}

> 监控时间: {当前时间}

## 一、风险快照
（天眼风险综合评估结果）

## 二、司法风险动态
| 类型 | 当前数量 | 最新变化 | 关注事项 |
| 法律诉讼 | | 新增X起 | |
| 被执行人 | | | |
| 失信被执行 | | | |

## 三、经营状态动态
| 类型 | 状态 | 变化情况 |
| 经营异常 | | |
| 严重违法 | | |

## 四、工商变更预警
| 变更事项 | 变更前 | 变更后 | 变更时间 |
（重点关注: 法人变更、注册资本减少、股东退出）

## 五、股东变动
| 股东名 | 变动类型 | 时间 |

## 六、监控结论
- 风险变化趋势: 恶化/稳定/改善
- 预警等级: 正常/关注/警告/危险
- 建议措施: ...
```

## 错误处理

- 各接口独立调用，单个接口失败不影响整体报告
- 标注各维度数据的获取时间，方便与上期对比

## 示例

输入: `keyword = "某建设工程有限公司"`

输出: 完整信贷风险监控报告。
