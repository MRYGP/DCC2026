# Skill系统开发培训手册 v1.0

**文档性质**：培训材料  
**创建日期**：2026-02-12  
**适用对象**：开发团队/运营团队/新成员  
**培训时长**：2-3小时

---

## 培训目标

学完本培训后，你将能够：
1. ✅ 理解DCC2026的AI原生架构
2. ✅ 独立编写产品/方案/策略Skill
3. ✅ 使用Skill Runner进行开发
4. ✅ 进行Skill热更新和调试
5. ✅ 理解审计日志和合规要求

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
| partner_id多租户 | 🚫 V0不做 | V0固定partner_id="DEFAULT" | V1实现租户隔离 |
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

## 第一部分：核心概念（30分钟）

### 1.1 为什么需要Skill系统？

#### 传统方式的问题

```python
# 传统方式：硬编码
def match_products(customer):
    results = []
    
    # 招商银行税贷（硬编码）
    if (customer.monthly_revenue >= 100000 and  # 硬编码阈值
        customer.annual_tax >= 10000 and
        customer.credit_query_6m <= 6):
        results.append("招商银行税贷")
    
    # 每次修改阈值都要：
    # 1. 改代码
    # 2. 测试
    # 3. 发版
    # 4. 重启服务
    
    return results
```

**问题清单**：
- ❌ 产品库更新 → 改代码 → 发版（2-3天）
- ❌ 调整阈值 → 改代码 → 发版（2-3天）
- ❌ 优化逻辑 → 改代码 → 发版（2-3天）
- ❌ 无法基于回填数据自动优化

---

#### Skill系统的解决方案

```yaml
# Skill方式：配置化
skill_id: "PRODUCT_CMB_TAX_001"
skill_name: "招商银行税贷"

eligibility:
  logic: "all"
  conditions:
    - field: "monthly_revenue"
      operator: ">="
      value: 100000               # 配置化阈值
    - field: "annual_tax"
      operator: ">="
      value: 10000
```

```python
# Skill Runner：动态执行
class ProductMatcherRunner:
    def match(self, customer):
        # 动态加载Skill配置
        products = self.skill_repo.load_all_products()
        
        for product in products:
            # 动态评估准入条件
            if self._check_eligibility(customer, product.eligibility):
                results.append(product)
        
        return results
```

**好处清单**：
- ✅ 修改阈值 → 只改YAML → 立即生效（1分钟）
- ✅ 新增产品 → 只加YAML → 立即生效（1分钟）
- ✅ 可基于回填数据优化（AI建议 + 人工审核）
- ✅ 所有变更可追溯、可回滚

---

### 1.2 Skill系统三层架构

```
┌─────────────────────────────────────────────┐
│ 第1层：Skill定义层（声明式配置）             │
│                                             │
│ - 产品库（YAML）                            │
│ - 方案库（YAML）                            │
│ - 策略库（YAML）                            │
│                                             │
│ 特点：可热更新、版本化、可审计              │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ 第2层：Skill Runner（确定性执行引擎）       │
│                                             │
│ - ProductMatcherRunner（产品匹配）          │
│ - SolutionGeneratorRunner（方案生成）       │
│ - ReminderRunner（提醒调度）                │
│                                             │
│ 特点：确定性、可复现、可测试                │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ 第3层：LLM辅助层（建议不自动生效）          │
│                                             │
│ - RuleGenerator（规则生成建议）             │
│ - ContentGenerator（内容生成）              │
│ - OptimizationAdvisor（优化建议）           │
│                                             │
│ 特点：辅助性、需人工审核、不直接生效        │
└─────────────────────────────────────────────┘
```

---

### 1.3 关键术语表

| 术语 | 定义 | 示例 |
|------|------|------|
| **Skill** | 声明式配置单元 | 产品配置YAML文件 |
| **Skill Runner** | 确定性执行引擎 | ProductMatcherRunner类 |
| **热更新** | 修改配置立即生效，无需重启 | 改YAML后1分钟生效 |
| **准入条件（eligibility）** | 客户必须满足的条件 | 流水≥10万、纳税≥1万 |
| **操作符（operator）** | 比较运算符 | `>=`, `<=`, `in`, `not_in` |
| **表达式（expression）** | 数学公式 | `annual_tax * 30` |
| **审计日志** | 所有操作的可追溯记录 | 谁在什么时间做了什么 |

---

## 第二部分：如何编写Skill（60分钟）

### 2.1 产品Skill编写实战

#### 任务：为工商银行税贷创建Skill

**需求**：
- 银行：工商银行
- 产品：税贷
- 准入条件：
  - 月均流水 ≥ 5万元
  - 年纳税 ≥ 5千元
  - 近6月征信查询 ≤ 10次
  - 企业成立 ≥ 1年
  - 不支持行业：房地产、金融、娱乐
- 额度计算：年纳税 × 25倍，最高50万

---

#### Step 1：创建YAML文件

```bash
# 文件路径
skills/products/banks/icbc/PRODUCT_ICBC_TAX_001.yaml
```

