# Skill格式规范 v1.0

**文档性质**：技术标准文档  
**创建日期**：2026-02-12  
**适用对象**：开发团队/运营团队  
**维护状态**：活跃维护

---

## V0能力矩阵（强制遵守）

| 能力点 | V0状态 | 说明 | V1+计划 |
|--------|--------|------|---------|
| **Skill定义** | | | |
| `eligibility.logic` (all/any) | ✅ 已实现 | 仅支持顶层单层logic | V1支持嵌套logic |
| `operator`: >=, <=, ==, !=, in, not_in | ✅ 已实现 | 完整支持 | |
| `operator`: between | ✅ 已实现 | 双端点包含 [min, max] | |
| `operator`: contains, matches | ✅ 已实现 | 字符串匹配 | |
| `amount_calculation`: expression | ✅ 已实现 | 基础算术+min/max/abs/round | V1支持更多函数 |
| `amount_calculation`: table | ✅ 已实现 | 左闭右开 [min, max) | |
| `amount_calculation`: ml_model | 🚫 V0禁止 | 预留字段，V0校验器拒绝 | V1实现 |
| **热更新** | | | |
| 手动reload API | ✅ 已实现 | POST /api/v1/reload-skills | |
| 缓存TTL自动重载 | ✅ 已实现 | 默认60秒 | |
| 文件监听自动重载 | 🚫 V0不做 | | V1实现watchdog |
| **租户隔离** | | | |
| partner_id多租户 | 🚫 V0不做 | V0 固定 partner_id="DEFAULT"，字段保留供 V1 扩展；代码/测试一律用 DEFAULT | V1实现租户隔离 |
| **内容自动化** | | | |
| content_generation/LLM | 🚫 V0不做 | 移至附录 | V2.0公众号自动化 |
| **字段契约** | | | |
| 13字段SSOT | ✅ 已实现 | 单位统一为"元" | |
| CustomerV0最小字段集 | ✅ 已实现 | 见下方定义 | V1扩展更多字段 |

### V0字段契约（强制）

**CustomerV0最小字段集**（必需）：
```python
customer_id: str              # 客户ID
monthly_revenue: float        # 月均流水（元）
annual_tax: float             # 年纳税（元）
credit_query_6m: int          # 近6月征信查询次数
credit_overdue_m3: int        # M3+逾期次数
company_age_years: int        # 企业成立年限（年）
industry: str                 # 行业
created_at: datetime
updated_at: datetime
```

**CustomerV0可选字段**：
- annual_revenue, revenue_stability, customer_concentration
- annual_invoice, tax_grade
- 其他征信/资产/资质字段

**单位约定**（强制）：
- 金额类字段统一为"元"（不是万元）
- 时间类字段统一为"年"、"月"、"天"
- 比例类字段统一为小数（0-1），如0.85表示85%

---

## 一、Skill定义概述

### 什么是Skill

Skill是DCC2026系统中的**声明式配置单元**，用于定义业务规则、匹配逻辑、执行策略等。

**核心特点**：
- 声明式（描述"要做什么"，而非"怎么做"）
- 可热更新（修改后立即生效，无需重启）
- 版本化（所有变更可追溯、可回滚）
- 可审计（谁在什么时间做了什么修改）

**三种Skill类型**：
1. **产品Skill**：定义银行产品的准入条件、额度计算规则
2. **方案Skill**：定义改造方案的适用场景、执行步骤
3. **策略Skill**：定义提醒策略、合规规则等

---

## 二、通用元数据规范

所有Skill必须包含以下元数据。**`metadata` 中的字段为唯一权威源**，系统加载器、Runner、API 均以 `metadata` 为准。

