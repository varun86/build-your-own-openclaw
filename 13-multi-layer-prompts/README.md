# Step 13: Multi-Layer Prompts

System prompts are now built from multiple layers for better organization and flexibility.

## Prerequisites

Same as previous steps - copy the config file and add your API key:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Edit config.user.yaml to add your API key
```

## What We Built

### Architecture

```
Agent Definition
├── AGENT.md (base prompt - capabilities, tools)
├── SOUL.md (personality layer - tone, style)
└── [MEMORY.md] (knowledge layer - user info)
      ↓
PromptBuilder
      ↓
Assembled System Prompt
├── Layer 1: Identity (agent_md)
├── Layer 2: Soul (soul_md)
├── Layer 3: Bootstrap (BOOTSTRAP.md + AGENTS.md + cron list)
├── Layer 4: Runtime (agent ID + timestamp)
└── Layer 5: Channel (platform hint)
```

### Key Components

- **AgentDef**: Now includes `soul_md` field for personality
- **AgentLoader**: Loads optional SOUL.md alongside AGENT.md
- **PromptBuilder**: Assembles all prompt layers into final system prompt

## Key Changes

[src/mybot/core/agent_loader.py](src/mybot/core/agent_loader.py) - Added soul_md field

```python
class AgentDef(BaseModel):
    """Loaded agent definition with merged settings."""

    id: str
    name: str
    description: str = ""
    agent_md: str
    soul_md: str = ""  # NEW: Personality layer (optional)
    llm: LLMConfig
    allow_skills: bool = False
```

[src/mybot/core/agent_loader.py](src/mybot/core/agent_loader.py) - Load SOUL.md

```python
def _load_soul_md(self, agent_id: str) -> str:
    """Load SOUL.md file for an agent if it exists."""
    soul_path = self.config.agents_path / agent_id / "SOUL.md"
    if soul_path.exists():
        return soul_path.read_text().strip()
    return ""
```

[src/mybot/core/prompt_builder.py](src/mybot/core/prompt_builder.py) - NEW: Layer assembly

```python
class PromptBuilder:
    """Assembles system prompt from layered sources."""

    def build(self, state: "SessionState") -> str:
        """Build the full system prompt from layers."""
        layers = []

        # Layer 1: Identity
        layers.append(state.agent.agent_def.agent_md)

        # Layer 2: Soul (optional)
        if state.agent.agent_def.soul_md:
            layers.append(f"## Personality\n\n{state.agent.agent_def.soul_md}")

        # Layer 3: Bootstrap context (BOOTSTRAP.md + AGENTS.md + crons)
        bootstrap = self._load_bootstrap_context()
        if bootstrap:
            layers.append(bootstrap)

        # Layer 4: Runtime context
        layers.append(self._build_runtime_context(agent_id, timestamp))

        # Layer 5: Channel hint
        layers.append(self._build_channel_hint(source))

        return "\n\n".join(layers)
```

[src/mybot/core/context.py](src/mybot/core/context.py) - Added prompt_builder

```python
class SharedContext:
    def __init__(self, config: Config, channels: list[Channel[Any]] | None = None):
        # ... other initialization ...
        self.prompt_builder = PromptBuilder(self)
```

[src/mybot/core/session_state.py](src/mybot/core/session_state.py) - Use prompt_builder

```python
def build_messages(self) -> list[Message]:
    """Build messages list with system prompt."""
    system_prompt = self.shared_context.prompt_builder.build(self)
    messages: list[Message] = [{"role": "system", "content": system_prompt}]
    messages.extend(self.messages)
    return messages
```

## Example Files

[default_workspace/agents/pickle/AGENT.md](../default_workspace/agents/pickle/AGENT.md) - Base instructions

```yaml
---
name: Pickle
description: A friendly cat assistant
allow_skills: true
---

You are Pickle, a friendly cat assistant. You help with daily tasks.

## Capabilities
- Answer questions and explain concepts
- Help with coding and technical tasks
```

[default_workspace/agents/pickle/SOUL.md](../default_workspace/agents/pickle/SOUL.md) - Personality

```markdown
# Personality

You are Pickle, a friendly cat assistant. Be warm and genuinely helpful with subtle cat mannerisms. Not overly cutesy—just a gentle, approachable presence.
```

## How to Run

```bash
cd 13-multi-layer-prompts
uv run my-bot chat

# The agent's system prompt now includes:
# 1. Base instructions from AGENT.md
# 2. Personality from SOUL.md (if exists)
# 3. Bootstrap context from BOOTSTRAP.md
# 4. Runtime info (agent ID, timestamp)
# 5. Channel hint (e.g., "You are responding via telegram")

# Create a personality file for your agent:
cat > ../default_workspace/agents/pickle/SOUL.md << 'EOF'
# Personality

You are Pickle, a helpful assistant with a dry sense of humor.
You get straight to the point but remain friendly.
EOF

# The new personality will be loaded on next agent creation
```

## Prompt Layers

| Layer | Source | Purpose |
|-------|--------|---------|
| Identity | AGENT.md body | Core capabilities and behavior |
| Soul | SOUL.md | Personality, tone, style |
| Bootstrap | BOOTSTRAP.md + AGENTS.md | Workspace context, available agents |
| Runtime | Generated | Agent ID, current timestamp |
| Channel | Generated | Platform hint (CLI, Telegram, etc.) |

## Benefits

1. **Separation of Concerns**: Capabilities vs personality are separate
2. **Easy Customization**: Change personality without touching core instructions
3. **Context Injection**: Bootstrap layer provides workspace context
4. **Runtime Awareness**: Agent knows when/where it's running

## What's Next

[Step 14: Post Message Back](../14-post-message-back/) - Agent-initiated communication
