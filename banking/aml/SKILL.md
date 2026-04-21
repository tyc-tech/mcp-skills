---
name: tyc-aml
description: AML客户尽职调查，覆盖UBO穿透、个人风险、司法扫描
category: banking
version: 1.0
---

# AML 客户尽职调查

## 触发条件

用户需要对企业客户进行反洗钱尽职调查(CDD)或增强尽职调查(EDD)时触发。

关键词: AML、反洗钱、CDD、EDD、尽职调查、客户识别

## 输入要求

- **keyword** (必填): 企业名称、公司ID、注册号或统一社会信用代码

## 执行流程

### Step 1: 客户基本信息

- 调用 `t1_ic_baseinfoV3` (参数: keyword) — 获取企业基本信息和主要人员
- 调用 `t1_ic_companyType_v3` (参数: keyword) — 确认企业类型

### Step 2: 受益所有人(UBO)穿透

- 调用 `t1_ic_holder` (参数: keyword, pageNum, pageSize) — 获取股东列表
- 调用 `t1_ic_actualControl` (参数: keyword) — 识别疑似实际控制人
- 调用 `t1_ic_humanholding` (参数: keyword, pageNum, pageSize) — 识别最终受益人及持股比例

### Step 3: 投资关系网络

- 调用 `t1_services_v3_open_investtree` (参数: keyword) — 获取上下游投资关系树
- 调用 `t1_ic_inverst` (参数: keyword, pageNum, pageSize) — 获取对外投资信息

### Step 4: 司法风险扫描

- 调用 `t1_jr_lawSuit` (参数: keyword, pageNum, pageSize) — 法律诉讼
- 调用 `t1_jr_zhixinginfo` (参数: keyword, pageNum, pageSize) — 被执行人
- 调用 `t1_jr_dishonest` (参数: keyword, pageNum, pageSize) — 失信被执行人

### Step 5: 关键人员风险

- 从 Step 1 结果中提取法定代表人姓名(humanName)和企业名称(name)
- 调用 `t1_human_companyholding` (参数: name, humanName) — 法人控股企业
- 调用 `t1_services_v4_open_humanRiskInfo` (参数: name, humanName) — 法人个人风险
- 调用 `t1_services_v4_open_human_dishonest` (参数: name, humanName, pageNum, pageSize) — 法人失信记录

### Step 6: 综合风险评估

- 调用 `t1_risk_riskInfo` (参数: keyword) — 天眼风险综合评估

## 输出格式

```markdown
# AML 客户尽职调查报告 — {企业名称}

## 一、客户基本信息
| 字段 | 值 |
|------|-----|

## 二、受益所有人(UBO)穿透
### 股权结构
### 疑似实际控制人
### 最终受益人

## 三、投资关系网络
### 对外投资
### 投资关系树

## 四、司法风险
| 风险类型 | 数量 | 详情摘要 |

## 五、关键人员风险
### 法定代表人
| 姓名 | 控股企业数 | 个人风险项 | 失信记录 |

## 六、风险评级与建议
- 客户风险等级: 低/中/高
- 是否需要增强尽职调查(EDD): 是/否
- 建议措施: ...
```

## 错误处理

- 若关键人员风险接口调用失败，仅输出企业维度风险，标注"人员维度数据暂不可用"
- 若投资关系树接口不可用，使用对外投资数据替代

## 示例

输入: `keyword = "北京某科技有限公司"`

输出: 完整 AML 尽调报告，覆盖客户信息、UBO穿透、司法风险、人员风险及评级建议。
