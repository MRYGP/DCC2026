# V0架构重构方案 - AI原生化（务实版）v1.0

**文档性质**：技术架构级、执行用  
**创建日期**：2026-02-12  
**适用对象**：技术团队/产品团队  
**关联文档**：
- 《项目进度追踪器-v2.4.md》
- 《贷款中介业务基座-完整框架设计-v3.2.md》
- 《诊断逻辑说明.md》
- 《开源+商业模式-v1.0.md》

---

## 一、核心问题诊断

### 当前V0架构的风险

**技术栈**：FastAPI + React + SQLite  
**开发模式**：实习生 + Claude Code（AI辅助开发）

**核心症结**：
```python
# 现在的思路：写死诊断逻辑
def match_products(customer):
    results = []
    for product in PRODUCT_LIST:  # 硬编码产品列表
        if (customer.revenue >= 100000 and  # 硬编码阈值
            customer.tax >= 10000 and
            customer.credit_score >= 600):
            results.append(product)
    return results
```

**问题清单**：
1. ❌ 产品库更新 → 需要改代码、发版
2. ❌ 匹配阈值优化 → 需要改代码、发版
3. ❌ 提醒规则调整 → 需要改代码、发版
4. ❌ 合规口径变化 → 需要改代码、发版
5. ❌ 违背SSOT的"结果库回填优化"要求（无法基于数据自动优化）

**本质**：把AI当成"写代码工具"，而不是"规则引擎执行者"

---

## 二、AI原生化定义（校正版）

### 错误理解（避坑）

❌ **错误1**：让LLM读取YAML并自由执行
- 风险：不可预测、不可审计、合规风险高
- 例子：LLM自己决定匹配逻辑、自己调参数

❌ **错误2**：AI自动优化立即生效
- 风险：黑盒、误伤、无法回滚
- 例子："拒批率>80%→自动提高阈值"

❌ **错误3**：AI管理租户隔离/审计
- 风险：安全关键不能有不确定性

### 正确理解（DCC2026版）

✅ **AI原生化 = 数据/规则热更新 + 确定性执行 + 可审计**

**三层架构**：
```
┌─────────────────────────────────────────────┐
│ Skill定义层（声明式配置）                    │
│ - 产品库（YAML/DB）                         │
│ - 匹配规则（表达式/DSL）                    │
│ - 提醒策略（事件触发规则）                  │
│ - 方案模板（结构化模板）                    │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ Skill Runner（确定性执行引擎）              │
│ - 解析规则                                  │
│ - 执行匹配                                  │
│ - 生成结果                                  │
│ - 记录日志                                  │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ LLM辅助层（建议不自动生效）                 │
│ - 辅助生成规则                              │
│ - 生成解释文案                              │
│ - 提出调参建议                              │
│ - 个性化表述                                │
└─────────────────────────────────────────────┘
```

**LLM的角色**（明确边界）：
- ✅ 辅助抽取政策文本 → 结构化字段（人工审核后入库）
- ✅ 生成解释文案（必须过合规闸门）
- ✅ 提出调参建议（生成候选，人工批准后生效）
- ❌ 不能自由执行
- ❌ 不能自动调参生效
- ❌ 不能绕过合规检查

---

## 三、分层架构设计（V0版本）

### 第1层：传统代码层（确定性，必须稳定）

**不可Skill化的模块**（保持传统代码）：

| 模块 | 为什么不能Skill化 | 技术选型 |
|------|------------------|---------|
| **CRUD操作** | 需要确定性、高性能 | SQLAlchemy ORM |
| **用户认证** | 安全关键，不能有不确定性 | FastAPI Security + JWT |
| **租户隔离** | 合规要求，必须确定性 | Row-level Security |
| **操作留痕** | 审计要求，必须可追溯 | Audit Log Table |
| **数据导出/删除** | GDPR要求，必须确定性 | 标准API |
| **免责声明固化** | 合规红线，系统级硬约束 | 硬编码在模板中 |
| **支付与分佣** | 涉及金钱，必须确定性 | 传统代码 + 强审计 |

