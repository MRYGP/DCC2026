# ProductMatcherRunner 原型代码 v1.0

**文档性质**：技术实现文档  
**创建日期**：2026-02-12  
**适用对象**：开发团队  
**编程语言**：Python 3.10+

---

## 一、完整代码实现

### 1.1 目录结构

```
dcc2026/
├── skill_runner/
│   ├── __init__.py
│   ├── base.py                    # 基础Runner类
│   ├── product_matcher.py         # 产品匹配Runner
│   ├── solution_generator.py      # 方案生成Runner
│   └── reminder_scheduler.py      # 提醒调度Runner
├── models/
│   ├── __init__.py
│   ├── customer.py                # 客户数据模型
│   ├── product.py                 # 产品数据模型
│   └── matched_result.py          # 匹配结果模型
├── repositories/
│   ├── __init__.py
│   ├── skill_repository.py        # Skill加载器
│   └── audit_repository.py        # 审计日志
├── utils/
│   ├── __init__.py
│   ├── expression_evaluator.py   # 表达式求值器
│   └── audit_logger.py            # 审计日志工具
└── tests/
    ├── __init__.py
    └── test_product_matcher.py    # 单元测试
```

---

### 1.2 数据模型定义

```python
# models/customer.py
from dataclasses import dataclass
from typing import Optional
from datetime import datetime

@dataclass
class Customer:
    """客户数据模型"""
    
    # 基本信息
    customer_id: str
    name: str
    partner_id: str                # 所属中介ID
    
    # 企业信息
    company_name: str
    company_age_years: int         # 企业成立年限
    industry: str                  # 行业
    region: str                    # 地区
    
    # 流水体质
    monthly_revenue: float         # 月均流水
    annual_revenue: float          # 年营收
    revenue_stability: float       # 流水稳定性（0-1）
    customer_concentration: float  # 客户集中度（0-1）
    
    # 票税体质
    annual_invoice: float          # 年开票额
    annual_tax: float              # 年纳税额
    tax_grade: str                 # 纳税等级（A/B/C/D）
    
    # 征信体质
    credit_score: int              # 征信分数
    credit_query_6m: int           # 近6月查询次数
    credit_overdue_m1: int         # M1逾期次数
    credit_overdue_m3: int         # M3+逾期次数
    debt_ratio: float              # 负债率
    small_loan_count: int          # 小贷笔数
    
    # 资产体质
    has_mortgage: bool             # 是否有抵押物
    mortgage_value: float          # 抵押物价值
    mortgage_ratio: float          # 抵押率
    
    # 资质体质
    high_tech_certification: bool  # 是否高新企业
    specialized_certification: bool # 是否专精特新
    
    # 元数据
    created_at: datetime
    updated_at: datetime
    
    def to_dict(self) -> dict:
        """转换为字典（供表达式求值使用）"""
        return {
            "customer_id": self.customer_id,
            "monthly_revenue": self.monthly_revenue,
            "annual_revenue": self.annual_revenue,
            "annual_tax": self.annual_tax,
            "credit_query_6m": self.credit_query_6m,
            "credit_overdue_m3": self.credit_overdue_m3,
            "company_age_years": self.company_age_years,
            "industry": self.industry,
            "high_tech_certification": self.high_tech_certification,
            # ... 其他字段
        }
```