#### Step 2：填写基本信息

```yaml
# ============================================
# 产品Skill：工商银行税贷
# ============================================

# 基本信息（必需）
skill_id: "PRODUCT_ICBC_TAX_001"        # 唯一ID（格式：PRODUCT_{BANK}_{TYPE}_{SEQUENCE}）
skill_name: "工商银行税贷"               # 显示名称
skill_type: "product"                   # 类型：product

# 产品属性（必需）
product:
  bank: "工商银行"
  bank_code: "ICBC"
  category: "税贷"
  product_code: "TAX_LOAN_001"
  
  # 产品特点（可选，描述性）
  features:
    - "无需抵押"
    - "纯信用贷款"
    - "线上申请"
  
  # 利率信息（可选）
  interest_rate:
    type: "range"
    min: 0.042                          # 年化利率4.2%
    max: 0.08                           # 年化利率8%
    description: "年化利率4.2%-8%"
  
  # 期限信息（可选）
  term:
    unit: "month"
    options: [12, 24, 36]
    description: "最长3年"
```

---

#### Step 3：定义准入条件（核心）

```yaml
# ============================================
# 准入条件（核心规则）
# ============================================
eligibility:
  # 逻辑类型（必需）
  logic: "all"                          # all（所有条件必须满足）/ any（任意条件满足）
  
  # 具体条件（必需）
  conditions:
    # 条件1：月均流水
    - field: "monthly_revenue"          # 客户字段名（必需）
      operator: ">="                    # 操作符（必需）
      value: 50000                      # 阈值（必需）
      description: "月均流水≥5万元"      # 可读描述（必需）
      dimension: "流水体质"              # 六宫格维度（必需）
      error_message: "您的月均流水为{actual}元，未达到5万元的要求"  # 错误提示（可选）
    
    # 条件2：年纳税
    - field: "annual_tax"
      operator: ">="
      value: 5000
      description: "年纳税≥5千元"
      dimension: "票税体质"
    
    # 条件3：征信查询
    - field: "credit_query_6m"
      operator: "<="
      value: 10
      description: "近6个月征信查询≤10次"
      dimension: "征信体质"
    
    # 条件4：企业年限
    - field: "company_age_years"
      operator: ">="
      value: 1
      description: "企业成立≥1年"
      dimension: "资质体质"
    
    # 条件5：行业限制（黑名单）
    - field: "industry"
      operator: "not_in"
      value:
        - "房地产开发"
        - "金融投资"
        - "娱乐业"
      description: "不支持以上行业"
      dimension: "行业体质"
```

**关键点**：
- `field`：必须是Customer对象中实际存在的字段
- `operator`：支持的操作符见下表
- `value`：根据operator类型不同，可以是数字、字符串、列表
- `description`：必须清晰易懂，用户会看到
- `dimension`：必须是六宫格之一

**支持的操作符**：

| 操作符 | 说明 | value类型 | 示例 |
|--------|------|-----------|------|
| `>=` | 大于等于 | 数字 | `monthly_revenue >= 50000` |
| `<=` | 小于等于 | 数字 | `credit_query_6m <= 10` |
| `>` | 大于 | 数字 | `company_age_years > 1` |
| `<` | 小于 | 数字 | `debt_ratio < 0.7` |
| `==` | 等于 | 任意 | `credit_overdue_m3 == 0` |
| `!=` | 不等于 | 任意 | `industry != "房地产"` |
| `in` | 在列表中 | 列表 | `region in ["北京", "上海"]` |
| `not_in` | 不在列表中 | 列表 | `industry not_in ["房地产", "金融"]` |

---

#### Step 4：定义额度计算规则

```yaml
# ============================================
# 额度计算规则
# ============================================
amount_calculation:
  # 公式类型（必需）
  formula_type: "expression"            # expression（表达式）/ table（查表）
  
  # 计算表达式（formula_type=expression时必需）
  formula: "min(annual_tax * 25, 500000)"
  
  # 表达式说明（可选）
  formula_description: |
    额度 = min(年纳税额 × 25倍, 50万)
    即：以年纳税额的25倍作为基础额度，但不超过50万
  
  # 额度上下限（可选）
  amount_range:
    min: 30000                          # 最小3万
    max: 500000                         # 最大50万
    unit: "元"
```

**表达式规则**：
- ✅ 允许：基本算术运算（`+`, `-`, `*`, `/`）
- ✅ 允许：内置函数（`min`, `max`, `abs`, `round`）
- ✅ 允许：客户字段（如`annual_tax`, `monthly_revenue`）
- ❌ 禁止：任意Python代码（安全风险）
- ❌ 禁止：外部函数调用

**额度计算的另一种方式：查表**

```yaml
amount_calculation:
  formula_type: "table"
  lookup_field: "annual_tax"            # 查找字段
  
  # 查找表
  table:
    - range: [0, 5000]                  # 年纳税0-5千
      amount: 0                         # 不符合
    - range: [5000, 20000]              # 年纳税5千-2万
      amount: 125000                    # 额度12.5万
    - range: [20000, 50000]             # 年纳税2万-5万
      amount: 300000                    # 额度30万
    - range: [50000, 999999999]         # 年纳税5万+
      amount: 500000                    # 额度50万（封顶）
```