**代码示例**：
```python
# 传统代码层：CRUD（保持不变）
class CustomerRepository:
    def create(self, customer: Customer) -> Customer:
        # 标准SQLAlchemy操作
        pass
    
    def get_by_id(self, customer_id: str) -> Customer:
        pass
    
    # 租户隔离（硬编码）
    def get_by_partner(self, partner_id: str) -> List[Customer]:
        return db.query(Customer).filter(
            Customer.partner_id == partner_id
        ).all()
```

---

### 第2层：Skill层（可热更新，业务逻辑）

**可以Skill化的模块**（核心业务逻辑）：

| 模块 | Skill化方式 | 热更新能力 | V0优先级 |
|------|------------|-----------|---------|
| **产品匹配** | 产品库YAML + 规则表达式 | 修改规则立即生效 | P0 ⭐⭐⭐⭐⭐ |
| **方案生成** | 方案模板 + 映射规则 | 新增方案立即可用 | P0 ⭐⭐⭐⭐⭐ |
| **提醒策略** | 事件触发规则 + 频率限制 | 调整规则立即生效 | P0 ⭐⭐⭐⭐⭐ |
| **合规检查** | 禁止表述黑名单 + 统一措辞 | 新增禁用词立即生效 | P0 ⭐⭐⭐⭐⭐ |
| **六宫格评分** | 评分规则表达式 | 调整权重立即生效 | P1 ⭐⭐⭐ |
| **内容生成** | 内容桶结构 + 模板 | 新增模板立即可用 | P1 ⭐⭐⭐ |

---

## 四、Skill定义规范（V0版本）

### Skill标准格式（YAML）

**产品匹配Skill示例**：
```yaml
# 产品库：招商银行税贷
product_id: CMB_TAX_001
product_name: 招商银行税贷
bank: 招商银行
category: 税贷

# 准入条件（使用JSONLogic或简化DSL）
eligibility:
  # 流水要求
  monthly_revenue:
    operator: ">="
    value: 100000
    description: "月均流水≥10万"
  
  # 纳税要求
  annual_tax:
    operator: ">="
    value: 10000
    description: "年纳税≥1万"
  
  # 征信要求
  credit_query_6m:
    operator: "<="
    value: 6
    description: "近6月查询≤6次"
  
  # 行业限制
  industry_blacklist:
    - "房地产"
    - "金融投资"
    - "娱乐业"

# 额度计算规则
amount_formula:
  type: "simple"
  base: "annual_tax * 30"
  max: 1000000
  description: "年纳税×30倍，最高100万"

# 元数据
metadata:
  source: "招商银行官网"
  verified_date: "2026-02-10"
  verified_by: "张三"
  region: "全国"
  confidence: "高"

# 版本控制
version: "1.2"
updated_at: "2026-02-10T15:30:00Z"
updated_by: "admin@dcc2026.com"
changelog:
  - version: "1.2"
    date: "2026-02-10"
    changes: "提高流水要求从5万到10万"
  - version: "1.1"
    date: "2026-02-01"
    changes: "初始版本"
```

**方案模板Skill示例**：
```yaml
# 改造方案：流水优化术
solution_id: SOLUTION_CASHFLOW_001
solution_name: 流水优化术
target_dimension: 流水体质

# 适用场景
applicable_when:
  gap_category: "流水不足"
  gap_severity:
    operator: ">="
    value: 50000
    description: "差距≥5万"

# 执行步骤模板
steps:
  - step_id: 1
    title: "梳理现有客户结构"
    description: "分析客户集中度，识别大客户依赖风险"
    duration_days: 3
    cost: 0
    output: "客户集中度分析表"
  
  - step_id: 2
    title: "优化收款方式"
    description: "引导客户通过对公账户支付，减少私户收款"
    duration_days: 7
    cost: 0
    output: "收款方式优化方案"
  
  - step_id: 3
    title: "增加流水稳定性"
    description: "签订长期合同，固定月度收款"
    duration_days: 30
    cost: 0
    output: "长期合同样本"

# 预估成果
estimated_result:
  time_cost: "1-2个月"
  money_cost: "0-5000元"
  success_rate: 0.75
  success_rate_source: "历史案例库（样本量50）"

# 风险提示
risks:
  - "客户可能不配合对公收款"
  - "需要一定时间积累，无法立即见效"

# 版本控制
version: "1.0"
updated_at: "2026-02-01T10:00:00Z"
updated_by: "admin@dcc2026.com"
```

