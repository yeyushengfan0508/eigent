# Eigent Agent 任务执行流程解析

## 📋 总体流程

```
用户提交任务
    ↓
ChatHistory 创建（保存任务信息）
    ↓
ListenChatAgent 初始化（创建 AI Agent）
    ↓
Agent 循环执行：思考 → 调用工具 → 获得结果
    ↓
每一步执行记录到 ChatStep
    ↓
任务完成，更新 ChatHistory 状态
```

---

## 🔑 核心数据结构

### 1. ChatHistory（任务信息）
**文件**: `server/app/model/chat/chat_history.py`

```python
class ChatHistory(AbstractModel, DefaultTimes, table=True):
    id: int                      # 任务记录ID
    user_id: int                 # 用户ID
    task_id: str                 # 任务唯一标识（UUID）
    project_id: str              # 项目ID
    question: str                # 用户输入的任务问题
    language: str                # 使用的语言
    model_platform: str          # 模型平台（如 OpenAI, Anthropic）
    model_type: str              # 模型类型（如 gpt-4, claude-3）
    api_key: str                 # API 密钥
    api_url: str                 # API 地址
    max_retries: int             # 最大重试次数
    installed_mcp: str           # 已安装的 MCP 工具列表（JSON）
    status: int                  # 任务状态（1=ongoing, 2=done）
    tokens: int                  # 使用的 token 数
    spend: float                 # 花费的金额
    created_at: datetime         # 创建时间
    updated_at: datetime         # 更新时间
```

### 2. ChatStep（执行步骤）
**文件**: `server/app/model/chat/chat_step.py`

```python
class ChatStep(AbstractModel, DefaultTimes, table=True):
    id: int                      # 步骤ID
    task_id: str                 # 所属任务ID
    step: str                    # 步骤类型（如 "thought", "tool_call", "result"）
    data: dict                   # 步骤的详细数据（JSON）
    timestamp: float             # 执行时间戳
```

**ChatStep 的 step 类型**:
- `thought` — Agent 的思考过程
- `tool_call` — 调用某个工具
- `tool_result` — 工具执行结果
- `message` — 最终回复

---

## 🤖 Agent 执行核心类

### ListenChatAgent
**文件**: `backend/app/agent/listen_chat_agent.py`

这是继承自 CAMEL 框架 `ChatAgent` 的自定义 Agent 类。

**关键方法**:

```python
class ListenChatAgent(ChatAgent):
    def __init__(
        self,
        api_task_id: str,           # API 任务ID
        agent_name: str,            # Agent 名称
        system_message: str,        # 系统提示词
        model: BaseModelBackend,    # 语言模型
        tools: List[FunctionTool],  # 可用工具列表
        memory: AgentMemory,        # 对话记忆
        max_iteration: int,         # 最大迭代次数
        tool_execution_timeout: float,  # 工具执行超时时间
    ):
        # 初始化 Agent
        
    async def step(self):
        # 执行一步：LLM思考 → 调用工具 → 获得结果
        
    def run(self):
        # 运行 Agent 直到完成
```

---

## 📡 API 接口

### 1. 创建/保存任务
**端点**: `POST /api/chat/history`

```json
{
  "task_id": "uuid-xxx",
  "project_id": "project-001",
  "question": "帮我分析这个文件",
  "language": "en",
  "model_platform": "openai",
  "model_type": "gpt-4",
  "api_key": "sk-xxx",
  "api_url": "https://api.openai.com/v1",
  "max_retries": 3,
  "installed_mcp": "[\"mcp_file_handler\", \"mcp_web_search\"]",
  "status": 1
}
```

**响应**: 返回 `ChatHistoryOut` 对象

---

### 2. 获取任务列表
**端点**: `GET /api/chat/histories`

**响应**: 分页返回用户的所有任务

```json
{
  "items": [
    {
      "id": 1,
      "task_id": "uuid-xxx",
      "question": "...",
      "status": 2,
      "created_at": "2026-02-02T10:00:00",
      "tokens": 2500,
      "spend": 0.05
    }
  ],
  "total": 42,
  "page": 1,
  "size": 10
}
```

