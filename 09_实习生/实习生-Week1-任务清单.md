# Week 1 每日任务清单

**目标**：完成合规底座（4项）+ 业务功能（6项）

---

## Day 2：合规底座（功能1-2）

### 功能1：委托处理协议模板（4小时）

**需求**：
- 固定的协议模板
- 可替换变量：中介公司名称、日期
- 生成PDF文件

**Claude Code提示词**：
```
实现委托处理协议功能：

1. 创建协议模板（Markdown格式）
2. 变量替换功能（公司名称、日期）
3. PDF生成（使用wkhtmltopdf或类似工具）
4. API接口：POST /api/v1/agreement/generate

参数：
- tenant_id: 租户ID
- company_name: 公司名称

响应格式：符合.claude/CLAUDE.md规范

协议内容参考：.claude/business-rules.md
```

**验收**：
- [ ] API能调用
- [ ] PDF能生成
- [ ] 变量替换正确
- [ ] 包含免责声明

---

### 功能2：租户隔离+操作留痕（4小时）

**需求**：
- 所有表都有tenant_id
- 所有查询自动加租户过滤
- 所有操作记录审计日志

**Claude Code提示词**：
```
实现租户隔离和审计日志：

1. 数据库设计：
   - 租户表（tenants）
   - 审计日志表（audit_logs）
   - 所有业务表加tenant_id

2. 中间件实现：
   - 从token提取tenant_id
   - 注入到请求上下文
   - 所有DB查询自动过滤

3. 审计日志：
   - 记录所有操作（view/create/update/delete）
   - 记录用户、时间、IP、资源

参考：.claude/CLAUDE.md 和 .claude/business-rules.md
```

**验收**：
- [ ] 租户表已创建
- [ ] 中间件能工作
- [ ] 审计日志能记录
- [ ] 不同租户数据隔离

---

## Day 3：合规底座（功能3-4）

### 功能3：数据导出/删除机制（3小时）

**需求**：
- 导出所有数据（Excel）
- 删除所有数据（软删除+30天冷却）
- 导出/删除都需要确认

**Claude Code提示词**：
```
实现数据导出和删除：

1. 导出功能：
   - API: GET /api/v1/data/export
   - 导出租户所有数据到Excel
   - 包含所有表

2. 删除功能：
   - API: POST /api/v1/data/delete
   - 软删除（标记deleted_at）
   - 30天后物理删除（定时任务）

3. 安全机制：
   - 需要二次确认
   - 记录审计日志

技术：使用pandas导出Excel
```

**验收**：
- [ ] 能导出Excel
- [ ] 能删除数据
- [ ] 有二次确认
- [ ] 有审计记录

---

### 功能4：免责声明（固化）（1小时）

**需求**：
- 固化在代码中
- 不可修改、不可删除
- 所有报告自动带上

**Claude Code提示词**：
```
实现免责声明：

1. 在constants.py定义免责声明常量
2. 内容参考：.claude/business-rules.md
3. 所有报告生成时自动添加
4. 不接受API参数控制

确保：
- 无法通过API修改
- 无法通过配置修改
- 固化在代码中
```

**验收**：
- [ ] 常量已定义
- [ ] 报告带免责声明
- [ ] 无法修改

---

## Day 4：业务功能（功能5）

### 功能5：批量导入Excel（8小时）

**需求**：
- 上传xlsx文件
- 验证13个必填字段
- 批量插入数据库
- 返回成功/失败统计

**Claude Code提示词**：
```
实现批量导入Excel：

1. API接口：POST /api/v1/customers/upload
2. 接收multipart/form-data（Excel文件）
3. 使用pandas读取
4. 验证13个必填字段（参考.claude/business-rules.md）
5. 批量插入customers表
6. 返回统计信息

必填字段：
- company_name, registered_capital, established_date
- legal_person_name, annual_invoice, annual_tax
- corporate_flow, legal_credit_status, main_business
- industry, loan_demand

验证规则：
- 字段不能为空
- 数字字段必须是数字
- 日期格式正确
- 枚举值有效

响应：
{
  "code": 200,
  "data": {
    "total": 100,
    "success": 95,
    "failed": 5,
    "errors": [...]
  }
}
```

**验收**：
- [ ] 能上传Excel
- [ ] 字段验证正确
- [ ] 能批量插入
- [ ] 错误信息清晰

---

## Day 5：业务功能（功能6）

### 功能6：自动诊断分层（8小时）

**需求**：
- 根据13个字段评分
- 分为A/B/C三层
- 可以看分层理由

