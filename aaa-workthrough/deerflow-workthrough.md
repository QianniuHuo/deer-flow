# DeerFlow 技术设计与实现路线分析

## 概述

DeerFlow 是一个基于 LangGraph 的 AI Super Agent 系统，采用全栈架构。核心运行机制围绕 LangGraph 的 `create_agent` 构建，通过 middleware 链实现复杂的状态管理和功能扩展。

---

## 一、核心运行机制：Main Loop 与 Middleware 链

### 1.1 LangGraph Server 启动流程

**配置文件**: `backend/langgraph.json`

```json
{
  "graphs": {
    "lead_agent": "deerflow.agents:make_lead_agent"
  },
  "checkpointer": {
    "path": "./packages/harness/deerflow/agents/checkpointer/async_provider.py:make_checkpointer"
  }
}
```

**启动命令**: `langgraph dev --no-browser --no-reload --n-jobs-per-worker 10`

### 1.2 Lead Agent 创建入口

**文件**: `packages/harness/deerflow/agents/lead_agent/agent.py`

```python
def make_lead_agent(config: RunnableConfig):
    model_name = _resolve_model_name(...)
    return create_agent(
        model=create_chat_model(name=model_name, thinking_enabled=thinking_enabled),
        tools=get_available_tools(...),
        middleware=_build_middlewares(config, model_name=model_name),
        system_prompt=apply_prompt_template(...),
        state_schema=ThreadState,
    )
```

### 1.3 Middleware 链构建

**文件**: `packages/harness/deerflow/agents/lead_agent/agent.py` (lines 205-277)

Middleware 按固定顺序组装，**顺序至关重要**：

```
before_agent hooks (按顺序执行):
1.  ThreadDataMiddleware      → 创建线程目录结构
2.  UploadsMiddleware         → 注入上传文件信息
3.  SandboxMiddleware         → 获取沙箱环境
4.  DanglingToolCallMiddleware → 修补缺失的 ToolMessage
5.  LLMErrorHandlingMiddleware → 规范化模型调用错误
6.  GuardrailMiddleware       → 工具调用授权 (可选)
7.  SandboxAuditMiddleware    → 沙箱操作安全审计
8.  ToolErrorHandlingMiddleware → 工具异常转换为 ToolMessage
9.  SummarizationMiddleware   → 上下文压缩 (可选)
10. TodoListMiddleware        → 任务跟踪 (plan mode 可选)
11. TokenUsageMiddleware      → Token 使用记录 (可选)
12. TitleMiddleware           → 生成会话标题
13. MemoryMiddleware          → 异步记忆更新
14. ViewImageMiddleware       → 图片注入 (vision 模型可选)
15. DeferredToolFilterMiddleware → 隐藏延迟工具 (tool_search)
16. SubagentLimitMiddleware   → 并发子代理限制
17. LoopDetectionMiddleware   → 循环检测与打断
18. ClarificationMiddleware   → 澄清请求拦截 (始终最后)
```

### 1.4 关键 Middleware 详解

#### 1.4.1 ThreadDataMiddleware

**文件**: `packages/harness/deerflow/agents/middlewares/thread_data_middleware.py`

- `before_agent`: 设置线程目录路径
- 状态更新: `{"thread_data": {"workspace_path", "uploads_path", "outputs_path"}}`

**目录结构**:
```
{base_dir}/threads/{thread_id}/user-data/
├── workspace/   # 沙箱工作目录
├── uploads/     # 上传文件
└── outputs/     # 生成输出
```

#### 1.4.2 SandboxMiddleware

**文件**: `packages/harness/deerflow/sandbox/middleware.py`

- `before_agent`: 获取沙箱 (`lazy_init=True` 时延迟到首次工具调用)
- `after_agent`: 释放沙箱
- 状态更新: `{"sandbox": {"sandbox_id": "..."}}`

#### 1.4.3 ClarificationMiddleware (中断机制)

**文件**: `packages/harness/deerflow/agents/middlewares/clarification_middleware.py`

拦截 `ask_clarification` 工具调用，通过 `Command(goto=END)` 中断执行流：

```python
def _handle_clarification(self, request: ToolCallRequest) -> Command:
    # ... 格式化澄清问题 ...
    return Command(
        update={"messages": [tool_message]},
        goto=END,  # 跳到 END 节点，终止本轮执行
    )
```

