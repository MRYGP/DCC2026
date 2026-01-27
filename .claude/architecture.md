# 项目架构说明

**用途**：让Claude Code理解项目整体架构

---

## 技术架构（三层）

```
┌─────────────────────────────────────┐
│         前端（React + Ant Design）   │
│  - 客户管理界面                      │
│  - 诊断结果展示                      │
│  - 方案生成界面                      │
│  - 配置管理界面                      │
└─────────────┬───────────────────────┘
              │ HTTP/REST API
              │
┌─────────────▼───────────────────────┐
│      后端（FastAPI + SQLAlchemy）    │
│  - API路由层                         │
│  - 业务逻辑层                        │
│  - 数据访问层                        │
│  - 中间件（认证、日志、租户隔离）    │
└─────────────┬───────────────────────┘
              │ SQL
              │
┌─────────────▼───────────────────────┐
│          数据库（SQLite）            │
│  - 租户表                            │
│  - 客户表                            │
│  - 诊断记录表                        │
│  - 改造方案表                        │
│  - 审计日志表                        │
└─────────────────────────────────────┘
```

---

## 后端目录结构

```
backend/
├── app/
│   ├── main.py                  # FastAPI入口
│   │   - 创建app实例
│   │   - 注册路由
│   │   - 配置中间件
│   │   - 启动事件
│   │
│   ├── config.py                # 配置文件
│   │   - 数据库配置
│   │   - API配置
│   │   - 常量定义
│   │
│   ├── database.py              # 数据库连接
│   │   - 创建engine
│   │   - Session管理
│   │   - 依赖注入
│   │
│   ├── models/                  # SQLAlchemy模型
│   │   ├── __init__.py
│   │   ├── tenant.py           # 租户模型
│   │   ├── customer.py         # 客户模型
│   │   ├── diagnosis.py        # 诊断记录模型
│   │   ├── plan.py             # 改造方案模型
│   │   └── audit_log.py        # 审计日志模型
│   │
│   ├── schemas/                 # Pydantic模式（数据验证）
│   │   ├── __init__.py
│   │   ├── tenant.py
│   │   ├── customer.py
│   │   ├── diagnosis.py
│   │   └── response.py         # 统一响应格式
│   │
│   ├── api/                     # API路由
│   │   ├── __init__.py
│   │   ├── customers.py        # 客户管理API
│   │   │   - POST /customers/upload    # 批量导入
│   │   │   - GET /customers            # 客户列表
│   │   │   - GET /customers/:id        # 客户详情
│   │   │   - PUT /customers/:id        # 更新客户
│   │   │   - DELETE /customers/:id     # 删除客户
│   │   │
│   │   ├── diagnosis.py        # 诊断API
│   │   │   - POST /diagnosis/batch     # 批量诊断
│   │   │   - GET /diagnosis/:id        # 诊断结果
│   │   │   - POST /diagnosis/:id/override  # 人工改判
│   │   │
│   │   ├── plans.py            # 方案API
│   │   │   - POST /plans/:customer_id  # 生成方案
│   │   │   - GET /plans/:id            # 方案详情
│   │   │   - GET /plans/:id/pdf        # 导出PDF
│   │   │
│   │   └── config.py           # 配置API
│   │       - GET /config/tenant        # 获取租户配置
│   │       - PUT /config/tenant        # 更新租户配置
│   │
│   ├── services/                # 业务逻辑
│   │   ├── __init__.py
│   │   ├── diagnosis_service.py  # 诊断逻辑
│   │   │   - 客户体质评分
│   │   │   - 产品匹配算法
│   │   │   - A/B/C分层
│   │   │
│   │   ├── plan_service.py       # 方案生成
│   │   │   - 缺口识别
│   │   │   - 方案推荐
│   │   │   - ROI计算
│   │   │
│   │   ├── export_service.py     # 导出功能
│   │   │   - Excel导出
│   │   │   - PDF生成
│   │   │
│   │   └── reminder_service.py   # 提醒功能
│   │       - A类立即提醒
│   │       - B类到期提醒
│   │
│   ├── utils/                   # 工具函数
│   │   ├── __init__.py
│   │   ├── auth.py             # 认证中间件
│   │   ├── logger.py           # 日志工具
│   │   └── constants.py        # 常量定义
│   │
│   └── middleware/              # 中间件
│       ├── __init__.py
│       ├── tenant_context.py   # 租户上下文
│       └── audit_logger.py     # 审计日志
│
├── data/                        # 数据目录
│   ├── database.db             # SQLite数据库
│   ├── uploads/                # 上传文件
│   └── exports/                # 导出文件
│
├── tests/                       # 测试
│   ├── test_api/
│   ├── test_services/
│   └── test_models/
│
├── alembic/                     # 数据库迁移
│   └── versions/
│
├── requirements.txt             # Python依赖
└── README.md
```

---