```yaml
# ============================================
# 元数据（权威字段源）
# ============================================
metadata:
  skill_id: "UNIQUE_SKILL_ID"          # 必需，唯一标识符
  skill_name: "Skill显示名称"          # 必需，显示名称
  skill_type: "product"                # 必需，类型：product/solution/strategy
  
  # 来源信息
  source: "招商银行官网"               # 信息来源（必需）
  source_url: "https://..."            # 来源URL（可选）
  verified_date: "2026-02-10"          # 核验日期（必需，YYYY-MM-DD）
  verified_by: "张三"                  # 核验人（必需）
  
  # 适用范围
  region: "全国"                       # 适用地区：全国/省份/城市（必需）
  confidence: "高"                     # 可信度：高/中/低（必需）
  
  # 版本控制
  version: "1.2.0"                     # 版本号（必需，遵循语义化版本）
  created_at: "2026-02-01T10:00:00Z"   # 创建时间（ISO 8601格式）
  updated_at: "2026-02-10T15:30:00Z"   # 更新时间（ISO 8601格式）
  created_by: "admin@dcc2026.com"      # 创建人（必需）
  updated_by: "admin@dcc2026.com"      # 更新人（必需）
  
  # 变更日志
  changelog:
    - version: "1.2.0"
      date: "2026-02-10"
      changes: "提高流水要求从5万到10万"
      reason: "回填数据显示拒批率过高"
      approved_by: "产品负责人"
    - version: "1.1.0"
      date: "2026-02-05"
      changes: "新增行业限制：房地产、金融"
      reason: "银行政策调整"
      approved_by: "合规负责人"
    - version: "1.0.0"
      date: "2026-02-01"
      changes: "初始版本"
      reason: "新产品上线"
      approved_by: "产品负责人"
  
  # 状态标记
  status: "active"                     # 状态：active/inactive/deprecated（必需）
  deprecated_reason: ""                # 废弃原因（status=deprecated时必需）
  replacement_skill_id: ""             # 替代Skill ID（status=deprecated时可选）
```

**说明**：顶层可以重复写 `skill_id`、`skill_name`、`skill_type` 作为快捷访问，但必须与 `metadata` 中的值完全一致，否则校验失败。

### 元数据权威性规则（V0强制）

1. **权威源**：`metadata` 中的字段为唯一权威。
2. **顶层快捷字段**（可选）：
   - 可在顶层写 `skill_id`、`skill_name`、`skill_type`；
   - 若存在，必须与 `metadata` 中的值完全一致；
   - 校验器发现不一致时直接拒绝加载。
3. **系统行为**：
   - Skill 加载器、Runner、API 全部使用 `metadata` 中的值；
   - 顶层字段仅用于 YAML 可读性（快速定位）。

---

## 三、产品Skill规范

### 3.1 完整示例

```yaml
# ============================================
# 产品Skill：招商银行税贷
# ============================================

# 基本信息
skill_id: "PRODUCT_CMB_TAX_001"
skill_name: "招商银行税贷"
skill_type: "product"

# 产品属性
product:
  bank: "招商银行"
  bank_code: "CMB"
  category: "税贷"
  product_code: "TAX_LOAN_001"
  
  # 产品特点（描述性信息）
  features:
    - "无需抵押"
    - "纯信用贷款"
    - "线上申请"
    - "快速审批"
  
  # 利率信息
  interest_rate:
    type: "range"                      # range/fixed/floating
    min: 0.036                         # 年化利率3.6%
    max: 0.072                         # 年化利率7.2%
    description: "年化利率3.6%-7.2%，具体以审批为准"
  
  # 期限信息
  term:
    unit: "month"                      # month/year
    options: [12, 24, 36]              # 可选期限
    description: "最长3年"
  
  # 还款方式
  repayment_methods:
    - "等额本息"
    - "先息后本"
    - "随借随还"

# ============================================
# 准入条件（核心规则）
# ============================================
eligibility:
  # 规则类型：all（所有条件必须满足）/ any（任意条件满足）
  logic: "all"
  
  # 具体条件
  conditions:
    # 流水要求
    - field: "monthly_revenue"
      operator: ">="
      value: 100000
      description: "月均流水≥10万元"
      dimension: "流水体质"              # 对应六宫格维度
    
    # 纳税要求
    - field: "annual_tax"
      operator: ">="
      value: 10000
      description: "年纳税≥1万元"
      dimension: "票税体质"
    
    # 征信要求
    - field: "credit_query_6m"
      operator: "<="
      value: 6
      description: "近6个月征信查询≤6次"
      dimension: "征信体质"
    
    - field: "credit_overdue_m3"
      operator: "=="
      value: 0
      description: "近2年无M3+逾期"
      dimension: "征信体质"
    
    # 企业年限
    - field: "company_age_years"
      operator: ">="
      value: 2
      description: "企业成立≥2年"
      dimension: "资质体质"
    
    # 行业限制（黑名单）
    - field: "industry"
      operator: "not_in"
      value:
        - "房地产开发"
        - "金融投资"
        - "娱乐业"
        - "批发零售"
      description: "不支持以上行业"
      dimension: "行业体质"

# ============================================
# 额度计算规则
# ============================================
amount_calculation:
  # 公式类型
  formula_type: "expression"           # expression/table/ml_model
  
  # 计算表达式
  formula: "min(annual_tax * 30, 1000000)"
  
  # 表达式说明
  formula_description: |
    额度 = min(年纳税额 × 30倍, 100万)
    即：以年纳税额的30倍作为基础额度，但不超过100万
  
  # 可用变量（供表达式使用）
  variables:
    - name: "annual_tax"
      description: "年纳税额"
      unit: "元"
    - name: "monthly_revenue"
      description: "月均流水"
      unit: "元"
  
  # 额度上下限
  amount_range:
    min: 50000                         # 最小5万
    max: 1000000                       # 最大100万
    unit: "元"
  
  # 特殊规则（可选）
  special_rules:
    - condition: "high_tech_certification == true"
      adjustment: "* 1.2"
      description: "高新技术企业额度上浮20%"

# ============================================
# 合规与免责
# ============================================
compliance:
  # 免责声明（系统级固定，不可修改）
  disclaimer: |
    1. 以上额度仅供初步评估，实际额度以银行审批为准
    2. 不构成贷款承诺或要约
    3. 银行有权根据风险评估调整额度或拒绝申请
  
  # 风险提示
  risk_warnings:
    - "征信查询次数过多会影响审批"
    - "流水不稳定可能导致额度降低"
    - "企业成立时间不足可能无法通过审批"
  
  # 禁止场景
  forbidden_scenarios:
    - "不得用于投资理财"
    - "不得用于房地产投机"
    - "不得用于还旧债"

# ============================================
# 元数据
# ============================================
metadata:
  skill_id: "PRODUCT_CMB_TAX_001"
  skill_name: "招商银行税贷"
  skill_type: "product"
  
  source: "招商银行官网"
  source_url: "https://www.cmbchina.com/loan/tax"
  verified_date: "2026-02-10"
  verified_by: "张三"
  
  region: "全国"
  confidence: "高"
  
  version: "1.2.0"
  created_at: "2026-02-01T10:00:00Z"
  updated_at: "2026-02-10T15:30:00Z"
  created_by: "admin@dcc2026.com"
  updated_by: "admin@dcc2026.com"
  
  changelog:
    - version: "1.2.0"
      date: "2026-02-10"
      changes: "提高流水要求从5万到10万"
      reason: "回填数据显示拒批率过高"
      approved_by: "产品负责人"
    - version: "1.1.0"
      date: "2026-02-05"
      changes: "新增行业限制"
      reason: "银行政策调整"
      approved_by: "合规负责人"
    - version: "1.0.0"
      date: "2026-02-01"
      changes: "初始版本"
      reason: "新产品上线"
      approved_by: "产品负责人"
  
  status: "active"
```

