# Step 03: Persistence - Remember Conversations

Save and restore conversation history so the agent remembers past interactions.

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
                         HistoryStore
                              ↓
                     .sessions/sessions/{id}.jsonl
```

File System Structure:

```
.sessions/
├── index.jsonl              # Session metadata
└── sessions/
    └── {session_id}.jsonl   # Messages (one file per session)
```

### Key Components

- **.session/index.jsonl**: JSONL file-based index for sessions, including metadata
- **.session/sessions/{id}.jsonl**: JSONL file-based storage for messages

## Key Changes

[src/core/history.py](src/core/history.py) - New file

```python
class HistoryStore:
    """JSONL file-based history storage."""

    def create_session(self, agent_id: str, session_id: str) -> dict:
        """Create a new conversation session."""

    def save_message(self, session_id: str, message: HistoryMessage) -> None:
        """Save a message to history."""

    def get_messages(self, session_id: str) -> list[HistoryMessage]:
        """Get all messages for a session."""
```


## How to Run

```bash
cd 03-persistence
uv run my-bot chat

# Each run starts a new session
# Messages are saved to .sessions/ directory
```

## What's Next

[Step 04: Compaction](../04-compaction/) - Handle long conversations with context management