**提醒策略Skill示例**：
```yaml
# 提醒策略：A类客户立即提醒
reminder_strategy_id: REMINDER_A_CLASS
strategy_name: A类客户立即提醒

# 触发条件（事件驱动）
trigger_event: "customer_diagnosed"
trigger_condition:
  customer_grade: "A"

# 执行规则
action:
  type: "send_notification"
  channel: "wechat_service"
  template_id: "TEMPLATE_A_CLASS"
  
  # 去重规则
  deduplication:
    enabled: true
    window_hours: 24
    description: "24小时内不重复发送"

# 文案生成规则
content_generation:
  use_llm: true
  llm_prompt: |
    基于以下客户信息，生成提醒文案：
    - 客户姓名：{customer_name}
    - 可贷额度：{estimated_amount}
    - 匹配产品：{matched_products}
    
    要求：
    1. 简洁明了（<100字）
    2. 突出额度优势
    3. 包含行动指引
    4. 必须包含免责声明："额度仅供参考，以实际审批为准"
  
  # 合规检查
  compliance_check:
    enabled: true
    forbidden_words:
      - "保证通过"
      - "包下款"
      - "必批"
    required_phrases:
      - "仅供参考"
      - "以实际审批为准"

# 频率限制
rate_limit:
  max_per_customer_per_day: 1
  max_per_customer_per_week: 3

# 版本控制
version: "1.0"
updated_at: "2026-02-05T14:00:00Z"
updated_by: "admin@dcc2026.com"
```

---

## 五、Skill Runner设计（确定性执行引擎）

### 核心职责

**Skill Runner不是LLM**，是一个确定性的Python引擎：
1. 解析Skill定义（YAML）
2. 执行规则（表达式求值）
3. 调用LLM（仅用于文案生成，结果仍需校验）
4. 记录执行日志（审计用）
5. 返回结构化结果

### 产品匹配Runner示例

```python
# skill_runner/product_matcher.py
from typing import List, Dict
import jsonlogic

class ProductMatcherRunner:
    """产品匹配Skill Runner（确定性执行）"""
    
    def __init__(self, product_repo: ProductRepository):
        self.product_repo = product_repo
    
    def match(self, customer: Customer) -> List[MatchedProduct]:
        """
        执行产品匹配（确定性）
        
        不使用LLM自由执行，而是：
        1. 加载产品库（YAML格式）
        2. 逐个评估准入条件（JSONLogic）
        3. 计算额度（表达式求值）
        4. 排序返回
        """
        # 1. 加载产品库
        products = self.product_repo.load_all_products()
        
        results = []
        
        # 2. 逐个评估
        for product in products:
            # 评估准入条件（确定性）
            eligible = self._check_eligibility(
                customer=customer,
                rules=product.eligibility
            )
            
            if not eligible:
                continue
            
            # 计算额度（确定性）
            amount = self._calculate_amount(
                customer=customer,
                formula=product.amount_formula
            )
            
            results.append(MatchedProduct(
                product_id=product.product_id,
                product_name=product.product_name,
                estimated_amount=amount,
                confidence="以实际审批为准"
            ))
        
        # 3. 排序（按额度从高到低）
        results.sort(key=lambda x: x.estimated_amount, reverse=True)
        
        # 4. 记录审计日志
        self._log_audit(
            customer_id=customer.customer_id,
            matched_count=len(results),
            timestamp=datetime.now()
        )
        
        return results
    
    def _check_eligibility(self, customer: Customer, rules: Dict) -> bool:
        """
        评估准入条件（使用JSONLogic或简化DSL）
        
        不使用LLM，确保确定性
        """
        customer_data = {
            "monthly_revenue": customer.monthly_revenue,
            "annual_tax": customer.annual_tax,
            "credit_query_6m": customer.credit_query_6m,
            "industry": customer.industry
        }
        
        # 使用JSONLogic求值（确定性）
        for field, rule in rules.items():
            if field == "industry_blacklist":
                if customer.industry in rule:
                    return False
            else:
                operator = rule["operator"]
                value = rule["value"]
                customer_value = customer_data.get(field)
                
                if not self._evaluate_operator(customer_value, operator, value):
                    return False
        
        return True
    
    def _calculate_amount(self, customer: Customer, formula: Dict) -> float:
        """
        计算额度（确定性）
        
        不使用LLM，使用简单表达式求值
        """
        if formula["type"] == "simple":
            # 简单公式：年纳税 × 30倍
            base_formula = formula["base"]  # "annual_tax * 30"
            amount = eval(base_formula, {"annual_tax": customer.annual_tax})
            
            # 限制最大值
            max_amount = formula.get("max", float("inf"))
            return min(amount, max_amount)
        
        # 可扩展其他公式类型
        raise NotImplementedError(f"Formula type {formula['type']} not supported")
    
    def _evaluate_operator(self, value, operator, threshold) -> bool:
        """求值操作符（确定性）"""
        if operator == ">=":
            return value >= threshold
        elif operator == "<=":
            return value <= threshold
        elif operator == ">":
            return value > threshold
        elif operator == "<":
            return value < threshold
        elif operator == "==":
            return value == threshold
        else:
            raise ValueError(f"Unknown operator: {operator}")
    
    def _log_audit(self, customer_id: str, matched_count: int, timestamp):
        """记录审计日志"""
        AuditLog.create(
            action="product_match",
            customer_id=customer_id,
            result_count=matched_count,
            timestamp=timestamp
        )
```