```python
# models/product.py
from dataclasses import dataclass
from typing import List, Dict, Optional

@dataclass
class ProductSkill:
    """产品Skill数据模型"""
    
    # 基本信息
    skill_id: str
    skill_name: str
    skill_type: str = "product"
    
    # 产品属性
    bank: str
    bank_code: str
    category: str
    product_code: str
    
    # 准入条件
    eligibility: Dict              # 准入条件配置
    
    # 额度计算
    amount_calculation: Dict       # 额度计算配置
    
    # 元数据
    metadata: Dict
    
    @classmethod
    def from_dict(cls, data: dict) -> 'ProductSkill':
        """从字典创建ProductSkill对象"""
        return cls(
            skill_id=data['skill_id'],
            skill_name=data['skill_name'],
            skill_type=data.get('skill_type', 'product'),
            bank=data['product']['bank'],
            bank_code=data['product']['bank_code'],
            category=data['product']['category'],
            product_code=data['product']['product_code'],
            eligibility=data['eligibility'],
            amount_calculation=data['amount_calculation'],
            metadata=data['metadata']
        )

@dataclass
class MatchedProduct:
    """匹配结果数据模型"""
    
    # 产品信息
    skill_id: str
    product_name: str
    bank: str
    category: str
    
    # 匹配结果
    estimated_amount: float        # 预估额度
    confidence: str                # 置信度
    
    # 准入情况
    eligible: bool                 # 是否满足准入条件
    failed_conditions: List[str]   # 未满足的条件（如果eligible=False）
    
    # 详细信息
    interest_rate_range: Optional[tuple] = None
    term_options: Optional[List[int]] = None
    features: Optional[List[str]] = None
    
    # 免责声明
    disclaimer: str = "以上额度仅供初步评估，实际额度以银行审批为准"
    
    def to_dict(self) -> dict:
        """转换为字典"""
        return {
            "skill_id": self.skill_id,
            "product_name": self.product_name,
            "bank": self.bank,
            "category": self.category,
            "estimated_amount": self.estimated_amount,
            "confidence": self.confidence,
            "eligible": self.eligible,
            "failed_conditions": self.failed_conditions,
            "disclaimer": self.disclaimer
        }
```

---

### 1.3 Skill仓库（加载器）

```python
# repositories/skill_repository.py
import yaml
from pathlib import Path
from typing import List, Optional
from models.product import ProductSkill
import logging

logger = logging.getLogger(__name__)

class SkillRepository:
    """Skill仓库：负责加载和缓存Skill配置"""
    
    def __init__(self, skills_dir: str = "skills"):
        self.skills_dir = Path(skills_dir)
        self._cache = {}
        self._last_reload = None
    
    def load_all_products(self, force_reload: bool = False) -> List[ProductSkill]:
        """
        加载所有产品Skill
        
        Args:
            force_reload: 是否强制重新加载（默认使用缓存）
        
        Returns:
            List[ProductSkill]: 产品Skill列表
        """
        cache_key = "all_products"
        
        # 检查缓存
        if not force_reload and cache_key in self._cache:
            logger.info(f"从缓存加载产品Skill，共{len(self._cache[cache_key])}个")
            return self._cache[cache_key]
        
        # 加载所有产品Skill文件
        products = []
        products_dir = self.skills_dir / "products"
        
        if not products_dir.exists():
            logger.error(f"产品Skill目录不存在: {products_dir}")
            return []
        
        # 递归查找所有.yaml文件
        for yaml_file in products_dir.rglob("*.yaml"):
            try:
                product = self._load_product_skill(yaml_file)
                
                # 只加载active状态的产品
                if product.metadata.get('status') == 'active':
                    products.append(product)
                    logger.debug(f"已加载产品: {product.skill_name}")
                else:
                    logger.debug(f"跳过非active产品: {product.skill_name}")
            
            except Exception as e:
                logger.error(f"加载产品Skill失败: {yaml_file}, 错误: {e}")
                continue
        
        # 缓存结果
        self._cache[cache_key] = products
        logger.info(f"成功加载{len(products)}个产品Skill")
        
        return products
    
    def load_product_by_id(self, skill_id: str) -> Optional[ProductSkill]:
        """
        根据ID加载单个产品Skill
        
        Args:
            skill_id: Skill ID
        
        Returns:
            ProductSkill or None
        """
        # 先尝试从缓存获取
        all_products = self.load_all_products()
        
        for product in all_products:
            if product.skill_id == skill_id:
                return product
        
        logger.warning(f"未找到产品Skill: {skill_id}")
        return None
    
    def _load_product_skill(self, yaml_file: Path) -> ProductSkill:
        """
        从YAML文件加载产品Skill
        
        Args:
            yaml_file: YAML文件路径
        
        Returns:
            ProductSkill对象
        """
        with open(yaml_file, 'r', encoding='utf-8') as f:
            data = yaml.safe_load(f)
        
        # 验证必需字段
        required_fields = ['skill_id', 'skill_name', 'product', 'eligibility', 'metadata']
        for field in required_fields:
            if field not in data:
                raise ValueError(f"缺少必需字段: {field}")
        
        # 创建ProductSkill对象
        return ProductSkill.from_dict(data)
    
    def reload_skills(self):
        """重新加载所有Skill（热更新）"""
        logger.info("重新加载Skill配置...")
        self._cache.clear()
        self.load_all_products(force_reload=True)
        logger.info("Skill配置重新加载完成")
```

