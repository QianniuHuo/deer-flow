# DeerFlow 异步记忆系统设计

## 概述

DeerFlow 的记忆系统采用"事实提取+置信度过滤"的异步设计，不直接存储对话历史，而是通过 LLM 动态提取结构化事实。记忆更新在后台静默执行，不阻塞主对话流程，解决了长对话中背景信息丢失的核心问题。

---

## 一、核心架构

### 1.1 组件关系

```
MemoryMiddleware
    │  (after_agent hook)
    ▼
MemoryUpdateQueue (30s 防抖)
    │  (定时触发)
    ▼
MemoryUpdater (LLM 提取)
    │
    ▼
FileMemoryStorage (memory.json)
```

### 1.2 目录结构

**文件**: `packages/harness/deerflow/agents/memory/`

```
memory/
├── __init__.py
├── message_processing.py   # 消息过滤
├── prompt.py              # LLM prompt 模板
├── queue.py               # 防抖队列
├── storage.py             # memory.json 读写
├── summarization_hook.py  # 压缩前 flush hook
└── updater.py            # LLM 更新逻辑
```

---

## 二、数据结构

### 2.1 memory.json Schema

**文件**: `packages/harness/deerflow/agents/memory/storage.py` (Lines 24-40)

```json
{
  "version": "1.0",
  "lastUpdated": "2026-01-01T00:00:00Z",
  "user": {
    "workContext": {"summary": "", "updatedAt": ""},
    "personalContext": {"summary": "", "updatedAt": ""},
    "topOfMind": {"summary": "", "updatedAt": ""}
  },
  "history": {
    "recentMonths": {"summary": "", "updatedAt": ""},
    "earlierContext": {"summary": "", "updatedAt": ""},
    "longTermBackground": {"summary": "", "updatedAt": ""}
  },
  "facts": [
    {
      "id": "fact_abc12345",
      "content": "User prefers Python over Java",
      "category": "preference",
      "confidence": 0.85,
      "createdAt": "2026-01-01T00:00:00Z",
      "source": "thread_123"
    }
  ]
}
```

### 2.2 Fact 分类与置信度

**文件**: `packages/harness/deerflow/agents/memory/prompt.py` (Lines 70-80)

| Category | 描述 | 典型置信度 |
|----------|------|-----------|
| `preference` | 工具、风格偏好 | 0.9-1.0 |
| `knowledge` | 专业知识、技术栈 | 0.9-1.0 |
| `context` | 背景（工作、项目） | 0.9-1.0 |
| `behavior` | 工作模式 | 0.7-0.8 |
| `goal` | 目标、指标 | 0.7-0.8 |
| `correction` | 明确错误（带 sourceError） | >= 0.95 |

### 2.3 用户上下文结构

**文件**: `packages/harness/deerflow/agents/memory/prompt.py` (Lines 43-65)

| Section | 内容 | 长度 |
|---------|------|------|
| `workContext` | 专业角色、公司、项目、技术栈 | 2-3 句 |
| `personalContext` | 语言偏好、兴趣爱好 | 1-2 句 |
| `topOfMind` | 多个进行中的优先级 | 3-5 句（详细） |
| `recentMonths` | 最近 1-3 个月 | 4-6 句 |
| `earlierContext` | 3-12 个月前 | 3-5 句 |
| `longTermBackground` | 基础信息 | 2-4 句 |

---

## 三、MemoryMiddleware

**文件**: `packages/harness/deerflow/agents/middlewares/memory_middleware.py`

### 3.1 after_agent 钩子

```python
def after_agent(self, state: AgentState, runtime: Runtime) -> dict | None:
    # 1. 提取 thread_id
    thread_id = runtime.context.get("thread_id")
    
    # 2. 获取消息列表
    messages = state.get("messages", [])
    
    # 3. 过滤消息 (移除工具调用、strip 上传标签)
    filtered = filter_messages_for_memory(messages)
    
    # 4. 检测信号
    correction = detect_correction(messages)
    reinforcement = detect_reinforcement(messages)
    
    # 5. 加入更新队列 (30s 防抖)
    queue.add(thread_id, filtered, agent_name, correction, reinforcement)
```

### 3.2 消息过滤

**文件**: `packages/harness/deerflow/agents/memory/message_processing.py` (Lines 56-85)

