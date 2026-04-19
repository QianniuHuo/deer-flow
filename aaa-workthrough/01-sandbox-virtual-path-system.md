# DeerFlow 虚拟路径系统与沙箱隔离设计

## 概述

DeerFlow 通过虚拟路径系统实现 Agent 与宿主机文件系统的安全隔离。Agent 在对话中只能通过虚拟路径访问受限的工作目录，无法直接读写宿主机上的任意文件。这一设计既保证了安全性，又实现了线程级别的资源隔离。

---

## 一、核心架构：Provider 模式

### 1.1 抽象接口

**文件**: `packages/harness/deerflow/sandbox/sandbox.py` (Lines 1-93)

```python
class Sandbox(metaclass=ABCMeta):
    @abstractmethod
    def execute_command(self, command: str, cwd: str | None = None) -> CommandResult: ...
    
    @abstractmethod
    def read_file(self, path: str, offset: int = 0, limit: int | None = None) -> str: ...
    
    @abstractmethod
    def list_dir(self, path: str, max_depth: int = 2) -> list[str]: ...
    
    @abstractmethod
    def write_file(self, path: str, content: str, append: bool = False) -> None: ...
    
    @abstractmethod
    def glob(self, pattern: str, path: str = ".") -> list[str]: ...
    
    @abstractmethod
    def grep(self, pattern: str, path: str = ".", is_regex: bool = True) -> list[str]: ...
    
    @abstractmethod
    def update_file(self, virtual_path: str, content: bytes) -> None: ...
```

### 1.2 Provider 模式

**文件**: `packages/harness/deerflow/sandbox/sandbox_provider.py` (Lines 8-98)

```python
class SandboxProvider(metaclass=ABCMeta):
    @abstractmethod
    def acquire(self, thread_id: str) -> str:  # 返回 sandbox_id
        """获取沙箱实例，返回沙箱 ID"""
    
    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox:
        """根据 ID 获取沙箱实例"""
    
    @abstractmethod
    def release(self, sandbox_id: str) -> None:
        """释放沙箱实例"""
```

**单例管理**:
```python
def get_sandbox_provider() -> SandboxProvider:
    global _default_sandbox_provider
    if _default_sandbox_provider is None:
        config = get_app_config()
        cls = resolve_class(config.sandbox.use, SandboxProvider)
        _default_sandbox_provider = cls(**kwargs)
    return _default_sandbox_provider
```

### 1.3 实现变体

| Provider | 特点 | `uses_thread_data_mounts` |
|----------|------|---------------------------|
| `LocalSandboxProvider` | 单例，本地文件系统 | `True` |
| `AioSandboxProvider` | 容器化隔离，Docker 模式 | `False` |

---

## 二、路径映射系统

### 2.1 虚拟路径前缀

**文件**: `packages/harness/deerflow/config/paths.py` (Line 7)

```python
VIRTUAL_PATH_PREFIX = "/mnt/user-data"
```

### 2.2 三层路径映射

**文件**: `packages/harness/deerflow/sandbox/local/local_sandbox_provider.py` (Lines 20-100)

| 虚拟路径 | 宿主机路径 (Local) | 容器内路径 (AIO) |
|----------|-------------------|-----------------|
| `/mnt/user-data/workspace` | `{base}/threads/{id}/user-data/workspace` | 已挂载 |
| `/mnt/user-data/uploads` | `{base}/threads/{id}/user-data/uploads` | 已挂载 |
| `/mnt/user-data/outputs` | `{base}/threads/{id}/user-data/outputs` | 已挂载 |
| `/mnt/skills` | `{skills_host_path}` | 已挂载 |
| `/mnt/acp-workspace` | `{base}/threads/{id}/acp-workspace` | 已挂载 |

**预留前缀保护** (Line 51):
```python
RESERVED_PREFIXES = [
    "/mnt/skills",
    "/mnt/acp-workspace", 
    "/mnt/user-data"
]
```

### 2.3 路径转换函数

**replace_virtual_path()** - `tools.py` (Lines 396-429)

```python
def replace_virtual_path(path: str, thread_data: ThreadDataState) -> str:
    """
    将虚拟路径转换为实际路径
    
    映射规则:
      /mnt/user-data/workspace/* → thread_data['workspace_path']/*
      /mnt/user-data/uploads/*  → thread_data['uploads_path']/*
      /mnt/user-data/outputs/*   → thread_data['outputs_path']/*
    """
    mappings = _thread_virtual_to_actual_mappings(thread_data)
    
    # 最长前缀匹配
    for virtual_base, actual_base in sorted(mappings.items(), 
                                            key=lambda item: len(item[0]), 
                                            reverse=True):
        if path == virtual_base or path.startswith(virtual_base + "/"):
            relative = path[len(virtual_base):]
            return actual_base + relative
    
    return path
```

**replace_virtual_paths_in_command()** - `tools.py` (Lines 699-744)

```python
def replace_virtual_paths_in_command(command: str, ...) -> str:
    """替换命令中所有虚拟路径为实际路径"""
    # 1. /mnt/skills/* → skills 目录
    # 2. /mnt/acp-workspace/* → per-thread ACP workspace
    # 3. 自定义挂载路径
    # 4. /mnt/user-data/* → thread workspace/uploads/outputs
```

### 2.4 反向转换 (输出掩码)

**mask_local_paths_in_output()** - `tools.py` (Lines 462-533)

将实际路径转换回虚拟路径，防止 Agent 看到真实路径信息：

```python
def mask_local_paths_in_output(output: str, thread_data: ThreadDataState) -> str:
    """将输出中的实际路径替换为虚拟路径"""
    # 反向映射，保持对称性
```

---

## 三、目录结构