---

#### Step 5：添加合规与免责

```yaml
# ============================================
# 合规与免责（必需）
# ============================================
compliance:
  # 免责声明（系统级固定，不可修改）
  disclaimer: |
    1. 以上额度仅供初步评估，实际额度以银行审批为准
    2. 不构成贷款承诺或要约
    3. 银行有权根据风险评估调整额度或拒绝申请
  
  # 风险提示（可选）
  risk_warnings:
    - "征信查询次数过多会影响审批"
    - "流水不稳定可能导致额度降低"
  
  # 禁止场景（可选）
  forbidden_scenarios:
    - "不得用于投资理财"
    - "不得用于房地产投机"
```

---

#### Step 6：填写元数据（必需）

```yaml
# ============================================
# 元数据（必需）
# ============================================
metadata:
  skill_id: "PRODUCT_ICBC_TAX_001"
  skill_name: "工商银行税贷"
  skill_type: "product"
  
  # 来源信息（必需）
  source: "工商银行官网"
  source_url: "https://www.icbc.com.cn/loan/tax"
  verified_date: "2026-02-12"           # 核验日期（YYYY-MM-DD）
  verified_by: "张三"
  
  # 适用范围（必需）
  region: "全国"
  confidence: "高"                      # 高/中/低
  
  # 版本控制（必需）
  version: "1.0.0"                      # 语义化版本
  created_at: "2026-02-12T10:00:00Z"    # ISO 8601格式
  updated_at: "2026-02-12T10:00:00Z"
  created_by: "admin@dcc2026.com"
  updated_by: "admin@dcc2026.com"
  
  # 变更日志（必需）
  changelog:
    - version: "1.0.0"
      date: "2026-02-12"
      changes: "初始版本"
      reason: "新产品上线"
      approved_by: "产品负责人"
  
  # 状态（必需）
  status: "active"                      # active/inactive/deprecated
```

---

#### Step 7：验证与测试

**验证YAML格式**：
```bash
# 使用验证工具
python tools/validate_skill.py skills/products/banks/icbc/PRODUCT_ICBC_TAX_001.yaml
```

**输出示例**：
```
✓ YAML格式正确
✓ 必需字段完整
✓ skill_id格式正确
✓ 版本号格式正确
✓ 操作符合法
验证通过！
```

**测试Skill逻辑**：
```bash
# 使用测试客户数据
python tools/test_skill.py \
  skills/products/banks/icbc/PRODUCT_ICBC_TAX_001.yaml \
  --customer test_data/customer_good.json
```

**输出示例**：
```
测试客户：customer_good.json
├─ 准入条件检查
│  ✓ 月均流水≥5万元（实际：8万）
│  ✓ 年纳税≥5千元（实际：1.2万）
│  ✓ 近6个月征信查询≤10次（实际：3次）
│  ✓ 企业成立≥1年（实际：3年）
│  ✓ 不支持以上行业（实际：软件开发）
│  结果：通过准入 ✓
│
├─ 额度计算
│  公式：min(annual_tax * 25, 500000)
│  计算：min(12000 * 25, 500000) = 300000
│  结果：预估额度 ¥300,000元 ✓
│
└─ 测试通过！
```

---

### 2.2 常见错误与调试

#### 错误1：字段名拼写错误

```yaml
# ❌ 错误
conditions:
  - field: "montly_revenue"             # 拼写错误
    operator: ">="
    value: 50000
```

**错误提示**：
```
运行时错误：Customer对象没有'montly_revenue'字段
```

**解决**：
```yaml
# ✅ 正确
conditions:
  - field: "monthly_revenue"            # 正确拼写
    operator: ">="
    value: 50000
```

---

#### 错误2：操作符与值类型不匹配

```yaml
# ❌ 错误
conditions:
  - field: "industry"
    operator: ">="                      # 字符串不能用>=
    value: "软件开发"
```

**错误提示**：
```
运行时错误：无法比较字符串 >= 字符串
```

**解决**：
```yaml
# ✅ 正确
conditions:
  - field: "industry"
    operator: "=="                      # 字符串用==或in/not_in
    value: "软件开发"
```

---

#### 错误3：公式中使用未定义变量

```yaml
# ❌ 错误
amount_calculation:
  formula: "annual_income * 30"         # annual_income不存在
```

**错误提示**：
```
运行时错误：表达式中包含未定义变量'annual_income'
可用变量：monthly_revenue, annual_revenue, annual_tax, ...
```

**解决**：
```yaml
# ✅ 正确
amount_calculation:
  formula: "annual_tax * 30"            # 使用正确的字段名
```

---

### 2.3 方案Skill编写实战（简化版）

#### 任务：为纳税提升方案创建Skill

