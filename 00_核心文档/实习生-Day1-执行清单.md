# 实习生Day 1执行清单

**今天目标**：环境配置完成 + 运行Hello World + 理解项目

---

## ⏰ 时间安排

- **09:00-12:00**：环境配置（3小时）
- **13:00-14:00**：阅读文档（1小时）
- **14:00-16:00**：第一次开发尝试（2小时）
- **16:00-17:00**：总结和准备明天（1小时）

---

## 📋 上午任务（09:00-12:00）

### Task 1：安装基础软件（30分钟）

```bash
# 检查Python版本
python --version
# 应该显示：Python 3.11.x 或更高

# 检查Node.js版本
node --version
# 应该显示：v18.x 或更高

# 检查Git版本
git --version
# 应该有版本号
```

**如果没有安装**：
- Python：https://www.python.org/downloads/
- Node.js：https://nodejs.org/
- Git：https://git-scm.com/

**完成标志**：三个命令都能返回版本号

---

### Task 2：订阅Claude Max（15分钟）

1. 访问：https://claude.ai/
2. 注册/登录账号
3. 点击"Upgrade to Max"
4. 选择Max计划（$200/月）
5. 完成支付

**为什么Max而不是Pro**：
- 使用量大（是Pro的5-20倍）
- 响应优先级高
- V0开发需要高频使用

**完成标志**：能看到Max标识

---

### Task 3：安装Claude Code（30分钟）

**访问官网下载**：
https://claude.com/product/claude-code

**或者命令行安装**：
```bash
# Mac/Linux
curl -fsSL https://code.anthropic.com/install.sh | sh

# Windows - 下载安装包
# 从官网下载 .exe 文件并安装
```

**验证安装**：
```bash
claude --version
# 应该显示版本号
```

**完成标志**：`claude --version`有输出

---

### Task 4：克隆项目（5分钟）

```bash
# 进入E盘
cd E:\

# 克隆项目
git clone https://github.com/MRYGP/DCC2026.git

# 进入项目
cd DCC2026

# 查看项目结构
dir
# 或 ls -la
```

**完成标志**：能看到项目文件夹

---

### Task 5：后端环境配置（40分钟）

```bash
# 创建backend目录
mkdir backend
cd backend

# 创建虚拟环境
python -m venv venv

# 激活虚拟环境
# Windows:
venv\Scripts\activate
# Mac/Linux:
source venv/bin/activate

# 看到(venv)前缀表示成功

# 安装依赖
pip install fastapi uvicorn sqlalchemy pydantic python-multipart pandas openpyxl alembic apscheduler python-dotenv

# 创建基本结构
mkdir app
cd app
type nul > __init__.py
type nul > main.py
# Mac/Linux: touch __init__.py main.py
```

**完成标志**：依赖安装成功，没有报错

---

### Task 6：前端环境配置（40分钟）

```bash
# 回到项目根目录
cd E:\DCC2026

# 创建前端项目
npm create vite@latest frontend -- --template react-ts

# 进入前端目录
cd frontend

# 安装依赖
npm install

# 安装UI库和工具
npm install antd axios @tanstack/react-query dayjs
```

**完成标志**：`npm install`成功完成

---

### Task 7：午休前检查（5分钟）

- [ ] Python能运行
- [ ] Node.js能运行
- [ ] Claude Code已安装
- [ ] 项目已克隆
- [ ] 后端虚拟环境已创建
- [ ] 前端项目已创建

**都完成了？去吃午饭！** 🍜

---

## 📋 下午任务（13:00-17:00）

### Task 8：阅读核心文档（60分钟）

**按顺序阅读**：

1. **总纲**（15分钟）
   - 文件：`00_核心文档/贷款中介业务基座-总纲-v1.0.md`
   - 重点：项目定位、目标用户、商业模式

2. **V0开发计划**（30分钟）
   - 文件：`05_开发文档/V0开发计划.md`
   - 重点：14项功能、开发排期、验收标准

3. **项目规范**（15分钟）
   - 文件：`.claude/CLAUDE.md`
   - 重点：代码规范、业务规则、工作流程

**读完后问自己**：
- 项目要做什么？
- 我要开发哪14项功能？
- 开发规范是什么？

---

### Task 9：运行Hello World（30分钟）

