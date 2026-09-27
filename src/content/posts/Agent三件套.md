---
title: '2026年Agent三件套'
published: 2026-06-12
description: '这次来谈谈A2A，MCP和Skills，写的很乱，算是给自己复习一遍吧。'
tags: [AI]
category: note
draft: false
---

这次来谈谈A2A，MCP和Skills，写的很乱，算是给自己复习一遍吧。

# 一、全局概念

先看一下A2A、MCP、Skills三者的协同架构图：

<img src="https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260612144502626.png" alt="image-20260612144502543" style="zoom: 50%;" />

|  概念  |                     角色                      | 方向 |       提出者        |
| :----: | :-------------------------------------------: | :--: | :-----------------: |
|  A2A   |   通信协议——Agent 与 Agent 之间的协作标准。   | 横向 |   Google (2025.4)   |
|  MCP   | 连接器——Agent 和外部工具/数据之间的标准接口。 | 纵向 | Anthropic (2024.11) |
| Skills |   知识包——封装了领域流程知识的可复用模块。    | 内部 | Anthropic (2025.10) |

在 Skills 出现之前，MCP 承担了大量本应由 Skills 完成的技能封装的工作。但 Skills 引入的渐进式披露思想，有效解决了 MCP 在复杂场景下容易出现的Token 爆炸的问题，因此热度迅速超过了 MCP。此后，MCP 也逐步回归其核心定位，专注于 Agent 与外部工具 / 数据之间的连接。不过，三者本质上工作在 AI 系统的不同层次，是相互补充而非替代的关系。在我看来，目前最理想的架构是：多个Agent通过A2A协作，每个Agent内部用Skills指导自己怎么做，通过MCP连接需要用到的工具和数据。

# 二、A2A

A2A 全称 **Agent2Agent Protocol**，是Google提出的一套让不同厂商、不同框架、不同服务器上的 AI Agent 互相发现、互相通信、互相协作而不需要暴露内部记忆、工具和实现细节的开放协议。

谈A2A之前必须要说的是：现在的行业落地里，A2A还不是主流工程方案，LangGraph / CrewAI / AutoGen / 自研 Orchestrator或者 MCP 仍然是当前多 Agent 落地更常见的工程方式。原因也比较简单。第一，目前公开可见的大规模、跨组织 Agent 协作项目还不算多，很多所谓多 Agent 系统，其实都运行在同一个项目、同一个进程或者同一个服务集群里。比如用 LangGraph 写 Multi-Agent，本质上直接函数调用或者框架内部状态图编排就能完成；即使不同 Agent 挂成独立服务，LangGraph 或自研调度器也可以通过 HTTP/API 去编排。第二，如果只是想远程调用某个 Agent 去处理一个具体任务，也可以把这个 Agent 包装成 MCP Server，对外暴露成一个工具。这样主 Agent 通过 MCP 调用它，也能完成类似 Agent-to-Agent 的效果。所以从短期工程落地角度看，A2A 在很多内部项目里并不是刚需。但是 A2A 的价值在于，它不是简单地把 Agent 当工具调用，而是把远程 Agent 当成一个有任务状态、有上下文、有产物返回能力的独立协作对象。特别是面对长任务状态跟踪、中途补充输入、流式进度反馈、异步通知、最终 Artifact 返回这类场景时，MCP 做起来就会比较别扭，而 A2A 里的 Task、Message、Artifact 这些设计会更自然。所以A2A 现在可能还不是主流落地方案，但它解决的是未来 Agent 生态互联的问题。随着 Agent 服务越来越多，跨框架、跨团队、跨平台调用 Agent 的需求增加，专门的 Agent 通信协议大概率会有应用空间。也正是因为这个趋势，我觉得 A2A 还是值得提前学习和理解的。当然这只是个人观点。

ok下面正式开始谈一下A2A。

## A2A的通信方式

A2A 采用经典的C/S架构，使用 **JSON-RPC 2.0 over HTTP(S)**，支持同步请求响应、SSE 流式响应、异步推送通知，并能交换文本、文件和结构化 JSON 数据。

常见模式有三种：

**第一种：普通请求响应**

适合很快完成的任务。

```
Client → Server：查一下今天成都天气
Server → Client：返回结果
```

**第二种：SSE 流式返回**

适合边生成边返回，比如写报告、写代码。

```
Client → Server：生成一份报告
Server → Client：正在检索资料
Server → Client：正在生成大纲
Server → Client：正在生成正文
Server → Client：完成
```

官方文档也说明，A2A 支持通过 Server-Sent Events 接收实时增量结果或状态更新。

**第三种：异步 Push**

适合超长任务。比如远程 Agent 跑实验，跑完后通过 webhook 通知我们。

```
Client：你慢慢跑，跑完通知我
Server：收到
……
Server → webhook：任务完成
```

## A2A 的工作流程

A2A的核心机制：

| 机制          | 说明                                                         | 层次   |
| ------------- | ------------------------------------------------------------ | ------ |
| Agent Card    | 每个 Agent 的名片，用 JSON 格式描述自己的能力 —— 其他 Agent 据此判断该找谁合作 | 协议层 |
| Message       | Agent 之间交换的消息，可以携带文本、文件、结构化数据         | 协议层 |
| Task          | 任务管理机制，有完整的生命周期（创建 → 进行中 → 完成 / 失败） | 协议层 |
| Artifact      | 任务的交付物 ——Agent 之间交换的工作成果                      | 协议层 |
| AgentExecutor | A2A 请求进入业务逻辑的执行入口                               | 业务层 |

这些定义都不难理解。下面以Google 官方 codelab 里的 purchasing concierge 为例。

这个场景是：一个购买助理 Agent 去远程调用Burger Seller Agent。

先看一下完整流程：

```
Burger Seller Agent 启动
  ↓
暴露 Agent Card：/.well-known/agent.json
  ↓
购买助理 Agent 启动
  ↓
读取远程 Agent Card，保存到 cards / remote_agent_connections
  ↓
用户说：我想买汉堡
  ↓
购买助理根据 Agent Card 判断该找 Burger Seller Agent
  ↓
购买助理把用户请求封装成 Message
  ↓
通过 A2A Client 发给 Burger Seller Agent
  ↓
Burger Seller Agent 收到请求，形成 Task 或直接返回 Message
  ↓
Burger Seller Agent 的 AgentExecutor 执行业务逻辑
  ↓
返回确认问题 / 订单结果 / Artifact
  ↓
购买助理把结果转给用户
```

Agent Card 是调用前读取的；Message 是调用时发过去的；Task 是远程 Agent 接住任务后管理状态的；Artifact 是任务完成后的交付物；AgentExecutor 是远程 Agent 收到任务后真正干活的代码。

**第一步：Burger Seller Agent 先把自己暴露成 A2A Server**

Burger Seller Agent 是被调用方。它启动时要做三件事：

1. 声明自己会什么：AgentSkill
2. 声明自己是谁、在哪、有什么能力：AgentCard
3. 实现收到请求后怎么处理：AgentExecutor

（首先说明：接下来A2A这一节提及的 skill 为通用功能定义，和业界热门的 AI Skills 只是名称巧合，二者无关。）

接下来我们定义一个售卖汉堡的 skill。

```
sell_burger_skill = AgentSkill(
    id="sell_burger",
    name="Sell Burger",
    description="Handle burger ordering requests",
    tags=["burger", "food", "order"],
    examples=[
        "I want a cheeseburger",
        "我想买一个汉堡"
    ]
)
```

然后把这个 skill 放进 Agent Card：

```
agent_card = AgentCard(
    name="Burger Seller Agent",
    description="A seller agent that handles burger purchase requests",
    url="http://localhost:10001/",
    version="1.0.0",
    skills=[sell_burger_skill],
)
```

服务启动后，A2A 框架会把这个 `AgentCard` 序列化成 JSON，并通过固定路径`/.well-known/agent.json`暴露出来：

```
{
  "name": "Burger Seller Agent",
  "description": "A seller agent that handles burger purchase requests",
  "url": "http://localhost:10001/",
  "skills": [
    {
      "id": "sell_burger",
      "name": "Sell Burger",
      "description": "Handle burger ordering requests"
    }
  ]
}
```

通过这个Agent Card，其他 Agent 在调用前就可以知道这个远程 Agent 是谁、在哪里、能做什么。

**第二步：购买助理 Agent 读取远程 Agent Card**

购买助理 Agent 是调用方。它会先从配置中拿到候选远程 Agent 的地址：

```
remote_agent_addresses = [
    "http://localhost:10001",  # Burger Seller Agent
    "http://localhost:10002",  # Pizza Seller Agent
]
```

