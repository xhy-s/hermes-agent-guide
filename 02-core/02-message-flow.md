# 消息流转机制

## 2.1 概述

本文档详细分析 Hermes Agent 中消息的完整流转过程，从用户输入到 Agent 响应的各个阶段。

---

## 2.2 消息流转总览

```
┌─────────────────────────────────────────────────────────────────┐
│                      消息流转总览                                 │
└─────────────────────────────────────────────────────────────────┘

用户输入
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 输入解析 (Input Parsing)                                    │
│     - CLI: hermes_cli/main.py → cli.py                        │
│     - Gateway: gateway/run.py → platform adapter               │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 会话关联 (Session Association)                              │
│     - 获取或创建 Session                                         │
│     - 关联用户和平台信息                                         │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. AIAgent 处理 (run_agent.py)                                │
│     - 构建 system prompt                                         │
│     - 组装消息历史                                               │
│     - 调用模型                                                   │
│     - 处理工具调用                                               │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 响应返回 (Response Delivery)                                │
│     - CLI: 直接输出                                              │
│     - Gateway: 通过 platform adapter 发送                        │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
用户
```

---

## 2.3 上下文管理

### 2.3.1 多层上下文

```
┌─────────────────────────────────────────────────────────────────┐
│                      上下文层次                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: System Prompt                                         │
│  - 身份定义 (DEFAULT_AGENT_IDENTITY)                           │
│  - 平台提示 (PLATFORM_HINTS)                                   │
│  - 记忆引导 (MEMORY_GUIDANCE)                                   │
│  - 技能指导 (SKILLS_GUIDANCE)                                   │
│  - 工具使用规则 (TOOL_USE_ENFORCEMENT_GUIDANCE)                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 2: Session Context                                       │
│  - 当前会话消息历史                                              │
│  - FTS5 搜索上下文 (SESSION_SEARCH_GUIDANCE)                    │
│  - Honcho 上下文 (如果启用)                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 3: 上下文引用 (Context References)                        │
│  - 相关文件内容                                                  │
│  - 相关会话片段                                                  │
│  - 外部知识引用                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3.2 上下文压缩触发条件

```python
# agent/context_compressor.py
CONTEXT_COMPRESSION_THRESHOLDS = {
    # 消息 token 总数超过此值时触发压缩
    "message_count": 50,

    # 消息总 token 数超过此值时触发压缩
    "token_count": 100000,

    # 单个消息超过此值时触发压缩
    "single_message_tokens": 80000,
}

def should_compress(messages: List[dict]) -> bool:
    """检查是否需要压缩上下文"""
    total_tokens = estimate_messages_tokens(messages)
    return total_tokens > CONTEXT_COMPRESSION_THRESHOLDS["token_count"]
```

---

## 2.4 Prompt 构建

### 2.4.1 构建流程

```python
# agent/prompt_builder.py
def build_system_prompt(
    identity: str = DEFAULT_AGENT_IDENTITY,
    platform_hints: str = None,
    memory_guidance: str = None,
    skills_guidance: str = None,
    context_files: list = None,
) -> str:
    """
    构建完整的 system prompt。

    组成:
    1. 身份定义
    2. 平台提示
    3. 记忆引导
    4. 技能指导
    5. 上下文文件内容
    """
    parts = [identity]

    if platform_hints:
        parts.append(platform_hints)

    if memory_guidance:
        parts.append(memory_guidance)

    if skills_guidance:
        parts.append(skills_guidance)

    if context_files:
        parts.append(build_context_files_prompt(context_files))

    return "\n\n".join(parts)
```

### 2.4.2 各部分内容

| 组件 | 来源 | 说明 |
|------|------|------|
| `DEFAULT_AGENT_IDENTITY` | `agent/prompt_builder.py` | Agent 身份定义 |
| `PLATFORM_HINTS` | `agent/prompt_builder.py` | 平台特定提示 |
| `MEMORY_GUIDANCE` | `agent/prompt_builder.py` | 记忆系统使用引导 |
| `SESSION_SEARCH_GUIDANCE` | `agent/prompt_builder.py` | 会话搜索引导 |
| `SKILLS_GUIDANCE` | `agent/prompt_builder.py` | 技能使用引导 |
| `TOOL_USE_ENFORCEMENT_GUIDANCE` | `agent/prompt_builder.py` | 工具使用规则 |

---

## 2.5 工具调用消息流

### 2.5.1 完整工具调用流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    工具调用消息流                                 │
└─────────────────────────────────────────────────────────────────┘

模型响应:
{
    "role": "assistant",
    "content": null,
    "tool_calls": [
        {
            "id": "call_xxx",
            "type": "function",
            "function": {
                "name": "Read",
                "arguments": "{\"path\": \"README.md\"}"
            }
        }
    ]
}
    │
    ▼
model_tools.handle_function_call("Read", {"path": "README.md"})
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  工具执行                                                       │
│     │                                                          │
│     ├── 同步工具 (file_tools, terminal_tool)                    │
│     │     └─ 直接返回结果                                       │
│     │                                                          │
│     └── 异步工具 (browser_tool, mcp_tool)                       │
│           └─ _run_async() → 返回结果                            │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
工具结果消息:
{
    "role": "tool",
    "tool_call_id": "call_xxx",
    "content": "文件内容..."
}
    │
    ▼
追加到消息历史，继续下一轮迭代
```

### 2.5.2 错误处理

```python
# model_tools.py
def handle_function_call(function_name: str, function_args: dict, ...) -> str:
    try:
        # 执行工具
        result = _execute_tool(function_name, function_args)
        return result
    except ToolNotFoundError:
        return f"Error: Tool '{function_name}' not found"
    except ValidationError as e:
        return f"Error: Invalid arguments - {e}"
    except ExecutionError as e:
        return f"Error executing {function_name}: {e}"
    except Exception as e:
        return f"Error: {type(e).__name__}: {e}"
```

---

## 2.6 SessionDB 消息存储

### 2.6.1 消息存储时机

```python
# hermes_state.py
class SessionDB:
    def add_message(self, session_id: str, role: str, content: str, metadata: dict = None):
        """
        存储消息到数据库。

        存储时机:
        1. 每次工具调用完成后
        2. 最终响应返回前
        3. 上下文压缩触发前
        """
        self.conn.execute(
            """INSERT INTO messages (session_id, role, content, metadata)
               VALUES (?, ?, ?, ?)""",
            (session_id, role, content, json.dumps(metadata or {}))
        )
        self.conn.commit()

        # 同步到 FTS 索引
        self.conn.execute(
            "INSERT INTO message_fts (content, session_id) VALUES (?, ?)",
            (content, session_id)
        )
```

---

## 2.7 关键源码索引

| 功能 | 文件 | 关键函数 |
|------|------|---------|
| Prompt 构建 | `agent/prompt_builder.py` | `build_system_prompt()` |
| 上下文压缩 | `agent/context_compressor.py` | `should_compress()`, `compress()` |
| 工具执行 | `model_tools.py` | `handle_function_call()` |
| 消息存储 | `hermes_state.py` | `SessionDB.add_message()` |
| 会话关联 | `gateway/session.py` | `SessionStore.get_or_create_session()` |

---

## 2.8 下一步

- [工具注册与调度](../03-tool-system/01-registry.md) - tools/registry.py 详解
- [CLI 系统架构](../04-cli-system/01-main.md) - hermes_cli 模块分析
- [Gateway 系统](../05-gateway/01-overview.md) - 网关消息处理

---

[← 返回总目录](../SUMMARY.md) | [← AIAgent 核心](01-aiagent.md) | [工具注册 →](../03-tool-system/01-registry.md)