### 3.2 字段详解

#### eligibility.conditions（准入条件）

**支持的操作符**：

| 操作符 | 说明 | 示例 |
|--------|------|------|
| `>=` | 大于等于 | `monthly_revenue >= 100000` |
| `<=` | 小于等于 | `credit_query_6m <= 6` |
| `>` | 大于 | `company_age_years > 1` |
| `<` | 小于 | `debt_ratio < 0.7` |
| `==` | 等于 | `credit_overdue_m3 == 0` |
| `!=` | 不等于 | `industry != "房地产"` |
| `in` | 在列表中 | `region in ["北京", "上海"]` |
| `not_in` | 不在列表中 | `industry not_in ["房地产", "金融"]` |
| `between` | 在范围内 | `annual_revenue between [100000, 500000]` |
| `contains` | 包含（字符串） | `company_name contains "科技"` |
| `matches` | 正则匹配 | `phone matches "^1[3-9]\d{9}$"` |

#### 操作符语义详解（V0契约）

**1. between 操作符**：
- 语义：双端点包含 `[min, max]`
- 示例：`annual_tax between [10000, 50000]` → 10000 ≤ annual_tax ≤ 50000
- 原因：业务阈值更符合“达到即满足”

**2. table 查表区间**：
- 语义：左闭右开 `[min, max)`
- 示例：
```yaml
  table:
    - range: [0, 10000]      # 0 ≤ x < 10000
      amount: 0
    - range: [10000, 50000]  # 10000 ≤ x < 50000
      amount: 300000
    - range: [50000, 999999999]  # 50000 ≤ x < 999999999
      amount: 500000
```
- 要求：区间不得重叠，必须覆盖所有可能值
- 校验：Validator 必须检查区间连续性

**3. contains / matches 操作符**：
- `contains`：字符串包含（不区分大小写）
- `matches`：正则表达式完全匹配

**单元测试覆盖要求**：
- between 端点：至少 2 条测试（边界值 min、max）
- table 区间：至少 3 条测试（区间内、区间边界、未覆盖）

**condition对象结构**：