**后端Hello World**：

编辑：`backend/app/main.py`
```python
from fastapi import FastAPI

app = FastAPI(title="贷款中介业务基座")

@app.get("/")
def read_root():
    return {
        "code": 200,
        "message": "success",
        "data": {"status": "V0开发启动！"}
    }

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

启动后端：
```bash
cd E:\DCC2026\backend
venv\Scripts\activate
uvicorn app.main:app --reload
```

测试：
- 浏览器访问：http://localhost:8000
- 应该看到JSON响应
- 访问：http://localhost:8000/docs
- 应该看到API文档

**前端Hello World**：

```bash
cd E:\DCC2026\frontend
npm run dev
```

测试：
- 浏览器访问：http://localhost:5173
- 应该看到Vite + React页面

**完成标志**：两个服务都能运行

---

### Task 10：第一次使用Claude Code（60分钟）

**启动Claude Code**：
```bash
cd E:\DCC2026
claude
```

**对话1：了解项目**
```
分析这个项目：
1. 项目目标是什么
2. 有哪些核心模块
3. 我应该从哪里开始开发

参考.claude/目录的文档
```

**对话2：生成第一个功能**
```
实现"委托处理协议模板"功能：

需求：
- 创建一个固定的协议模板
- 可以替换公司名称、日期等变量
- 返回PDF文件

技术要求：
- FastAPI后端
- 符合统一响应格式（参考.claude/CLAUDE.md）
- 协议内容在.claude/business-rules.md中

实现步骤：
1. 先给我实现计划
2. 确认后再生成代码
```

**对话3：测试功能**
```
刚才生成的协议功能，帮我：
1. 写一个测试脚本
2. 验证能否正常生成PDF
3. 检查变量替换是否正确
```

**完成标志**：
- 成功生成了代码
- 代码能运行
- 功能基本可用

---

### Task 11：总结和准备（30分钟）

**今天完成情况**：

环境配置：
- [ ] Python已安装并能运行
- [ ] Node.js已安装并能运行
- [ ] Claude Code已安装
- [ ] 项目已克隆
- [ ] 后端环境已配置
- [ ] 前端环境已配置

功能验证：
- [ ] 后端Hello World能运行
- [ ] 前端Hello World能运行
- [ ] Claude Code能对话
- [ ] 用Claude Code生成了第一个功能

理解程度：
- [ ] 知道项目要做什么
- [ ] 知道14项功能是什么
- [ ] 知道开发规范
- [ ] 会用Claude Code

**提交代码**：
```bash
cd E:\DCC2026
git add .
git commit -m "Day 1: 环境配置完成 + Hello World"
git push
```

**准备明天**：
- 阅读：`05_开发文档/V0开发计划.md` Day 2部分
- 明天要做：合规底座的功能1-2

---

## 🆘 遇到问题怎么办

### 环境配置问题

**Python安装失败**：
```
问Claude Code：
我在安装Python时遇到错误：[粘贴错误信息]
帮我解决
```

**Claude Code安装失败**：
- 检查网络连接
- 尝试从官网下载安装包
- 问Claude Desktop怎么解决

### 代码问题

**代码运行报错**：
```
问Claude Code：
我运行这段代码报错：
[粘贴代码]

错误信息：
[粘贴完整错误]

帮我修复
```

### 需求不清楚

**不理解某个功能**：
```
问Claude Code：
关于"委托处理协议"功能，我不确定：
1. 协议模板在哪里
2. 需要替换哪些变量
3. 如何生成PDF

参考.claude/目录的文档，帮我理解
```

---

## ✅ Day 1完成标准

**必须完成**：
- ✅ 所有软件已安装
- ✅ 后端和前端都能运行
- ✅ Claude Code能使用
- ✅ 理解项目目标和任务

**加分项**：
- ⭐ 用Claude Code生成了第一个功能
- ⭐ 代码已提交到Git

**未完成也没关系**：
- 明天继续
- 重要的是理解流程
- 不要有压力

---

## 📅 明天预告

**Day 2任务**：
- 合规底座功能1：委托处理协议模板
- 合规底座功能2：租户隔离+操作留痕

**需要准备**：
- 今晚好好休息
- 明早准时开始
- 带着信心来！

---

**加油！第一天最关键！** 🚀
