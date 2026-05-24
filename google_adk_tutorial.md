# Google ADK 新手使用教程

本文是一份面向新手的 Google ADK 使用教程，示例基于 `google-adk-1.14.x` 的常见 Python 用法整理。文档只讲 ADK 组件、功能、使用方式和通用 demo。

## 1. Google ADK 是什么

Google ADK (Agent Development Kit) 是一个用于构建智能体应用的 Python 框架。它把一个智能体应用拆成几类核心对象：

- `Agent` / `LlmAgent`：负责调用模型，按 instruction 完成任务，也可以调用工具函数。
- `SequentialAgent`：按照顺序执行多个子智能体。
- `ParallelAgent`：并行执行多个子智能体。
- `BaseAgent`：用于自定义更复杂的编排逻辑。
- `Runner`：负责启动一次智能体运行，并持续产生事件。
- `SessionService`：负责保存会话和会话状态。
- `ToolContext`：工具函数访问会话状态的入口。
- `CallbackContext`：回调函数访问会话状态、agent 名称、运行上下文的入口。
- `LlmRequest` / `LlmResponse` / `google.genai.types`：用于描述模型请求、模型响应和消息内容。
- `LiteLLM`：模型适配器，可通过 LiteLLM 接入 OpenAI、Azure OpenAI、Vertex AI 等兼容模型服务。

一个典型 ADK 应用的数据流如下：

```text
用户输入
-> SessionService 创建 session，并写入初始 state
-> Runner 运行 root agent
-> Agent / Tool / Callback 持续读写 session.state
-> Runner 产生 Event 流
-> 调用方从最终 Event 或 state_delta 中取得结果
```

## 2. 安装与基础目录

安装：

```bash
pip install google-adk==1.14.1
```

一个适合新手的最小目录可以这样组织：

```text
demo_adk_app/
  app.py
  model.py
  agents.py
  tools.py
  callbacks.py
```

## 3. LiteLLM：模型适配器

### 功能

`LiteLLM` 用来创建 ADK 可识别的模型对象。它不是智能体本身，而是被传给 `Agent` 或 `LlmAgent` 的 `model` 参数。

它常用于以下场景：
- 使用非 Gemini 或 OpenAI 兼容模型。
- 使用自定义 `base_url`。
- 统一配置 API key、重试次数等参数。
- 让多个 agent 复用同一个模型配置。

### 运用方式

通常在单独的 `model.py` 中初始化一次模型对象，然后在多个 `agent` 中导入复用。

### Demo 代码

```python
# model.py
import os

from google.adk.models.lite_llm import LiteLLM


model = LiteLLM(
    model=os.getenv("LLM_MODEL", "openai/gpt-4.1-mini"),
    base_url=os.getenv("LLM_BASE_URL"),
    api_key=os.getenv("LLM_API_KEY"),
    num_retries=int(os.getenv("LLM_RETRY", "2")),
)
```

```python
# agents.py
from google.adk import Agent
from model import model

simple_agent = Agent(
    name="simple_agent",
    model=model,
    description="A simple assistant.",
    instruction="Answer the user's question clearly and concisely.",
    output_key="simple_agent_output",
)
```

## 4. Agent：最常用的 LLM 智能体

### 功能

`Agent` 是最常用的 ADK 智能体类型。它会根据 `instruction` 调用模型，必要时调用 `tools`，并把输出写入 `session.state`。

常用参数：
- `name`：agent 名称。事件流、日志、回调里都会用到。
- `model`：模型对象，例如 `LiteLLM(...)`。
- `description`：agent 说明，帮助编排和调试。
- `instruction`：系统任务说明，即prompt。可以写普通文本，也可以使用 state 变量占位。
- `tools`：工具函数列表。
- `output_key`：模型最终文本输出写入 `session.state` 的 key。
- `before_agent_callback`：agent 执行前运行的回调。
- `after_agent_callback`：agent 执行后运行的回调。

### 运用方式

当一个任务可以由单个模型调用完成时，优先使用`Agent`

### Demo 代码

```python
from google.adk import Agent
from model import model

classification_agent = Agent(
    name="classification_agent",
    model=model,
    description="Classify a user request into a known category.",
    instruction="""
You are a request classifier.

Input:
{user_text}

Return one category:
- billing
- technical
- account
- other
""".strip(),
    output_key="category",
)
```