### 方案生成Runner示例

```python
# skill_runner/solution_generator.py
from typing import List

class SolutionGeneratorRunner:
    """改造方案生成Runner（半AI化）"""
    
    def __init__(self, solution_repo: SolutionRepository, llm_client: LLMClient):
        self.solution_repo = solution_repo
        self.llm_client = llm_client
    
    def generate(self, gap: Gap) -> List[Solution]:
        """
        生成改造方案（确定性检索 + AI文案）
        
        1. 确定性检索方案（基于gap维度）
        2. LLM生成个性化文案（但必须过合规闸门）
        """
        # 1. 确定性检索（不使用LLM）
        solutions = self.solution_repo.find_by_dimension(
            dimension=gap.dimension,
            severity=gap.severity
        )
        
        results = []
        
        for solution in solutions:
            # 2. LLM生成个性化表述（可选）
            personalized_desc = self._generate_personalized_description(
                solution=solution,
                gap=gap
            )
            
            # 3. 合规检查（强制）
            if not self._check_compliance(personalized_desc):
                # 不通过合规，回退到标准描述
                personalized_desc = solution.standard_description
            
            results.append(Solution(
                solution_id=solution.solution_id,
                solution_name=solution.solution_name,
                description=personalized_desc,
                steps=solution.steps,
                estimated_cost=solution.estimated_result["money_cost"],
                estimated_time=solution.estimated_result["time_cost"],
                success_rate=solution.estimated_result["success_rate"],
                disclaimer="仅供初步评估，以实际执行为准"
            ))
        
        return results
    
    def _generate_personalized_description(self, solution, gap) -> str:
        """
        使用LLM生成个性化描述（但必须过合规闸门）
        """
        prompt = f"""
        基于以下信息，生成改造方案描述：
        - 客户差距：{gap.description}
        - 标准方案：{solution.solution_name}
        - 执行步骤：{solution.steps}
        
        要求：
        1. 简洁明了（<200字）
        2. 个性化（针对客户具体情况）
        3. 必须包含免责声明："仅供参考，以实际执行为准"
        4. 禁止使用：保证、必成功、100%等承诺性表述
        """
        
        response = self.llm_client.complete(prompt)
        return response.text
    
    def _check_compliance(self, text: str) -> bool:
        """
        合规检查（硬编码规则，不使用LLM）
        """
        # 加载禁用词黑名单
        forbidden_words = ComplianceConfig.get_forbidden_words()
        
        for word in forbidden_words:
            if word in text:
                return False
        
        # 检查必须包含的免责声明
        required_phrases = ["仅供参考", "以实际", "为准"]
        for phrase in required_phrases:
            if phrase not in text:
                return False
        
        return True
```

