# Step 16: Concurrency Control

Per-agent concurrency limits using semaphores to prevent system overload.

## Prerequisites

Same as previous steps - copy the config file and add your API key:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Edit config.user.yaml to add your API key
```

## What We Built

### Architecture

```
Multiple InboundEvents for Agent A
              ↓
        Semaphore(2)
         ↙    ↘
    Task 1   Task 2   ← Running
              ↓
    Task 3   ← Waiting in queue
```

## Key Components

- **AgentDef.max_concurrency**: Configurable per-agent limit (default: 1)
- **AgentWorker._semaphores**: Dictionary of semaphores per agent
- **async with sem**: Blocks when concurrency limit reached



[src/mybot/core/agent_loader.py](src/mybot/core/agent_loader.py) - Add max_concurrency field

```python
class AgentDef(BaseModel):
    # ... existing fields ...
    max_concurrency: int = Field(default=1, ge=1)
```

[src/mybot/server/agent_worker.py](src/mybot/server/agent_worker.py) - Semaphore-based limiting

```python
class AgentWorker(SubscriberWorker):
    CLEANUP_THRESHOLD = 5

    def __init__(self, context: "SharedContext"):
        super().__init__(context)
        self._semaphores: dict[str, asyncio.Semaphore] = {}

    async def exec_session(self, event, agent_def: "AgentDef") -> None:
        sem = self._get_or_create_semaphore(agent_def)

        async with sem:  # Blocks if limit reached
            # ... execute session ...

        self._maybe_cleanup_semaphores(agent_def)

    def _get_or_create_semaphore(self, agent_def: "AgentDef") -> asyncio.Semaphore:
        if agent_def.id not in self._semaphores:
            self._semaphores[agent_def.id] = asyncio.Semaphore(
                agent_def.max_concurrency
            )
        return self._semaphores[agent_def.id]

    def _maybe_cleanup_semaphores(self, agent_def: "AgentDef") -> None:
        if agent_def.id in self._semaphores:
            if not self._semaphores[agent_def.id]._waiters:
                del self._semaphores[agent_def.id]
```

## How to Run

```bash
cd 16-concurrency-control
uv run my-bot chat

# Configure concurrency in AGENT.md frontmatter:
# ---
# name: my-agent
# max_concurrency: 3  # Allow 3 concurrent sessions
# ---
```

## Benefits

1. **Prevent Overload**: Limit concurrent LLM calls per agent
2. **Configurable**: Each agent can have different limits
3. **Memory Efficient**: Semaphores cleaned up when no waiters
4. **Fair Queuing**: asyncio.Semaphore ensures FIFO ordering

## What's Next

[Step 17: Memory](../17-memory/) - Long-term knowledge system