```yaml
# ============================================
# 方案Skill：纳税提升术
# ============================================

skill_id: "SOLUTION_TAX_001"
skill_name: "纳税提升术"
skill_type: "solution"

solution:
  category: "票税改造"
  target_dimension: "票税体质"

# 适用场景
applicable_when:
  logic: "all"
  conditions:
    - field: "gap_category"
      operator: "=="
      value: "纳税不足"
    - field: "gap_amount"
      operator: ">="
      value: 5000                       # 差距≥5千

# 执行步骤
steps:
  - step_id: "STEP_001"
    step_number: 1
    title: "核对税务申报记录"
    description: "检查是否有漏报、少报情况"
    duration:
      value: 3
      unit: "days"
    cost:
      value: 0
      unit: "元"
    output:
      type: "document"
      name: "税务核对清单"
  
  - step_id: "STEP_002"
    step_number: 2
    title: "补缴漏报税款"
    description: "对漏报、少报的税款进行补缴"
    duration:
      value: 7
      unit: "days"
    cost:
      value: 5000
      unit: "元"
    output:
      type: "receipt"
      name: "补税凭证"

# 预估成果
estimated_result:
  time_cost:
    min: 7
    max: 14
    unit: "days"
  money_cost:
    min: 0
    max: 10000
    unit: "元"
  success_rate:
    value: 0.80
    description: "80%"
    sample_size: 30

# 元数据（同产品Skill）
metadata:
  skill_id: "SOLUTION_TAX_001"
  skill_name: "纳税提升术"
  skill_type: "solution"
  version: "1.0.0"
  status: "active"
  # ... 其他字段
```

---

## 第三部分：如何使用Skill Runner（45分钟）

### 3.1 ProductMatcherRunner使用

#### 基本使用流程

```python
from skill_runner.product_matcher import ProductMatcherRunner
from repositories.skill_repository import SkillRepository
from utils.audit_logger import AuditLogger
from models.customer import Customer
from datetime import datetime

# Step 1：初始化组件
skill_repo = SkillRepository(skills_dir="skills")
audit_logger = AuditLogger()
matcher = ProductMatcherRunner(
    skill_repo=skill_repo,
    audit_logger=audit_logger
)

# Step 2：创建客户对象（CustomerV0 完整版）
customer = Customer(
    # 必需字段
    customer_id="CUST_001",
    name="张三",
    partner_id="DEFAULT",              # V0 固定为 DEFAULT
    company_name="某某科技有限公司",
    company_age_years=3,
    industry="软件开发",
    region="北京",
    
    # 流水体质（13 字段中的核心字段）
    monthly_revenue=150000,            # 月均流水 15 万元（注意：单位是元）
    annual_revenue=1800000,            # 年营收 180 万元
    revenue_stability=0.85,            # 流水稳定性 85%
    customer_concentration=0.4,        # 客户集中度 40%
    
    # 票税体质
    annual_invoice=1500000,            # 年开票 150 万元
    annual_tax=18000,                  # 年纳税 1.8 万元
    tax_grade="B",                     # 纳税等级 B
    
    # 征信体质（13 字段中的核心字段）
    credit_score=720,                  # 征信分数
    credit_query_6m=3,                 # 近 6 月查询 3 次
    credit_overdue_m1=0,               # M1 逾期 0 次
    credit_overdue_m3=0,               # M3+ 逾期 0 次
    debt_ratio=0.45,                   # 负债率 45%
    small_loan_count=1,                # 小贷笔数
    
    # 资产体质
    has_mortgage=False,                # 无抵押物
    mortgage_value=0,
    mortgage_ratio=0,
    
    # 资质体质
    high_tech_certification=True,      # 有高新认证
    specialized_certification=False,   # 无专精特新
    
    # 时间戳
    created_at=datetime.now(),
    updated_at=datetime.now()
)

# 注意：V0 所有金额单位为「元」（不是万元）
# 注意：V0 partner_id 固定为「DEFAULT」（V1 才做多租户）

# Step 3：执行匹配
matched_products = matcher.match(customer, top_n=5)

# Step 4：处理结果
for product in matched_products:
    print(f"产品：{product.product_name}")
    print(f"银行：{product.bank}")
    print(f"预估额度：¥{product.estimated_amount:,.0f}元")
    print(f"{product.disclaimer}\n")
```

---

### 3.2 Web API集成

#### 创建FastAPI路由

```python
# api/routes.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter()

class MatchRequest(BaseModel):
    customer_id: str
    monthly_revenue: float
    annual_tax: float
    credit_query_6m: int
    company_age_years: int
    industry: str

@router.post("/api/v1/match")
async def match_products(request: MatchRequest):
    """产品匹配API"""
    try:
        # 构建Customer对象
        customer = Customer(
            customer_id=request.customer_id,
            monthly_revenue=request.monthly_revenue,
            annual_tax=request.annual_tax,
            # ... 填充字段
        )
        
        # 执行匹配
        matched = matcher.match(customer, top_n=5)
        
        # 返回结果
        return {
            "customer_id": request.customer_id,
            "matched_count": len(matched),
            "products": [p.to_dict() for p in matched]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

#### 调用API

```bash
# 使用curl
curl -X POST http://localhost:8000/api/v1/match \
  -H "Content-Type: application/json" \
  -d '{
    "customer_id": "CUST_001",
    "monthly_revenue": 150000,
    "annual_tax": 18000,
    "credit_query_6m": 3,
    "company_age_years": 3,
    "industry": "软件开发"
  }'