然后逐个请求它们的 Agent Card：

```
for address in remote_agent_addresses:
    card = get(f"{address}/.well-known/agent.json")
    cards[address] = card
```

用户说“我想买汉堡”时，购买助理就可以根据这些 Agent Card 判断应该调用 Burger Seller Agent。

**第三步：购买助理把用户请求封装成 Message**

购买助理选择 Burger Seller Agent 后，构造 A2A Message发送给Burger Seller Agent：

```
{
  "role": "user",
  "parts": [
    {
      "kind": "text",
      "text": "Order lunch for the team."
    },
    {
      "kind": "data",
      "data": {
        "people": 5,
        "budget": 80,
        "dietaryRestrictions": ["no pork", "one vegetarian"]
      }
    }
  ],
  "messageId": "msg-001"
}
```

**第四步：Burger Seller Agent 接收 Message，并通过 AgentExecutor 处理 Task**

Burger Seller Agent 收到这个 Message 后，A2A Server 会负责解析协议请求，并把它交给 `AgentExecutor` 处理。

`AgentExecutor` 可以理解成远程 Agent 的执行入口：A2A 请求进来以后，真正进入业务逻辑的地方就是这里。

```
class BurgerAgentExecutor(AgentExecutor):
    async def execute(self, context, event_queue):
        # 1. 从 context 中取出购买助理发来的 Message
        user_message = context.message

        # 2. 调用自己的业务逻辑，例如检查菜单、计算价格、处理饮食限制
        result = await burger_seller.handle_order(user_message)

        # 3. 把处理结果通过 event_queue 返回给调用方
        await event_queue.enqueue_event(
            new_agent_text_message(result)
        )
```

Burger Seller Agent 可以从`context.message`里面拿到两类信息：一类是文本请求，比如 `Order lunch for the team.`；另一类是结构化数据，比如人数、预算和饮食限制。

因为Burger Seller Agent 可能不能马上给出最终结果。它可能需要先检查菜单，再组合套餐，再判断预算，最后还要让用户确认：

```
I can order 4 cheeseburgers and 1 veggie burger for 78 dollars. Please confirm.
```

所以这次远程调用就需要被当成一个 Task 来管理，而不是一次性 Message 回复。Task 的状态可能是：

```
submitted → working → input_required → working → completed
```

例如 Burger Seller Agent 处理到一半，需要用户确认订单，就可以返回一个 `input_required` 状态：

```
{
  "id": "task-123",
  "status": {
    "state": "input_required",
    "message": {
      "role": "agent",
      "parts": [
        {
          "kind": "text",
          "text": "I can order 4 cheeseburgers and 1 veggie burger for 78 dollars. Please confirm."
        }
      ]
    }
  }
}
```

购买助理 Agent 收到这个状态后，不会把任务当成已经完成，而是把确认问题转给用户。用户确认后，购买助理再把确认信息发回 Burger Seller Agent，远程商家 Agent 继续处理同一个 Task，直到状态变成 `completed` 或 `failed`。

所以，A2A 并不是简单地把远程 Agent 当成一个 HTTP 接口调用，而是把这类跨 Agent 协作建模成一个可跟踪、可暂停、可继续的任务。

**第五步：任务完成后返回 Artifact**

当 Burger Seller Agent 完成订单后，返回 Artifact。

```
{
  "artifactId": "receipt-001",
  "name": "burger-order-receipt",
  "parts": [
    {
      "kind": "data",
      "data": {
        "orderId": "B123",
        "items": ["4 cheeseburgers", "1 veggie burger"],
        "total": 78,
        "status": "confirmed"
      }
    }
  ]
}
```

如果从程序员角度看，A2A 落地可以总结成两句话：

```
服务端 Agent 要做： 
定义 AgentSkill / AgentCard，配置 TaskStore，实现 AgentExecutor，并借助 A2A SDK 提供的 A2A Server 将自己暴露成可远程调用的 Agent。

客户端 Agent 要做：
获取远程 Agent 地址，读取 Agent Card，根据 Agent Card 选择合适的远程 Agent，构造 Message，发送 A2A 请求，并处理返回的 Message / Task 状态 / Artifact。
```

这就是 A2A 的核心工程模型。

## A2A的具体实现

一个demo。

**server.py：Burger Seller Agent**

```
import os

import uvicorn

from a2a.helpers import (
    get_message_text,
    new_task_from_user_message,
    new_text_message,
    new_text_part,
)
from a2a.server.agent_execution import AgentExecutor, RequestContext
from a2a.server.events import EventQueue
from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.routes import create_agent_card_routes, create_jsonrpc_routes
from a2a.server.tasks import InMemoryTaskStore, TaskUpdater
from a2a.types import AgentCapabilities, AgentCard, AgentInterface, AgentSkill
from a2a.types.a2a_pb2 import TaskState
from starlette.applications import Starlette


A2A_HOST = os.getenv("A2A_HOST", "127.0.0.1")
A2A_PORT = int(os.getenv("A2A_PORT", "10001"))


class BurgerSeller:
    """真正的业务逻辑。

    A2A 只负责 Agent 之间如何发现、发消息、维护任务状态。
    至于“卖汉堡”这件事本身怎么实现，属于业务代码。
    """

    async def prepare_quote(self, user_request: str) -> str:
        """根据用户请求生成报价。

        这里简化为直接返回固定报价。
        真实项目里，这一步可能会查库存、算价格、调用支付/餐厅系统等。
        """
        return (
            "I can order 4 cheeseburgers and 1 veggie burger "
            "for 78 dollars. Please confirm."
        )

    async def place_order(self) -> str:
        """用户确认后真正下单。"""
        return (
            "Order confirmed. I placed the burger order and the restaurant "
            "will deliver it in about 30 minutes."
        )


class BurgerAgentExecutor(AgentExecutor):
    """A2A 服务端真正处理消息的入口。
    每次客户端发送 Message，A2A SDK 都会调用 execute()。
    """

    def __init__(self) -> None:
        self.seller = BurgerSeller()

    async def execute(
        self,
        context: RequestContext,
        event_queue: EventQueue,
    ) -> None:
        """处理一次 A2A message:send 请求。

        context 里包含本次请求的 Message、当前 Task、context_id 等信息。
        event_queue 是服务端向 A2A 框架发布 Task 变化的通道。
        """
        # 如果客户端传来了已有 task_id，A2A SDK 会从 TaskStore 中恢复 current_task。
        # 这就是“长任务可以继续”的关键：第二次确认不是新任务，而是同一个任务。
        if context.current_task:
            task = context.current_task
        else:
            # 第一次请求还没有 Task，需要从用户 Message 创建一个新的 Task。
            # new_task_from_user_message 会生成 task_id/context_id，并把用户消息放入 history。
            task = new_task_from_user_message(context.message)
            await event_queue.enqueue_event(task)

        # TaskUpdater 是 SDK 提供的辅助类，用来发布状态更新和 artifact。
        # 后面所有 WORKING、INPUT_REQUIRED、COMPLETED 都通过它写回同一个 Task。
        task_updater = TaskUpdater(
            event_queue=event_queue,
            task_id=task.id,
            context_id=task.context_id,
        )

        # A2A Message 可以有多个 part，这里只演示 text/plain，所以直接提取文本。
        user_text = get_message_text(context.message).strip()

        # 如果当前 Task 已经处于 INPUT_REQUIRED，说明上一次服务端在等待用户输入。
        # 此时这条新消息就应该被当作“用户确认/补充信息”，而不是新订单。
        if task.status.state == TaskState.TASK_STATE_INPUT_REQUIRED:
            await self._handle_confirmation(user_text, task_updater)
            return

        # 第一次处理订单：先把任务状态改成 WORKING，表示 Agent 正在处理。
        await task_updater.update_status(
            state=TaskState.TASK_STATE_WORKING,
            message=new_text_message("Burger Seller Agent is preparing a quote..."),
        )

        # 生成报价，并把报价作为 artifact 写入 Task。
        # artifact 更像任务产物；status.message 更像当前状态说明。
        quote = await self.seller.prepare_quote(user_text)
        await task_updater.add_artifact(
            parts=[
                new_text_part(
                    text=quote,
                    media_type="text/plain",
                )
            ]
        )

        # INPUT_REQUIRED 表示任务被中断/暂停，正在等待用户继续提供输入。
        # 客户端拿到这个状态后，可以提示用户输入 confirm，再带着同一个 task_id 续发消息。
        await task_updater.requires_input(
            message=new_text_message("Please confirm if you want me to place this order."),
        )

    async def _handle_confirmation(
        self,
        user_text: str,
        task_updater: TaskUpdater,
    ) -> None:
        """处理 INPUT_REQUIRED 状态下用户发来的确认消息。"""
        if user_text.lower() not in {"confirm", "yes", "y", "ok", "sure"}:
            await task_updater.requires_input(
                message=new_text_message(
                    "Please reply with 'confirm' to place the order."
                ),
            )
            return

        # 用户确认后，任务从 INPUT_REQUIRED 回到 WORKING。
        await task_updater.update_status(
            state=TaskState.TASK_STATE_WORKING,
            message=new_text_message("Confirmation received. Placing the order..."),
        )

        # 执行真正的下单动作，并把最终结果追加成新的 artifact。
        # 所以客户端最终会看到两个 artifact：报价结果 + 下单结果。
        result = await self.seller.place_order()
        await task_updater.add_artifact(
            parts=[
                new_text_part(
                    text=result,
                    media_type="text/plain",
                )
            ]
        )

        # 任务完成后进入终态 COMPLETED。
        # 进入终态后，这个 Task 就不应该再继续处理新的业务步骤。
        await task_updater.update_status(
            state=TaskState.TASK_STATE_COMPLETED,
            message=new_text_message("Order task is completed."),
        )

    async def cancel(
        self,
        context: RequestContext,
        event_queue: EventQueue,
    ) -> None:
        # 这里暂时没有实现取消逻辑。
        raise NotImplementedError("Cancel is not supported.")


def build_app() -> Starlette:
    """创建 A2A HTTP 服务。

    这里使用 Starlette 承载 A2A SDK 生成的路由。
    对外暴露两类能力：
    1. Agent Card 路由：让客户端发现这个 Agent 是谁、会什么、怎么访问。
    2. JSON-RPC 路由：让客户端发送 A2A message:send 请求。
    """
    # AgentSkill 描述这个 Agent 的一个具体能力。
    # 客户端或调度系统可以通过 Skill 判断是否应该把某类任务发给这个 Agent。
    sell_burger_skill = AgentSkill(
        id="sell_burger",
        name="Sell Burger",
        description="Handle burger ordering requests with manual confirmation",
        input_modes=["text/plain"],
        output_modes=["text/plain"],
        tags=["burger", "food", "order"],
        examples=[
            "I want a cheeseburger",
            "Order lunch for the team",
            "Confirm the burger order",
        ],
    )

    # AgentCard 是 A2A 的“名片”。
    # 客户端通常先读取 AgentCard，再根据里面的协议地址、输入输出类型、能力描述来创建客户端。
    public_agent_card = AgentCard(
        name="Burger Seller Agent",
        description="A seller agent that quotes and confirms burger purchase requests.",
        version="1.0.0",
        default_input_modes=["text/plain"],
        default_output_modes=["text/plain"],
        capabilities=AgentCapabilities(streaming=False),
        supported_interfaces=[
            AgentInterface(
                protocol_binding="JSONRPC",
                url=f"http://{A2A_HOST}:{A2A_PORT}",
            )
        ],
        skills=[sell_burger_skill],
    )

    # RequestHandler 把协议层和业务执行器串起来。
    # InMemoryTaskStore 用内存保存 Task，所以服务端重启后历史任务会丢失。
    # 真实系统里通常会换成数据库或持久化 TaskStore。
    request_handler = DefaultRequestHandler(
        agent_executor=BurgerAgentExecutor(),
        task_store=InMemoryTaskStore(),
        agent_card=public_agent_card,
    )

    # create_agent_card_routes 暴露 /.well-known/agent-card.json 等发现路由。
    # create_jsonrpc_routes 暴露 JSON-RPC 请求入口，本 demo 绑定在根路径 "/"。
    routes = []
    routes.extend(create_agent_card_routes(public_agent_card))
    routes.extend(create_jsonrpc_routes(request_handler, "/"))

    return Starlette(routes=routes)


if __name__ == "__main__":
    app = build_app()
    uvicorn.run(app, host=A2A_HOST, port=A2A_PORT)

```