---

## 六、LLM辅助层（建议不自动生效）

### LLM的三个角色（明确边界）

#### 角色1：辅助生成规则（人工审核后入库）

**场景**：从政策文本抽取结构化字段

```python
# llm_assistant/rule_generator.py

class RuleGenerator:
    """LLM辅助生成规则（不自动生效）"""
    
    def generate_product_config(self, policy_text: str) -> Dict:
        """
        从政策文本生成产品配置（建议）
        
        注意：生成的配置不自动入库，需要人工审核批准
        """
        prompt = f"""
        从以下银行政策文本中，抽取结构化信息：
        
        {policy_text}
        
        输出JSON格式：
        {{
            "product_name": "...",
            "monthly_revenue": {{"operator": ">=", "value": 100000}},
            "annual_tax": {{"operator": ">=", "value": 10000}},
            "industry_blacklist": ["房地产", "金融"]
        }}
        """
        
        response = self.llm_client.complete(prompt)
        suggested_config = json.loads(response.text)
        
        # 返回建议，不自动入库
        return {
            "suggested_config": suggested_config,
            "source": policy_text,
            "status": "待审核",
            "auto_applied": False
        }
```

**工作流**：
```
政策文本 
  → LLM生成建议配置
  → 人工审核（检查准确性）
  → 批准后入库
  → 版本化、可回滚
```

---

#### 角色2：生成解释文案（必须过合规闸门）

**场景**：个性化提醒文案、方案描述

```python
# llm_assistant/content_generator.py

class ContentGenerator:
    """LLM生成内容（必须过合规检查）"""
    
    def generate_reminder_text(self, customer: Customer, products: List[Product]) -> str:
        """
        生成提醒文案（必须过合规闸门）
        """
        prompt = f"""
        基于以下信息生成提醒文案：
        - 客户：{customer.name}
        - 可贷额度：{products[0].estimated_amount}
        - 产品：{products[0].name}
        
        要求：
        1. 简洁（<100字）
        2. 必须包含："额度仅供参考，以实际审批为准"
        3. 禁止：保证通过、包下款
        """
        
        text = self.llm_client.complete(prompt).text
        
        # 强制合规检查
        if not ComplianceChecker.check(text):
            # 不通过，使用标准模板
            text = self._get_standard_template(customer, products)
        
        return text
    
    def _get_standard_template(self, customer, products) -> str:
        """标准模板（兜底）"""
        return f"""
        【{customer.name}】您好，
        根据初步评估，您可申请{products[0].name}，
        预估额度{products[0].estimated_amount}万元。
        
        注：额度仅供参考，以实际审批为准。
        """
```

---

#### 角色3：提出调参建议（生成候选，人工批准）

**场景**：基于回填数据优化规则

```python
# llm_assistant/optimization_advisor.py

class OptimizationAdvisor:
    """LLM提出优化建议（不自动生效）"""
    
    def suggest_threshold_adjustment(self, product_id: str) -> Dict:
        """
        基于回填数据，提出阈值调整建议
        
        注意：建议不自动生效，需要人工审核批准
        """
        # 1. 加载回填数据
        feedback_data = FeedbackRepository.get_by_product(product_id)
        
        # 分析拒批原因
        reject_reasons = self._analyze_reject_reasons(feedback_data)
        
        # 2. LLM生成调参建议
        prompt = f"""
        基于以下数据，提出阈值调整建议：
        
        产品：{product_id}
        回填样本：{len(feedback_data)}条
        拒批率：{reject_reasons['reject_rate']}
        主要拒批原因：{reject_reasons['top_reasons']}
        
        当前阈值：
        - 流水要求：≥10万
        - 纳税要求：≥1万
        
        请分析：
        1. 是否需要调整阈值？
        2. 建议调整为多少？
        3. 调整的证据是什么？
        4. 预期影响（通过率变化、影响客户数）？
        """
        
        suggestion = self.llm_client.complete(prompt).text
        
        # 3. 返回建议（不自动生效）
        return {
            "product_id": product_id,
            "current_threshold": {...},
            "suggested_threshold": {...},
            "evidence": feedback_data,
            "reasoning": suggestion,
            "status": "待审核",
            "auto_applied": False,  # 关键：不自动生效
            "created_at": datetime.now()
        }
    
    def _analyze_reject_reasons(self, feedback_data) -> Dict:
        """分析拒批原因（确定性统计）"""
        total = len(feedback_data)
        rejected = [f for f in feedback_data if f.result == "rejected"]
        
        reasons = {}
        for feedback in rejected:
            reason = feedback.reject_reason  # 六宫格归因
            reasons[reason] = reasons.get(reason, 0) + 1
        
        return {
            "reject_rate": len(rejected) / total,
            "top_reasons": sorted(reasons.items(), key=lambda x: x[1], reverse=True)[:3]
        }
```