```

**响应示例**：
```json
{
  "customer_id": "CUST_001",
  "matched_count": 3,
  "products": [
    {
      "skill_id": "PRODUCT_CMB_TAX_001",
      "product_name": "招商银行税贷",
      "bank": "招商银行",
      "estimated_amount": 540000,
      "disclaimer": "以上额度仅供初步评估..."
    },
    {
      "skill_id": "PRODUCT_ICBC_TAX_001",
      "product_name": "工商银行税贷",
      "bank": "工商银行",
      "estimated_amount": 450000,
      "disclaimer": "以上额度仅供初步评估..."
    }
  ]
}
```

---

### 3.3 热更新操作

#### 场景：修改产品阈值

**需求**：招商银行税贷的流水要求从10万提高到12万

**Step 1：修改Skill文件**

```yaml
# skills/products/banks/cmb/PRODUCT_CMB_TAX_001.yaml

# 修改前
conditions:
  - field: "monthly_revenue"
    operator: ">="
    value: 100000                       # 10万

# 修改后
conditions:
  - field: "monthly_revenue"
    operator: ">="
    value: 120000                       # 12万

# 更新版本号和变更日志
metadata:
  version: "1.1.0"                      # 从1.0.0升级到1.1.0
  updated_at: "2026-02-13T09:00:00Z"
  updated_by: "admin@dcc2026.com"
  
  changelog:
    - version: "1.1.0"
      date: "2026-02-13"
      changes: "提高流水要求从10万到12万"
      reason: "回填数据显示拒批率过高"
      approved_by: "产品负责人"
    # ... 保留旧版本记录
```

**Step 2：触发热更新**

```bash
# 方式1：调用API
curl -X POST http://localhost:8000/api/v1/reload-skills
```

```python
# 方式2：代码调用
skill_repo.reload_skills()
```

**Step 3：验证生效**

```bash
# 使用测试客户验证
python tools/test_skill.py \
  skills/products/banks/cmb/PRODUCT_CMB_TAX_001.yaml \
  --customer test_data/customer_revenue_110k.json
```

**输出**：
```
测试客户：customer_revenue_110k.json (月均流水11万)
├─ 准入条件检查
│  ✗ 月均流水≥12万元（实际：11万）
│  结果：未通过准入 ✗
│
└─ 符合预期（阈值已提高到12万）✓
```

---

### 3.4 调试技巧

#### 技巧1：查看详细日志

```python
import logging

# 启用DEBUG级别日志
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# 执行匹配
matched = matcher.match(customer)
```

**输出示例**：
```
2026-02-12 10:00:01 - ProductMatcherRunner - INFO - 开始匹配产品，客户ID: CUST_001
2026-02-12 10:00:01 - SkillRepository - INFO - 已加载20个产品Skill
2026-02-12 10:00:01 - ProductMatcherRunner - DEBUG - ✓ 招商银行税贷: 通过准入，预估额度540000元
2026-02-12 10:00:01 - ProductMatcherRunner - DEBUG - ✗ 工商银行企业贷: 未通过准入，原因: 月均流水不足
2026-02-12 10:00:01 - ProductMatcherRunner - INFO - 匹配完成，匹配3个产品
```

---

#### 技巧2：单独测试准入条件

```python
# 单独测试某个产品的准入条件
product = skill_repo.load_product_by_id("PRODUCT_CMB_TAX_001")

eligible, failed_conditions = matcher._check_eligibility(
    customer=customer,
    product=product
)

print(f"是否满足准入：{eligible}")
if not eligible:
    print("未满足的条件：")
    for condition in failed_conditions:
        print(f"  - {condition}")
```

**输出示例**：
```
是否满足准入：False
未满足的条件：
  - 月均流水≥12万元 (实际: 11万, 要求: >= 12万)
```

---

#### 技巧3：单独测试额度计算

```python
# 单独测试额度计算
product = skill_repo.load_product_by_id("PRODUCT_CMB_TAX_001")

amount = matcher._calculate_amount(
    customer=customer,
    product=product
)

print(f"预估额度：¥{amount:,.0f}元")
```

---

## 第四部分：合规与审计（15分钟）

### 4.1 合规要求（强制）

#### 免责声明（系统级，不可修改）

所有产品Skill必须包含：

```yaml
compliance:
  disclaimer: |
    1. 以上额度仅供初步评估，实际额度以银行审批为准
    2. 不构成贷款承诺或要约
    3. 银行有权根据风险评估调整额度或拒绝申请
