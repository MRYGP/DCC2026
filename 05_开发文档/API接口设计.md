# API接口设计 / API Design

**版本**：v0.1（占位）  
**更新日期**：2026-01-27  
**状态**：📝 待开发时详细补充

---

## 说明

本文档将在V0开发启动时详细补充，包含：

1. **RESTful API规范**
   - URL设计规范
   - HTTP方法使用规范
   - 状态码规范
   - 错误响应格式

2. **认证与授权**
   - JWT Token机制
   - 租户隔离实现
   - 权限控制

3. **核心接口清单**
   - 客户管理接口
   - 诊断接口
   - 方案生成接口
   - 配置管理接口
   - 订单管理接口

4. **接口文档**
   - 请求参数
   - 响应格式
   - 示例代码

---

## 快速预览（核心接口）

### 基础URL

```
开发环境：http://localhost:8000/api/v1
生产环境：https://api.{tenant-domain}/api/v1
```

### 客户管理

```
POST   /customers/upload      # 批量导入Excel
GET    /customers             # 客户列表
GET    /customers/:id         # 客户详情
PUT    /customers/:id         # 更新客户
DELETE /customers/:id         # 删除客户
```

### 诊断功能

```
POST   /diagnosis/batch       # 批量诊断
GET    /diagnosis/:customer_id # 获取诊断结果
POST   /diagnosis/:id/override # 人工改判
```

### 方案生成

```
POST   /plans/:customer_id    # 生成改造方案
GET    /plans/:id             # 获取方案详情
GET    /plans/:id/pdf         # 导出PDF
```

### 配置管理

```
GET    /config/tenant         # 获取租户配置
PUT    /config/tenant         # 更新租户配置
GET    /config/products       # 获取产品库配置
PUT    /config/products       # 更新产品库配置
```

---

## 响应格式标准

### 成功响应

```json
{
  "code": 200,
  "message": "success",
  "data": {
    // 业务数据
  }
}
```

### 错误响应

```json
{
  "code": 400,
  "message": "参数错误",
  "errors": [
    {
      "field": "company_name",
      "message": "企业名称不能为空"
    }
  ]
}
```

---

**待补充**：完整的接口文档将在开发时添加

**参考文档**：
- [V0开发计划](V0开发计划.md)
- [技术栈选择](技术栈选择.md)
- [数据库设计](数据库设计.md)

**API文档生成**：
- FastAPI会自动生成交互式文档（/docs）
- 开发时可以直接在浏览器测试接口
