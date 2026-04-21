---
name: tyc-trade-compliance
description: 跨境贸易与国际结算场景下的进出口企业合规核查
category: banking
version: 1.0
---

# 贸易融资合规核查

## 触发条件

用户需要对进出口企业进行贸易融资合规准入审查时触发。

关键词: 贸易融资、进出口、合规核查、开证、议付

## 输入要求

- **keyword** (必填): 企业名称、公司ID、注册号或统一社会信用代码
- **code** (可选): 注册号/统一社会信用代码（用于三要素核验）
- **name** (可选): 公司名称（用于三要素核验）
- **legalPersonName** (可选): 法人姓名（用于三要素核验）

## 执行流程

### Step 1: 主体核验

- 调用 `t1_ic_baseinfoV3` (参数: keyword) — 获取企业基本信息
- 调用 `t1_ic_verify` (参数: code, name, legalPersonName) — 三要素一致性验证（如提供了三要素参数）

### Step 2: 进出口信用

- 调用 `t1_m_importAndExport` (参数: keyword) — 获取海关注册编码、注册海关、经营类别等进出口信用信息

### Step 3: 经营资质许可

- 调用 `t1_m_getLicense` (参数: keyword, pageNum, pageSize) — 行政许可(工商)
- 调用 `t1_m_getAdministrativeLicense` (参数: keyword, pageNum, pageSize) — 行政许可(含有效期)

### Step 4: 处罚与异常

- 调用 `t1_mr_punishmentInfo` (参数: keyword, pageNum, pageSize) — 行政处罚
- 调用 `t1_mr_abnormal` (参数: keyword, pageNum, pageSize) — 经营异常

### Step 5: 失信与被执行

- 调用 `t1_jr_dishonest` (参数: keyword, pageNum, pageSize) — 失信被执行人
- 调用 `t1_jr_zhixinginfo` (参数: keyword, pageNum, pageSize) — 被执行人

### Step 6: 综合风险

- 调用 `t1_risk_riskInfo` (参数: keyword) — 天眼风险综合评估

## 输出格式

```markdown
# 贸易融资合规核查报告 — {企业名称}

## 一、主体核验
| 字段 | 值 |
（三要素验证结果）

## 二、进出口信用
| 海关注册编码 | 注册海关 | 经营类别 | 信用等级 |

## 三、经营资质
| 许可名称 | 许可机关 | 有效期 | 状态 |

## 四、风险扫描
| 风险类型 | 数量 |
| 行政处罚 | |
| 经营异常 | |
| 失信被执行 | |
| 被执行人 | |

## 五、综合评估
- 合规准入结论: 通过/不通过/需补充审查
- 风险等级: ...
- 建议: ...
```

## 错误处理

- 三要素核验接口仅在用户提供完整三要素参数时调用，否则跳过
- 进出口信用接口返回空时标注"未查询到进出口登记信息"

## 示例

输入: `keyword = "某进出口贸易有限公司"`

输出: 完整贸易融资合规核查报告。