```python
def filter_messages_for_memory(messages) -> list[Message]:
    """只保留用户输入和最终 AI 响应"""
    # 移除 tool_calls 消息
    # 移除 <uploaded_files> 标签
    # 跳过紧跟纯上传消息的 AI 响应
```

### 3.3 信号检测

**纠正检测** (Lines 88-97):
```python
CORRECTION_PATTERNS = [
    "that's wrong", "you misunderstood", "try again", "redo",
    "不对", "你理解错了", ...  # 中英双语
]
```

**强化检测** (Lines 100-109):
```python
REINFORCEMENT_PATTERNS = [
    "yes exactly", "perfect", "that's right", "keep doing that",
    "对", "就是这样", "完全正确", ...
]
```

---

## 四、防抖队列

**文件**: `packages/harness/deerflow/agents/memory/queue.py`

### 4.1 ConversationContext

```python
@dataclass
class ConversationContext:
    thread_id: str
    messages: list[Any]
    timestamp: datetime
    agent_name: str | None = None
    correction_detected: bool = False
    reinforcement_detected: bool = False
```

### 4.2 Per-Thread 去重

**Lines 109-124**: 当同一 thread_id 新消息到达时，**替换**旧上下文而非追加：

```python
def _enqueue_locked(self, context: ConversationContext):
    # 移除旧的同 thread_id 条目
    self._queue = [c for c in self._queue if c.thread_id != thread_id]
    # 添加新条目
    self._queue.append(context)
    # correction/reinforcement 标志 OR 合并
```

### 4.3 防抖机制

**Lines 126-144**: 30 秒无新消息后触发处理：

```python
def _schedule_timer(self):
    self._timer = threading.Timer(
        config.debounce_seconds,  # 默认 30s
        self._process_queue
    )
    self._timer.start()

def add(self, thread_id, messages, ...):
    # 重置防抖计时器
    self._reset_timer()
```

### 4.4 队列处理

**Lines 146-193**:

```python
def _process_queue(self):
    with self._lock:
        if self._processing:
            # 已在处理中，立即触发下一次
            self._schedule_timer(0)
            return
        
        self._processing = True
        queue_snapshot = self._queue.copy()
        self._queue.clear()
    
    for context in queue_snapshot:
        MemoryUpdater.update_memory(context)
        time.sleep(0.5)  # 避免限流
```

---

## 五、MemoryUpdater

**文件**: `packages/harness/deerflow/agents/memory/updater.py`

### 5.1 更新流程

```python
def update_memory(self, context: ConversationContext):
    # 1. 加载当前 memory
    current = FileMemoryStorage.load(agent_name)
    
    # 2. 构造 prompt
    prompt = self._prepare_update_prompt(current, context)
    
    # 3. 调用 LLM
    response = self._call_llm(prompt)
    
    # 4. 解析 JSON
    updates = json.loads(response)
    
    # 5. 应用更新
    self._apply_updates(current, updates, context)
    
    # 6. 原子写入
    FileMemoryStorage.save(current, agent_name)
```

### 5.2 Prompt 模板

**文件**: `packages/harness/deerflow/agents/memory/prompt.py` (Lines 15-131)

**MEMORY_UPDATE_PROMPT** 结构:
1. **当前 Memory 状态** - 作为上下文参考
2. **新对话** - 需要分析的新消息
3. **LLM 输出格式** - JSON with:
   - `userContext.*` - 更新的摘要
   - `history.*` - 更新的历史
   - `newFacts` - 新提取的事实
   - `factsToRemove` - 需要删除的事实 ID

### 5.3 去重机制

**文件**: `packages/harness/deerflow/agents/memory/updater.py` (Lines 290-296)

```python
def _fact_content_key(content: Any) -> str | None:
    """空白符归一化 + 大小写不敏感"""
    stripped = content.strip()
    return stripped.casefold()

# 在 _apply_updates 中使用:
existing_keys = {_fact_content_key(f["content"]) for f in current_facts}
if new_fact_key not in existing_keys:
    facts.append(new_fact)
```

### 5.4 置信度过滤

**Lines 501-530**: 仅存储 `confidence >= config.fact_confidence_threshold` (默认 0.7) 的事实：

```python
if fact["confidence"] >= config.fact_confidence_threshold:
    facts.append(fact)
```

### 5.5 最大数量限制

**Lines 532-539**: 超过 `max_facts` (默认 100) 时移除最低置信度：

```python
if len(facts) > config.max_facts:
    facts.sort(key=lambda f: f["confidence"], reverse=True)
    facts = facts[:config.max_facts]
```