**工作流**：
```
回填数据积累
  → LLM分析并生成调参建议
  → 人工审核（检查合理性）
  → 批准后更新配置
  → 版本化、可回滚
  → 记录审计日志
```

---

## 七、V0实施路线图（3周）

### Week 1（Day 1-7）：基础设施 + 产品匹配Skill化

**Day 1-2：基础设施**
- [ ] 创建Skill定义目录结构（`/skills/products/`, `/skills/solutions/`, `/skills/reminders/`）
- [ ] 设计YAML Schema（产品配置/方案模板/提醒策略）
- [ ] 实现Skill Loader（加载YAML并验证格式）
- [ ] 实现版本控制（Git + 审计日志）

**Day 3-5：产品匹配Skill化**
- [ ] 将现有20个产品转换为YAML格式
- [ ] 实现ProductMatcherRunner（确定性执行引擎）
- [ ] 实现准入条件评估（JSONLogic或简化DSL）
- [ ] 实现额度计算（表达式求值）

**Day 6-7：测试与验证**
- [ ] 单元测试（准入条件评估/额度计算）
- [ ] 集成测试（完整匹配流程）
- [ ] 验收：修改产品YAML → 重新匹配立即生效（不改代码）

**验收标准**：
- ✅ 20个产品全部YAML化
- ✅ 修改阈值立即生效（不需要发版）
- ✅ 所有操作有审计日志

---

### Week 2（Day 8-14）：方案生成 + 提醒策略Skill化

**Day 8-10：方案生成Skill化**
- [ ] 将12个改造方案转换为YAML格式
- [ ] 实现SolutionGeneratorRunner（确定性检索 + LLM文案）
- [ ] 实现合规检查模块（禁用词黑名单/必需短语）
- [ ] 集成LLM（仅用于个性化表述）

**Day 11-13：提醒策略Skill化**
- [ ] 定义提醒策略YAML（A类立即/B类到期等）
- [ ] 实现ReminderRunner（事件触发 + 频率限制）
- [ ] 实现去重机制（24小时窗口）
- [ ] 集成微信服务号推送

**Day 14：测试与验证**
- [ ] 测试方案生成（个性化 + 合规检查）
- [ ] 测试提醒策略（触发条件 + 去重）
- [ ] 验收：调整规则立即生效

**验收标准**：
- ✅ 12个方案全部YAML化
- ✅ LLM生成的文案100%通过合规检查
- ✅ 提醒规则可动态调整

---

### Week 3（Day 15-21）：LLM辅助层 + 完整测试

**Day 15-17：LLM辅助层**
- [ ] 实现RuleGenerator（政策文本 → 结构化配置）
- [ ] 实现OptimizationAdvisor（回填数据 → 调参建议）
- [ ] 实现人工审核工作流（建议 → 审核 → 批准 → 生效）

**Day 18-20：完整集成测试**
- [ ] 端到端测试（导入 → 诊断 → 匹配 → 方案 → 提醒 → 回填）
- [ ] 性能测试（100个客户批量诊断）
- [ ] 安全测试（Prompt injection防护）