![image-20260614222103909](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260614222104209.png)

**client.py：Purchasing Agent**

```
import asyncio
import os

import httpx
from a2a.client import A2ACardResolver, ClientConfig, create_client
from a2a.helpers import new_text_message
from a2a.types.a2a_pb2 import Role, SendMessageRequest, StreamResponse, Task, TaskState


A2A_HOST = os.getenv("A2A_HOST", "127.0.0.1")
A2A_PORT = int(os.getenv("A2A_PORT", "10001"))
BURGER_AGENT_URL = f"http://{A2A_HOST}:{A2A_PORT}"


async def get_agent_card():
    """读取远程 Agent Card"""
    async with httpx.AsyncClient() as httpx_client:
        resolver = A2ACardResolver(
            httpx_client=httpx_client,
            base_url=BURGER_AGENT_URL,
        )
        return await resolver.get_agent_card()


def get_artifact_text(task: Task) -> list[str]:
    """从 Task 的 artifacts 中提取文本结果。"""
    texts = []
    for artifact in task.artifacts:
        for part in artifact.parts:
            if part.text:
                texts.append(part.text)
    return texts


def print_task_summary(task: Task) -> None:
    """
    SDK 返回的原始 protobuf 对象信息很完整，但有些繁杂。
    这里提取 task_id、context_id、状态、状态消息和 artifacts，方便观察长任务流转。
    """
    state_name = TaskState.Name(task.status.state)
    print(f"\nTask: {task.id}")
    print(f"Context: {task.context_id}")
    print(f"State: {state_name}")

    # status.message 表示“当前 Task 状态附带的一句话说明”。
    if task.status.message.parts:
        print("Status message:")
        for part in task.status.message.parts:
            if part.text:
                print(f"  {part.text}")

    # artifacts 表示“任务已经产生的结果”。
    # 第一次请求会产生报价 artifact；确认后会追加下单成功 artifact。
    artifact_texts = get_artifact_text(task)
    if artifact_texts:
        print("Artifacts:")
        for text in artifact_texts:
            print(f"  {text}")


async def send_message(
    client,
    user_text: str,
    task_id: str | None = None,
    context_id: str | None = None,
) -> Task:
    """向 A2A Agent 发送一条用户消息，并返回服务端给出的 Task。

    第一次发送消息时不传 task_id/context_id，服务端会创建新 Task。
    当服务端返回 INPUT_REQUIRED 后，第二次发送确认消息时必须传回同一个
    task_id/context_id，这样服务端才知道是在继续同一个长任务。
    """
    message = new_text_message(
        text=user_text,
        role=Role.ROLE_USER,
        task_id=task_id,
        context_id=context_id,
    )
    request = SendMessageRequest(message=message)

    # client.send_message 返回的是一个异步迭代器。
    # 即使当前配置是非流式，SDK 也会把结果包装成 StreamResponse，保持统一接口。
    last_task = None
    async for chunk in client.send_message(request):
        if isinstance(chunk, Task):
            last_task = chunk
        elif isinstance(chunk, StreamResponse) and chunk.HasField("task"):
            last_task = chunk.task

    if last_task is None:
        raise RuntimeError("The agent did not return a task.")

    return last_task


async def send_order_message(user_text: str):
    """完整演示一次“报价 -> 等待确认 -> 继续完成”的 A2A 长任务。"""
    agent_card = await get_agent_card()

    print("Read remote Agent Card:")
    print(f"- name: {agent_card.name}")
    print(f"- description: {agent_card.description}")

    # 服务端每次请求返回一个聚合后的 Task，而不是持续推送事件流。
    config = ClientConfig(streaming=False)
    client = await create_client(
        agent=agent_card,
        client_config=config,
    )

    try:
        # 第一次发送订单请求。
        # 服务端会创建 Task，生成报价 artifact，然后把状态置为 INPUT_REQUIRED。
        print("\nSending A2A message:")
        print(user_text)
        task = await send_message(client, user_text)
        print_task_summary(task)

        # 只要服务端说 INPUT_REQUIRED，客户端就应该把控制权交还给用户。
        # 用户输入 confirm 后，客户端带着同一个 task_id/context_id 继续发消息。
        while task.status.state == TaskState.TASK_STATE_INPUT_REQUIRED:
            confirmation = input("\nType 'confirm' to continue this A2A task: ").strip()
            task = await send_message(
                client,
                confirmation,
                task_id=task.id,
                context_id=task.context_id,
            )
            print_task_summary(task)
    finally:
        # 显式关闭客户端底层 httpx 连接。
        await client.close()


async def main():
    # 这个文本就是用户的自然语言任务请求。
    user_text = (
        "Order lunch for the team. "
        "people=5, budget=80, dietaryRestrictions=no pork and one vegetarian."
    )

    await send_order_message(user_text)


if __name__ == "__main__":
    asyncio.run(main())

```