```

**强制执行**：
- ✅ 系统会自动附加免责声明到所有输出
- ❌ 任何人都无法删除或修改免责声明

---

#### 禁止表述（黑名单）

**绝对禁止**：
- "保证通过"
- "包下款"
- "必批"、"秒批"
- "100%成功"
- "稳过"
- "内部渠道"

**检查机制**：
```python
# 系统会自动检查所有文案
forbidden_words = ["保证通过", "包下款", "必批"]

if any(word in text for word in forbidden_words):
    raise ComplianceError(f"包含禁用词：{word}")
```

---

### 4.2 审计日志

#### 自动记录的内容

每次产品匹配都会自动记录：

```json
{
  "action": "product_match",
  "customer_id": "CUST_001",
  "partner_id": "PARTNER_001",
  "details": {
    "matched_count": 3,
    "failed_count": 17,
    "top_products": [
      {
        "skill_id": "PRODUCT_CMB_TAX_001",
        "product_name": "招商银行税贷",
        "estimated_amount": 540000
      }
    ],
    "total_products": 20
  },
  "timestamp": "2026-02-12T10:00:01Z"
}
```

#### 查询审计日志

```python
# 查询某客户的所有匹配记录
logs = audit_logger.query(
    action="product_match",
    customer_id="CUST_001",
    start_date="2026-02-01",
    end_date="2026-02-28"
)

for log in logs:
    print(f"{log['timestamp']}: 匹配{log['details']['matched_count']}个产品")
```

---

### 4.3 版本回滚

#### 场景：发现新版本有问题，需要回滚

**Step 1：找到旧版本**

```bash
# 查看Git历史
git log skills/products/banks/cmb/PRODUCT_CMB_TAX_001.yaml

# 输出
commit abc123...
Author: admin@dcc2026.com
Date:   2026-02-12 10:00:00
    v1.1.0: 提高流水要求从10万到12万

commit def456...
Author: admin@dcc2026.com
Date:   2026-02-10 15:00:00
    v1.0.0: 初始版本
```

**Step 2：恢复旧版本**

```bash
# 恢复到旧版本
git checkout def456 -- skills/products/banks/cmb/PRODUCT_CMB_TAX_001.yaml
```

**Step 3：更新版本信息**

```yaml
metadata:
  version: "1.1.1"                      # 回滚也需要新版本号
  updated_at: "2026-02-13T15:00:00Z"
  
  changelog:
    - version: "1.1.1"
      date: "2026-02-13"
      changes: "回滚到v1.0.0版本"
      reason: "v1.1.0导致拒批率过高"
      approved_by: "产品负责人"
    - version: "1.1.0"
      date: "2026-02-13"
      changes: "提高流水要求从10万到12万"
    # ...
```

**Step 4：提交并热更新**

```bash
git commit -m "回滚到v1.0.0版本"
git push

# 触发热更新
curl -X POST http://localhost:8000/api/v1/reload-skills
```

---

## 第五部分：最佳实践（15分钟）

### 5.1 Skill命名规范

#### 产品Skill

格式：`PRODUCT_{BANK}_{TYPE}_{SEQUENCE}`

**示例**：
- `PRODUCT_CMB_TAX_001`：招商银行税贷001
- `PRODUCT_ICBC_INVOICE_001`：工商银行发票贷001
- `PRODUCT_ABC_MORTGAGE_001`：农业银行抵押贷001

#### 方案Skill

格式：`SOLUTION_{CATEGORY}_{SEQUENCE}`

**示例**：
- `SOLUTION_CASHFLOW_001`：流水优化术001
- `SOLUTION_TAX_001`：纳税提升术001
- `SOLUTION_CREDIT_001`：征信修复术001

#### 策略Skill

格式：`STRATEGY_{TYPE}_{NAME}`

**示例**：
- `STRATEGY_REMINDER_A_CLASS`：A类客户提醒策略
- `STRATEGY_COMPLIANCE_CHECK`：合规检查策略

---

### 5.2 版本管理

#### 语义化版本

格式：`MAJOR.MINOR.PATCH`

**规则**：
- **MAJOR**（主版本号）：不兼容的修改
  - 例：修改字段结构，导致旧版Runner无法解析
- **MINOR**（次版本号）：向下兼容的新增
  - 例：新增检查条件、新增字段
- **PATCH**（修订号）：向下兼容的修正
  - 例：修正阈值、修正描述文字

**示例**：
- `1.0.0` → `1.0.1`：修正流水阈值（10万 → 12万）
- `1.0.1` → `1.1.0`：新增征信查询限制条件
- `1.1.0` → `2.0.0`：修改eligibility结构（不兼容）

---

### 5.3 代码复用

#### 使用变量避免重复

```yaml
# ❌ 不好：重复写相同的描述
conditions:
  - field: "monthly_revenue"
    operator: ">="
    value: 100000
    description: "月均流水≥10万元，低于此标准无法申请"
  
  - field: "monthly_revenue"
    operator: "<"
    value: 500000
    description: "月均流水<50万元，超过此标准请选择其他产品"