在运行时，如果 `session.state["user_text"] = "I cannot log in"`，ADK 会把 `{user_text}` 注入到 prompt 中。

## 5. LlmAgent：显式的 LLM 智能体

### 功能

`LlmAgent` 和 `Agent` 的使用方式非常接近，都是 LLM-backed agent。很多代码中二者可以承担类似职责。选择建议：
- 想写最常见的智能体，使用 `Agent`。
- 想在代码上显式表达“这是一个 LLM agent”，使用 `LlmAgent`。

### 运用方式

`LlmAgent` 特别适合搭配工具函数，让模型按步骤决定何时调用工具。

### Demo 代码

```python
from google.adk.agents import LlmAgent

from model import model
from tools import lookup_product, check_inventory


shopping_agent = LlmAgent(
    name="shopping_agent",
    model=model,
    description="Help the user find product information.",
    instruction="""
Help the user answer product questions.
Use tools when you need fresh product details or inventory.
""".strip(),
    tools=[lookup_product, check_inventory],
    output_key="shopping_answer",
)
```

## 6. ToolContext：工具函数访问状态

### 功能

工具函数是普通 Python 函数。只要函数被放进 `agent` 的 `tools` 列表，模型就可以调用它。

`ToolContext` 是工具函数访问运行状态的入口，最常见的用法是：
- 从 `tool_context.state` 读取输入。
- 调用数据库、HTTP API、文件系统或内部函数。
- 把中间结果写回 `tool_context.state`，供后续 `agent`、`callback` 或 `tool` 使用。
- 返回字符串，作为工具调用结果反馈给模型。

### 运用方式

工具函数通常要保持小而明确：一个工具只做一件事，返回模型容易理解的文本或 JSON 字符串。

### Demo 代码

```python
# tools.py
import json

from google.adk.tools import ToolContext


def search_knowledge_base(tool_context: ToolContext) -> str:
    """Search internal knowledge by the current user query."""
    query = tool_context.state.get("user_query", "")

    # Demo: replace this with a real database or HTTP call.
    records = [
        {
            "title": "Password reset",
            "content": "Use the reset link on the sign-in page."
        },
        {
            "title": "MFA setup",
            "content": "Open settings and add an authenticator app."
        },
    ]

    matched = [item for item in records if query.lower() in item["content"].lower()]
    tool_context.state["kb_results"] = matched

    return json.dumps(matched, ensure_ascii=False)
```

```python
# agents.py
from google.adk import Agent

from model import model
from tools import search_knowledge_base


qa_agent = Agent(
    name="qa_agent",
    model=model,
    description="Answer questions using a knowledge base.",
    instruction="""
Answer the user question.
If the answer needs knowledge base information, call search_knowledge_base first.
""".strip(),
    tools=[search_knowledge_base],
    output_key="qa_answer"
)
```

## 7. CallbackContext：回调函数访问状态

### 功能

回调函数用于在 agent 执行前后做“非模型逻辑”。例如：
- 准备 prompt 所需的 state。
- 初始化状态容器。
- 把输入拆成多个批次。
- 解析模型输出中的标签或 JSON。
- 聚合多个并行 agent 的输出。
- 记录运行耗时、token 使用量、调试信息。
- 在无数据可处理时提前返回一个响应。

`CallbackContext` 常用属性：
- `callback_context.state`：当前 session 的共享状态。
- `callback_context.agent_name`：当前 agent 名称。
- `callback_context.invocation_id`：当前调用 ID。
- `callback_context.session`：当前 session 对象。

### 运用方式

回调分两类：
- `before_agent_callback`：在 agent 调用模型前运行。
- `after_agent_callback`：在 agent 调用模型后运行。

如果回调只修改`state`，返回`None`即可。如果需要覆盖可见输出或提前结束，可返回 ADK 支持的响应对象，例如 `types.Content` 或 `LlmResponse`。具体返回类型以当前 ADK 版本签名为准。

### Demo 代码：执行前准备 state

```python
# callbacks.py
from google.adk.agents.callback_context import CallbackContext


def prepare_user_context(callback_context: CallbackContext):
    state = callback_context.state

    user_profile = state.get("user_profile") or {}
    state["user_tier"] = user_profile.get("tier", "free")
    state["language"] = user_profile.get("language", "zh-CN")

    return None
```