---

### 1.4 表达式求值器

```python
# utils/expression_evaluator.py
import operator
import re
from typing import Any, Dict

class ExpressionEvaluator:
    """表达式求值器：用于评估准入条件"""
    
    # 支持的操作符
    OPERATORS = {
        '>=': operator.ge,
        '<=': operator.le,
        '>': operator.gt,
        '<': operator.lt,
        '==': operator.eq,
        '!=': operator.ne,
    }
    
    @staticmethod
    def evaluate_condition(
        field_value: Any,
        operator_str: str,
        threshold_value: Any
    ) -> bool:
        """
        评估单个条件
        
        Args:
            field_value: 字段值（来自客户数据）
            operator_str: 操作符（如 ">="）
            threshold_value: 阈值（来自Skill配置）
        
        Returns:
            bool: 是否满足条件
        """
        # 处理特殊操作符
        if operator_str == 'in':
            return field_value in threshold_value
        
        if operator_str == 'not_in':
            return field_value not in threshold_value
        
        if operator_str == 'between':
            if not isinstance(threshold_value, list) or len(threshold_value) != 2:
                raise ValueError("between操作符需要长度为2的列表")
            return threshold_value[0] <= field_value <= threshold_value[1]
        
        if operator_str == 'contains':
            return threshold_value in str(field_value)
        
        if operator_str == 'matches':
            return bool(re.match(threshold_value, str(field_value)))
        
        # 标准比较操作符
        if operator_str in ExpressionEvaluator.OPERATORS:
            op_func = ExpressionEvaluator.OPERATORS[operator_str]
            return op_func(field_value, threshold_value)
        
        raise ValueError(f"不支持的操作符: {operator_str}")
    
    @staticmethod
    def evaluate_expression(
        expression: str,
        context: Dict[str, Any]
    ) -> float:
        """
        求值数学表达式（用于额度计算）
        
        Args:
            expression: 表达式字符串（如 "annual_tax * 30"）
            context: 上下文变量（如 {"annual_tax": 15000}）
        
        Returns:
            float: 计算结果
        
        安全性说明：
            只允许基本算术运算，不允许执行任意Python代码
        """
        # 安全检查：只允许数字、运算符、变量名
        allowed_chars = set('0123456789+-*/()._ ')
        allowed_chars.update(set('abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'))
        
        if not all(c in allowed_chars for c in expression):
            raise ValueError(f"表达式包含非法字符: {expression}")
        
        # 检查是否包含危险函数
        dangerous_keywords = ['import', '__', 'eval', 'exec', 'open', 'file']
        if any(keyword in expression.lower() for keyword in dangerous_keywords):
            raise ValueError(f"表达式包含危险关键字: {expression}")
        
        # 替换变量
        safe_expression = expression
        for var_name, var_value in context.items():
            safe_expression = safe_expression.replace(var_name, str(var_value))
        
        # 求值（限制在安全的数学运算）
        try:
            # 只允许基本数学函数
            allowed_names = {
                'min': min,
                'max': max,
                'abs': abs,
                'round': round,
            }
            
            result = eval(safe_expression, {"__builtins__": {}}, allowed_names)
            return float(result)
        
        except Exception as e:
            raise ValueError(f"表达式求值失败: {expression}, 错误: {e}")
```

---

### 1.5 产品匹配Runner（核心实现）