## 前端目录结构

```
frontend/
├── src/
│   ├── main.tsx                # 入口文件
│   │   - React渲染
│   │   - Router配置
│   │
│   ├── App.tsx                 # 根组件
│   │   - 布局
│   │   - 路由
│   │
│   ├── api/                    # API封装
│   │   ├── client.ts           # Axios配置
│   │   │   - baseURL
│   │   │   - 拦截器
│   │   │   - 错误处理
│   │   │
│   │   ├── customer.ts         # 客户API
│   │   ├── diagnosis.ts        # 诊断API
│   │   ├── plan.ts             # 方案API
│   │   └── config.ts           # 配置API
│   │
│   ├── components/             # 通用组件
│   │   ├── Layout/             # 布局组件
│   │   │   ├── Header.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   └── Footer.tsx
│   │   │
│   │   ├── UploadExcel.tsx     # Excel上传
│   │   ├── CustomerTable.tsx   # 客户列表
│   │   ├── DiagnosisCard.tsx   # 诊断结果卡片
│   │   └── PlanGenerator.tsx   # 方案生成器
│   │
│   ├── pages/                  # 页面
│   │   ├── Dashboard.tsx       # 仪表盘
│   │   ├── CustomerList.tsx    # 客户列表
│   │   ├── CustomerDetail.tsx  # 客户详情
│   │   ├── DiagnosisList.tsx   # 诊断列表
│   │   ├── PlanList.tsx        # 方案列表
│   │   └── Settings.tsx        # 配置页面
│   │
│   ├── hooks/                  # 自定义Hooks
│   │   ├── useCustomers.ts     # 客户数据
│   │   ├── useDiagnosis.ts     # 诊断数据
│   │   └── usePlans.ts         # 方案数据
│   │
│   ├── types/                  # TypeScript类型
│   │   ├── customer.ts
│   │   ├── diagnosis.ts
│   │   ├── plan.ts
│   │   └── api.ts              # API响应类型
│   │
│   ├── utils/                  # 工具函数
│   │   ├── format.ts           # 格式化
│   │   ├── validation.ts       # 验证
│   │   └── constants.ts        # 常量
│   │
│   └── styles/                 # 样式
│       └── global.css
│
├── public/                     # 静态资源
│   ├── logo.png
│   └── favicon.ico
│
├── package.json
├── vite.config.ts
├── tsconfig.json
└── README.md
```

---

## 数据流（示例：批量导入）

```
1. 用户上传Excel
   ↓
2. 前端：UploadExcel组件
   - 文件选择
   - 前端验证（文件类型、大小）
   - 调用API
   ↓
3. 后端：POST /api/v1/customers/upload
   - 接收文件
   - 读取Excel（pandas）
   - 数据验证（Pydantic）
   - 租户注入（middleware）
   - 批量插入（SQLAlchemy）
   - 记录审计日志
   - 返回统一响应
   ↓
4. 前端：显示结果
   - 成功：显示导入统计
   - 失败：显示错误详情
```

---

## 中间件执行顺序

```
请求 → 中间件1 → 中间件2 → 中间件3 → API处理器 → 响应
        ↓          ↓          ↓
     租户上下文  审计日志   异常处理
```

**中间件1：租户上下文**
- 从token/session提取tenant_id
- 注入到请求上下文
- 后续所有DB查询自动带租户过滤

**中间件2：审计日志**
- 记录请求信息
- 记录响应结果
- 写入audit_logs表

**中间件3：异常处理**
- 捕获所有异常
- 转换为统一响应格式
- 记录错误日志

---

## 核心设计模式

### 1. 统一响应格式

**所有API都返回**：
```json
{
  "code": 200,
  "message": "success",
  "data": { /* 业务数据 */ }
}
```

**好处**：
- 前端统一处理
- 错误处理标准化
- 易于调试

### 2. 依赖注入

**数据库Session**：
```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/customers")
def get_customers(db: Session = Depends(get_db)):
    # db会自动注入
    customers = db.query(Customer).all()
    return customers
```

### 3. 分层架构

```
API层（路由）
  ↓ 调用
Service层（业务逻辑）
  ↓ 调用
DAO层（数据访问）
  ↓ 操作
数据库
```

**好处**：
- 职责清晰
- 易于测试
- 易于维护

---

## 关键技术决策

### 为什么用FastAPI？
- 自动生成API文档（/docs）
- Pydantic数据验证（减少bug）
- 异步支持（性能好）
- 类型提示（IDE友好）

### 为什么用SQLite？
- V0阶段够用（支持100个中介）
- 无需安装配置（开发快）
- 文件数据库（易备份）
- 后期可无缝升级PostgreSQL

### 为什么用Ant Design？
- 组件丰富（表格、表单、上传等）
- 专为B端设计
- 中文文档完善
- 社区活跃

---

**最后更新**：2026-01-27