```python
from google.adk import Agent

from callbacks import prepare_user_context
from model import model


support_agent = Agent(
    name="support_agent",
    model=model,
    description="Answer support questions.",
    instruction="""
User tier: {user_tier}
Preferred language: {language}

Answer the user request:
{user_query}
""".strip(),
    output_key="support_answer",
    before_agent_callback=[prepare_user_context],
)
```

### Demo 代码：执行后解析模型输出

```python
# callbacks.py
import re

from google.adk.agents.callback_context import CallbackContext
from google.genai import types


def extract_tag(text: str, tag: str) -> str:
    match = re.search(rf"<{tag}>(.*?)</{tag}>", text, flags=re.DOTALL | re.I)
    return match.group(1).strip() if match else ""


def parse_category(callback_context: CallbackContext):
    raw_output = callback_context.state.get("raw_category_output", "")
    category = extract_tag(raw_output, "category")

    callback_context.state["category"] = category or "other"

    return types.Content(
        role="model",
        parts=[types.Part(text=f"Category saved: {callback_context.state['category']}")],
    )
```

```python
from google.adk import Agent

from callbacks import parse_category
from model import model


category_agent = Agent(
    name="category_agent",
    model=model,
    description="Classify the request.",
    instruction="""
Classify the request.
Return only:
<category>billing|technical|account|other</category>

Request:
{user_query}
""".strip(),
    output_key="raw_category_output",
    after_agent_callback=[parse_category],
)
```

## 8. output_key 与 session.state

### 功能

`output_key` 是 ADK 的一个重要参数。它会把 `agent` 的输出保存到 `session.state[output_key]`。

这带来三个好处：
- 后续 agent 可以通过 `{output_key}` 在 prompt 中引用前一步结果。
- callback 可以从 `state[output_key]` 读取并解析结果。
- 最终调用方可以从 `event` 或 `state` 里拿到结构化结果。

### 运用方式

为每个关键步骤设置清晰的 `output_key`，不要让多个 `agent` 写同一个 `key`，除非你明确希望覆盖。

### Demo 代码

```python
from google.adk import Agent
from google.adk.agents import SequentialAgent

from model import model


draft_agent = Agent(
    name="draft_agent",
    model=model,
    instruction="Draft a short answer for: {user_query}",
    output_key="draft_text",
)

review_agent = Agent(
    name="review_agent",
    model=model,
    instruction="""
Improve this draft:
{draft_text}

Return the final version.
""".strip(),
    output_key="final_text",
)

pipeline = SequentialAgent(
    name="draft_then_review",
    sub_agents=[draft_agent, review_agent],
)
```

## 9. SequentialAgent：顺序编排

### 功能

`SequentialAgent` 会按 `sub_agents` 的顺序逐个运行 sub agent。前一个 agent 写入 state 的结果，可以被后一个 agent 使用。

适合以下场景：
- 先分类，再检索，再总结。
- 先准备数据，再执行模型任务，再聚合结果。
- 多个步骤必须严格按顺序执行。

### 运用方式

把每个步骤拆成独立 agent，通过 `output_key` 和 `callback` 共享状态。

### Demo 代码

```python
from google.adk import Agent
from google.adk.agents import SequentialAgent

from model import model


intent_agent = Agent(
    name="intent_agent",
    model=model,
    instruction="Classify the user intent: {user_query}",
    output_key="intent",
)

answer_agent = Agent(
    name="answer_agent",
    model=model,
    instruction="""
User intent:
{intent}

User query:
{user_query}

Write a helpful answer.
""".strip(),
    output_key="answer"
)

root_agent = SequentialAgent(
    name="root_agent",
    description="Classify first, answer second.",
    sub_agents=[intent_agent, answer_agent],
)
```

## 10. ParallelAgent：并行编排

### 功能

`ParallelAgent` 会并行运行多个 sub agent。它适合把互相独立的任务同时执行，提高吞吐和整体速度。

适合以下场景：
- 同时生成多个角度的分析。
- 对多个批次、多个行、多个文档并行处理。
- 同时调用多个互不依赖的工具型 agent。

### 运用方式

每个并行分支应写入不同的 `output_key`，避免状态互相覆盖。并行结束后，通常用 `after_agent_callback` 聚合结果。

### Demo 代码