```yaml
- field: "字段名"                      # 必需
  operator: "操作符"                   # 必需
  value: 值                           # 必需（根据operator类型不同）
  description: "可读描述"              # 必需（供用户理解）
  dimension: "六宫格维度"              # 必需（流水/票税/征信/资产/资质/行业）
  error_message: "不满足时的提示"      # 可选（自定义错误提示）
```

#### amount_calculation（额度计算）

**支持的公式类型**：

**1. expression（表达式）**：✅ V0 已实现
```yaml
formula_type: "expression"
formula: "annual_tax * 30"
```

**2. table（查表）**：✅ V0 已实现
```yaml
formula_type: "table"
lookup_field: "annual_tax"
table:
  - range: [0, 10000]      # 左闭右开 [min, max)
    amount: 0
  - range: [10000, 50000]
    amount: 300000
  - range: [50000, 100000]
    amount: 1500000
  - range: [100000, 999999999]
    amount: 3000000
```

**3. ml_model（机器学习模型）**：🚫 V0 未实现（预留）
```yaml
formula_type: "ml_model"
model_id: "amount_predictor_v1"
# V0 校验器会拒绝加载此类 Skill
```

**V0 验证行为**：
- 若 `formula_type: "ml_model"`，校验器抛出错误：
  ```
  ValidationError: formula_type 'ml_model' is reserved for V1+
  ```
- V1 版本实现后移除此限制。

---

## 四、方案Skill规范

### 4.1 完整示例