```python
# skill_runner/product_matcher.py
from typing import List, Dict, Optional
from models.customer import Customer
from models.product import ProductSkill, MatchedProduct
from repositories.skill_repository import SkillRepository
from utils.expression_evaluator import ExpressionEvaluator
from utils.audit_logger import AuditLogger
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

class ProductMatcherRunner:
    """
    产品匹配Runner（确定性执行引擎）
    
    职责：
    1. 加载产品Skill
    2. 逐个评估准入条件
    3. 计算预估额度
    4. 排序返回结果
    5. 记录审计日志
    """
    
    def __init__(
        self,
        skill_repo: SkillRepository,
        audit_logger: AuditLogger
    ):
        self.skill_repo = skill_repo
        self.audit_logger = audit_logger
        self.evaluator = ExpressionEvaluator()
    
    def match(
        self,
        customer: Customer,
        top_n: int = 5
    ) -> List[MatchedProduct]:
        """
        执行产品匹配（核心方法）
        
        Args:
            customer: 客户对象
            top_n: 返回前N个产品（默认5个）
        
        Returns:
            List[MatchedProduct]: 匹配结果列表（按额度降序）
        """
        logger.info(f"开始匹配产品，客户ID: {customer.customer_id}")
        
        # 1. 加载产品库
        products = self.skill_repo.load_all_products()
        logger.info(f"已加载{len(products)}个产品Skill")
        
        matched_results = []
        failed_results = []
        
        # 2. 逐个评估产品
        for product in products:
            try:
                # 2.1 评估准入条件
                eligible, failed_conditions = self._check_eligibility(
                    customer=customer,
                    product=product
                )
                
                # 2.2 如果满足准入条件，计算额度
                if eligible:
                    estimated_amount = self._calculate_amount(
                        customer=customer,
                        product=product
                    )
                    
                    matched_results.append(MatchedProduct(
                        skill_id=product.skill_id,
                        product_name=product.skill_name,
                        bank=product.bank,
                        category=product.category,
                        estimated_amount=estimated_amount,
                        confidence="以实际审批为准",
                        eligible=True,
                        failed_conditions=[]
                    ))
                    
                    logger.debug(
                        f"✓ {product.skill_name}: "
                        f"通过准入，预估额度{estimated_amount:.0f}元"
                    )
                
                else:
                    # 记录未通过的产品（用于分析）
                    failed_results.append({
                        "skill_id": product.skill_id,
                        "product_name": product.skill_name,
                        "failed_conditions": failed_conditions
                    })
                    
                    logger.debug(
                        f"✗ {product.skill_name}: "
                        f"未通过准入，原因: {', '.join(failed_conditions)}"
                    )
            
            except Exception as e:
                logger.error(
                    f"评估产品{product.skill_name}时出错: {e}",
                    exc_info=True
                )
                continue
        
        # 3. 排序（按额度降序）
        matched_results.sort(
            key=lambda x: x.estimated_amount,
            reverse=True
        )
        
        # 4. 取前N个
        top_results = matched_results[:top_n]
        
        # 5. 记录审计日志
        self._log_audit(
            customer=customer,
            matched_count=len(matched_results),
            failed_count=len(failed_results),
            top_results=top_results
        )
        
        logger.info(
            f"匹配完成，客户ID: {customer.customer_id}, "
            f"匹配{len(matched_results)}个产品，返回前{len(top_results)}个"
        )
        
        return top_results
    
    def _check_eligibility(
        self,
        customer: Customer,
        product: ProductSkill
    ) -> tuple[bool, List[str]]:
        """
        评估准入条件（确定性）
        
        Args:
            customer: 客户对象
            product: 产品Skill
        
        Returns:
            (是否满足, 未满足的条件列表)
        """
        eligibility = product.eligibility
        logic = eligibility.get('logic', 'all')  # all / any
        conditions = eligibility.get('conditions', [])
        
        failed_conditions = []
        satisfied_conditions = []
        
        # 获取客户数据字典
        customer_data = customer.to_dict()
        
        # 逐个评估条件
        for condition in conditions:
            field = condition['field']
            operator_str = condition['operator']
            threshold_value = condition['value']
            description = condition.get('description', field)
            
            # 获取客户字段值
            field_value = customer_data.get(field)
            
            if field_value is None:
                # 字段缺失
                failed_conditions.append(
                    f"{description} (字段缺失)"
                )
                continue
            
            # 求值
            try:
                satisfied = self.evaluator.evaluate_condition(
                    field_value=field_value,
                    operator_str=operator_str,
                    threshold_value=threshold_value
                )
                
                if satisfied:
                    satisfied_conditions.append(description)
                else:
                    # 生成可读的错误消息
                    error_msg = condition.get('error_message')
                    if error_msg:
                        # 替换占位符
                        error_msg = error_msg.format(
                            actual=field_value,
                            required=threshold_value,
                            gap=abs(field_value - threshold_value) if isinstance(field_value, (int, float)) else None
                        )
                        failed_conditions.append(error_msg)
                    else:
                        failed_conditions.append(
                            f"{description} (实际: {field_value}, 要求: {operator_str} {threshold_value})"
                        )
            
            except Exception as e:
                logger.error(
                    f"评估条件失败: {condition}, 错误: {e}"
                )
                failed_conditions.append(
                    f"{description} (评估失败)"
                )
        
        # 根据logic判断是否满足
        if logic == 'all':
            # 所有条件都满足
            eligible = len(failed_conditions) == 0
        elif logic == 'any':
            # 任意条件满足
            eligible = len(satisfied_conditions) > 0
        else:
            raise ValueError(f"不支持的logic类型: {logic}")
        
        return eligible, failed_conditions
    
    def _calculate_amount(
        self,
        customer: Customer,
        product: ProductSkill
    ) -> float:
        """
        计算预估额度（确定性）
        
        Args:
            customer: 客户对象
            product: 产品Skill
        
        Returns:
            float: 预估额度
        """
        amount_config = product.amount_calculation
        formula_type = amount_config.get('formula_type', 'expression')
        
        if formula_type == 'expression':
            # 表达式求值
            formula = amount_config['formula']
            customer_data = customer.to_dict()
            
            # 求值
            amount = self.evaluator.evaluate_expression(
                expression=formula,
                context=customer_data
            )
            
            # 限制上下限
            amount_range = amount_config.get('amount_range', {})
            min_amount = amount_range.get('min', 0)
            max_amount = amount_range.get('max', float('inf'))
            
            amount = max(min_amount, min(amount, max_amount))
            
            return amount
        
        elif formula_type == 'table':
            # 查表法
            lookup_field = amount_config['lookup_field']
            table = amount_config['table']
            
            field_value = getattr(customer, lookup_field)
            
            # 查找对应区间
            for row in table:
                range_min, range_max = row['range']
                if range_min <= field_value < range_max:
                    return float(row['amount'])
            
            # 未找到匹配区间
            return 0.0
        
        elif formula_type == 'ml_model':
            # 机器学习模型（暂不实现）
            raise NotImplementedError("ML模型暂不支持")
        
        else:
            raise ValueError(f"不支持的formula_type: {formula_type}")
    
    def _log_audit(
        self,
        customer: Customer,
        matched_count: int,
        failed_count: int,
        top_results: List[MatchedProduct]
    ):
        """
        记录审计日志
        
        Args:
            customer: 客户对象
            matched_count: 匹配产品数
            failed_count: 未匹配产品数
            top_results: 返回的TOP结果
        """
        self.audit_logger.log(
            action="product_match",
            customer_id=customer.customer_id,
            partner_id=customer.partner_id,
            details={
                "matched_count": matched_count,
                "failed_count": failed_count,
                "top_products": [
                    {
                        "skill_id": r.skill_id,
                        "product_name": r.product_name,
                        "estimated_amount": r.estimated_amount
                    }
                    for r in top_results
                ],
                "total_products": matched_count + failed_count
            },
            timestamp=datetime.now()
        )
```

