# Step 01: Tools - Read, Write, Bash is Powerful Enough

Give the agent the ability to execute tools (read, write, edit, bash) and interact with the filesystem.

## Prerequisites

Same as Step 00 - copy the config file and add your API key:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Edit config.user.yaml to add your API key
```

## What We will Build?

### Architecture

```
User Input → ChatLoop → AgentSession → Agent → LLM
                              ↓              ↑
                         ToolRegistry ← Tool Calls
                              ↓
                         Tool Execution
```

### Key Components

- **Stop Reason**: Chat Loop can stop because of "end_turn" or "tool_use"
- **ToolRegistry**: Manages available tools and executes tool calls
- **Builtin Tools**: read, write, edit, bash
- **Tool Calling Loop**: Agent calls tools, adds results to history, continues conversation

## Key Changes

[src/tools/base.py](src/tools/base.py)

```python
class BaseTool(ABC):
    """Abstract base class for all tools."""

    name: str
    description: str
    parameters: dict[str, Any]

    @abstractmethod
    async def execute(self, session: "AgentSession", **kwargs: Any) -> str:
        """Execute the tool."""

    def get_tool_schema(self) -> dict[str, Any]:
        """Get the tool/function schema for LiteLLM."""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            },
        }
```

[src/core/agent.py](src/core/agent.py) - Tool calling loop

```python
async def chat(self, message: str) -> str:
    """Send a message with tool calling support."""
    user_msg: Message = {"role": "user", "content": message}
    self.state.add_message(user_msg)

    tool_schemas = self.tools.get_tool_schemas()

    while True:
        messages = self.state.build_messages()
        content, tool_calls = await self.agent.llm.chat(messages, tool_schemas)

        # Add assistant message with tool calls
        assistant_msg: Message = {
            "role": "assistant",
            "content": content,
            "tool_calls": [...],
        }
        self.state.add_message(assistant_msg)

        if not tool_calls:
            break

        # Execute tools and add results to history
        await self._handle_tool_calls(tool_calls)

    return content
```


## How to Run

```bash
cd 01-tools
uv run my-bot chat

# You: Hey Can you read your README.md please?
# pickle: I found and read the README.md file! 🐱

# # Step 01: Tools - Read, Write, Bash is Powerful Enough

# Give the agent the ability to execute tools (read, write, edit, bash) and interact with the filesystem.
# [More lines]
```

## What's Next

[Step 02: Skills](../02-skills/) - Add dynamic capability loading with SKILL.md files