```yaml
# ============================================
# 方案Skill：流水优化术
# ============================================

# 基本信息
skill_id: "SOLUTION_CASHFLOW_001"
skill_name: "流水优化术"
skill_type: "solution"

# 方案属性
solution:
  category: "流水改造"
  target_dimension: "流水体质"          # 主要作用于哪个维度
  
  # 方案标签
  tags:
    - "无成本"
    - "1-2个月"
    - "成功率75%"

# ============================================
# 适用场景
# ============================================
applicable_when:
  # 适用条件（多条件用logic控制）
  logic: "all"                         # all/any
  
  conditions:
    # 差距类型
    - field: "gap_category"
      operator: "=="
      value: "流水不足"
      description: "诊断结果显示流水不足"
    
    # 差距严重程度
    - field: "gap_amount"
      operator: ">="
      value: 50000
      description: "差距金额≥5万元"
    
    # 客户配合度
    - field: "customer_cooperation"
      operator: "=="
      value: "高"
      description: "客户配合度高（可选条件）"

# ============================================
# 执行步骤
# ============================================
steps:
  - step_id: "STEP_001"
    step_number: 1
    title: "梳理现有客户结构"
    
    description: |
      详细分析企业的客户结构，识别大客户依赖风险。
      
      具体包括：
      1. 列出前10大客户及其贡献占比
      2. 计算客户集中度（前5大客户占比）
      3. 识别单一客户依赖风险
    
    # 预期产出
    output:
      type: "document"
      name: "客户集中度分析表"
      template_url: "/templates/customer_concentration.xlsx"
    
    # 执行时间
    duration:
      value: 3
      unit: "days"
      description: "预计3个工作日"
    
    # 执行成本
    cost:
      value: 0
      unit: "元"
      description: "无额外成本"
    
    # 关键行动
    actions:
      - "收集近12个月对公流水明细"
      - "统计每个客户的累计收款金额"
      - "计算客户集中度指标"
    
    # 注意事项
    notes:
      - "需要完整的对公流水数据"
      - "建议使用Excel透视表功能"
  
  - step_id: "STEP_002"
    step_number: 2
    title: "优化收款方式"
    
    description: |
      引导客户通过对公账户支付，减少私户收款。
      
      具体措施：
      1. 与主要客户沟通，说明对公收款的重要性
      2. 在合同中明确约定对公账户收款
      3. 设置激励机制（如小额优惠）引导对公付款
    
    output:
      type: "plan"
      name: "收款方式优化方案"
      template_url: "/templates/payment_optimization.docx"
    
    duration:
      value: 7
      unit: "days"
      description: "预计1周"
    
    cost:
      value: 1000
      unit: "元"
      description: "激励成本预估"
    
    actions:
      - "起草客户沟通话术"
      - "修改销售合同模板"
      - "设计激励方案"
      - "逐个客户沟通推进"
    
    notes:
      - "部分客户可能不配合"
      - "需要一定沟通技巧"
      - "可能影响短期收款速度"
  
  - step_id: "STEP_003"
    step_number: 3
    title: "增加流水稳定性"
    
    description: |
      通过签订长期合同，固定月度收款，提高流水稳定性。
      
      具体措施：
      1. 与主要客户商谈年度合作协议
      2. 设定月度最低采购量
      3. 约定固定月结日期
    
    output:
      type: "contract"
      name: "长期合作协议模板"
      template_url: "/templates/longterm_contract.docx"
    
    duration:
      value: 30
      unit: "days"
      description: "预计1个月"
    
    cost:
      value: 5000
      unit: "元"
      description: "合同审查及商务成本"
    
    actions:
      - "筛选适合的客户"
      - "起草长期协议草案"
      - "商务谈判"
      - "签订正式协议"
    
    notes:
      - "不是所有客户都适合长期协议"
      - "需要给客户一定优惠条件"
      - "协议执行需要跟进"

# ============================================
# 预估成果
# ============================================
estimated_result:
  # 时间成本
  time_cost:
    min: 30
    max: 60
    unit: "days"
    description: "1-2个月"
  
  # 金钱成本
  money_cost:
    min: 0
    max: 10000
    unit: "元"
    description: "0-1万元"
  
  # 成功率
  success_rate:
    value: 0.75
    description: "75%"
    sample_size: 50
    data_source: "历史案例库"
  
  # 预期改善
  improvement:
    - dimension: "流水体质"
      metric: "月均流水"
      improvement_range: "提升20%-50%"
    - dimension: "流水体质"
      metric: "流水稳定性"
      improvement_range: "显著改善"

# ============================================
# 风险与注意事项
# ============================================
risks:
  - risk_id: "RISK_001"
    description: "客户可能不配合对公收款"
    probability: "中"               # 高/中/低
    impact: "中"                    # 高/中/低
    mitigation: "提前沟通，说明对公收款对企业融资的重要性"
  
  - risk_id: "RISK_002"
    description: "需要一定时间积累，无法立即见效"
    probability: "高"
    impact: "低"
    mitigation: "设定合理预期，告知客户需要1-2个月"
  
  - risk_id: "RISK_003"
    description: "可能影响与客户的关系"
    probability: "低"
    impact: "高"
    mitigation: "强调是为了企业发展，而非强制要求"

# ============================================
# 成功案例（可选）
# ============================================
success_cases:
  - case_id: "CASE_001"
    company_name: "客户A（脱敏）"
    industry: "软件开发"
    before:
      monthly_revenue: 80000
      customer_concentration: 0.65
    after:
      monthly_revenue: 120000
      customer_concentration: 0.45
    duration_days: 45
    key_factors:
      - "客户配合度高"
      - "有多个潜在大客户"

# ============================================
# 合规与免责
# ============================================
compliance:
  disclaimer: |
    1. 以上方案仅供参考，执行效果因企业而异
    2. 不保证一定能达到预期效果
    3. 需要企业主积极配合执行
  
  legal_notes:
    - "合同修改需咨询法律顾问"
    - "不得虚构流水或交易"
    - "所有操作必须基于真实业务"

# ============================================
# 元数据
# ============================================
metadata:
  skill_id: "SOLUTION_CASHFLOW_001"
  skill_name: "流水优化术"
  skill_type: "solution"
  
  source: "服务商经验总结"
  verified_date: "2026-02-01"
  verified_by: "李四"
  
  region: "全国"
  confidence: "高"
  
  version: "1.0.0"
  created_at: "2026-02-01T10:00:00Z"
  updated_at: "2026-02-01T10:00:00Z"
  created_by: "admin@dcc2026.com"
  updated_by: "admin@dcc2026.com"
  
  changelog:
    - version: "1.0.0"
      date: "2026-02-01"
      changes: "初始版本"
      reason: "方案库建设"
      approved_by: "产品负责人"
  
  status: "active"
```

---

## 五、策略Skill规范

### 5.1 提醒策略示例