![image-20260614225104944](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260614225105142.png)

# 三、MCP

MCP全称**Model Context Protocol**，由Anthropic在2024年11月推出，它的思路很简单：定义一个统一标准，让所有AI应用用同一种方式连接所有工具。就像USB-C统一了充电接口一样。

MCP和A2A一样采用C/S架构，中间用 **JSON-RPC 2.0** 协议通信，这是一个已经被广泛验证的成熟技术，上文已经提过，这里就不做赘述了。

```
┌───────────────────────────────────────┐
│              AI 应用  	 			   │  ← MCP Client
└───────────────────────────────────────┘
                  │
                  │  MCP 协议 (JSON-RPC 2.0)
                  │
     ┌────────────┼───────────────┐
     ▼            ▼               ▼
┌────────────┐ ┌────────────┐ ┌────────────┐
│ Google     │ │ 数据库      │ │ Slack      │  ← MCP Servers
│ Drive      │ │ Server     │ │ Server     │
└────────────┘ └────────────┘ └────────────┘
```

## MCP的基础概念

**MCP client**

`MCP Client`，比如cursor，它是 MCP 架构中的关键组件，主要负责和 MCP 服务器建立连接并进行通信。它能自动匹配服务器的协议版本，确认可用功能，并负责数据传输和 JSON-RPC 交互。主要用于调用**远程**或者**本地**资源，远程：高德地图，本地：数据库、表等。

主要负责：

1. 接收来自LLM的请求
2. 将请求转发到相应的MCP server
3. 将MCP server的结果返回给LLM

**MCP Server**

`MCP Server`，是整个 MCP 架构的核心部分，主要用来为客户端提供各种工具、资源和功能支持。它负责处理客户端的请求，包括解析协议、提供工具、管理资源以及处理各种交互信息。他的本质是运行在电脑上的一个nodejs或python程序。

![](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260614210727413.png)

主要暴露三类东西：

| 概念      | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Tools     | 模型可以调用的函数 / 操作（如「搜索数据库」「发送邮件」）    |
| Resources | 应用程序可以读取的数据源（如文件、数据库记录）               |
| Prompts   | 预定义的提示模板（如「用这种格式总结文档」）。用户或 Host 可以直接调用这些模板。 |

## MCP的工作流程

别人的图

![6cdc0672d18e299f2dffe117e717ffca](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260614210851501.png)

![c7ef289d8f65f0609b77f708f3cd8bae](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260614210958901.png)

## MCP的具体实现

下面是一个简单的demo，场景如下：一个本地智能舆情分析系统，通过nlp与多工具协作，实现用户查询意图的自动理解、新闻检索、情绪分析、结构化输出与邮件推送。

**Server.py**

```
import json
import os
import smtplib
from datetime import datetime
from email.message import EmailMessage
from pathlib import Path

import httpx
from dotenv import load_dotenv
from mcp.server.fastmcp import FastMCP
from openai import OpenAI


BASE_DIR = Path(__file__).resolve().parent
GOOGLE_NEWS_DIR = BASE_DIR / "google_news"
SENTIMENT_REPORTS_DIR = BASE_DIR / "sentiment_reports"

# 加载环境变量。
load_dotenv()

# 初始化 MCP 服务器
mcp = FastMCP("NewsServer")


def _safe_filename(filename: str, default_prefix: str, suffix: str) -> str:
    """把模型或用户给出的文件名清理成 Windows/Linux 都安全的文件名。"""
    filename = filename.strip()
    if not filename:
        filename = f"{default_prefix}_{datetime.now().strftime('%Y%m%d_%H%M%S')}{suffix}"

    filename = os.path.basename(filename)
    filename = "".join(ch for ch in filename if ch not in '\\/:*?"<>|')

    if not filename.endswith(suffix):
        filename = f"{filename}{suffix}"

    return filename


def _resolve_report_path(filename: str = "", attachment_path: str = "") -> Path:
    """把邮件工具收到的文件名或路径解析成真实报告路径。"""
    raw_path = attachment_path or filename
    if not raw_path:
        raise ValueError("filename 或 attachment_path 不能为空")

    path = Path(raw_path)
    if path.is_absolute():
        return path

    # 如果传入的是 sentiment_reports/xxx.md 这种相对路径，就按 MCP 目录解析。
    candidate = BASE_DIR / path
    if candidate.exists():
        return candidate

    # 如果只传入 xxx.md，就默认它在 sentiment_reports 目录下。
    return SENTIMENT_REPORTS_DIR / path.name


@mcp.tool()
async def search_google_news(keyword: str) -> str:
    """使用 Serper API 搜索 Google News，并返回前 5 条新闻。

    参数:
        keyword: 新闻搜索关键词，例如“小米汽车”。

    返回:
        JSON 字符串，包含标题、摘要和链接；同时会在本地保存一份 JSON 文件。
    """
    api_key = os.getenv("SERPER_API_KEY")
    if not api_key:
        return "未配置 SERPER_API_KEY，请在 .env 文件中设置。"

    url = "https://google.serper.dev/news"
    headers = {
        "X-API-KEY": api_key,
        "Content-Type": "application/json",
    }
    payload = {"q": keyword}

    async with httpx.AsyncClient() as client:
        response = await client.post(url, headers=headers, json=payload)
        response.raise_for_status()
        data = response.json()

    if "news" not in data:
        return "未获取到搜索结果。"

    articles = [
        {
            "title": item.get("title"),
            "desc": item.get("snippet"),
            "url": item.get("link"),
        }
        for item in data["news"][:5]
    ]

    GOOGLE_NEWS_DIR.mkdir(parents=True, exist_ok=True)
    filename = f"google_news_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
    file_path = GOOGLE_NEWS_DIR / filename

    file_path.write_text(
        json.dumps(articles, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )

    return (
        f"已获取与 [{keyword}] 相关的前 5 条 Google 新闻：\n"
        f"{json.dumps(articles, ensure_ascii=False, indent=2)}\n"
        f"已保存到：{file_path}"
    )


@mcp.tool()
async def analyze_sentiment(text: str, filename: str = "") -> str:
    """对新闻文本做舆情分析，并保存成 Markdown 报告。

    参数:
        text: 新闻内容或搜索工具返回的新闻 JSON。
        filename: 希望保存的 Markdown 文件名。可以不传，不传时自动生成。

    返回:
        报告文件的完整路径。后续 send_email_with_attachment 可以直接使用这个路径。
    """
    openai_key = os.getenv("DASHSCOPE_API_KEY")
    model = os.getenv("MODEL")
    base_url = os.getenv("BASE_URL")

    if not openai_key:
        return "未配置 DASHSCOPE_API_KEY，请在 .env 文件中设置。"
    if not model:
        return "未配置 MODEL，请在 .env 文件中设置。"

    client = OpenAI(api_key=openai_key, base_url=base_url)

    prompt = (
        "请对以下新闻内容进行舆情分析，要求包含：\n"
        "1. 总体情绪倾向\n"
        "2. 正面因素\n"
        "3. 负面或风险因素\n"
        "4. 对企业或品牌的影响\n"
        "5. 简短结论\n\n"
        f"{text}"
    )

    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
    )
    result = response.choices[0].message.content.strip()

    markdown = f"""# 舆情分析报告

**分析时间：** {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

---

## 原始新闻材料

{text}

---

## 分析结果

{result}
"""

    SENTIMENT_REPORTS_DIR.mkdir(parents=True, exist_ok=True)

    # 不管 filename 是否为空，都必须生成 file_path。
    safe_name = _safe_filename(filename, "sentiment", ".md")
    file_path = SENTIMENT_REPORTS_DIR / safe_name
    file_path.write_text(markdown, encoding="utf-8")

    return str(file_path)


@mcp.tool()
async def send_email_with_attachment(
    to: str,
    subject: str,
    body: str,
    filename: str = "",
    attachment_path: str = "",
) -> str:
    """发送带附件的邮件。

    参数:
        to: 收件人邮箱。
        subject: 邮件标题。
        body: 邮件正文。
        filename: sentiment_reports 目录下的报告文件名。
        attachment_path: 报告文件完整路径。若传了这个参数，会优先使用它。

    返回:
        邮件发送结果说明。
    """
    smtp_server = os.getenv("SMTP_SERVER")
    smtp_port = int(os.getenv("SMTP_PORT", "465"))
    sender_email = os.getenv("EMAIL_USER")
    sender_pass = os.getenv("EMAIL_PASS")

    if not smtp_server or not sender_email or not sender_pass:
        return "SMTP 配置不完整，请检查 SMTP_SERVER、EMAIL_USER、EMAIL_PASS。"

    try:
        full_path = _resolve_report_path(filename=filename, attachment_path=attachment_path)
    except ValueError as exc:
        return f"附件参数无效：{exc}"

    if not full_path.exists():
        return f"附件路径无效，未找到文件：{full_path}"

    msg = EmailMessage()
    msg["Subject"] = subject
    msg["From"] = sender_email
    msg["To"] = to
    msg.set_content(body)

    try:
        file_data = full_path.read_bytes()
        msg.add_attachment(
            file_data,
            maintype="application",
            subtype="octet-stream",
            filename=full_path.name,
        )
    except Exception as exc:
        return f"附件读取失败：{exc}"

    try:
        with smtplib.SMTP_SSL(smtp_server, smtp_port) as server:
            server.login(sender_email, sender_pass)
            server.send_message(msg)
        return f"邮件已成功发送给 {to}，附件路径：{full_path}"
    except Exception as exc:
        return f"邮件发送失败：{exc}"


if __name__ == "__main__":
    mcp.run(transport="stdio")

```

