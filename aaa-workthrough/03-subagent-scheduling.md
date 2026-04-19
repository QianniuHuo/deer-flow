# DeerFlow Subagent 调度系统设计

## 概述

DeerFlow 通过 `task` 工具实现子代理派发，支持复杂任务分解与多 Agent 协作。系统采用三层线程池架构，通过 SSE 事件流实现父子 Agent 间的实时通信，并通过 Middleware 在 `after_model` 阶段强制截断超额调用，实现精细化的并发控制。

---

## 一、核心架构

### 1.1 组件关系

```
task_tool (task)
    │
    ▼
SubagentExecutor.execute_async()
    │
    ├── _scheduler_pool (3 workers) → 调度任务
    │
    └── _execution_pool (3 workers) → 执行子代理
             │
             ▼
        agent.astream() → SSE events → parent agent
```

### 1.2 目录结构

**文件**: `packages/harness/deerflow/subagents/`

```
subagents/
├── __init__.py
├── config.py          # SubagentConfig 数据类
├── executor.py        # SubagentExecutor 核心
├── registry.py        # 子代理注册表
└── builtins/
    ├── __init__.py    # BUILTIN_SUBAGENTS 字典
    ├── general_purpose.py
    └── bash_agent.py
```

---

## 二、Task 工具实现

**文件**: `packages/harness/deerflow/tools/builtins/task_tool.py`

### 2.1 工具签名

```python
@tool("task", parse_docstring=True)
async def task_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,       # 简短描述 (3-5 词)
    prompt: str,           # 子代理任务描述
    subagent_type: str,    # "general-purpose" 或 "bash"
    tool_call_id: Annotated[str, InjectedToolCallId],
    max_turns: int | None = None,  # 可选覆盖
) -> str:
```

### 2.2 执行流程

**Lines 114-248**:

```python
async def task_tool(...):
    # 1. 验证子代理类型
    config = get_subagent_config(subagent_type)
    
    # 2. 提取上下文
    sandbox_state = runtime.state.get("sandbox")
    thread_data = runtime.state.get("thread_data")
    thread_id = runtime.context.get("thread_id")
    
    # 3. 创建执行器
    executor = SubagentExecutor(
        config=config,
        tools=tools,
        parent_model=parent_model,
        sandbox_state=sandbox_state,
        thread_data=thread_data,
        thread_id=thread_id,
        trace_id=trace_id,
    )
    
    # 4. 异步启动
    task_id = executor.execute_async(prompt, task_id=tool_call_id)
    
    # 5. 轮询结果
    while True:
        result = get_background_task_result(task_id)
        
        if result.status == "COMPLETED":
            return f"Task Succeeded. Result: {result.result}"
        
        elif result.status == "FAILED":
            return f"Task Failed. Error: {result.error}"
        
        # 继续轮询 (5s 间隔)
        await asyncio.sleep(5)
```

---

## 三、线程池管理

**文件**: `packages/harness/deerflow/subagents/executor.py` (Lines 68-80)

### 3.1 三层池架构

```python
# 全局背景任务注册表
_background_tasks: dict[str, SubagentResult] = {}
_background_tasks_lock = threading.Lock()

# 调度池: 任务调度和编排
_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")

# 执行池: 实际子代理执行 (支持超时)
_execution_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-exec-")

# 隔离循环池: 同步调用时防止事件循环冲突
_isolated_loop_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-isolated-")
```

### 3.2 并发限制

**MAX_CONCURRENT_SUBAGENTS = 3** (Line 532)

通过 `SubagentLimitMiddleware` 在 `after_model` 阶段强制执行：

```python
# agent.py lines 262-266
if subagent_enabled:
    middlewares.append(SubagentLimitMiddleware(max_concurrent=max_concurrent_subagents))
```

**限制范围**: `[2, 4]` (clamped, Lines 14-21)

---

## 四、SSE 事件流

### 4.1 事件类型

**文件**: `packages/harness/deerflow/tools/builtins/task_tool.py` (Lines 137-209)

| 事件 | 触发时机 | 字段 |
|------|----------|------|
| `task_started` | 任务启动 | `task_id`, `description` |
| `task_running` | 每轮新消息 | `task_id`, `message`, `message_index`, `total_messages` |
| `task_completed` | 成功完成 | `task_id`, `result` |
| `task_failed` | 执行失败 | `task_id`, `error` |
| `task_cancelled` | 用户取消 | `task_id`, `error` |
| `task_timed_out` | 超时 | `task_id`, `error` |