```yaml
# ============================================
# 策略Skill：A类客户立即提醒
# ============================================

# 基本信息
skill_id: "STRATEGY_REMINDER_A_CLASS"
skill_name: "A类客户立即提醒策略"
skill_type: "strategy"

# 策略属性
strategy:
  category: "reminder"               # reminder/compliance/scoring
  trigger_type: "event"              # event/schedule/manual

# ============================================
# 触发条件
# ============================================
trigger:
  # 事件类型
  event: "customer_diagnosed"        # 客户诊断完成事件
  
  # 触发条件
  condition:
    logic: "all"
    conditions:
      - field: "customer_grade"
        operator: "=="
        value: "A"
        description: "客户分层为A类"
      
      - field: "estimated_amount"
        operator: ">"
        value: 0
        description: "有可贷额度"

# ============================================
# 执行规则
# ============================================
action:
  type: "send_notification"           # 动作类型
  
  # 通知渠道
  channel: "wechat_service"           # wechat_service/sms/email
  
  # 模板配置
  template:
    template_id: "TEMPLATE_A_CLASS"
    
    # 动态参数
    variables:
      - name: "customer_name"
        source: "customer.name"
      - name: "estimated_amount"
        source: "diagnosis.estimated_amount"
        format: "¥{value:,.0f}万元"
      - name: "matched_products"
        source: "diagnosis.matched_products"
        format: "list"
  
  # 去重规则
  deduplication:
    enabled: true
    window_hours: 24
    key_fields: ["customer_id", "template_id"]
    description: "24小时内不重复发送相同模板"
  
  # 频率限制
  rate_limit:
    max_per_customer_per_day: 1
    max_per_customer_per_week: 3
    max_per_customer_per_month: 10

# 注：content_generation（LLM 内容生成）为 V2 能力，见附录B

# ============================================
# 元数据
# ============================================
metadata:
  skill_id: "STRATEGY_REMINDER_A_CLASS"
  skill_name: "A类客户立即提醒策略"
  skill_type: "strategy"
  
  source: "产品策略"
  verified_date: "2026-02-05"
  verified_by: "运营负责人"
  
  region: "全国"
  confidence: "高"
  
  version: "1.0.0"
  created_at: "2026-02-05T14:00:00Z"
  updated_at: "2026-02-05T14:00:00Z"
  created_by: "admin@dcc2026.com"
  updated_by: "admin@dcc2026.com"
  
  changelog:
    - version: "1.0.0"
      date: "2026-02-05"
      changes: "初始版本"
      reason: "提醒策略建设"
      approved_by: "产品负责人"
  
  status: "active"
```

### 5.2 合规检查策略示例

```yaml
# ============================================
# 策略Skill：内容合规检查
# ============================================

skill_id: "STRATEGY_COMPLIANCE_CHECK"
skill_name: "内容合规检查策略"
skill_type: "strategy"

strategy:
  category: "compliance"
  trigger_type: "on_demand"          # 按需触发

# ============================================
# 检查规则
# ============================================
rules:
  # 1. 禁用词检查
  - rule_id: "RULE_001"
    rule_name: "禁用词检查"
    enabled: true
    
    forbidden_words:
      - pattern: "保证通过"
        severity: "error"
        message: "不得使用承诺性表述"
      - pattern: "包下款"
        severity: "error"
        message: "不得使用承诺性表述"
      - pattern: "必批|秒批|稳过"
        severity: "error"
        message: "不得使用承诺性表述"
      - pattern: "内部渠道|特殊关系"
        severity: "error"
        message: "不得暗示非正常渠道"
      - pattern: "最低利率(?!.*参考)"
        severity: "warning"
        message: "提及利率需标注'仅供参考'"
  
  # 2. 必需短语检查
  - rule_id: "RULE_002"
    rule_name: "免责声明检查"
    enabled: true
    
    required_patterns:
      - pattern: "(仅供|仅作).*参考"
        message: "必须包含'仅供参考'类表述"
      - pattern: "(实际|以.*)(审批|批复).*为准"
        message: "必须包含'以实际审批为准'类表述"
  
  # 3. 敏感信息检查
  - rule_id: "RULE_003"
    rule_name: "敏感信息检查"
    enabled: true
    
    sensitive_patterns:
      - pattern: "\\d{15}|\\d{18}"     # 身份证号
        severity: "error"
        message: "不得包含身份证号"
      - pattern: "\\d{16,19}"           # 银行卡号
        severity: "error"
        message: "不得包含银行卡号"
      - pattern: "1[3-9]\\d{9}"         # 完整手机号
        severity: "warning"
        message: "避免泄露完整手机号"

# ============================================
# 处理逻辑
# ============================================
processing:
  # 检查模式
  mode: "strict"                     # strict/lenient
  
  # 遇到error级别违规时
  on_error: "reject"                 # reject/warn/fix
  
  # 遇到warning级别违规时
  on_warning: "warn"                 # reject/warn/fix
  
  # 自动修复（可选）
  auto_fix:
    enabled: false
    rules:
      - pattern: "最低利率"
        replacement: "参考利率"

# 元数据
metadata:
  skill_id: "STRATEGY_COMPLIANCE_CHECK"
  skill_name: "内容合规检查策略"
  skill_type: "strategy"
  
  source: "合规要求"
  verified_date: "2026-02-01"
  verified_by: "合规负责人"
  
  version: "1.0.0"
  status: "active"
```

---

## 六、Skill验证规范

### 6.1 YAML Schema验证

所有Skill必须通过JSON Schema验证：