**Client.py**

整体链路：`用户输入 → Client 调用大模型规划工具流程 → Client 依次调用 MCP Server 工具 → 收集工具结果再丢给大模型生成最终回答。`

```
import asyncio
import os
import json
from typing import Optional, List
from contextlib import AsyncExitStack
from datetime import datetime
import re
from openai import OpenAI
from dotenv import load_dotenv
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

load_dotenv()

class MCPClient:
	# 基础依赖层：统一管理全局资源、大模型客户端、资源自动回收栈
    def __init__(self):
        self.exit_stack = AsyncExitStack()  # 异步资源管理器（核心）
        self.openai_api_key = os.getenv("DASHSCOPE_API_KEY") # 读取.env大模型配置
        self.base_url = os.getenv("BASE_URL")
        self.model = os.getenv("MODEL")
        if not self.openai_api_key:
            raise ValueError("❌ 未找到 OpenAI API Key，请在 .env 文件中设置 DASHSCOPE_API_KEY")
        self.client = OpenAI(api_key=self.openai_api_key, base_url=self.base_url)  # 初始化大模型客户端
        self.session: Optional[ClientSession] = None # MCP会话句柄
	
	# MCP 服务连接层：启动 MCP 服务子进程、建立双向 stdio 通信、初始化 MCP 会话、拉取服务端可用工具列表
    async def connect_to_server(self, server_script_path: str):
        # 对服务器脚本进行判断，只允许是 .py 或 .js
        is_python = server_script_path.endswith('.py')
        is_js = server_script_path.endswith('.js')
        if not (is_python or is_js):
            raise ValueError("服务器脚本必须是 .py 或 .js 文件")

        # 确定启动命令，.py 用 python，.js 用 node
        command = "python" if is_python else "node"

        # 构造 MCP 所需的服务器参数StdioServerParameters，包含启动命令、脚本路径参数、环境变量（为 None 表示默认）
        server_params = StdioServerParameters(command=command, args=[server_script_path], env=None)

        # 启动 MCP 工具服务进程，stdio_client 创建 stdio 双向传输通信通道（子进程 stdin 写、stdout 读）
        stdio_transport = await self.exit_stack.enter_async_context(stdio_client(server_params))

        # 拆包通信通道：self.stdio(读流), self.write(写流)。作为 MCP 会话底层通信载体；
        self.stdio, self.write = stdio_transport

        # 创建 ClientSession（MCP 客户端会话对象，封装协议方法：list_tools/call_tool）
        self.session = await self.exit_stack.enter_async_context(ClientSession(self.stdio, self.write))

        # 初始化会话，执行 MCP 握手协议，建立正式通信；
        await self.session.initialize()

        # list_tools() 拉取服务端所有工具，缓存下来供后续规划使用。
        response = await self.session.list_tools()
        tools = response.tools
        print("\n已连接到服务器，支持以下工具:", [tool.name for tool in tools])
        
        # 总的来说，底层通信模型就是：
        # MCPClient(主程序) <==stdio管道==> server.py(工具服务进程)
        # write → 往子进程stdin发请求（调用工具、查询工具列表）
        # stdio → 读取子进程stdout返回结果
        # ClientSession 封装了这套管道的MCP协议报文序列化/反序列化，不用手动拼JSON


	# 工具链规划模块（自研核心逻辑，不属于原生 MCP）
	# 设计目的：减少多轮 LLM 调用开销，一次性规划完整任务流程，适合固定流水线任务（情感分析→生成报告→发邮件）。
	async def plan_tool_usage(self, query: str, tools: List[dict]) -> List[dict]:
        # 构造系统提示词 system_prompt。
        # 将所有可用工具组织为文本列表插入提示中，并明确指出工具名，
        # 限定返回格式是 JSON，防止其输出错误格式的数据。
        print("\n📤 提交给大模型的工具定义:")
        print(json.dumps(tools, ensure_ascii=False, indent=2))
        tool_list_text = "\n".join([
            f"- {tool['function']['name']}: {tool['function']['description']}"
            for tool in tools
        ])
        system_prompt = {
            "role": "system",
            "content": (
                "你是一个智能任务规划助手，用户会给出一句自然语言请求。\n"
                "你只能从以下工具中选择（严格使用工具名称）：\n"
                f"{tool_list_text}\n"
                "如果多个工具需要串联，后续步骤中可以使用 {{上一步工具名}} 占位。\n"
                "返回格式：JSON 数组，每个对象包含 name 和 arguments 字段。\n"
                "不要返回自然语言，不要使用未列出的工具名。"
            )
        }

        # 构造对话上下文并调用模型。
        # 将系统提示和用户的自然语言一起作为消息输入，并选用当前的模型。
        planning_messages = [
            system_prompt,
            {"role": "user", "content": query}
        ]

        response = self.client.chat.completions.create(
            model=self.model,
            messages=planning_messages,
            tools=tools,
            tool_choice="none"
        )

        # 提取出模型返回的 JSON 内容
        content = response.choices[0].message.content.strip()
        match = re.search(r"```(?:json)?\\s*([\s\S]+?)\\s*```", content)
        if match:
            json_text = match.group(1)
        else:
            json_text = content

        # 在解析 JSON 之后返回调用计划
        try:
            plan = json.loads(json_text)
            return plan if isinstance(plan, list) else []
        except Exception as e:
            print(f"❌ 工具调用链规划失败: {e}\n原始返回: {content}")
            return []

    async def cleanup(self):
        await self.exit_stack.aclose()
        
        
	# 业务主逻辑，整条流水线：
	# 用户输入 → 生成报告文件名 → LLM 规划工具链 → 循环执行工具 → 汇总结果给 LLM 生成最终回答 → 本地保存对话日志
    async def process_query(self, query: str) -> str:
        # 准备初始消息和获取工具列表
        messages = [{"role": "user", "content": query}]
        response = await self.session.list_tools()

        available_tools = [
            {
                "type": "function",
                "function": {
                    "name": tool.name,
                    "description": tool.description,
                    "input_schema": tool.inputSchema
                }
            } for tool in response.tools
        ]

        # 提取问题的关键词，对文件名进行生成。
        # 在接收到用户提问后就应该生成出最后输出的 md 文档的文件名，
        # 因为导出时若再生成文件名会导致部分组件无法识别该名称。
        keyword_match = re.search(r'(关于|分析|查询|搜索|查看)([^的\s，。、？\n]+)', query)
        keyword = keyword_match.group(2) if keyword_match else "分析对象"
        safe_keyword = re.sub(r'[\\/:*?"<>|]', '', keyword)[:20]
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        md_filename = f"sentiment_{safe_keyword}_{timestamp}.md"
        md_path = os.path.join("./sentiment_reports", md_filename)

        # 更新查询，将文件名添加到原始查询中，使大模型在调用工具链时可以识别到该信息
        # 然后调用 plan_tool_usage 获取工具调用计划
        query = query.strip() + f" [md_filename={md_filename}] [md_path={md_path}]"
        messages = [{"role": "user", "content": query}]

        tool_plan = await self.plan_tool_usage(query, available_tools)

        tool_outputs = {}
        messages = [{"role": "user", "content": query}]

        # 依次执行工具调用，并收集结果
        for step in tool_plan:
            tool_name = step["name"]
            tool_args = step["arguments"]

            for key, val in tool_args.items():
                if isinstance(val, str) and val.startswith("{{") and val.endswith("}}"):
                    ref_key = val.strip("{} ")
                    resolved_val = tool_outputs.get(ref_key, val)
                    tool_args[key] = resolved_val

            # 注入统一的文件名或路径（用于分析和邮件）
            if tool_name == "analyze_sentiment" and "filename" not in tool_args:
                tool_args["filename"] = md_filename
            if (
                tool_name == "send_email_with_attachment"
                and "filename" not in tool_args
                and "attachment_path" not in tool_args
            ):
                tool_args["attachment_path"] = md_path

            result = await self.session.call_tool(tool_name, tool_args)

            tool_outputs[tool_name] = result.content[0].text
            messages.append({
                "role": "tool",
                "tool_call_id": tool_name,
                "content": result.content[0].text
            })

        # 调用大模型生成回复信息，并输出保存结果
        final_response = self.client.chat.completions.create(
            model=self.model,
            messages=messages
        )
        final_output = final_response.choices[0].message.content

        # 对辅助函数进行定义，目的是把文本清理成合法的文件名
        def clean_filename(text: str) -> str:
            text = text.strip()
            text = re.sub(r'[\\/:*?\"<>|]', '', text)
            return text[:50]

        # 使用清理函数处理用户查询，生成用于文件命名的前缀，并添加时间戳、设置输出目录
        # 最后构建出完整的文件路径用于保存记录
        safe_filename = clean_filename(query)
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        filename = f"{safe_filename}_{timestamp}.txt"
        output_dir = "./llm_outputs"
        os.makedirs(output_dir, exist_ok=True)
        file_path = os.path.join(output_dir, filename)

        # 将对话内容写入 md 文档，其中包含用户的原始提问以及模型的最终回复结果
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(f"🗣 用户提问：{query}\n\n")
            f.write(f"🤖 模型回复：\n{final_output}\n")

        print(f"📄 对话记录已保存为：{file_path}")

        return final_output

	# 交互入口
    async def chat_loop(self):
        # 初始化提示信息
        print("\n🤖 MCP 客户端已启动！输入 'quit' 退出")

        # 进入主循环中等待用户输入
        while True:
            try:
                query = input("\n你: ").strip()
                if query.lower() == 'quit':
                    break

                # 处理用户的提问，并返回结果
                response = await self.process_query(query)
                print(f"\n🤖 AI: {response}")

            except Exception as e:
                print(f"\n⚠️ 发生错误: {str(e)}")

    


async def main():
    server_script_path = "./server.py"
    client = MCPClient()
    try:
        await client.connect_to_server(server_script_path)
        await client.chat_loop()
    finally:
    	# 资源释放：关闭 AsyncExitStack，自动关闭 MCP 会话、杀掉 server 子进程，防止资源泄漏；
        await client.cleanup()


if __name__ == "__main__":
    asyncio.run(main())

```