---

## 六、存储实现

**文件**: `packages/harness/deerflow/agents/memory/storage.py`

### 6.1 原子写入

```python
def save(self, data: dict, agent_name: str | None = None):
    file_path = self._get_memory_file_path(agent_name)
    temp_path = file_path.with_suffix(f".{uuid.uuid4().hex}.tmp")
    
    with open(temp_path, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)
    
    temp_path.replace(file_path)  # POSIX 原子替换
```

### 6.2 缓存失效

```python
def load(self, agent_name: str | None = None) -> dict:
    file_path = self._get_memory_file_path(agent_name)
    current_mtime = file_path.stat().st_mtime
    
    if agent_name in self._memory_cache:
        cached_data, cached_mtime = self._memory_cache[agent_name]
        if current_mtime == cached_mtime:
            return cached_data
    
    # 重新加载并缓存
    data = json.loads(file_path.read_text())
    self._memory_cache[agent_name] = (data, current_mtime)
    return data
```

---

## 七、Prompt 注入

**文件**: `packages/harness/deerflow/agents/memory/prompt.py` (Lines 201-317)

### 7.1 format_memory_for_injection()

```python
def format_memory_for_injection(memory: dict, max_tokens: int = 2000) -> str:
    """
    将 memory.json 格式化为注入 prompt 的字符串
    使用 tiktoken 精确计算 token 数
    """
    sections = []
    
    # 1. User Context (workContext, personalContext, topOfMind)
    # 2. History (recentMonths, earlierContext, longTermBackground)
    # 3. Facts (按置信度排序)
    
    # token 限制内尽可能多注入
    # correction 类事实特殊格式: "[correction | 0.95] content (avoid: sourceError)"
```

### 7.2 注入位置

**文件**: `packages/harness/deerflow/agents/lead_agent/prompt.py` (Lines 533-536)

```python
SYSTEM_PROMPT_TEMPLATE = """
...
<memory>
{memory_context}
</memory>
...
"""
```

---

## 八、配置参数

**文件**: `packages/harness/deerflow/config/memory_config.py`

| 参数 | 默认值 | 范围 | 描述 |
|------|--------|------|------|
| `enabled` | True | - | 启用记忆系统 |
| `storage_path` | "" | - | memory.json 路径 |
| `debounce_seconds` | 30 | 1-300 | 防抖延迟 |
| `model_name` | None | - | 记忆更新模型（默认同主模型） |
| `max_facts` | 100 | 10-500 | 最大事实数 |
| `fact_confidence_threshold` | 0.7 | 0.0-1.0 | 最小置信度 |
| `injection_enabled` | True | - | 启用记忆注入 |
| `max_injection_tokens` | 2000 | 100-8000 | 注入 token 预算 |

---

## 九、设计模式总结

### 9.1 异步非阻塞

```
主对话流程          记忆更新流程
    │                    │
    ▼                    │
MemoryMiddleware        │
    │                    ▼
    ▼                MemoryUpdateQueue (后台)
queue.add()             │
    │                   ▼
    ▼                MemoryUpdater (LLM)
返回                    │
    │                   ▼
继续对话            memory.json
```

### 9.2 事实型记忆 vs 对话存储

| 对话存储 | DeerFlow 记忆 |
|----------|--------------|
| 原始消息列表 | LLM 提取的事实 |
| 无结构 | 分类 + 置信度 |
| 重复信息多 | 空白符归一化去重 |
| Token 消耗高 | 固定 token 预算 |

### 9.3 信号感知

- **Correction**: 用户纠正 → 更高置信度的 correction 类事实
- **Reinforcement**: 用户确认 → 更高置信度的 preference/behavior 类事实

### 9.4 原子性保证

- 临时文件 + POSIX `replace()` 保证写入原子性
- mtime 缓存失效保证多实例一致性

---

## 十、关键文件索引

| 组件 | 文件 | 关键行号 |
|------|------|---------|
| 中间件 | `middlewares/memory_middleware.py` | 45-97 |
| 消息过滤 | `memory/message_processing.py` | 56-109 |
| Prompt 模板 | `memory/prompt.py` | 15-363 |
| 防抖队列 | `memory/queue.py` | 27-236 |
| LLM 更新器 | `memory/updater.py` | 299-541 |
| 存储 | `memory/storage.py` | 24-174 |
| 配置 | `config/memory_config.py` | - |