```yaml
# skill_schema.json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["skill_id", "skill_name", "skill_type", "metadata"],
  "properties": {
    "skill_id": {
      "type": "string",
      "pattern": "^(PRODUCT|SOLUTION|STRATEGY)_[A-Z0-9_]+$"
    },
    "skill_name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "skill_type": {
      "type": "string",
      "enum": ["product", "solution", "strategy"]
    },
    "metadata": {
      "type": "object",
      "required": ["version", "status", "verified_date"],
      "properties": {
        "version": {
          "type": "string",
          "pattern": "^\\d+\\.\\d+\\.\\d+$"
        },
        "status": {
          "type": "string",
          "enum": ["active", "inactive", "deprecated"]
        }
      }
    }
  }
}
```

### 6.2 验证工具

```python
# skill_validator.py
import yaml
import jsonschema

def validate_skill(skill_file: str) -> bool:
    """验证Skill文件格式"""
    with open(skill_file) as f:
        skill = yaml.safe_load(f)
    
    with open('skill_schema.json') as f:
        schema = json.load(f)
    
    try:
        jsonschema.validate(skill, schema)
        return True
    except jsonschema.ValidationError as e:
        print(f"验证失败: {e.message}")
        return False
```

---

## 七、Skill命名规范

### 7.1 Skill ID命名

格式：`{TYPE}_{CATEGORY}_{SEQUENCE}`

**示例**：
- `PRODUCT_CMB_TAX_001`：产品 - 招商银行税贷 - 序号001
- `SOLUTION_CASHFLOW_001`：方案 - 流水优化 - 序号001
- `STRATEGY_REMINDER_A_CLASS`：策略 - 提醒A类 - 描述性名称

**规则**：
- 全大写
- 下划线分隔
- TYPE：PRODUCT/SOLUTION/STRATEGY
- CATEGORY：简短分类（不超过20字符）
- SEQUENCE：3位数字序号或描述性名称

### 7.2 文件命名

格式：`{skill_id}.yaml`

**示例**：
- `PRODUCT_CMB_TAX_001.yaml`
- `SOLUTION_CASHFLOW_001.yaml`
- `STRATEGY_REMINDER_A_CLASS.yaml`

### 7.3 目录结构

```
skills/
├── products/                      # 产品Skill
│   ├── banks/                    # 按银行分类
│   │   ├── cmb/                  # 招商银行
│   │   │   ├── PRODUCT_CMB_TAX_001.yaml
│   │   │   └── PRODUCT_CMB_INVOICE_001.yaml
│   │   ├── icbc/                 # 工商银行
│   │   └── abc/                  # 农业银行
│   └── categories/               # 按产品类型分类
│       ├── tax/                  # 税贷
│       ├── invoice/              # 发票贷
│       └── mortgage/             # 抵押贷
│
├── solutions/                     # 方案Skill
│   ├── cashflow/                 # 流水改造
│   ├── tax/                      # 纳税改造
│   └── credit/                   # 征信改造
│
└── strategies/                    # 策略Skill
    ├── reminders/                # 提醒策略
    ├── compliance/               # 合规策略
    └── scoring/                  # 评分策略
```

---

## 八、版本管理规范

### 8.1 语义化版本

格式：`MAJOR.MINOR.PATCH`

**版本号说明**：
- **MAJOR**（主版本号）：不兼容的API修改
  - 例：修改eligibility结构，导致旧版Runner无法解析
- **MINOR**（次版本号）：向下兼容的功能性新增
  - 例：新增字段、新增检查规则
- **PATCH**（修订号）：向下兼容的问题修正
  - 例：修正阈值、修正描述文字

**示例**：
- `1.0.0` → `1.0.1`：修正流水阈值（10万 → 12万）
- `1.0.1` → `1.1.0`：新增征信查询限制条件
- `1.1.0` → `2.0.0`：修改eligibility结构（不兼容）

### 8.2 变更日志规范

每次修改必须在`changelog`中记录：

```yaml
changelog:
  - version: "1.2.0"
    date: "2026-02-10"
    changes: "提高流水要求从5万到10万"
    reason: "回填数据显示拒批率过高（样本50，拒批率65%）"
    approved_by: "产品负责人-张三"
    impact: "预计影响20%现有客户"
```

---

## 九、最佳实践

### 9.1 DO（推荐）

✅ **使用描述性字段名**
```yaml
field: "monthly_revenue"          # 好
field: "mr"                       # 差
```

✅ **提供完整的description**
```yaml
description: "月均流水≥10万元"    # 好
description: "流水要求"            # 差
```