```python
from google.adk import Agent
from google.adk.agents import ParallelAgent, SequentialAgent

from callbacks import merge_parallel_outputs
from model import model


pros_agent = Agent(
    name="pros_agent",
    model=model,
    instruction="List the advantages of this idea: {idea}",
    output_key="pros",
)

cons_agent = Agent(
    name="cons_agent",
    model=model,
    instruction="List the risks of this idea: {idea}",
    output_key="cons",
)

parallel_review = ParallelAgent(
    name="parallel_review",
    sub_agents=[pros_agent, cons_agent],
    description="Analyze advantages and risks at the same time.",
)

final_agent = Agent(
    name="final_agent",
    model=model,
    instruction="""
Advantages:
{pros}

Risks:
{cons}

Create a balanced recommendation.
""".strip(),
    output_key="recommendation",
)

root_agent = SequentialAgent(
    name="parallel_then_final",
    sub_agents=[parallel_review, final_agent],
)
```

```python
# callbacks.py
from google.adk.agents.callback_context import CallbackContext


def merge_parallel_outputs(callback_context: CallbackContext):
    state = callback_context.state
    state["review_summary"] = {
        "pros": state.get("pros", ""),
        "cons": state.get("cons", ""),
    }
    return None
```

## 11. BaseAgent：自定义编排器

### 功能

当 `SequentialAgent` 和 `ParallelAgent` 不够用时，可以继承 `BaseAgent` 实现自己的编排逻辑。

常见用途：
- 根据运行时 state 动态创建 sub agent。
- 把列表拆成多个批次，每个批次创建一个 agent。
- 动态选择并行分支数量。
- 在自定义逻辑里调用 `ParallelAgent.run_async(ctx)`。
- 只传递 sub agent 的事件，不产生额外可见文本。

### 关键点

自定义 `BaseAgent` 时通常要实现：
- 类字段 `name: str`
- 类字段 `description: str`
- `async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]`

`ctx.session.state` 是当前 session 的状态容器。自定义编排器可以读写它。

### Demo 代码

```python
from __future__ import annotations

from typing import AsyncGenerator, ClassVar

from google.adk import Agent
from google.adk.agents import BaseAgent, ParallelAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events import Event

from model import model


class BatchSummaryOrchestrator(BaseAgent):
    name: str = "batch_summary_orchestrator"
    description: str = "Create one summary agent per input batch and run them in parallel."

    SUMMARY_TEMPLATE: ClassVar[str] = """
Summarize this batch in three bullet points:

{batch_text}
""".strip()

    async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        state = ctx.session.state
        batches: list[str] = state.get("batches", [])

        if not batches:
            return

        sub_agents = []
        for index, batch_text in enumerate(batches):
            state_key = f"batch_{index}"
            output_key = f"batch_summary_{index}"
            state[state_key] = batch_text

            sub_agents.append(
                Agent(
                    name=f"summary_agent_{index}",
                    model=model,
                    description=f"Summarize batch {index}.",
                    instruction=self.SUMMARY_TEMPLATE.replace("{batch_text}", "{" + state_key + "}"),
                    output_key=output_key,
                )
            )

        parallel = ParallelAgent(
            name="parallel_batch_summary",
            sub_agents=sub_agents,
            description="Run all batch summaries concurrently.",
        )

        async for event in parallel.run_async(ctx):
            yield event
```

## 12. InvocationContext：自定义编排中的上下文

### 功能

`InvocationContext` 是 agent 运行时上下文。新手最常用的是它的 `session`：

```python
state = ctx.session.state
```

在自定义 `BaseAgent` 里，它是读取输入、创建动态 sub agent、传递上下文的关键对象。

### Demo 代码

```python
from typing import AsyncGenerator

from google.adk.agents import BaseAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events import Event


class NoopAgent(BaseAgent):
    name: str = "noop_agent"
    description: str = "Record that this step was reached."

    async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        ctx.session.state["noop_reached"] = True
        return
```

## 13. Event：读取运行过程与最终结果

### 功能

`Runner.run_async(...)` 返回的是异步事件流。每个事件通常包含：
- `event.author`：产生事件的 `agent` 名称。
- `event.content`：本次事件的文本内容。
- `event.actions.state_delta`：本次事件写入 `state` 的增量。
- `event.is_final_response()`：是否是最终响应。

### 调用方式

调用方可以遍历事件流：
- 打印中间进度。
- 只捕获某个最终 `agent` 的最终输出。
- 从 `state_delta` 中提取结构化结果。

### Demo 代码