**Day 21：朋友1验证准备**
- [ ] 准备演示环境
- [ ] 准备测试数据（10个真实客户）
- [ ] 准备文档（Skill使用手册）

**验收标准**：
- ✅ 完整闭环可运行
- ✅ 热更新能力验证通过
- ✅ 朋友1可以开始使用

---

## 八、验收标准（可测试、可对账）

### 功能验收

**1. 热更新能力验证**
- [ ] 修改产品库YAML → 重新匹配立即生效（不改代码、不重启）
- [ ] 新增方案模板 → 立即可被推荐（不改代码、不重启）
- [ ] 调整提醒策略 → 次日调度生效（不改代码、不重启）
- [ ] 新增禁用词 → 立即生效（不改代码、不重启）

**2. 确定性验证**
- [ ] 相同客户数据 → 相同匹配结果（可复现）
- [ ] 准入条件评估 → 可追溯（知道为什么通过/拒绝）
- [ ] 额度计算 → 可解释（知道公式和参数）

**3. 合规验收**
- [ ] LLM生成的任何文案 → 100%通过合规闸门
- [ ] 免责声明 → 系统级强制，不可删除/修改
- [ ] 禁用词检测 → 100%准确率

**4. 审计验收**
- [ ] 所有Skill变更 → 有版本记录
- [ ] 所有执行 → 有审计日志
- [ ] 所有LLM调用 → 有输入输出记录
- [ ] 可回滚到任意历史版本

---

### 性能验收

**1. 响应时间**
- [ ] 单个客户诊断 → <2秒
- [ ] 批量100个客户 → <30秒
- [ ] Skill热更新 → <1秒生效

**2. 准确性**
- [ ] 产品匹配准确率 → >85%（基于回填数据）
- [ ] 合规检查漏检率 → <1%

---

## 九、风险与应对

### 风险1：Skill定义复杂度

**风险**：YAML配置过于复杂，维护困难

**应对**：
- V0只做简单规则（大于/小于/等于）
- V1再扩展复杂表达式（JSONLogic）
- 提供Skill编辑器（可视化配置）

**验证指标**：运营人员能在10分钟内修改一个产品配置

---

### 风险2：LLM成本

**风险**：每次提醒/方案都调用LLM，成本高

**应对**：
- 标准场景使用模板（不调用LLM）
- 只在个性化场景调用LLM
- 缓存LLM生成的文案（相似客户复用）

**验证指标**：LLM成本 < 总成本的10%

---

### 风险3：Skill恶意篡改

**风险**：租户隔离不严格，中介A修改中介B的Skill

**应对**：
- Skill配置按partner_id隔离
- 所有修改需要认证+授权
- 审计日志记录所有变更

**验证指标**：无跨租户访问事件

---

### 风险4：LLM Prompt Injection

**风险**：恶意客户数据触发Prompt injection

**应对**：
- 所有客户数据先转义再拼接到Prompt
- LLM输出强制过合规闸门
- 异常输出告警（如包含代码/SQL）

**验证指标**：Prompt injection攻击成功率 < 0.1%

---

## 十、对比：改造前 vs 改造后

### 改造前（传统架构）

```python
# 硬编码匹配逻辑
def match_products(customer):
    results = []
    
    # 招商银行税贷（硬编码）
    if (customer.monthly_revenue >= 100000 and
        customer.annual_tax >= 10000 and
        customer.credit_query_6m <= 6):
        results.append({
            "name": "招商银行税贷",
            "amount": customer.annual_tax * 30
        })
    
    # 工商银行税贷（硬编码）
    if (customer.monthly_revenue >= 50000 and
        customer.annual_tax >= 5000):
        results.append({
            "name": "工商银行税贷",
            "amount": customer.annual_tax * 25
        })
    
    # ... 20个if-else
    
    return results
```

**问题**：
- 修改阈值 → 改代码
- 新增产品 → 改代码
- 优化逻辑 → 改代码
- 每次都要发版、测试、部署

---

### 改造后（AI原生架构）