### 3.1 线程目录布局

**文件**: `packages/harness/deerflow/config/paths.py` (Lines 137-172)

```
{base_dir}/
└── threads/
    └── {thread_id}/
        ├── user-data/           ← 挂载为 /mnt/user-data/
        │   ├── workspace/       ← /mnt/user-data/workspace/
        │   ├── uploads/        ← /mnt/user-data/uploads/
        │   └── outputs/        ← /mnt/user-data/outputs/
        └── acp-workspace/       ← 挂载为 /mnt/acp-workspace/
```

### 3.2 SandboxMiddleware 集成

**文件**: `packages/harness/deerflow/sandbox/middleware.py` (Lines 21-83)

```python
class SandboxMiddleware(AgentMiddleware[SandboxMiddlewareState]):
    def __init__(self, lazy_init: bool = True):
        self._lazy_init = lazy_init  # 默认延迟获取
    
    def before_agent(self, state, runtime) -> dict | None:
        if self._lazy_init:
            return super().before_agent(...)  # 跳过，延迟到工具调用
        # 立即获取沙箱
        sandbox_id = self._acquire_sandbox(thread_id)
        return {"sandbox": {"sandbox_id": sandbox_id}}
    
    def after_agent(self, state, runtime) -> dict | None:
        # 释放沙箱
        get_sandbox_provider().release(sandbox_id)
```

**懒初始化时机**: 沙箱获取延迟到首次工具调用，目录创建也在此时触发。

---

## 四、安全机制

### 4.1 路径遍历防护

**文件**: `packages/harness/deerflow/sandbox/tools.py` (Lines 536-542)

```python
def _reject_path_traversal(path: str) -> None:
    """拒绝包含 '..' 段的路径"""
    normalised = path.replace("\\", "/")
    for segment in normalised.split("/"):
        if segment == "..":
            raise PermissionError("Access denied: path traversal detected")
```

### 4.2 工具路径验证

**validate_local_tool_path()** - `tools.py` (Lines 545-596)

```python
def validate_local_tool_path(path: str) -> None:
    """验证路径在允许的家族中"""
    # 允许的路径:
    # - /mnt/user-data/* (完全访问)
    # - /mnt/skills/* (只读)
    # - /mnt/acp-workspace/* (只读)
    # - 自定义挂载路径 (遵循 read_only 标志)
```

### 4.3 Bash 命令验证

**validate_local_bash_command_paths()** - `tools.py` (Lines 638-696)

```python
def validate_local_bash_command_paths(command: str) -> None:
    """验证 Bash 命令中的路径"""
    # 阻止 file:// URL
    # 验证 /mnt/user-data 路径
    # 验证 /mnt/skills 路径
    # 验证 /mnt/acp-workspace 路径
    # 允许系统路径: /bin/, /usr/bin/, /usr/sbin/, /sbin/, ...
```

### 4.4 本地沙箱安全声明

**文件**: `packages/harness/deerflow/sandbox/security.py` (Lines 1-45)

```python
LOCAL_HOST_BASH_DISABLED_MESSAGE = """
LocalSandboxProvider is NOT a secure sandbox - it is intended for 
local development only. Do not use in production with untrusted code.
"""

def is_host_bash_allowed() -> bool:
    """仅在显式配置或非 LocalSandbox 时允许"""
    return not is_local_sandbox() or config.sandbox.allow_host_bash
```

---

## 五、工具实现

**文件**: `packages/harness/deerflow/sandbox/tools.py`

| 工具 | 行号 | 关键行为 |
|------|------|---------|
| `bash_tool()` | 989-1035 | 路径验证，CWD 前缀，输出截断 |
| `ls_tool()` | 1038-1079 | 解析路径后列出，结果掩码 |
| `glob_tool()` | 1082-1129 | 路径模式匹配，结果掩码 |
| `grep_tool()` | 1132-1199 | 正则/字面匹配，结果掩码 |
| `read_file_tool()` | 1202-1254 | 行范围读取，截断 |
| `write_file_tool()` | 1257-1294 | 文件锁，写入 |
| `str_replace_tool()` | 1297-1345 | 读-改-写原子操作 |

---

## 六、设计模式总结

### 6.1 抽象工厂模式

```
SandboxProvider (抽象)
    ├── LocalSandboxProvider (单例)
    └── AioSandboxProvider (per-thread)
```

通过 `config.sandbox.use` 类路径解析实现插件架构。

### 6.2 虚拟化隔离

- Agent 只能看到 `/mnt/user-data/*` 系列虚拟路径
- 所有操作都在线程隔离的目录内
- 输出中的真实路径被掩码替换

### 6.3 懒初始化

- 沙箱获取延迟到首次工具调用
- 目录创建按需触发
- 避免启动时过慢

### 6.4 双向路径映射

| 方向 | 函数 | 用途 |
|------|------|------|
| 虚拟→物理 | `replace_virtual_path()` | 实际操作文件系统 |
| 物理→虚拟 | `mask_local_paths_in_output()` | 隐藏真实路径 |

---

## 七、关键文件索引

| 组件 | 文件 | 关键行号 |
|------|------|---------|
| 抽象 Sandbox | `sandbox.py` | 6-93 |
| Provider 抽象 | `sandbox_provider.py` | 8-98 |
| 本地沙箱 | `local/local_sandbox.py` | 23-398 |
| 本地 Provider | `local/local_sandbox_provider.py` | 13-121 |
| 路径转换 | `tools.py` | 396-744 |
| 安全验证 | `tools.py` | 536-696 |
| 沙箱中间件 | `middleware.py` | 21-83 |
| 路径配置 | `config/paths.py` | 1-306 |
| 线程状态 | `agents/thread_state.py` | 10-55 |