```python
async for event in runner.run_async(
    user_id=user_id,
    session_id=session.id,
    new_message=user_message,
    run_config=run_config,
):
    print("event author:", event.author)

    if event.content and event.content.parts:
        print("text:", event.content.parts[0].text)

    if event.is_final_response():
        print("final response from:", event.author)
```

如果某个 callback 在 state 中写入了最终结果，也可以从 `state_delta` 中读取：

```python
async for event in runner.run_async(
    user_id=user_id,
    session_id=session.id,
    new_message=user_message,
):
    if event.is_final_response():
        result = event.actions.state_delta.get("final_result")
        if result is not None:
            print(result)
```

## 14. InMemorySessionService：内存会话服务

### 功能

`InMemorySessionService` 用内存保存 session。它适合：
- 本地开发。
- 单进程 demo。
- 单元测试。
- 临时任务。

不适合：
- 多进程共享状态。
- 服务重启后保留历史会话。
- 生产环境长期会话存储。

### 运用方式

每次请求开始时创建一个 `session`，把初始输入写入 `state`。

### Demo 代码

```python
from google.adk.sessions import InMemorySessionService


session_service = InMemorySessionService()

session = await session_service.create_session(
    app_name="demo_app",
    user_id="user_001",
    session_id="session_001",
    state={
        "user_query": "How do I reset my password?",
        "user_profile": {"tier": "pro", "language": "en-US"},
    },
)
```

## 15. Runner 与 RunConfig：启动一次运行

### 功能

`Runner` 负责把 root agent、session service 和 app 名称连接起来，并启动运行。

`RunConfig` 用于配置运行行为，例如指定响应模态：
```python
RunConfig(response_modalities=["TEXT"])
```

### 运用方式

典型步骤：
1. 创建 `session_service`。
2. 创建 session，并写入初始 state。
3. 创建 `Runner(app_name, agent, session_service)`。
4. 创建用户消息 `types.Content(...)`。
5. 调用 `runner.run_async(...)`。
6. 遍历事件流，提取最终结果。

### Demo 代码

```python
import asyncio
import json
import uuid

from google.adk.runners import Runner, RunConfig
from google.adk.sessions import InMemorySessionService
from google.genai import types

from agents import root_agent


APP_NAME = "demo_app"
USER_ID = "user_001"


async def run_demo(user_query: str):
    session_service = InMemorySessionService()
    session_id = f"session_{uuid.uuid4().hex}"

    initial_state = {"user_query": user_query}
    session = await session_service.create_session(
        app_name=APP_NAME,
        user_id=USER_ID,
        session_id=session_id,
        state=initial_state,
    )

    runner = Runner(
        app_name=APP_NAME,
        agent=root_agent,
        session_service=session_service,
    )

    user_message = types.Content(
        role="user",
        parts=[types.Part(text=json.dumps(initial_state, ensure_ascii=False))],
    )

    run_config = RunConfig(response_modalities=["TEXT"])

    final_text = None
    async for event in runner.run_async(
        user_id=session.user_id,
        session_id=session.id,
        new_message=user_message,
        run_config=run_config,
    ):
        if event.is_final_response() and event.content and event.content.parts:
            final_text = event.content.parts[0].text

    return final_text


if __name__ == "__main__":
    print(asyncio.run(run_demo("How do I reset my password?")))
```

## 16. google.genai.types.Content 与 Part：构造消息

### 功能

ADK 使用 `google.genai.types` 表达消息内容。最常见的是：
- `types.Content`：一条消息，包含角色和多个 `part`。
- `types.Part`：消息中的一段内容，最常见是 `text`。
- `types.Candidate`：模型候选响应。
- `types.GenerateContentResponse`：生成式模型响应结构。

### 运用方式

调用 `Runner` 时，用户输入可以包装成 `types.Content`：

```python
from google.genai import types


user_message = types.Content(
    role="user",
    parts=[types.Part(text="Please summarize this document.")],
)
```

回调中如果要替换可见输出，也可以返回 `types.Content`：

```python
from google.adk.agents.callback_context import CallbackContext
from google.genai import types


def replace_visible_output(callback_context: CallbackContext):
    callback_context.state["done"] = True

    return types.Content(
        role="model",
        parts=[types.Part(text="Done.")],
    )
```

## 17. LlmRequest 与 LlmResponse：控制模型请求和响应

### 功能