![image-20260614235044516](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260614235044671.png)

![image-20260614235020234](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260614235020379.png)

## 配置现有的MCP

MCP现在的生态很丰富，下面以cluade code 为例说一下怎么配置一个自己写好的或者别人的MCP Server。

1. 本地Stdio MCP

   比如我本地已经写好了一个MCP server, 在项目目录里执行

   ```
   claude mcp add 你的server名字 --transport stdio -- "你的python路径" "你的mcp server位置"
   ```

   如果有环境变量需要配置，可以参考下面这个：

   ```
   claude mcp add news-server --transport stdio --scope project ^
   -e SERPER_API_KEY= ^
   -e DASHSCOPE_API_KEY= ^
   -e BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1 ^
   -e MODEL=qwen-plus ^
   -- "E:\Work\agent_study\.venv\Scripts\python.exe" "E:\Work\agent_study\MCP\server.py"
   ```

   这个是配到`.mcp.json`里的（项目共享级），如果想配到`.claude.json`下（项目私有级）或者`C:\Users\用户名\.claude.json`下（全局用户级），把`--scope`后的参数改成`loacl/user`即可。

   ![887ed8ac08e5b92b77459236854c0c44](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260615010310949.png)

   通过`claude mcp list`查看：

   ![image-20260615005017806](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260615005017987.png)

   当然，你可以下载别人的mcp，然后配置到你的cc里，以高德为例：

   ```
   claude mcp add amap-maps --transport stdio --scope project ^
   -e AMAP_MAPS_API_KEY= ^
   -- npx -y @amap/amap-maps-mcp-server
   ```

2. 远程 HTTP MCP

   先找到远程MCP地址，比如Notion的： https://mcp.notion.com/mcp

   在项目目录里执行：

   ```
   claude mcp add notion --transport http --scope project https://mcp.notion.com/mcp
   ```

   然后再cc里输入/mcp选择新配置的mcp进行认证就好了。

# 四、Skills

Skills是2026年Agent中最火的东西，也是目前 Agent 体系中最简单的一个模块。它本质上只是一个遵循特定内容规范的文件夹，里面存在了一些文件。可以这么理解：把某一类常见任务的处理方式按照你的偏好提前整理好，之后在合适的时候直接调用，相当于是扩展了 AI 的能力，给它一份说明书，让它能在对应场景下自动切换，调用对应的技能，完成相关的任务。

我最开始的时候觉得Skills就是提示词外置，因为那个时候MCP已经很强大了，但落地使用后明显能感觉到，Skills之于MCP确实是更加稳定。并且，凭借按需加载，通用 Agent 能够以较低成本适配不同垂直业务场景，变身专业 Agent。所以我觉得现在用LangGraph开发或者自己手写一些agent已经意义不大了，写Skills放进hermes，claude code里效果好还简单。

## Skills的基础概念

一个标准的 Skill 文件夹长这样：

```
your-skill-name/           # 技能文件夹
├── SKILL.md               # 必填: 核心指令文件, 告诉 AI 该怎么做
├── scripts/               # 可选: 可执行代码目录
│   └── process_data.py    # 示例: Python 处理脚本
├── references/             # 可选: 参考文档目录
│   └── api-guide.md       # 示例: API 接口说明文档
└── assets/                # 可选: 资源文件目录
    └── report-template.md # 示例: 报告生成模板
```

**SKILL.md**（必填核心文件）

这是整个 Skill 的灵魂，Agent 通过读取它来了解技能的作用和执行步骤。模型最先读取的就是这个文件，来判断是否要使用这个 Skill。

它由两部分组成：头部元数据 + 详细说明 / 步骤

- 命名要求：必须精确拼写为 `SKILL.md`。
- 内容结构：
  1. **YAML 前言（Frontmatter）**：位于文件最顶部，包含 `name` 和 `description`，用于告诉 Agent 何时触发该技能。
  2. **Markdown 指令主体**：用于告诉 Agent 触发后如何一步步执行任务。