---

### 3. 获取任务的执行步骤
**端点**: `GET /api/chat/steps?task_id=uuid-xxx`

**响应**: 返回该任务的所有执行步骤

```json
[
  {
    "id": 1,
    "task_id": "uuid-xxx",
    "step": "thought",
    "data": {
      "thought": "用户要求分析文件，我需要先读取文件内容"
    },
    "timestamp": 1706862000.123
  },
  {
    "id": 2,
    "task_id": "uuid-xxx",
    "step": "tool_call",
    "data": {
      "tool_name": "read_file",
      "arguments": {"file_path": "/path/to/file"}
    },
    "timestamp": 1706862001.456
  },
  {
    "id": 3,
    "task_id": "uuid-xxx",
    "step": "tool_result",
    "data": {
      "tool_name": "read_file",
      "result": "文件内容..."
    },
    "timestamp": 1706862002.789
  }
]
```

---

### 4. 播放任务执行过程（SSE 流）
**端点**: `GET /api/chat/steps/playback/{task_id}?delay_time=1`

**响应**: Server-Sent Events 流，逐步回放任务执行过程

```
data: {"id": 1, "step": "thought", "data": {...}}

data: {"id": 2, "step": "tool_call", "data": {...}}

data: {"id": 3, "step": "tool_result", "data": {...}}
```

---

## 🔄 代码执行链路

### 1️⃣ 用户提交任务

**API 端点**: `POST /api/chat/history`

```python
# server/app/controller/chat/history_controller.py
@router.post("/history", name="save chat history")
def create_chat_history(data: ChatHistoryIn, session, auth):
    data.user_id = auth.user.id
    chat_history = ChatHistory(**data.model_dump())
    session.add(chat_history)
    session.commit()
    return chat_history
```

**关键操作**:
- 保存用户的任务信息到 SQLite 数据库
- 返回 `ChatHistory` 对象（包含新分配的 ID）

---

### 2️⃣ 初始化 Agent

**文件**: `backend/app/agent/agent_model.py`

```python
def agent_model(
    agent_name: str,
    system_message: str,
    options: Chat,  # 包含模型配置、tools 等
    tools: list = None,
):
    task_lock = get_task_lock(options.project_id)
    agent_id = str(uuid.uuid4())
    
    # 创建语言模型
    model_factory = ModelFactory()
    model = model_factory.create(
        platform=ModelPlatformType(options.model_platform),
        model_type=ModelType(options.model_type),
        api_key=options.api_key,
        url=options.api_url
    )
    
    # 创建 ListenChatAgent 实例
    agent = ListenChatAgent(
        api_task_id=options.task_id,
        agent_name=agent_name,
        system_message=system_message,
        model=model,
        tools=tools,  # 从 MCP 加载的工具
        max_iteration=10,
        tool_execution_timeout=300
    )
    
    return agent
```

---

### 3️⃣ Agent 循环执行

**文件**: `backend/app/agent/listen_chat_agent.py`

```python
class ListenChatAgent(ChatAgent):
    async def step(self):
        """执行一步"""
        
        # 1. Agent 思考：调用 LLM
        response = await self.model.get_chat_response(
            messages=self.memory.get_context_messages()
        )
        
        # 2. 记录思考步骤
        self._save_step(
            step_type="thought",
            data={"thought": response.thought}
        )
        
        # 3. 如果 LLM 决定调用工具
        if response.tool_calls:
            for tool_call in response.tool_calls:
                # 4. 执行工具
                tool_result = await self._execute_tool(
                    tool_name=tool_call.name,
                    arguments=tool_call.arguments
                )
                
                # 5. 记录工具调用和结果
                self._save_step(
                    step_type="tool_call",
                    data={"tool": tool_call.name, "args": tool_call.arguments}
                )
                self._save_step(
                    step_type="tool_result",
                    data={"result": tool_result}
                )
                
                # 6. 将结果加入对话记忆
                self.memory.add(tool_result)
    
    def run(self):
        """运行 Agent 直到完成"""
        for i in range(self.max_iteration):
            asyncio.run(self.step())
            
            # 检查是否应该停止
            if self._should_stop():
                break
        
        # 返回最终结果
        return self.memory.get_latest_response()
```