`LlmRequest` 和 `LlmResponse` 是 ADK 对模型请求与响应的抽象。新手通常不会直接构造 `LlmRequest`，但可能会在 callback 的函数签名中接收它。

`LlmResponse` 常用于：
- 在 before callback 中判断没有数据可处理，提前返回固定响应。
- 在 after callback 中覆盖模型响应。
- 为跳过某个步骤提供明确的模型输出。

### 运用方式

当无需提前结束时，`callback` 返回 `None`。当需要直接返回内容时，构造 `LlmResponse`。

### Demo 代码

```python
from typing import Optional

from google.adk.agents.callback_context import CallbackContext
from google.adk.models import LlmRequest, LlmResponse
from google.genai import types


def stop_if_no_items(
    callback_context: CallbackContext,
    llm_request: Optional[LlmRequest] = None,
    **kwargs,
):
    items = callback_context.state.get("items", [])

    if items:
        return None

    callback_context.state["final_result"] = []

    return LlmResponse(
        content=types.Content(
            role="model",
            parts=[types.Part(text="No items to process.")],
        )
    )
```

不同 ADK patch 版本对 `LlmResponse.content` 的内部结构可能有差异。如果遇到类型错误，可以改用 `types.GenerateContentResponse` + `types.Candidate` 的形式：

```python
return LlmResponse(
    content=types.GenerateContentResponse(
        candidates=[
            types.Candidate(
                content=types.Content(
                    role="model",
                    parts=[types.Part(text="No items to process.")],
                )
            )
        ]
    )
)
```

## 18. 组合模式一：预处理 -> 并行处理 -> 聚合

这是 ADK 中很常见的模式：

1. `before_agent_callback` 把输入拆成多个批次，写入 state。
2. 自定义 `BaseAgent` 根据批次数量动态创建多个子 `agent`。
3. `ParallelAgent` 并行处理。
4. `after_agent_callback` 汇总所有输出，写入最终 `state`。

### Demo 代码

```python
# callbacks.py
from google.adk.agents.callback_context import CallbackContext


def prepare_batches(callback_context: CallbackContext):
    items = callback_context.state.get("items", [])
    batches = [items[i : i + 5] for i in range(0, len(items), 5)]

    plan = []
    for index, batch in enumerate(batches):
        batch_key = f"batch_{index}"
        output_key = f"batch_output_{index}"

        callback_context.state[batch_key] = batch
        plan.append({"batch_key": batch_key, "output_key": output_key})

    callback_context.state["batch_plan"] = plan
    return None


def aggregate_batches(callback_context: CallbackContext):
    state = callback_context.state
    plan = state.get("batch_plan", [])

    outputs = []
    for item in plan:
        output = state.get(item["output_key"], "")
        if output:
            outputs.append(output)

    state["final_result"] = outputs
    return None
```

```python
# agents.py
from typing import AsyncGenerator

from google.adk import Agent
from google.adk.agents import BaseAgent, ParallelAgent, SequentialAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events import Event

from callbacks import aggregate_batches, prepare_batches
from model import model


class BatchProcessor(BaseAgent):
    name: str = "batch_processor"
    description: str = "Run one processor agent per batch."

    async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        state = ctx.session.state
        plan = state.get("batch_plan", [])

        agents = []
        for index, item in enumerate(plan):
            agents.append(
                Agent(
                    name=f"batch_agent_{index}",
                    model=model,
                    instruction="Process this batch:\n{" + item["batch_key"] + "}",
                    output_key=item["output_key"],
                )
            )

        if not agents:
            return

        parallel = ParallelAgent(
            name="parallel_batches",
            sub_agents=agents,
        )

        async for event in parallel.run_async(ctx):
            yield event


root_agent = SequentialAgent(
    name="batch_pipeline",
    sub_agents=[BatchProcessor()],
    before_agent_callback=[prepare_batches],
    after_agent_callback=[aggregate_batches],
)
```

## 19. 组合模式二：工具调用 -> 标签解析 -> 状态传递

### 功能

当模型输出需要被程序稳定消费时，不建议只依赖自然语言。可以要求模型输出 XML-like 标签或 JSON，然后用 after callback 解析。

### Demo 代码

```python
from google.adk import Agent

from callbacks import extract_decision
from model import model


decision_agent = Agent(
    name="decision_agent",
    model=model,
    instruction="""
Read the input and decide whether it should be approved.

Return exactly:
<decision>approve or reject</decision>
<reason>short reason</reason>

Input:
{request_text}
""".strip(),
    output_key="decision_raw",
    after_agent_callback=[extract_decision],
)
```