#### 1.4.4 LoopDetectionMiddleware

**文件**: `packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py`

两层检测:
1. **Hash-based**: 相同工具调用集合达到 `hard_limit=5` 次时硬停止
2. **Per-tool-type**: 同一工具类型达到 `tool_freq_hard_limit=50` 次时硬停止

硬停止时清除 `tool_calls`，强制文本回答。

### 1.5 Middleware 钩子类型

| 钩子 | 时机 | 用途 |
|------|------|------|
| `before_agent` | Agent 执行前 | 状态预处理、目录创建、文件注入 |
| `before_model` | 模型调用前 | 上下文补充（如图片、任务提醒） |
| `after_model` | 模型调用后 | 响应后处理（如标题生成、Token 统计） |
| `after_agent` | Agent 完成后 | 清理、记忆更新、沙箱释放 |
| `wrap_tool_call` | 工具执行时 | 错误处理、安全审计、循环检测 |
| `wrap_model_call` | 模型调用时 | 工具过滤、错误规范化 |

---

## 二、统一的用户资料摄取管线：文件上传与注入

### 2.1 上传网关 API

**文件**: `app/gateway/routers/uploads.py`

**端点**: `POST /api/threads/{thread_id}/uploads`

**处理流程**:
```
1. validate_thread_id()         → 验证线程 ID
2. ensure_uploads_dir()         → 创建上传目录
3. normalize_filename()         → 规范化文件名 (防路径遍历)
4. file.read() + write_bytes()  → 存储原文件
5. sandbox.update_file()        → 同步到沙箱
6. convert_file_to_markdown()   → 转换可识别格式
```

### 2.2 支持转换的文件格式

**定义**: `deerflow/utils/file_conversion.py`

```python
CONVERTIBLE_EXTENSIONS = {".pdf", ".ppt", ".pptx", ".xls", ".xlsx", ".doc", ".docx"}
```

**转换策略** (PDF 为例):
1. 优先使用 `pymupdf4llm` (更好的标题检测、更快)
2. 若输出过短 (<50 chars/page)，判定为图片型 PDF，回退到 MarkItDown
3. 直接使用 MarkItDown 作为备选

### 2.3 文件注入机制

**文件**: `packages/harness/deerflow/agents/middlewares/uploads_middleware.py`

**触发时机**: `UploadsMiddleware.before_agent()`

**注入流程**:
```
1. 从当前消息的 additional_kwargs.files 提取新文件列表
2. 扫描上传目录获取历史文件 (排除新文件)
3. 为每个文件提取文档大纲 (从转换的 .md 文件)
4. 生成 <uploaded_files> 标签内容
5. 追加到最后一个 HumanMessage 的 content 前面
```

### 2.4 上传文件标签格式

```
<uploaded_files>
The following files were uploaded in this message:
- report.pdf (2.3 MB)
  Path: /mnt/user-data/uploads/report.pdf
  Document outline (use `read_file` with line ranges to read sections):
    L45: Executive Summary
    L120: Financial Analysis

The following files were uploaded in previous messages and are still available:
- presentation.pptx (1.1 MB)
  ...

To work with these files:
- Read from the file first
- Use grep to search for keywords
- Use glob to find files by name pattern
</uploaded_files>
```

### 2.5 前端集成

**前端文件**: `frontend/src/core/threads/hooks.ts`

**上传流程**:
```typescript
// 1. 调用上传 API
const uploadedFileInfo = await uploadFiles(threadId, files);

// 2. 提交消息时携带文件元数据
additional_kwargs: { files: filesForSubmit }
```

### 2.6 完整数据流

```
前端: 用户选择文件 → uploadFiles() → API 返回 fileInfo[]
     用户发送消息 → additional_kwargs.files 携带文件元数据

Gateway:
     POST /api/threads/{thread_id}/uploads
     - 规范化文件名 → 存储原文件
     - 同步到沙箱 (sandbox.update_file)
     - 转换格式 (convert_file_to_markdown)
     - 提取大纲 (extract_outline)

UploadsMiddleware.before_agent():
     - 提取 additional_kwargs.files 中的新文件
     - 扫描上传目录获取历史文件
     - 提取文档大纲
     - 追加 <uploaded_files> 标签到 HumanMessage

Lead Agent:
     模型接收到的 HumanMessage 已包含 <uploaded_files> 标签
     模型知道有哪些文件可用，通过 read_file/grep/glob 工具访问
```