**scripts/**（可选）

存放可执行的代码文件（如 Python、Bash 脚本等）。

使用场景：如果你的工作流包含复杂的数据处理或需要调用外部 API，可以写成脚本放在这里，在 SKILL.md 中让 Agent 运行它们。

**references/**（可选） 

存放给 AI 看的详细参考文档。 

使用场景：如果你有必要的API 文档或设计规范，不要直接塞进 SKILL.md 里，把它们放在这个目录下，在SKILL.md中提示 Agent “按需查阅”。

**assets/**（可选）

存放素材资源，AI 可以直接拿来使用的，比如模板、图片等。 

使用场景：如果你的 skill 需要输出特定格式的报告，可以把 Markdown 模板放在这里；或者前端生成技能中用到的字体、图标等。

## Skills的运行机制

Skill 采用**渐进式加载**，它的核心设计理念非常清晰：分阶段、按需加载。

Skill 采用三层加载模型：

```
1. 用户发送请求
   ↓
2. Level 1 总是加载：读取 SKILL.md 头部 YAML 简介
   ↓
3. 判断用户请求是否匹配该技能
   ├─ 不匹配 → 走常规对话流程，结束
   └─ 匹配 → 进入 Level 2
   ↓
4. Level 2 触发时加载：读取 SKILL.md 完整指令、执行步骤
   ↓
5. AI 按照文档步骤开始执行任务
   ├─ 任务不需要外部文件资源 → 直接输出最终结果
   └─ 任务需要读取文档 / 运行脚本 → 进入 Level 3
   ↓
6. Level 3 引用时按需加载：读取外部资源目录
   ├─ references/ 参考文档：文档全文加载进上下文，再生成结果
   └─ scripts/ 可执行脚本：运行代码，仅将运行结果传入上下文，代码本身不占用上下文
   ↓
   汇总所有信息，输出最终结果
```

**Level 1: 元数据**

内容：SKILL.md 的 YAML 前言（Frontmatter，即包含 name 和 description 的部分）。

加载时机：启动时，直接加载到 Agent 系统提示（System Prompt）中。

Token 成本：极低，大约～100 tokens/Skill。

作用：让 Agent 知道系统里有哪些 Skills 可用，以及在什么情境下该唤醒它们。

**Level 2: 核心指令**

内容：SKILL.md 的 Markdown 主体部分（即具体的步骤和规范）。

加载时机：当用户的请求命中了 Level 1 中的 description（描述）时。

加载方式：系统会在后台自动将其调入当前对话。

Token 成本：中等，一般 < 5,000 tokens。

作用：为 AI 提供完成当前任务的详细操作指南和最佳实践（比如 “先写大纲，再分段”）。

**Level 3: 其他文件**

内容：存放在 references/ 的详细文档、scripts/ 的可执行脚本、assets/ 的模板文件等额外资源。

加载时机：只有当 Level 2 的指令中明确要求引用、执行这些文件时。

加载方式与 Token 成本：

- 额外文档：通过读取命令进入上下文，Token 成本视实际文件大小而定。
- 可执行脚本：通过执行命令运行，脚本代码本身不进入上下文，只有执行后的输出结果才会消耗 Token。

作用：提供近乎无限制的扩展能力，避免了把几万字参考资料直接塞给 AI 导致的上下文污染和幻觉。

## Skills的设计模式

下面是Google总结的5种 Skill 设计模式，我觉得挺好的，看完之后自己动手开发 Skill 时能有个大概的思路。

![Image](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260616000241025.jpeg)

**工具封装器（Tool Wrapper）**

侧重于应用知识，为 Agent 提供关于特定代码库的按需上下文。

SKILL.md 中定义了匹配规则，系统会监听用户提示词里特定代码库的关键字，命中后从 references/ 目录动态加载相关文档。

适用场景：适合将团队的内部编码指南或特定框架最佳实践直接分发到开发者工作流中。

![image-20260616001501226](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260616001501449.png)

SKILL.md示例：

```
---
name: api-expert
description: FastAPI 开发的最佳实践。在构建、审查或调试FastAPI应用、REST API或Pydantic模型时使用。
metadata:
  pattern: tool-wrapper
  domain: fastapi
---

你是 FastAPI 开发领域的专家。请将以下规范应用于用户的代码或问题中。

## 核心规范

加载 'references/conventions.md'以获取完整的FastAPI最佳实践。

## 审查代码时
1. 加载规范参考文件
2. 逐条对照规范检查用户的代码
3. 对于每一处违反规范的地方，引用具体规则并提供修复建议。

## 编写代码时
1. 加载规范参考文件
2. 严格遵循每一项规范
3. 为所有函数签名添加类型注解
4. 使用 Annotated 风格进行依赖注入
```

**生成器(Generator)**

用于强制执行一致的输出。

它利用了两个可选的目录：assets/ 用于存放你的输出模板，而 references/ 用于存放风格指南。指令在这里充当项目经理的角色，它告诉 Agent 加载模板、阅读风格指南、向用户询问缺失的变量，并填充文档。

适用场景：适合生成可预测的 API 文档、标准化提交信息或搭建项目架构。

![image-20260616003018328](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260616003018574.png)

SKILL.md示例：

```
---
name: report-generator
description: 生成结构化的Markdown格式技术报告。当用户要求编写、创建或起草报告、摘要或分析文档时使用。
metadata:
  pattern: generator
  output-format: markdown
---

你是一个技术报告生成器。请严格遵循以下步骤：

步骤 1: 加载 'references/style-guide.md'获取语气和排版规则。
步骤 2: 加载 'assets/report-template.md'获取所需的输出结构。
步骤 3: 向用户询问填充模板所需的任何缺少信息：
- 主题或内容
- 关键发现或数据点
- 目标受众（技术人员、管理层、普通受众）

步骤 4: 遵循样式指南的规则填充模板。模板中的每个部分都必须出现在最终的输出中。

步骤 5: 将完成的报告作为单个 Markdown文档返回。
```

Skill 文件本身并不包含实际的排版或语法规则。它仅仅负责协调这些资产的检索，并强制 Agent 逐步去执行它们。

references/ 目录下通常存放如 style-guide.md 等指导性文件。它规定了报告的语气、术语标准或排版倾向。当团队需要调整文风时，只需替换此目录下的静态文本，无需修改 SKILL.md 中的任何代码指令。assets/ 通常存放预先 “挖好空” 的模板文件（如 report-template.md）。

大模型不再构思文章脉络，而是将收集到的事实组装进模板中，从而在系统层面上保障了交付物结构的一致。

**审查器(Reviewer)**

将 “检查什么” 与 “如何检查” 分离开来。

无需编写冗长的系统提示词来详细说明每一种场景的评分标准，而是将模块化的评分标准存储在 review-checklist.md 文件中。执行过程中，Agent 会加载此清单并有条理地对提交内容进行评分。

适用场景：适合自动化审计相关的流程。

![image-20260616150920507](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260616150920976.png)

SKILL.md示例：

```
---
name: code-reviewer
description: 审查 Python 代码的质量、风格和常见Bug。当用户提交代码以供审查、要求获取代码反馈或希望进行代码审计时使用。
metadata:
  pattern: reviewer
  severity-levels: error,warning，info
---

你是一个 Python 代码审查员。请严格遵循以下审查协议：

步骤 1: 加载 'references/review-checkkist.md'获取完整的审查标准。

步骤 2: 仔细阅读用户的代码。在进行点评之前，先理解其意图。

步骤 3: 将清单中的每条规则应用于代码。对于发现的每一个违规之处：
- 记录行号（或大致位置）
- 对严重程度进行分类: error（必须修复）、warning(建议修复)、info(供参考)
- 解释“为什么”这是一个问题, 而不仅仅是指出”哪里“错了
- 提供具体的修复建议及修改后的代码

步骤 4: 生成包含以下部分的结构化审查报告：
- **总结**: 代码的功能说明，整体质量评估
- **发现的问题**: 按严重程度分组（error优先，其次是warning，最后是info）
- **评分**: 1-10 分，并附带简短理由
```

**反转 (Inversion)** 

Agent 往往本能地想要立刻进行猜测并生成结果。 反转模式颠覆了这种动态逻辑。不再是用户驱动提示词而 Agent 去执行，而是由 Agent 扮演面试官（访谈者）的角色。 它会按顺序提出结构化问题，并等待你的回答，然后再进入下一个阶段。 

适用场景：适合需要大量前置信息的复杂任务。

![image-20260616152150095](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260616152150310.png)

SKILL.md示例：

````
---
name: project-planner
description: 规划新软件项目，在生成计划之前通过结构化的问题收集需求。当用户说“我想构建”、“帮我做计划”、“设计系统”或“开始新项目”时使用。
metadata:
  pattern: inversion
  interaction: multi-turn
---

你正在进行一次结构化的需求访谈。在所有阶段完成之前，绝对不要开始构建或设计。

## 阶段 1 – 问题发现（一次只问一个问题，等待每个回答）
按顺序提出以下问题。不要跳过任何一个
- Q1: “这个项目为用户解决了什么问题？”
- Q2: “主要用户是谁？他们的技术水平如何？”
- Q3: “预期的规模有多大？（每日用户量、数据量、请求频率）”

## 阶段 2 – 技术约束（仅在阶段 1 被被完全解答后进行)
- Q4: “你打算使用什么部署环境？”
- Q5: “你对技术栈有什么要求或偏好吗？”
- Q6: “有哪些不可妥协的要求？（延迟、正常运行时间、合规性、预算) ”

## 阶段 3 – 总结（仅在所有问题都被解答后进行)
1. 加载 'assets/plan-template.md'获取输出格式
2. 使用收集到的需求填充模板的每个部分
3. 向用户展示完整的计划
4. 询问: “这个计划准确地涵盖了你的需求吗？你希望修改哪些地方？
5. 根据反馈不断迭代, 直到用户确认为止
```
````

通过结构化的提问表单与严格的阶段控制，它能够有效引导那些不知道自己不知道什么的用户，共同推导出一份高质量的执行方案。

**流水线(Pipeline)**

流水线模式强制执行一个带有硬性检查点、严格按顺序执行的工作流。

适用场景：适合复杂的任务，无法承受遗漏步骤的常见。

![image-20260616165843654](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260616165844104.png)

SKILL.md示例：

```
---
name: doc-pipeline
description: 通过多步流水线从 Python 源代码生成 API 文档。当用户要求为模块编写文档、生成API文档或从代码创建文档时使用。
metadata:
  pattern: pipeline
  steps: "4"
---

你正在运行一个文档生成流水线。请严格请严格按顺序执行每个步骤。绝对不要跳过任何步骤，也不要在某个步骤失败时继续前进。

## 步骤 1 – 解析与盘点
分析用户的 Python 代码，提取所有公共的类、函数和常量。将盘点清单作为检查列表展示给用户，并询问：“这是您希望编写文档的完整公共API吗？”

## 步骤 2 – 生成文档字符串（Docstring）
对于每一个缺少文档字符串的函数:
- 加载 'references/docstring-style.md' 获取所需的格式规范
- 严格遵循样式指南生成文档字符串
- 向用户展示每个生成的文档字符串以供审核批准
**在用户确认之前，绝对不要进入步骤 3。**

## 步骤 3 – 组装文档
加载 'assets/api-doc-template.md'获取输出结构。将所有的类、函数和文档字符串编译整合进一个单一的API参考文档中。

## 步骤 4 – 质量检查
根据 'references/quality-checklis'进行复核:
- 确保每一个公共符号都已记录
- 确保每一个参数都包含类型和描述
- 确保每个函数至少提供一个使用示例
汇报检查结果。在展示最终文档之前，必须先修复发现的所有问题。
```

## Skills的自进化

一旦 Skill 变成了资产，它就不再只是提示工程的产物，而开始接近软件工程里的模块、知识工程里的工件、以及强化学习里的可复用策略单元。顺着这个思路，skill + 自进化开辟出了一条全新的 Agent 研究路线。Hermes Agent就是这条路上的代表性实现。

很多人会下意识地将 “Agent的自进化” 理解成 "它记住了用户说过的话"，但 Hermes 的"学习"远不止于此。它真正在做的是：**每次对话结束后，agent 会评估这次解决问题的过程，决定要不要把工作流程提炼成一个新的 skill 文件**。

先看下hermes里的SKILL.md的Frontmatter是怎么写的：

```
---
name: systematic-debugging
description: "4-phase root cause debugging: understand bugs before fixing."
version: 1.1.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
metadata:
  hermes:
    tags: [debugging, troubleshooting, problem-solving, root-cause, investigation]
    related_skills: [test-driven-development, writing-plans, subagent-driven-development]
---
```

name和description就不说了。

version：语义化版本。Hermes 会跟踪每个 skill 的版本变化。当一个 skill 被 agent 自动优化后，版本号会递增。用户可以通过 `hermes skills log` 看到每次修改的历史，甚至可以回滚到之前的版本。这个设计决定了它能支持演化而不是单纯的覆盖。

author：来源追溯。Hermes 不是从零写 skill，而是从已有的社区 skill 库适配过来的。这也说明skill 是可以跨项目共享的。一个人写的好 skill，另一个人可以 import 来用，就像 npm 包或者 pip 库一样。

metadata.hermes.tags：检索索引。当 agent 遇到一个问题，它不会"遍历所有 skills 看哪个 description 匹配"——那样太慢。它会先用关键词匹配 tags，快速缩小候选集，再读候选的 description，再决定加载哪个。

metadata.hermes.related_skills：技能图谱。这是整个 skill 系统里最未来感的字段。它声明"这个 skill 和哪些其他 skill 相关"。比如 `systematic-debugging` 的 related 包括 `test-driven-development` 和 `writing-plans`——意思是 做根因分析的时候，可能也需要用到 TDD 和写计划的技能。Agent 加载一个 skill 时，会顺便把它的 related_skills 的 description 也加载（但不加载正文）。这样 agent 就建立了一张技能关联图，在解决复杂问题时可以沿着图跳转。这是 Hermes 从"孤立的工具集"向"有知识图谱的能力体系"演进的关键设计。

SKILL.md的设计可以说是自进化的前提保障，下面看看Skill具体是怎么做自进化的：

```
_SKILL_REVIEW_PROMPT = (
    "Review the conversation above and consider saving or updating "
    "a skill if appropriate.\n\n"
    "Focus on: was a non-trivial approach used to complete a task that "
    "required trial and error, or changing course due to experiential "
    "findings along the way, or did the user expect or desire a "
    "different method or outcome?\n\n"
    "If a relevant skill already exists, update it with what you learned. "
    "Otherwise, create a new skill if the approach is reusable.\n"
    "If nothing is worth saving, just say 'Nothing to save.' and stop."
)
翻译一下:
审查上面这段对话，决定要不要保存或更新一个 skill。
关注：这次任务是不是用了"非凡常规"的方法？是不是经历了试错或中途调整路线？用户是不是期待或想要一种不同的做法/结果？
如果已有相关 skill，用学到的东西更新它。否则，如果这个做法可复用，就创建一个新 skill。
如果没什么值得保存的，就说 “Nothing to save.” 然后停下。
```

这是**整个自进化机制的核心**。每轮对话结束后，Hermes 会派一个 background review agent，让它读这次对话的完整 transcript，然后用上面这段 prompt 决定要不要动 skill 库。review agent 自己判断"什么算非凡常规的方法"，没有写死的阈值。

review agent的pipeline:

1. 摘取。Agent 回顾本轮对话的完整工具调用序列，标记出"导致成功的关键步骤"。
2. 抽象。把具体任务抽象成可复用模式。抽象是这个流水线最难的环节，需要 agent 有足够的元认知能力——“我刚才做的事情里，哪些是这个问题特有的，哪些是可以推广的”。Hermes 在这里依赖的是基础模型本身的泛化能力。弱一点的模型在这步会失败，生成的 skill 会过于具体没法复用。
3. 格式化。把抽象出来的方法论填进 agentskills.io 标准的 skill 模板：
   - 写 description（按三段式：触发条件 + 核心承诺 + 关键约束）
   - 选 tags（基于任务类型）
   - 标 related_skills（搜索已有 skills 看哪些相关）
   - 写正文（把 4 个阶段的方法论写成 Markdown）
4. 质量审核。这是最关键的一步。Agent 不会直接保存生成的 skill，而是启动一个子 agent 专门审核这个 skill 的质量。如果审核不过，生成流程回退到阶段 2 重新抽象。最多尝试 3 次，还不过就放弃——宁可不产生 skill，也不产生垃圾 skill。

# 五、小结

我还记得第一次面试的时候，mentor 跟我说：“AI 是第四次工业革命，也是普及速度最快的一次革命。” 事实也印证了这句话 —— 这几年，几乎每一年都在被称作Agentic AI 的元年。这几年里，AI从单纯的能聊天变成了能干活，而Agent、LLM、A2A、MCP、Skills这几个概念，就是支撑这个转变的基础设施。如果说LLM是大脑，那MCP是手，Skills是经验，A2A是语言，Agent就是把这一切整合起来的人。一个人，有手可以操作工具，有经验可以把事情做好，有语言可以和同事协作——这就是 Agentic AI 的完整图景。

它们的关系很像互联网早期的TCP、IP、HTTP、DNS一样——单看每一个都很单薄，但放在一起，就构成了一个让无数应用得以运行的底层生态。AI的下一幕，不是更聪明的模型，而是更好的协作基础设施。而我们正在见证这个基础设施从零开始搭建。

# 六、参考链接

Agent2Agent (A2A) Protocol

https://github.com/a2aproject/A2A

Agent2Agent (A2A) 协议使用入门：Cloud Run 和 Agent Engine 上的购买助理和远程销售代理互动

https://codelabs.developers.google.com/intro-a2a-purchasing-concierge

在矛盾知识库里看到的狄师哥写的MCP的文章，是我最开始学MCP的时候看的，写的很清晰

[‍⁠﻿⁠‬‬﻿‬‌‍‍⁠⁠‍‍⁠‌‍‌﻿‬‬‍﻿‬‬⁠MCP安全-MCP基础 - 飞书云文档](https://spear-shield.feishu.cn/wiki/QClbwJtHtiZtr7kLsdhcVfpjn6c?fromScene=spaceOverview)

Model Context Protocol servers

https://github.com/modelcontextprotocol/servers

5 Agent Skill design patterns every ADK developer should know

[https://x.com/GoogleCloudTech/status/2033953579824758855](https://tool.lu/en_US/article/7JA/url)

Hermes Agent ☤

https://github.com/NousResearch/hermes-agent