```python
import re

from google.adk.agents.callback_context import CallbackContext


def _between(text: str, tag: str) -> str:
    match = re.search(rf"<{tag}>(.*?)</{tag}>", text, re.DOTALL | re.I)
    return match.group(1).strip() if match else ""


def extract_decision(callback_context: CallbackContext):
    raw = callback_context.state.get("decision_raw", "")
    callback_context.state["decision"] = _between(raw, "decision")
    callback_context.state["decision_reason"] = _between(raw, "reason")
    return None
```

## 20. 组合模式三：运行入口封装成 API

### 功能

ADK 应用常被封装成 HTTP API。API 层负责：
- 校验请求。
- 创建 session。
- 选择 root agent。
- 调用 Runner。
- 读取最终结果。
- 返回 JSON。

### Demo 代码

```python
import json
import uuid

from fastapi import FastAPI, HTTPException
from google.adk.runners import Runner, RunConfig
from google.adk.sessions import InMemorySessionService
from google.genai import types
from pydantic import BaseModel

from agents import root_agent


app = FastAPI()
session_service = InMemorySessionService()

APP_NAME = "demo_app"
USER_ID = "default_user"


class AnalyzeRequest(BaseModel):
    user_query: str
    mode: str = "default"


@app.post("/analyze")
async def analyze(req: AnalyzeRequest):
    session_id = f"session_{uuid.uuid4().hex}"
    initial_state = req.model_dump()

    session = await session_service.create_session(
        app_name=APP_NAME,
        user_id=USER_ID,
        session_id=session_id,
        state=initial_state,
    )

    runner = Runner(
        app_name=APP_NAME,
        agent=root_agent,
        session_service=session_service,
    )

    user_message = types.Content(
        role="user",
        parts=[types.Part(text=json.dumps(initial_state, ensure_ascii=False))],
    )

    final_result = None
    async for event in runner.run_async(
        user_id=session.user_id,
        session_id=session.id,
        new_message=user_message,
        run_config=RunConfig(response_modalities=["TEXT"]),
    ):
        if event.is_final_response():
            final_result = event.actions.state_delta.get("final_result")
            if final_result is None and event.content and event.content.parts:
                final_result = event.content.parts[0].text

    if final_result is None:
        raise HTTPException(status_code=500, detail="No final result")

    return {"result": final_result}
```

## 21. 新手实践建议

### 1. 先把 state 设计清楚

ADK 多 agent 编排的核心是 `session.state`。建议提前列出：
- 初始输入 key。
- 每个 agent 的 `output_key`。
- 每个 callback 写入的中间 key。
- 最终结果 key。

### 2. 每个 agent 只做一类任务

不要让一个 agent 同时完成分类、检索、过滤、总结、聚合。拆成多个 agent 后：
- prompt 更短。
- 输出更稳定。
- 失败点更容易定位。
- callback 更容易解析。

### 3. 并行分支必须避免写同一个 key

并行 agent 如果写同一个 `output_key`，结果可能互相覆盖。推荐使用：
```text
batch_output_0
batch_output_1
batch_output_2
...
```
再由聚合 callback 统一汇总。

### 4. 模型输出要便于解析

需要程序读取的内容，最好让模型输出：
```text
<keep_list>1,3,5</keep_list>
```
或：
```json
{"decision": "approve", "reason": "Matched required criteria"}
```
然后在 after callback 中解析并写入 state。

### 5. 工具函数负责确定性逻辑

模型适合理解、判断、生成。工具函数适合：
- 查数据库。
- 调 API。
- 读文件。
- 解析 JSON。
- 计算指标。
- 保存中间结果。

确定性逻辑放工具或 callback，模型输出会更可靠。

### 6. 回调里做好空数据分支

在真实应用中，经常会出现没有记录、没有匹配、工具返回空结果。建议 before callback 提前处理：

```python
if not callback_context.state.get("items"):
    callback_context.state["final_result"] = []
    return None
```

### 7. Runner 事件流要完整消费

如果你在事件流中很早拿到了最终结果，也建议继续消费完事件流，让底层运行过程自然结束。这样更利于 tracing、日志和资源清理。

## 22. 最小完整示例

