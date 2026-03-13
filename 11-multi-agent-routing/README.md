# Step 11: Multi-Agent Routing - Right Agent for Right Job

Route incoming messages to specialized agents based on source patterns.

## What We Will Build

```
┌────────────────────────────────────────────────────────────────────┐
│                            Server                                   │
│                                                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │EventBus     │  │AgentWorker   │  │DeliveryWorker│             │
│  │             │  │              │  │              │             │
│  └─────────────┘  └──────────────┘  └──────────────┘             │
│         ▲                  ▲                  │                    │
│         │                  │                  │                    │
│         │            ┌─────┴─────┐            │                    │
│         │            │ Agent     │            │                    │
│         │            │ Session   │            │                    │
│         │            └───────────┘            │                    │
│         │                                     │                    │
│  ┌──────┴─────────────────────────────────────┴──────┐            │
│  │              RoutingTable                         │            │
│  │                                                   │            │
│  │  Bindings:                                        │            │
│  │  - "platform-telegram:.*" -> cookie-agent         │            │
│  │  - "platform-cli:.*" -> pickle-agent              │            │
│  │  - ".*" -> default-agent                          │            │
│  └───────────────────────────────────────────────────┘            │
│         ▲                                                         │
│         │                                                         │
│  ┌──────┴──────────────────────────────────────────────┐         │
│  │              ChannelWorker                           │         │
│  │                                                      │         │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │         │
│  │  │CLI      │  │Telegram │  │WebSocket│             │         │
│  │  │Channel  │  │Channel  │  │Channel  │             │         │
│  │  └─────────┘  └─────────┘  └─────────┘             │         │
│  └─────────────────────────────────────────────────────┘         │
└────────────────────────────────────────────────────────────────────┘
```

**Key Components:**
- **RoutingTable** - Routes sources to agents using regex bindings with tiered specificity
- **Binding** - A source pattern + agent mapping with automatic tier computation
- **AgentLoader.discover_agents()** - Scans agents directory to find all available agents
- **CLI Commands** - `routing`, `binding`, `agents` for managing multi-agent setup



### 1. RoutingTable ([src/mybot/core/routing.py](src/mybot/core/routing.py))

```python
@dataclass
class Binding:
    """A routing binding that matches sources to agents."""
    agent: str
    value: str
    tier: int  # 0=exact, 1=specific regex, 2=wildcard
    pattern: Pattern  # Compiled regex

@dataclass
class RoutingTable:
    """Routes sources to agents using regex bindings."""

    def resolve(self, source: str) -> str:
        """Return agent_id for source, falling back to default_agent."""
        for binding in self._load_bindings():
            if binding.pattern.match(source):
                return binding.agent
        return self.context.config.default_agent

    def get_or_create_session_id(self, source: EventSource) -> str:
        """Get existing or create new session_id for source."""
        # Check cache first (session affinity)
        source_session = self.context.config.sources.get(str(source))
        if source_session:
            return source_session.session_id

        # Resolve agent and create new session
        agent_id = self.resolve(str(source))
        agent_def = self.context.agent_loader.load(agent_id)
        agent = Agent(agent_def, self.context)
        session = agent.new_session(source)

        # Cache for future messages
        self.context.config.set_runtime(
            f"sources.{str(source)}", SourceSessionConfig(session_id=session.session_id)
        )
        return session.session_id
```

### 2. Binding Tiers ([src/mybot/core/routing.py](src/mybot/core/routing.py))

Bindings are sorted by specificity:
- **Tier 0**: Exact literal (no regex special chars) - highest priority
- **Tier 1**: Specific regex (anchors, character classes)
- **Tier 2**: Wildcard (`.`, `.*`) - lowest priority

```python
def _compute_tier(self) -> int:
    """Compute specificity tier."""
    if not any(c in self.value for c in r".*+?[]()|^$"):
        return 0  # Exact match
    if ".*" in self.value:
        return 2  # Wildcard
    return 1  # Specific regex
```

### 3. Agent Discovery ([src/mybot/core/agent_loader.py](src/mybot/core/agent_loader.py))

```python
class AgentLoader:
    def discover_agents(self) -> list[AgentDef]:
        """Scan agents directory and return list of valid AgentDef."""
        return discover_definitions(
            self.config.agents_path, "AGENT.md", self._parse_agent_def
        )
```

### 4. Config Updates ([src/mybot/utils/config.py](src/mybot/utils/config.py))

```python
class Config(BaseModel):
    # ... existing fields ...
    routing: dict[str, Any] = Field(default_factory=lambda: {"bindings": []})
```

### 5. ChannelWorker Integration ([src/mybot/server/channel_worker.py](src/mybot/server/channel_worker.py))

```python
async def callback(message: str, source: EventSource) -> None:
    # ... validation ...

    # Use routing_table to resolve agent from bindings
    session_id = self.context.routing_table.get_or_create_session_id(source)

    # Publish event
    event = InboundEvent(session_id=session_id, source=source, content=message)
    await self.context.eventbus.publish(event)
```

## How to Run

**List available agents:**
```bash
cd 11-multi-agent-routing
uv run my-bot agents list
```

**Show routing bindings:**
```bash
uv run my-bot routing list
```

**Add a routing binding:**
```bash
# Route all Telegram messages to cookie agent
uv run my-bot routing add "platform-telegram:.*" cookie

# Route specific CLI user to pickle agent
uv run my-bot binding set "platform-cli:alice" pickle
```

**Start server with routing:**
```bash
uv run my-bot server
```

Messages will be routed based on your bindings. Unmatched sources fall back to `default_agent`.

## CLI Commands

### `routing` - Manage Routing Bindings

```bash
my-bot routing list              # Show all bindings
my-bot routing add <pattern> <agent>  # Add binding
my-bot routing clear             # Remove all bindings
```

### `binding` - Quick Source Binding

```bash
my-bot binding set <source> <agent>   # Bind source to agent
my-bot binding unset <source>         # Remove binding
```

### `agents` - Agent Management

```bash
my-bot agents list              # List all agents
my-bot agents show <agent-id>   # Show agent details
```

## Architecture Notes

**Routing Flow:**
1. Message arrives at ChannelWorker
2. ChannelWorker calls `routing_table.get_or_create_session_id(source)`
3. RoutingTable checks cache for existing session (session affinity)
4. If no cache, RoutingTable resolves agent from bindings
5. New session created with resolved agent
6. Session cached for future messages from same source

**Binding Configuration:**
Bindings are stored in `config.user.yaml`:

```yaml
routing:
  bindings:
    - agent: cookie
      value: "platform-telegram:.*"
    - agent: pickle
      value: "platform-cli:alice"
```

**Session Affinity:**
- First message creates session with resolved agent
- Subsequent messages from same source use cached session
- Ensures conversation continuity
- Cache stored in `config.runtime.yaml`

## What's Next

Step 12 will add **Cron + Heartbeat** - scheduled tasks and health monitoring.