---

### 1.6 审计日志工具

```python
# utils/audit_logger.py
from datetime import datetime
from typing import Dict, Any
import logging
import json

logger = logging.getLogger(__name__)

class AuditLogger:
    """审计日志工具"""
    
    def __init__(self, db_connection=None):
        """
        初始化审计日志工具
        
        Args:
            db_connection: 数据库连接（可选）
        """
        self.db = db_connection
    
    def log(
        self,
        action: str,
        customer_id: str,
        partner_id: str,
        details: Dict[str, Any],
        timestamp: datetime
    ):
        """
        记录审计日志
        
        Args:
            action: 操作类型（如 "product_match"）
            customer_id: 客户ID
            partner_id: 中介ID
            details: 详细信息
            timestamp: 时间戳
        """
        log_entry = {
            "action": action,
            "customer_id": customer_id,
            "partner_id": partner_id,
            "details": details,
            "timestamp": timestamp.isoformat()
        }
        
        # 写入日志文件
        logger.info(f"审计日志: {json.dumps(log_entry, ensure_ascii=False)}")
        
        # 写入数据库（如果有）
        if self.db:
            try:
                self._write_to_db(log_entry)
            except Exception as e:
                logger.error(f"写入审计日志到数据库失败: {e}")
    
    def _write_to_db(self, log_entry: dict):
        """写入数据库"""
        # TODO: 实现数据库写入逻辑
        pass
```