---

## 三、状态相关实现

### 3.1 ThreadState 定义

**文件**: `packages/harness/deerflow/agents/thread_state.py`

```python
class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]  # 带去重的合并
    todos: NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

### 3.2 自定义 Reducer

#### merge_artifacts

```python
def merge_artifacts(existing, new):
    # 合并 + 去重 + 保持顺序
    return list(dict.fromkeys(existing + new))
```

#### merge_viewed_images

```python
def merge_viewed_images(existing, new):
    # 空字典 {} 特殊处理：清除所有图片
    if len(new) == 0:
        return {}
    # 新值覆盖旧值
    return {**existing, **new}
```

### 3.3 状态流转图

```
LangGraph Checkpointer (持久化 ThreadState，支持跨轮次恢复)
                              │
                              ▼
Middleware Chain:
  before_agent:
    ThreadDataMiddleware    → 设置 thread_data
    UploadsMiddleware      → 设置 uploaded_files + 消息注入
    SandboxMiddleware      → 获取 sandbox
  before_model:
    TodoMiddleware         → 注入 todo 提醒 (若上下文丢失)
    ViewImageMiddleware   → 注入图片详情
  after_agent:
    MemoryMiddleware      → 记忆队列更新
    SandboxMiddleware    → 释放 sandbox
                              │
                              ▼
状态Reducer 合并:
  artifacts      → merge_artifacts (去重 + 保持顺序)
  viewed_images  → merge_viewed_images (覆盖 + 空清除)
  todos          → LangChain TodoListMiddleware 管理
  uploaded_files → 直接替换 (非合并)
```

### 3.4 Checkpointer 实现

**文件**: `packages/harness/deerflow/agents/checkpointer/`

支持的后端:
- `memory`: 内存存储 (默认)
- `sqlite`: SQLite 持久化
- `postgres`: PostgreSQL 持久化

---

## 四、关键技术难点总结

### 4.1 Middleware 链顺序敏感性

| 位置 | 设计原因 |
|------|---------|
| ThreadData → Uploads → Sandbox | 基础设施优先 |
| DanglingToolCall 在 Model 前 | 修补消息历史 |
| ToolErrorHandling 在 Clarification 前 | 确保工具错误被正确处理 |
| Clarification 最后 | 拦截所有澄清请求 |

### 4.2 懒初始化 (Lazy Init)

大部分 Middleware 支持 `lazy_init=True` (默认):
- 目录创建延迟到首次访问
- 沙箱获取延迟到首次工具调用
- 避免启动时过慢

### 4.3 状态隔离

- **线程隔离**: 每个线程独立的目录结构和状态
- **沙箱隔离**: `sandbox_id` 标记不同执行环境
- **文件隔离**: 上传文件存储在 `{thread_id}/user-data/uploads/`

### 4.4 文件注入的上下文管理

`<uploaded_files>` 标签解决了:
1. 模型不知道有哪些文件可用
2. 模型不知道文件结构和内容位置
3. 历史文件和新上传文件的区分

---

## 五、架构设计亮点

### 5.1 Harness/App 分层

```
packages/harness/deerflow/   → 可发布的 Agent 框架包
app/                         → 应用层 (FastAPI Gateway + Channels)

规则: App 可以 import Harness，Harness 禁止 import App
```

### 5.2 工具系统扩展性

```
get_available_tools()
├── Config 定义的工具
├── MCP 服务器工具 (懒加载 + mtime 缓存)
├── 内置工具 (present_files, ask_clarification, view_image)
├── 子代理工具 (task)
└── 社区工具 (tavily, jina_ai, firecrawl, image_search)
```

### 5.3 虚拟路径系统

沙箱内外路径映射:
- 沙箱内: `/mnt/user-data/{workspace,uploads,outputs}`
- 宿主机: `backend/.deer-flow/threads/{thread_id}/user-data/...`

---

*文档生成时间: 2026-04-19*