### 4.2 事件发送

```python
from langgraph.config import get_stream_writer

async def task_tool(...):
    writer = get_stream_writer()
    
    # 启动时
    writer({"type": "task_started", "task_id": task_id, "description": description})
    
    # 轮询期间
    writer({"type": "task_running", "task_id": task_id, "message": "...", ...})
    
    # 完成时
    writer({"type": "task_completed", "task_id": task_id, "result": "..."})
```

### 4.3 父 Agent 结果接收

子代理结果通过两个通道传递给父 Agent：

1. **工具返回值**: `task_tool` 返回 `"Task Succeeded. Result: ..."` 字符串
2. **SSE 事件**: 实时流事件供 UI 显示进度

---

## 五、SubagentExecutor 核心

**文件**: `packages/harness/deerflow/subagents/executor.py`

### 5.1 execute_async()

**Lines 465-529**: 异步执行入口

```python
def execute_async(self, task: str, task_id: str | None = None) -> str:
    task_id = task_id or str(uuid.uuid4())
    
    result = SubagentResult(
        task_id=task_id,
        status=SubagentStatus.PENDING,
        cancel_event=threading.Event(),
    )
    
    with _background_tasks_lock:
        _background_tasks[task_id] = result
    
    # 提交到调度池
    _scheduler_pool.submit(self._run_task, task, result)
    
    return task_id
```

### 5.2 _run_task()

**Lines 440-462**: 调度逻辑

```python
def _run_task(self, task: str, result: SubagentResult):
    # 提交到执行池
    execution_future = _execution_pool.submit(
        self.execute, task, result
    )
    
    try:
        # 等待结果 (超时: timeout_seconds + 60s)
        exec_result = execution_future.result(timeout=self.config.timeout_seconds + 60)
        result.result = exec_result
        result.status = SubagentStatus.COMPLETED
    except TimeoutError:
        result.status = SubagentStatus.TIMED_OUT
        result.error = f"Task timed out after {self.config.timeout_seconds}s"
    except Exception as e:
        result.status = SubagentStatus.FAILED
        result.error = str(e)
```

### 5.3 execute()

**Lines 330-378**: 同步执行主循环

```python
def execute(self, task: str, result: SubagentResult | None = None) -> str:
    if asyncio.get_event_loop().is_running():
        return self._execute_in_isolated_loop(task, result)
    
    return asyncio.run(self._aexecute(task, result))

def _execute_in_isolated_loop(self, task, result):
    """在隔离的事件循环中执行 (防止冲突)"""
    loop = asyncio.new_event_loop()
    try:
        asyncio.set_event_loop(loop)
        return loop.run_until_complete(self._aexecute(task, result))
    finally:
        loop.close()
        asyncio.set_event_loop(previous_loop)
```

### 5.4 _aexecute()

**Lines 200-327**: 异步执行主循环

```python
async def _aexecute(self, task: str, result: SubagentResult | None = None):
    # 1. 创建子代理
    agent = self._create_agent()
    
    # 2. 构造初始状态
    state = {
        "messages": [HumanMessage(content=task)],
        "sandbox": self.sandbox_state,
        "thread_data": self.thread_data,
    }
    
    # 3. 流式执行
    async for chunk in agent.astream(state, config=run_config, ...):
        # 检查取消信号
        if result.cancel_event.is_set():
            result.status = SubagentStatus.CANCELLED
            return "Task cancelled by parent"
        
        # 累积最终状态
        final_state = chunk
    
    # 4. 提取结果
    return extracted_result
```

---

## 六、SubagentLimitMiddleware

**文件**: `packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py`

### 6.1 截断逻辑

**Lines 40-67**: `after_model` 阶段截断超额调用

```python
def after_model(self, state: AgentState, runtime: Runtime) -> dict | None:
    messages = state.get("messages", [])
    last_msg = messages[-1]
    tool_calls = getattr(last_msg, "tool_calls", None)
    
    if not tool_calls:
        return None
    
    # 找出 task 工具调用索引
    task_indices = [
        i for i, tc in enumerate(tool_calls)
        if tc.get("name") == "task"
    ]
    
    if len(task_indices) <= self.max_concurrent:
        return None  # 无需截断
    
    # 保留前 max_concurrent 个，丢弃其余
    excess_count = len(task_indices) - self.max_concurrent
    indices_to_drop = set(task_indices[self.max_concurrent:])
    
    truncated_tool_calls = [
        tc for i, tc in enumerate(tool_calls)
        if i not in indices_to_drop
    ]
    
    logger.warning(f"Truncated {excess_count} excess task tool call(s)...")
    
    updated_msg = last_msg.model_copy(update={"tool_calls": truncated_tool_calls})
    return {"messages": [updated_msg]}
```