下面是一个可串起来理解的最小完整例子：用户输入一个问题，agent 调用工具查询知识库，最终生成回答。

```python
import asyncio
import json
import os
import uuid

from google.adk import Agent
from google.adk.models.lite_llm import LiteLLM
from google.adk.runners import Runner, RunConfig
from google.adk.sessions import InMemorySessionService
from google.adk.tools import ToolContext
from google.genai import types


model = LiteLLM(
    model=os.getenv("LLM_MODEL", "openai/gpt-4.1-mini"),
    base_url=os.getenv("LLM_BASE_URL"),
    api_key=os.getenv("LLM_API_KEY"),
    num_retries=2,
)


def search_docs(tool_context: ToolContext) -> str:
    query = tool_context.state.get("user_query", "")
    docs = [
        {"title": "Reset password", "text": "Click Forgot password on the sign-in page."},
        {"title": "Change email", "text": "Open account settings and choose Email."},
    ]
    matched = [doc for doc in docs if query.lower() in doc["text"].lower()]
    tool_context.state["search_results"] = matched
    return json.dumps(matched, ensure_ascii=False)


root_agent = Agent(
    name="qa_agent",
    model=model,
    description="Answer user questions with optional document search.",
    instruction="""
Answer the user's question.
If you need document information, call search_docs.

Question:
{user_query}
""".strip(),
    tools=[search_docs],
    output_key="final_answer"
)


async def main():
    session_service = InMemorySessionService()
    session_id = f"session_{uuid.uuid4().hex}"

    initial_state = {"user_query": "password"}
    session = await session_service.create_session(
        app_name="demo_app",
        user_id="user_001",
        session_id=session_id,
        state=initial_state,
    )

    runner = Runner(
        app_name="demo_app",
        agent=root_agent,
        session_service=session_service,
    )

    user_message = types.Content(
        role="user",
        parts=[types.Part(text=json.dumps(initial_state, ensure_ascii=False))],
    )

    final_text = None
    async for event in runner.run_async(
        user_id=session.user_id,
        session_id=session.id,
        new_message=user_message,
        run_config=RunConfig(response_modalities=["TEXT"]),
    ):
        if event.is_final_response() and event.content and event.content.parts:
            final_text = event.content.parts[0].text

    print(final_text)


if __name__ == "__main__":
    asyncio.run(main())
```

## 23. 组件速查表

| 组件 | 作用 | 常见使用位置 |
| --- | --- | --- |
| `LiteLLM` | 创建模型适配器 | `model.py` |
| `Agent` | 定义普通 LLM 智能体 | 单步任务、工具调用 |
| `LlmAgent` | 显式定义 LLM-backed agent | 工具型 agent、明确表达 LLM agent |
| `SequentialAgent` | 顺序运行多个 sub agent | 多步骤 pipeline |
| `ParallelAgent` | 并行运行多个 sub agent | 多批次、多角度、多分支处理 |
| `BaseAgent` | 自定义动态编排逻辑 | 根据 state 动态创建 agent |
| `InvocationContext` | 自定义 agent 的运行上下文 | `BaseAgent._run_async_impl` |
| `Event` | Runner 产生的运行事件 | 遍历 `runner.run_async(...)` |
| `InMemorySessionService` | 内存会话存储 | 本地开发、测试、简单服务 |
| `Runner` | 启动并驱动 root agent | API handler、CLI 入口 |
| `RunConfig` | 配置运行行为 | 设置响应模态等 |
| `ToolContext` | 工具函数访问 state | tool 函数参数 |
| `CallbackContext` | 回调函数访问 state 和上下文 | before/after callback |
| `LlmRequest` | 模型请求对象 | 高级 callback 签名 |
| `LlmResponse` | 模型响应对象 | callback 提前返回或覆盖响应 |
| `types.Content` | 消息内容 | 用户消息、回调返回 |
| `types.Part` | 消息片段 | 文本消息 |

## 24. 推荐学习顺序

1. 先学会 `Agent + LiteLLM + Runner + InMemorySessionService`，跑通单 agent。
2. 再学习 `ToolContext`，让 agent 能调用工具。
3. 接着学习 `output_key + session.state`，理解多步骤之间如何传数据。
4. 然后学习 `SequentialAgent`，完成线性 pipeline。
5. 再学习 `ParallelAgent`，处理并行分支。
6. 最后学习 `BaseAgent + InvocationContext + Event`，实现动态编排。