**关键概念**:

- **思考 (Thought)**: LLM 分析任务和当前状态
- **工具调用 (Tool Call)**: LLM 决定调用哪个工具和参数
- **工具结果 (Tool Result)**: 工具执行后的结果
- **记忆 (Memory)**: Agent 维护的对话历史，用于上下文理解

---

## 🛠️ 可用的工具（Tools）

工具通过 **MCP (Model Context Protocol)** 加载。

**示例工具**:
- `read_file` — 读取文件内容
- `write_file` — 写入文件
- `web_search` — 搜索网络信息
- `execute_code` — 执行代码
- `database_query` — 查询数据库

---

## 💾 数据持久化流程

### 存储执行步骤

每个 Agent 步骤都被保存到 `ChatStep` 表：

```python
# 伪代码
chat_step = ChatStep(
    task_id=task_id,
    step="thought",
    data={"thought": "..."},
    timestamp=time.time()
)
session.add(chat_step)
session.commit()
```

### 更新任务状态

任务完成后更新 `ChatHistory` 状态：

```python
chat_history = session.query(ChatHistory).filter(
    ChatHistory.task_id == task_id
).first()

chat_history.status = ChatStatus.done  # 2
chat_history.tokens = total_tokens_used
chat_history.spend = calculate_spend(total_tokens)
session.commit()
```

---

## 🔐 认证与授权

所有 API 都需要有效的 JWT Token：

```python
@router.get("/chat/histories")
def list_histories(auth: Auth = Depends(auth_must)):
    user_id = auth.user.id  # 从 Token 中提取用户ID
    # 只返回该用户的任务
```

---

## 📊 数据库关系图

```
User (1)
  ├── ChatHistory (*)  [user_id 索引]
  │     ├── task_id (唯一)
  │     ├── project_id (可选)
  │     └── ChatStep (*)  [task_id 索引]
  │           ├── step (思考/工具调用/结果)
  │           └── data (JSON 详细信息)
  │
  └── Config (*)  [存储 API 密钥等配置]
```

---

## 🚀 尝试这个功能

### 第一步：在 Swagger UI 创建任务

```bash
POST /api/chat/history
Content-Type: application/json

{
  "task_id": "test-001",
  "project_id": "my-project",
  "question": "What is 2+2?",
  "language": "en",
  "model_platform": "openai",
  "model_type": "gpt-4",
  "api_key": "your-api-key",
  "api_url": "https://api.openai.com/v1"
}
```

### 第二步：查看任务信息

```bash
GET /api/chat/histories
```

### 第三步：获取任务的执行步骤

```bash
GET /api/chat/steps?task_id=test-001
```

### 第四步：播放执行过程

```bash
GET /api/chat/steps/playback/test-001
```

---

## 📚 文件导航

| 功能 | 文件位置 |
|------|--------|
| 任务 API | `server/app/controller/chat/history_controller.py` |
| 步骤 API | `server/app/controller/chat/step_controller.py` |
| ChatHistory 模型 | `server/app/model/chat/chat_history.py` |
| ChatStep 模型 | `server/app/model/chat/chat_step.py` |
| Agent 实现 | `backend/app/agent/listen_chat_agent.py` |
| Agent 工厂 | `backend/app/agent/agent_model.py` |
| 工具管理 | `backend/app/agent/tools.py` |
| 提示词 | `backend/app/agent/prompt.py` |

---

## 🎯 核心概念总结

| 概念 | 说明 |
|------|------|
| **Task** | 用户提交的一个任务/问题 |
| **Agent** | AI 代理，负责分析任务和调用工具 |
| **Step** | Agent 执行过程中的一个步骤（思考/工具调用/结果） |
| **Tool** | Agent 可以调用的工具函数 |
| **Memory** | Agent 维护的对话历史 |
| **MCP** | Model Context Protocol，工具加载协议 |
| **Prompt** | 给 Agent 的系统提示词 |