### 6.2 效果

- 父 Agent 的 `task` 调用被限制为最多 3 个并行
- 超出限制的调用从 AI 消息中**完全移除**
- 模型收到的是修改后的请求，不会知道被截断

---

## 七、Cooperative Cancellation

**Lines 535-550**: 支持用户取消子代理

```python
def request_cancel_background_task(task_id: str) -> None:
    with _background_tasks_lock:
        result = _background_tasks.get(task_id)
        if result is not None:
            result.cancel_event.set()  # 设置取消信号

# 在 _aexecute 中检查:
if result.cancel_event.is_set():
    logger.info(f"Subagent {self.config.name} cancelled by parent")
    result.status = SubagentStatus.CANCELLED
    return "Task cancelled by parent"
```

---

## 八、内置子代理

### 8.1 配置

**文件**: `packages/harness/deerflow/subagents/config.py`

```python
@dataclass
class SubagentConfig:
    name: str
    description: str
    system_prompt: str
    tools: list[str] | None = None  # None = 继承父工具
    disallowed_tools: list[str] = field(default_factory=lambda: ["task"])
    model: str = "inherit"
    max_turns: int = 50
    timeout_seconds: int = 900  # 15 分钟
```

### 8.2 General-Purpose Agent

```python
{
    "name": "general-purpose",
    "description": "General purpose agent for complex tasks",
    "tools": None,  # 继承所有父工具
    "disallowed_tools": ["task", "ask_clarification", "present_files"],
    "max_turns": 100,
}
```

### 8.3 Bash Agent

```python
{
    "name": "bash",
    "description": "Bash command specialist",
    "tools": ["bash", "ls", "read_file", "write_file", "str_replace"],
    "disallowed_tools": ["task", "ask_clarification", "present_files"],
    "max_turns": 60,
}
```

---

## 九、轮询机制

**task_tool.py Lines 142-210**:

```python
# 轮询间隔: 5 秒
POLL_INTERVAL = 5

# 最大轮询次数: (timeout + 60) / 5
max_polls = (config.timeout_seconds + 60) // POLL_INTERVAL

for poll_count in range(max_polls):
    result = get_background_task_result(task_id)
    
    if result.status == SubagentStatus.RUNNING:
        if result.message_count > last_message_count:
            # 发送 task_running 事件
            writer({"type": "task_running", ...})
            last_message_count = result.message_count
    
    elif result.status != SubagentStatus.PENDING:
        break
    
    await asyncio.sleep(POLL_INTERVAL)
```

---

## 十、设计模式总结

### 10.1 生产者-消费者

```
_scheduler_pool (生产者)
    └── 提交任务到 _execution_pool (消费者)
    
执行结果放入 _background_tasks
task_tool 轮询消费
```

### 10.2 隔离事件循环

```python
# 防止与共享 async primitives (httpx) 冲突
if asyncio.get_event_loop().is_running():
    return self._execute_in_isolated_loop(task, result)
```

### 10.3 Middleware 截断

```
after_model hook:
    截断 excess task calls
    ↓
模型看到的是修改后的响应
    ↓
实际执行被限制在 max_concurrent
```

### 10.4 SSE 实时流

| 通道 | 内容 | 用途 |
|------|------|------|
| 工具返回值 | `"Task Succeeded. Result: ..."` | 父 Agent 获取结果 |
| SSE CUSTOM 事件 | `task_started/running/completed` | UI 实时进度 |

---

## 十一、关键文件索引

| 组件 | 文件 | 关键行号 |
|------|------|---------|
| Task 工具 | `tools/builtins/task_tool.py` | 23-248 |
| 执行器 | `subagents/executor.py` | 128-529 |
| 限制中间件 | `middlewares/subagent_limit_middleware.py` | 40-67 |
| 子代理配置 | `subagents/config.py` | - |
| 注册表 | `subagents/registry.py` | 13-99 |
| 内置代理 | `subagents/builtins/__init__.py` | - |