---

## 二、使用示例

### 2.1 基本使用

```python
# main.py
from skill_runner.product_matcher import ProductMatcherRunner
from repositories.skill_repository import SkillRepository
from utils.audit_logger import AuditLogger
from models.customer import Customer
from datetime import datetime

# 1. 初始化组件
skill_repo = SkillRepository(skills_dir="skills")
audit_logger = AuditLogger()
matcher = ProductMatcherRunner(
    skill_repo=skill_repo,
    audit_logger=audit_logger
)

# 2. 创建客户对象
customer = Customer(
    customer_id="CUST_001",
    name="张三",
    partner_id="PARTNER_001",
    company_name="某某科技有限公司",
    company_age_years=3,
    industry="软件开发",
    region="北京",
    monthly_revenue=150000,      # 月均流水15万
    annual_revenue=1800000,      # 年营收180万
    revenue_stability=0.85,
    customer_concentration=0.4,
    annual_invoice=1500000,      # 年开票150万
    annual_tax=18000,            # 年纳税1.8万
    tax_grade="B",
    credit_score=720,
    credit_query_6m=3,           # 近6月查询3次
    credit_overdue_m1=0,
    credit_overdue_m3=0,
    debt_ratio=0.45,
    small_loan_count=1,
    has_mortgage=False,
    mortgage_value=0,
    mortgage_ratio=0,
    high_tech_certification=True,  # 有高新认证
    specialized_certification=False,
    created_at=datetime.now(),
    updated_at=datetime.now()
)

# 3. 执行匹配
matched_products = matcher.match(customer, top_n=5)

# 4. 输出结果
print(f"\n为客户 {customer.name} 匹配到 {len(matched_products)} 个产品：\n")

for i, product in enumerate(matched_products, 1):
    print(f"{i}. {product.product_name} ({product.bank})")
    print(f"   预估额度: ¥{product.estimated_amount:,.0f}元")
    print(f"   {product.disclaimer}\n")
```

**输出示例**：
```
为客户 张三 匹配到 3 个产品：

1. 招商银行税贷 (招商银行)
   预估额度: ¥540,000元
   以上额度仅供初步评估，实际额度以银行审批为准

2. 工商银行税贷 (工商银行)
   预估额度: ¥450,000元
   以上额度仅供初步评估，实际额度以银行审批为准

3. 建设银行信用贷 (建设银行)
   预估额度: ¥300,000元
   以上额度仅供初步评估，实际额度以银行审批为准
```

---

### 2.2 Web API集成

```python
# api/routes.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from typing import List
from datetime import datetime

from skill_runner.product_matcher import ProductMatcherRunner
from repositories.skill_repository import SkillRepository
from utils.audit_logger import AuditLogger
from models.customer import Customer

router = APIRouter()

# 初始化（全局单例）
skill_repo = SkillRepository(skills_dir="skills")
audit_logger = AuditLogger()
matcher = ProductMatcherRunner(skill_repo, audit_logger)

class MatchRequest(BaseModel):
    """匹配请求"""
    customer_id: str
    monthly_revenue: float
    annual_tax: float
    credit_query_6m: int
    company_age_years: int
    industry: str
    # ... 其他字段

class MatchResponse(BaseModel):
    """匹配响应"""
    customer_id: str
    matched_count: int
    products: List[dict]

@router.post("/api/v1/match", response_model=MatchResponse)
async def match_products(request: MatchRequest):
    """
    产品匹配API
    
    POST /api/v1/match
    Body: MatchRequest
    Returns: MatchResponse
    """
    try:
        # 1. 构建Customer对象
        customer = Customer(
            customer_id=request.customer_id,
            name="",  # API请求中可能不需要
            partner_id="",  # 从JWT Token中获取
            monthly_revenue=request.monthly_revenue,
            annual_tax=request.annual_tax,
            credit_query_6m=request.credit_query_6m,
            company_age_years=request.company_age_years,
            industry=request.industry,
            # ... 填充其他字段
            created_at=datetime.now(),
            updated_at=datetime.now()
        )
        
        # 2. 执行匹配
        matched_products = matcher.match(customer, top_n=5)
        
        # 3. 返回结果
        return MatchResponse(
            customer_id=request.customer_id,
            matched_count=len(matched_products),
            products=[p.to_dict() for p in matched_products]
        )
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/v1/reload-skills")
async def reload_skills():
    """
    热更新Skill配置
    
    POST /api/v1/reload-skills
    Returns: {"status": "success", "message": "..."}
    """
    try:
        skill_repo.reload_skills()
        return {
            "status": "success",
            "message": "Skill配置已重新加载"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

## 三、单元测试

```python
# tests/test_product_matcher.py
import pytest
from datetime import datetime
from models.customer import Customer
from models.product import ProductSkill
from skill_runner.product_matcher import ProductMatcherRunner
from repositories.skill_repository import SkillRepository
from utils.audit_logger import AuditLogger