# ✅ 好：简洁清晰
conditions:
  - field: "monthly_revenue"
    operator: ">="
    value: 100000
    description: "月均流水≥10万"
  
  - field: "monthly_revenue"
    operator: "<"
    value: 500000
    description: "月均流水<50万"
```

---

### 5.4 文档注释

```yaml
# ============================================
# 产品Skill：招商银行税贷
# 
# 更新记录：
# - 2026-02-13: 提高流水要求到12万（原因：拒批率过高）
# - 2026-02-10: 初始版本
# 
# 维护人：张三 (zhangsan@company.com)
# ============================================

skill_id: "PRODUCT_CMB_TAX_001"
# ...
```

---

## 第六部分：常见问题FAQ（10分钟）

### Q1：修改Skill后多久生效？

**A**：V0 支持两种方式：

**方式1：手动触发（立即生效）**
```bash
curl -X POST http://localhost:8000/api/v1/reload-skills
```
响应示例：
```json
{
  "status": "success",
  "message": "成功加载20个Skill",
  "report": {
    "success_count": 20,
    "failed_count": 0,
    "failed_files": [],
    "errors": [],
    "timestamp": "2026-02-12T10:00:00Z"
  }
}
```

**方式2：缓存 TTL 自动重载（默认 60 秒）**
- 修改 YAML 后，等待 TTL 过期（默认 60 秒）
- Skill Runner 会自动重新加载
- 适合非紧急更新

**V1+ 扩展**：
- 文件监听自动重载（watchdog）
- 配置化 TTL（运维可调整）

**推荐**：生产环境使用方式1（可控），开发环境可依赖方式2。

---

### Q2：如何测试Skill不会影响生产环境？

**A**：使用测试环境：

```bash
# 本地测试
python tools/test_skill.py PRODUCT_CMB_TAX_001.yaml --customer test_customer.json

# 测试环境API
curl -X POST https://test.dcc2026.com/api/v1/match -d {...}

# 确认无误后，再部署到生产环境
git push origin main
```

---

### Q3：一个客户可以匹配多少个产品？

**A**：默认返回前5个（按额度降序），可以通过`top_n`参数调整：

```python
matched = matcher.match(customer, top_n=10)  # 返回前10个
```

---

### Q4：如何处理复杂的准入条件（如"A且B或C"）？

**A**：使用嵌套logic（V1.1版本支持）：

```yaml
eligibility:
  logic: "all"
  conditions:
    - field: "monthly_revenue"
      operator: ">="
      value: 100000
    
    # 嵌套条件：(纳税≥1万) 或 (高新企业)
    - logic: "any"
      conditions:
        - field: "annual_tax"
          operator: ">="
          value: 10000
        - field: "high_tech_certification"
          operator: "=="
          value: true
```

---

### Q5：能否让AI自动生成Skill？

**A**：可以，但需要人工审核：

```python
# AI生成建议（不自动生效）
suggestion = rule_generator.generate_product_config(policy_text)

# 人工审核
print(suggestion['suggested_config'])

# 审核通过后，手动创建YAML文件
```

---

## 第七部分：故障排查清单（10分钟）

### 排查清单（按优先级）

#### 1. Skill 加载失败

**症状**：调用 reload API 返回 failed_count > 0

**排查步骤**：
```bash
# 1. 查看 reload 报告
curl -X POST http://localhost:8000/api/v1/reload-skills

# 2. 检查 YAML 语法
python tools/validate_skill.py SKILL_FILE.yaml

# 3. 检查编码（必须 UTF-8）
file -i SKILL_FILE.yaml
```

**常见原因**：
- YAML 缩进错误（必须用空格，不能用 Tab）
- 中文引号（""）误用为英文引号（""）
- 文件编码非 UTF-8
- metadata.status != "active"（会被跳过）

---

#### 2. 匹配结果为空

**症状**：所有客户都匹配 0 个产品

**排查步骤**：
```python
# 单独测试准入条件
product = skill_repo.load_product_by_id("PRODUCT_XXX")
eligible, failed = matcher._check_eligibility(customer, product)
print(f"失败条件: {failed}")
```

**常见原因**：
- 字段单位错误（YAML 用万元，代码传元）
- 字段名拼写错误（montly_revenue vs monthly_revenue）
- 行业黑名单覆盖过宽
- 阈值设置过严

---

#### 3. 额度计算为 0

**症状**：匹配成功但 estimated_amount=0

**排查步骤**：
```python
# 单独测试额度计算
amount = matcher._calculate_amount(customer, product)
print(f"计算额度: {amount}")

# 检查公式
print(f"公式: {product.amount_calculation['formula']}")
print(f"客户数据: {customer.to_dict()}")
```

**常见原因**：
- table 区间未覆盖客户值
- 公式中变量名错误
- amount_range.max 限制过低
- 客户字段值为 None

---

#### 4. 热更新不生效

**症状**：修改 YAML 后结果未变化

**排查步骤**：
```bash
# 1. 确认调用了 reload
curl -X POST http://localhost:8000/api/v1/reload-skills

# 2. 检查缓存 TTL
# 若未调用 reload，等待 60 秒后自动生效