✅ **明确单位**
```yaml
value: 100000
unit: "元"
description: "月均流水≥10万元"
```

✅ **提供可读的错误提示**
```yaml
error_message: "您的月均流水为{actual}元，未达到10万元的要求，差距{gap}元"
```

✅ **版本号递增**
```yaml
# 修改阈值
version: "1.0.0" → "1.0.1"

# 新增条件
version: "1.0.1" → "1.1.0"
```

### 9.2 DON'T（避免）

❌ **硬编码中文**（应提取为变量）
```yaml
# 差
formula: "年纳税 * 30"

# 好
formula: "annual_tax * 30"
```

❌ **缺少免责声明**
```yaml
# 必须包含
compliance:
  disclaimer: "以上额度仅供初步评估..."
```

❌ **版本号不规范**
```yaml
version: "v1.2"                   # 差
version: "1.2.0"                  # 好
```

❌ **缺少变更日志**
```yaml
# 每次修改必须记录
changelog:
  - version: "1.2.0"
    date: "2026-02-10"
    changes: "..."
```

---

## 十、常见问题FAQ

### Q1：修改Skill后多久生效？

**A**：立即生效（热更新）。Skill Runner会定期（默认每分钟）重新加载Skill配置。

### Q2：如何回滚到旧版本？

**A**：
1. 从Git历史找到旧版本文件
2. 恢复文件内容
3. 修改`version`和`changelog`
4. 提交变更

### Q3：能否在生产环境直接修改Skill？

**A**：不推荐。建议流程：
1. 在测试环境修改
2. 验证无误后
3. 通过代码审查
4. 合并到生产环境

### Q4：Skill文件过大怎么办？

**A**：考虑拆分：
- 将复杂的eligibility条件拆成多个Skill
- 使用`include`引用公共配置（如果支持）

### Q5：如何测试Skill？

**A**：使用单元测试：
```python
def test_product_skill():
    skill = load_skill("PRODUCT_CMB_TAX_001")
    customer = mock_customer(monthly_revenue=120000, annual_tax=15000)
    assert skill.check_eligibility(customer) == True
```

---

## 十一、工具与资源

### 11.1 Skill编辑器（规划中）

功能：
- 可视化编辑Skill
- 实时验证
- 版本对比
- 一键发布

### 11.2 Skill测试工具

```bash
# 验证Skill格式
python tools/validate_skill.py skills/products/PRODUCT_CMB_TAX_001.yaml

# 测试Skill逻辑
python tools/test_skill.py skills/products/PRODUCT_CMB_TAX_001.yaml --customer test_data/customer_001.json
```

### 11.3 文档生成工具

```bash
# 生成Skill文档（Markdown）
python tools/generate_skill_doc.py skills/products/PRODUCT_CMB_TAX_001.yaml > docs/PRODUCT_CMB_TAX_001.md
```

---

## 十二、变更记录

### v1.1（2026-02-12）

**v1.0→v1.1 修订**：
- 新增《V0能力矩阵+版本门禁》（与培训手册、原型代码统一）
- 明确 metadata 为字段权威源，顶层快捷字段可选且须一致
- 明确 between=[min,max] 双端包含、table.range=[min,max) 左闭右开
- ml_model 标注 V0 禁止，校验器拒绝加载
- content_generation 移至附录B（V2 内容自动化规划）

### v1.0（2026-02-12）

**初始版本**：
- 定义三种Skill类型（产品/方案/策略）
- 制定元数据规范
- 制定命名规范
- 提供完整示例
- 制定验证规范
- 制定版本管理规范

**下一步**：
- 开发Skill编辑器
- 开发验证工具
- 收集团队反馈并迭代

---

## 附录B：V2内容自动化扩展（规划中）

以下特性在 **V2.0 公众号自动化** 阶段实现，V0/V1 不支持。

### content_generation（内容生成）

```yaml
content_generation:
  use_llm: true
  llm_config:
    model: "claude-sonnet-4"
    temperature: 0.3
  llm_prompt: |
    生成提醒文案...
  fallback_template: |
    【{customer_name}】您好，
    根据初步评估，您符合A类优质客户标准，
    预估可申请额度：{estimated_amount}
    匹配产品：{matched_products}
    点击查看详情 >
    注：额度仅供参考，以实际审批为准。
  compliance_check:
    enabled: true
    forbidden_words: ["保证通过", "包下款"]
```

**注意**：V0/V1 若在 Skill 中包含 `content_generation` 字段，校验器会发出警告（不阻塞）。

---

**文档版本**：v1.0  
**创建日期**：2026-02-12  
**维护人**：技术团队  
**审核状态**：待审核