@pytest.fixture
def mock_customer():
    """创建测试客户"""
    return Customer(
        customer_id="TEST_001",
        name="测试客户",
        partner_id="PARTNER_001",
        company_name="测试公司",
        company_age_years=3,
        industry="软件开发",
        region="北京",
        monthly_revenue=150000,
        annual_revenue=1800000,
        revenue_stability=0.85,
        customer_concentration=0.4,
        annual_invoice=1500000,
        annual_tax=18000,
        tax_grade="B",
        credit_score=720,
        credit_query_6m=3,
        credit_overdue_m1=0,
        credit_overdue_m3=0,
        debt_ratio=0.45,
        small_loan_count=1,
        has_mortgage=False,
        mortgage_value=0,
        mortgage_ratio=0,
        high_tech_certification=True,
        specialized_certification=False,
        created_at=datetime.now(),
        updated_at=datetime.now()
    )

@pytest.fixture
def matcher():
    """创建ProductMatcherRunner实例"""
    skill_repo = SkillRepository(skills_dir="tests/fixtures/skills")
    audit_logger = AuditLogger()
    return ProductMatcherRunner(skill_repo, audit_logger)

def test_match_returns_results(matcher, mock_customer):
    """测试匹配返回结果"""
    results = matcher.match(mock_customer, top_n=5)
    
    assert len(results) > 0
    assert all(r.eligible for r in results)
    assert all(r.estimated_amount > 0 for r in results)

def test_match_sorted_by_amount(matcher, mock_customer):
    """测试结果按额度降序排列"""
    results = matcher.match(mock_customer, top_n=5)
    
    amounts = [r.estimated_amount for r in results]
    assert amounts == sorted(amounts, reverse=True)

def test_eligibility_check(matcher, mock_customer):
    """测试准入条件检查"""
    skill_repo = matcher.skill_repo
    product = skill_repo.load_product_by_id("PRODUCT_CMB_TAX_001")
    
    eligible, failed_conditions = matcher._check_eligibility(
        customer=mock_customer,
        product=product
    )
    
    assert isinstance(eligible, bool)
    assert isinstance(failed_conditions, list)

def test_amount_calculation(matcher, mock_customer):
    """测试额度计算"""
    skill_repo = matcher.skill_repo
    product = skill_repo.load_product_by_id("PRODUCT_CMB_TAX_001")
    
    amount = matcher._calculate_amount(
        customer=mock_customer,
        product=product
    )
    
    assert amount > 0
    assert amount <= 1000000  # 假设max为100万

def test_failed_eligibility(matcher):
    """测试不满足准入条件的情况"""
    # 创建一个不满足条件的客户
    bad_customer = Customer(
        customer_id="TEST_BAD",
        name="不合格客户",
        partner_id="PARTNER_001",
        monthly_revenue=10000,      # 流水太低
        annual_tax=1000,            # 纳税太低
        credit_query_6m=15,         # 查询次数过多
        # ... 其他字段
        created_at=datetime.now(),
        updated_at=datetime.now()
    )
    
    results = matcher.match(bad_customer, top_n=5)
    
    # 可能没有匹配结果或者很少
    assert len(results) < 3
