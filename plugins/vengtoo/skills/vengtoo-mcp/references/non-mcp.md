# Vengtoo — Non-MCP Agent Authorization Reference

For LangChain, LangGraph, AutoGen, and custom agent loops.

---

## The pattern

Every tool invocation goes through a single `vengtoo_tool()` wrapper before executing:

```
agent decides to call tool → vengtoo_tool() checks → execute or raise
```

---

## Python — LangChain

### Custom tool decorator
```python
from vengtoo import Vengtoo
from langchain.tools import tool
from langchain_core.tools import ToolException
import os

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

def vengtoo_tool(agent_id: str):
    """Decorator that checks Vengtoo before running a LangChain tool."""
    def decorator(fn):
        @tool(fn.__name__, description=fn.__doc__ or "")
        def wrapper(**kwargs):
            resp = client.authorize({
                "subject": {"id": agent_id, "type": "agent"},
                "resource": {
                    "type": "tool",
                    "id": fn.__name__,
                    "properties": kwargs,
                },
                "action": {"name": "invoke"},
            })
            if not resp.decision:
                raise ToolException(f"Tool '{fn.__name__}' denied: {resp.context.reason}")
            return fn(**kwargs)
        return wrapper
    return decorator

AGENT_ID = "agent:research-bot"

@vengtoo_tool(AGENT_ID)
def search_web(query: str) -> str:
    """Search the web for information."""
    # ... actual implementation

@vengtoo_tool(AGENT_ID)
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email."""
    # ... actual implementation
```

### LangGraph node wrapper
```python
from langgraph.graph import StateGraph
from vengtoo import Vengtoo

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

def authorized_node(agent_id: str, tool_name: str, node_fn):
    """Wrap a LangGraph node with Vengtoo authorization."""
    def wrapper(state):
        resp = client.authorize({
            "subject": {"id": agent_id, "type": "agent"},
            "resource": {"type": "tool", "id": tool_name},
            "action": {"name": "invoke"},
        })
        if not resp.decision:
            return {**state, "error": f"Denied: {resp.context.reason}"}
        return node_fn(state)
    return wrapper

graph = StateGraph(State)
graph.add_node("send_email", authorized_node("agent:workflow", "send_email", send_email_node))
```

---

## Node.js — LangChain.js

```ts
import { DynamicTool } from '@langchain/core/tools'
import { Vengtoo } from '@vengtoo/sdk'

const client = new Vengtoo({ apiKey: process.env.VENGTOO_API_KEY! })
const AGENT_ID = 'agent:assistant'

function vengtooTool(name: string, description: string, fn: (input: string) => Promise<string>) {
  return new DynamicTool({
    name,
    description,
    func: async (input: string) => {
      const resp = await client.authorize({
        subject: { id: AGENT_ID, type: 'agent' },
        resource: { type: 'tool', id: name, properties: { input } },
        action: { name: 'invoke' },
      })
      if (!resp.decision) throw new Error(`Tool '${name}' denied: ${resp.context.reason}`)
      return fn(input)
    },
  })
}

const tools = [
  vengtooTool('search', 'Search the web', async (q) => searchWeb(q)),
  vengtooTool('write_file', 'Write a file to disk', async (args) => writeFile(JSON.parse(args))),
]
```

---

## Python — AutoGen

```python
import autogen
from vengtoo import Vengtoo

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

class VengtooAgent(autogen.AssistantAgent):
    def __init__(self, agent_id: str, **kwargs):
        super().__init__(**kwargs)
        self.agent_id = agent_id

    def generate_tool_call_output(self, tool_name: str, tool_input: dict) -> dict:
        resp = client.authorize({
            "subject": {"id": self.agent_id, "type": "agent"},
            "resource": {"type": "tool", "id": tool_name, "properties": tool_input},
            "action": {"name": "invoke"},
        })
        if not resp.decision:
            return {"error": f"Tool '{tool_name}' denied by policy: {resp.context.reason}"}
        return super().generate_tool_call_output(tool_name, tool_input)
```

---

## Custom agent loop (any language)

```python
class AuthorizedAgentLoop:
    """Intercepts tool calls in any agent loop."""

    def __init__(self, tools: dict, agent_id: str):
        self.tools = tools
        self.agent_id = agent_id
        self.vengtoo = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

    def call_tool(self, name: str, args: dict):
        # Check
        resp = self.vengtoo.authorize({
            "subject": {"id": self.agent_id, "type": "agent"},
            "resource": {"type": "tool", "id": name, "properties": args},
            "action": {"name": "invoke"},
        })
        if not resp.decision:
            raise PermissionError(f"Tool '{name}' denied: {resp.context.reason}")

        # Execute
        if name not in self.tools:
            raise ValueError(f"Unknown tool: {name}")
        return self.tools[name](**args)
```

---

## Batch pre-authorization (efficiency)

Before starting a task, pre-check all tools the agent may need:

```python
tool_names = ["search_web", "read_file", "send_email", "write_file"]

results = client.authorize_batch({
    "subject": {"id": agent_id, "type": "agent"},
    "items": [
        {"resource": {"type": "tool", "id": name}, "action": {"name": "invoke"}}
        for name in tool_names
    ]
})

allowed_tools = {
    tool_names[i]: results[i].decision
    for i in range(len(tool_names))
}

# Only expose allowed tools to the agent
available_tools = [t for t in all_tools if allowed_tools.get(t.name, False)]
```

---

## Human-in-the-loop escalation

When `reason_code == "INSUFFICIENT_PRIVILEGE"`, escalate to a human:

```python
resp = client.authorize({...})
if not resp.decision:
    if resp.context.reason_code == "INSUFFICIENT_PRIVILEGE":
        approved = await ask_human(
            f"Agent wants to call '{tool_name}'. Approve?",
            context=resp.context
        )
        if approved:
            result = tool_fn(**args)  # proceed after human approval
            return result
    raise PermissionError(resp.context.reason)
```

See `/vengtoo-ai-agents` for the full JIT and delegation pattern.