```python
# Skill化匹配逻辑
class ProductMatcherRunner:
    def match(self, customer):
        # 1. 动态加载产品库（YAML）
        products = self.skill_loader.load_products()
        
        # 2. 确定性执行
        results = []
        for product in products:
            if self._check_eligibility(customer, product.rules):
                amount = self._calculate_amount(customer, product.formula)
                results.append(MatchedProduct(product, amount))
        
        return results
```

**产品库（YAML）**：
```yaml
# skills/products/cmb_tax_001.yaml
product_id: CMB_TAX_001
eligibility:
  monthly_revenue: {operator: ">=", value: 100000}
  annual_tax: {operator: ">=", value: 10000}
amount_formula:
  base: "annual_tax * 30"
```

**好处**：
- 修改阈值 → 只改YAML（不改代码）
- 新增产品 → 只加YAML（不改代码）
- 优化逻辑 → 调整公式（不改代码）
- 立即生效（不需要发版）

---

## 十一、下一步行动

### P0（本周）

**1. 架构设计确认**
- [ ] 技术团队评审本方案
- [ ] 确认Skill格式（YAML Schema）
- [ ] 确认Runner接口设计
- [ ] 责任人：技术负责人
- [ ] 输出物：《Skill格式规范-v1.0》

**2. 原型验证**
- [ ] 实现1个产品的Skill化（招商银行税贷）
- [ ] 实现ProductMatcherRunner（MVP版本）
- [ ] 验证热更新能力
- [ ] 责任人：开发团队
- [ ] 输出物：可运行的原型代码

**3. 开发计划调整**
- [ ] 更新V0开发计划（整合Skill化改造）
- [ ] 调整时间表（可能需要3周 → 4周）
- [ ] 更新验收标准
- [ ] 责任人：项目经理
- [ ] 输出物：《V0开发计划-v1.5》

---

### P1（2周内）

**4. 全面Skill化**
- [ ] 20个产品Skill化
- [ ] 12个方案Skill化
- [ ] 提醒策略Skill化
- [ ] 责任人：开发团队

**5. LLM辅助层**
- [ ] 实现RuleGenerator
- [ ] 实现ContentGenerator
- [ ] 实现OptimizationAdvisor
- [ ] 责任人：开发团队

---

### P2（3周内）

**6. 完整测试**
- [ ] 端到端测试
- [ ] 性能测试
- [ ] 安全测试（Prompt injection）
- [ ] 责任人：测试团队

**7. 朋友1验证**
- [ ] 部署到测试环境
- [ ] 朋友1使用并反馈
- [ ] 迭代优化
- [ ] 责任人：产品团队

---

## 十二、变更记录

### v1.0（2026-02-12）

**核心变更**：
1. 定义AI原生化（DCC2026版）：数据/规则热更新 + 确定性执行 + 可审计
2. 设计两层架构：传统代码层（确定性）+ Skill层（可热更新）
3. 明确LLM角色：辅助生成规则/生成文案/提出建议（不自动生效）
4. 制定Skill定义规范（YAML格式）
5. 设计Skill Runner（确定性执行引擎）
6. 给出3周实施路线图

**影响范围**：
- 技术架构：从传统代码 → Skill化架构
- 开发方式：从硬编码 → 配置化
- 维护方式：从发版 → 热更新
- 优化方式：从人工改代码 → AI建议+人工审核

**关键决策**：
- Skill化聚焦P0模块（产品匹配/方案生成/提醒策略）
- 传统代码保留安全关键模块（CRUD/认证/支付）
- LLM不自由执行，只生成建议（人工审核后生效）
- 所有变更版本化、可审计、可回滚

**预期收益**：
- 产品库更新 → 不需要改代码、不需要发版
- 诊断逻辑优化 → 调整配置即可
- 真正的数据飞轮（基于回填优化）
- 6个月后不会被"AI原生竞品"淘汰

**风险提示**：
- 开发周期可能延长（3周 → 4周）
- 需要学习新范式（Skill定义/Runner设计）
- 初期可能不稳定（需要充分测试）

**下一步**：
- P0：架构设计确认（本周）
- P0：原型验证（本周）
- P1：全面Skill化（2周内）

---

**文档版本**：v1.0  
**创建日期**：2026-02-12  
**维护人**：技术团队  
**审核状态**：待审核