```

---

## 四、性能优化

### 4.1 缓存机制

```python
# 在SkillRepository中实现缓存
class SkillRepository:
    def __init__(self, skills_dir: str = "skills", cache_ttl: int = 60):
        self.skills_dir = Path(skills_dir)
        self._cache = {}
        self._last_reload = None
        self.cache_ttl = cache_ttl  # 缓存有效期（秒）
    
    def load_all_products(self, force_reload: bool = False) -> List[ProductSkill]:
        """加载所有产品（带缓存）"""
        now = datetime.now()
        
        # 检查缓存是否过期
        if (not force_reload and 
            self._last_reload and 
            (now - self._last_reload).seconds < self.cache_ttl):
            return self._cache.get("all_products", [])
        
        # 重新加载
        products = self._load_products_from_disk()
        self._cache["all_products"] = products
        self._last_reload = now
        
        return products
```

### 4.2 批量匹配

```python
class ProductMatcherRunner:
    def match_batch(
        self,
        customers: List[Customer],
        top_n: int = 5
    ) -> Dict[str, List[MatchedProduct]]:
        """
        批量匹配（性能优化）
        
        Args:
            customers: 客户列表
            top_n: 每个客户返回前N个产品
        
        Returns:
            Dict[customer_id -> matched_products]
        """
        # 只加载一次产品库
        products = self.skill_repo.load_all_products()
        
        results = {}
        
        for customer in customers:
            matched = []
            
            for product in products:
                eligible, _ = self._check_eligibility(customer, product)
                if eligible:
                    amount = self._calculate_amount(customer, product)
                    matched.append(MatchedProduct(...))
            
            matched.sort(key=lambda x: x.estimated_amount, reverse=True)
            results[customer.customer_id] = matched[:top_n]
        
        return results
```

---

## 五、部署说明

### 5.1 依赖安装

```bash
# requirements.txt
PyYAML>=6.0
jsonschema>=4.17.0
python-dateutil>=2.8.2
```

```bash
pip install -r requirements.txt
```

### 5.2 目录结构

```bash
# 创建目录
mkdir -p skills/products/banks/{cmb,icbc,abc}
mkdir -p skills/solutions/{cashflow,tax,credit}
mkdir -p skills/strategies/{reminders,compliance}

# 复制Skill文件
cp PRODUCT_CMB_TAX_001.yaml skills/products/banks/cmb/
```

### 5.3 测试运行

```bash
# 单元测试
pytest tests/test_product_matcher.py -v

# 运行示例
python examples/match_example.py
```

---

## 六、常见问题

### Q1：如何添加新产品？

**A**：
1. 创建新的YAML文件（参考Skill格式规范）
2. 放到 `skills/products/` 目录下
3. 调用热更新API：`POST /api/v1/reload-skills`

### Q2：性能瓶颈在哪里？

**A**：
- YAML文件加载（已通过缓存优化）
- 表达式求值（可考虑预编译）
- 大量客户批量匹配（可使用批量API）

### Q3：如何调试匹配失败？

**A**：
```python
# 启用详细日志
import logging
logging.basicConfig(level=logging.DEBUG)

# 查看未满足的条件
eligible, failed_conditions = matcher._check_eligibility(customer, product)
print(f"未满足的条件: {failed_conditions}")
```

---

## 七、下一步扩展

### 7.1 机器学习模型集成

```python
class MLAmountPredictor:
    """ML模型额度预测器"""
    
    def predict(self, features: Dict) -> float:
        """基于ML模型预测额度"""
        # TODO: 集成scikit-learn/TensorFlow模型
        pass
```

### 7.2 A/B测试支持

```python
class ABTestingMatcher:
    """支持A/B测试的匹配器"""
    
    def match(self, customer: Customer, variant: str = "A") -> List[MatchedProduct]:
        """根据A/B测试变体返回不同结果"""
        if variant == "A":
            # 使用公式A
            pass
        elif variant == "B":
            # 使用公式B
            pass
```

---

**文档版本**：v1.0  
**创建日期**：2026-02-12  
**维护人**：开发团队  
**代码仓库**：[待补充]