# 3. 检查 Skill 文件路径
ls -la skills/products/banks/cmb/
```

**常见原因**：
- 未调用 reload API
- 文件路径错误（Skill 不在 skills/ 目录下）
- 文件名不匹配（不是 .yaml 后缀）
- metadata.status = "inactive"

---

#### 5. 表达式求值失败

**症状**：RuntimeError: 表达式求值失败

**排查步骤**：
```python
# 测试表达式
from utils.expression_evaluator import ExpressionEvaluator

evaluator = ExpressionEvaluator()
context = {"annual_tax": 10000}

try:
    result = evaluator.evaluate_expression("annual_tax * 30", context)
    print(f"结果: {result}")
except Exception as e:
    print(f"错误: {e}")
```

**常见原因**：
- 表达式中使用未定义变量
- 表达式语法错误（如 `annual_tax x 30`）
- 变量名包含非法字符
- 除零错误

---

### 快速诊断命令

```bash
# 验证所有 Skill
python tools/validate_all_skills.py skills/products/

# 测试单个客户
python tools/test_customer.py --customer-id CUST_001

# 查看加载的 Skill 列表
curl http://localhost:8000/api/v1/skills/list

# 查看审计日志
tail -f logs/audit.log | grep product_match
```

---

## 第八部分：实战演练（30分钟）

### 演练任务

**任务1**：为建设银行创建一个税贷产品Skill
- 准入条件：流水≥8万，纳税≥8千，查询≤8次
- 额度：纳税×20倍，最高30万

**任务2**：测试该Skill是否正确

**任务3**：修改流水要求到10万，并热更新

---

### 演练答案（参考）

```yaml
# skills/products/banks/ccb/PRODUCT_CCB_TAX_001.yaml

skill_id: "PRODUCT_CCB_TAX_001"
skill_name: "建设银行税贷"
skill_type: "product"

product:
  bank: "建设银行"
  bank_code: "CCB"
  category: "税贷"
  product_code: "TAX_LOAN_001"

eligibility:
  logic: "all"
  conditions:
    - field: "monthly_revenue"
      operator: ">="
      value: 80000
      description: "月均流水≥8万"
      dimension: "流水体质"
    
    - field: "annual_tax"
      operator: ">="
      value: 8000
      description: "年纳税≥8千"
      dimension: "票税体质"
    
    - field: "credit_query_6m"
      operator: "<="
      value: 8
      description: "近6月查询≤8次"
      dimension: "征信体质"

amount_calculation:
  formula_type: "expression"
  formula: "min(annual_tax * 20, 300000)"
  amount_range:
    min: 20000
    max: 300000

compliance:
  disclaimer: |
    以上额度仅供初步评估，实际额度以银行审批为准

metadata:
  skill_id: "PRODUCT_CCB_TAX_001"
  skill_name: "建设银行税贷"
  skill_type: "product"
  version: "1.0.0"
  status: "active"
  source: "建设银行官网"
  verified_date: "2026-02-12"
  verified_by: "学员姓名"
  region: "全国"
  confidence: "高"
  created_at: "2026-02-12T14:00:00Z"
  updated_at: "2026-02-12T14:00:00Z"
  created_by: "student@dcc2026.com"
  updated_by: "student@dcc2026.com"
  changelog:
    - version: "1.0.0"
      date: "2026-02-12"
      changes: "初始版本"
      reason: "培训演练"
      approved_by: "培训讲师"
```

---

## 附录A：工具清单

### 验证工具

```bash
# 验证Skill格式
python tools/validate_skill.py SKILL_FILE.yaml

# 批量验证
python tools/validate_all_skills.py skills/products/
```

### 测试工具

```bash
# 测试单个Skill
python tools/test_skill.py SKILL_FILE.yaml --customer CUSTOMER.json

# 批量测试
python tools/test_all_skills.py skills/products/ --customers test_data/
```

### 热更新工具

```bash
# 热更新API
curl -X POST http://localhost:8000/api/v1/reload-skills

# 查看当前加载的Skill
curl http://localhost:8000/api/v1/skills/list
```

---

## 附录B：参考资料

1. **Skill格式规范-v1.0.md**：完整的YAML Schema定义
2. **ProductMatcherRunner原型代码-v1.0.md**：Runner实现细节
3. **V0架构重构方案-AI原生化-v1.0.md**：整体架构说明

---

## 附录C：培训检查清单

- [ ] 理解传统架构 vs Skill架构的区别
- [ ] 能独立编写产品Skill
- [ ] 能独立编写方案Skill
- [ ] 会使用验证工具
- [ ] 会使用测试工具
- [ ] 会触发热更新
- [ ] 理解合规要求
- [ ] 理解审计日志
- [ ] 会调试Skill问题
- [ ] 会使用故障排查清单
- [ ] 完成实战演练

---

**培训完成！欢迎加入DCC2026开发团队！**

---

**文档版本**：v1.0  
**创建日期**：2026-02-12  
**维护人**：培训团队  
**反馈渠道**：training@dcc2026.com