**Claude Code提示词**：
```
实现自动诊断分层：

1. 评分算法（参考.claude/business-rules.md）：
   - 流水体质（200分）
   - 票税体质（200分）
   - 征信体质（200分）
   - 成立时间（100分）
   - 其他（100分）

2. 分层规则：
   - A类：总分≥600
   - B类：400≤总分<600
   - C类：总分<400

3. API：POST /api/v1/diagnosis/batch
4. 返回分层结果和评分明细

实现在：services/diagnosis_service.py
```

**验收**：
- [ ] 评分算法正确
- [ ] 分层准确
- [ ] 有评分明细
- [ ] 可以批量诊断

---

## Day 6：业务功能（功能7-8）

### 功能7：三段式输出（4小时）

**需求**：
- 能做什么（A类产品）
- 值得做什么（B类+改造方案）
- 不建议什么（C类）

**Claude Code提示词**：
```
实现三段式输出：

根据诊断结果生成报告：

1. A类客户：
   - 列出符合条件的产品
   - 预估额度
   - 申请流程

2. B类客户：
   - 当前不符合的产品
   - 改造后可做的产品
   - 改造方案概要

3. C类客户：
   - 为什么不建议
   - 给出建议

格式参考：.claude/business-rules.md

API：GET /api/v1/diagnosis/:id/report
```

**验收**：
- [ ] 三段式格式正确
- [ ] 内容准确
- [ ] 带免责声明

---

### 功能8：改造方案生成（4小时）

**需求**：
- 5个模块：缺口+任务+服务商+效果+成本
- 标准输出包
- 可导出PDF

**Claude Code提示词**：
```
实现改造方案生成：

5个模块（参考.claude/business-rules.md）：

1. 缺口清单：客户缺什么
2. 任务清单：需要做什么
3. 服务商推荐：推荐谁
4. 效果预览：改造后的效果
5. 成本估算：需要多少钱

API：POST /api/v1/plans/:customer_id

返回标准方案包（Markdown格式）
```

**验收**：
- [ ] 5个模块齐全
- [ ] 内容准确
- [ ] 可导出PDF

---

## Day 7：业务功能（功能9-10）

### 功能9：自动提醒（3小时）

**需求**：
- A类：立即提醒
- B类：到期提醒
- 定时任务检查

**Claude Code提示词**：
```
实现自动提醒：

1. 提醒规则：
   - A类：导入后立即提醒
   - B类：改造完成日期到期提醒

2. 定时任务（APScheduler）：
   - 每天检查一次
   - 找出需要提醒的客户
   - 发送通知（先实现内部列表）

3. API：GET /api/v1/reminders/pending

实现在：services/reminder_service.py
```

**验收**：
- [ ] A类立即提醒
- [ ] B类到期提醒
- [ ] 定时任务运行

---

### 功能10：可解释+人工复核（3小时）

**需求**：
- 每个分层能说清为什么
- 可以人工改判
- 改判要记录理由

**Claude Code提示词**：
```
实现可解释和人工复核：

1. 可解释：
   - 返回评分明细
   - 每项得分和理由

2. 人工复核：
   - API：POST /api/v1/diagnosis/:id/override
   - 参数：new_tier, reason
   - 记录改判历史

3. 查看改判记录：
   - API：GET /api/v1/diagnosis/:id/history

实现在：services/diagnosis_service.py
```

**验收**：
- [ ] 能看评分明细
- [ ] 能人工改判
- [ ] 改判有记录

---

## 每日工作流程

### 开始工作（09:00）

```bash
# 1. 拉取最新代码
cd E:\DCC2026
git pull

# 2. 激活环境
cd backend
venv\Scripts\activate

# 3. 启动Claude Code
cd E:\DCC2026
claude

# 4. 查看今天任务
cat 00_核心文档/实习生-Week1-任务清单.md
```

---

### 工作中

- 每完成一个小功能就commit
- 遇到问题先问Claude Code
- 不要堆积多个功能

---

### 结束工作（18:00）

```bash
# 1. 提交代码
git add .
git commit -m "Day X: [今天完成的功能]"
git push

# 2. 更新进度
# 在任务清单上打勾

# 3. 记录明天要做的
```

---

## ✅ Week 1目标

**Day 7结束时应该完成**：
- ✅ 合规底座（4项功能）
- ✅ 业务功能（6项功能）
- ✅ 所有功能可运行
- ✅ 通过基本测试

**完成标准**：
- 所有API能调用
- 功能符合需求
- 代码有注释
- Git有提交记录

---

**加油！一周很快就过去了！** 🚀
