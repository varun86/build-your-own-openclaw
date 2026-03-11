# Step 12: Cron + Heartbeat - Scheduled Tasks

Agent can now initiate conversations on a schedule using CRON.md definitions.

## Prerequisites

Same as previous steps - copy the config file and add your API key:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Edit config.user.yaml to add your API key
```

## What We will Build?

### Architecture

```
CronWorker (every minute)
    ↓
Find due cron jobs
    ↓
Create DispatchEvent
    ↓
AgentWorker executes
    ↓
DispatchResultEvent
```

### Key Components

- **CronLoader**: Loads CRON.md files with schedule definitions
- **CronWorker**: Background worker that checks every minute for due jobs
- **DispatchEvent**: Event type for internal agent-to-agent delegation
- **DispatchResultEvent**: Result event returned from dispatched jobs
- **CronEventSource**: EventSource for cron-triggered events

## Key Changes

[src/mybot/core/cron_loader.py](src/mybot/core/cron_loader.py)

```python
class CronDef(BaseModel):
    """Loaded cron job definition."""
    id: str
    name: str
    description: str
    agent: str
    schedule: str
    prompt: str
    one_off: bool = False

    @field_validator("schedule")
    @classmethod
    def validate_schedule(cls, v: str) -> str:
        """Validate cron expression and enforce 5-minute minimum granularity."""
        if not croniter.is_valid(v):
            raise ValueError(f"Invalid cron expression: {v}")
        # ... validation logic
        return v
```

[src/mybot/server/cron_worker.py](src/mybot/server/cron_worker.py)

```python
class CronWorker(Worker):
    """Finds due cron jobs, publishes DISPATCH events."""

    async def run(self) -> None:
        """Check every minute for due jobs."""
        self.logger.info("CronWorker started")

        while True:
            try:
                await self._tick()
            except Exception as e:
                self.logger.error(f"Error in tick: {e}")

            await asyncio.sleep(60)

    async def _tick(self) -> None:
        """Find and dispatch due jobs via EventBus."""
        jobs = self.context.cron_loader.discover_crons()
        due_jobs = find_due_jobs(jobs)

        for cron_def in due_jobs:
            # Create agent session and dispatch event
            event = DispatchEvent(
                session_id=session.session_id,
                source=CronEventSource(cron_id=cron_def.id),
                content=cron_def.prompt,
            )
            await self.context.eventbus.publish(event)
```

[src/mybot/core/events.py](src/mybot/core/events.py) - New event types

```python
@dataclass
class CronEventSource(EventSource):
    """Source for cron-triggered events."""
    _namespace = "cron"
    cron_id: str

@dataclass
class DispatchEvent(Event):
    """Event for internal agent-to-agent delegation."""
    parent_session_id: str = ""
    retry_count: int = 0

@dataclass
class DispatchResultEvent(Event):
    """Event for result of a dispatched job."""
    parent_session_id: str = ""
    success: bool = True
    error: str | None = None
```

[src/mybot/server/agent_worker.py](src/mybot/server/agent_worker.py) - Handle DispatchEvent

```python
class AgentWorker(SubscriberWorker):
    """Dispatches events to session executors."""

    def __init__(self, context):
        super().__init__(context)
        # Subscribe to both InboundEvent and DispatchEvent
        self.context.eventbus.subscribe(InboundEvent, self.dispatch_event)
        self.context.eventbus.subscribe(DispatchEvent, self.dispatch_event)
```

## Example CRON.md

[default_workspace/crons/hello-world/CRON.md](../default_workspace/crons/hello-world/CRON.md)

```yaml
---
name: Hello World
description: A simple scheduled task that greets every 5 minutes
agent: default
schedule: "*/5 * * * *"
one_off: false
---

Please greet the user and tell them what time it is. This is a scheduled task that runs every 5 minutes.
```

**CRON.md Fields:**
- `name`: Human-readable name
- `description`: What this cron job does
- `agent`: Which agent to invoke
- `schedule`: Cron expression (5-field format, minimum 5-minute granularity)
- `one_off`: Delete after first run (default: false)
- Body: Prompt to send to the agent

## How to Run

```bash
cd 12-cron-heartbeat
uv run my-bot chat

# The agent will automatically execute scheduled tasks
# Watch the logs for cron execution messages:
# INFO:mybot.server.cron_worker:Dispatched cron job: hello-world
# INFO:mybot.server.agent_worker:Session completed: ...

# Create a new cron job:
mkdir -p ../default_workspace/crons/my-task
cat > ../default_workspace/crons/my-task/CRON.md << 'EOF'
---
name: My Task
description: Runs every 10 minutes
agent: default
schedule: "*/10 * * * *"
---

Tell me a fun fact about AI.
EOF

# The new cron will be picked up automatically on next tick
```

## Notes

**Cron Schedule Format:**
- Uses standard 5-field cron syntax: `minute hour day month weekday`
- Minimum 5-minute granularity enforced (e.g., `*/5 * * * *` works, `*/1 * * * *` fails)
- Examples:
  - `*/5 * * * *` - Every 5 minutes
  - `0 * * * *` - Every hour
  - `0 9 * * 1-5` - Weekdays at 9:00 AM

**One-off Tasks:**
- Set `one_off: true` to delete the cron after first execution
- Useful for scheduled reminders or delayed tasks

**Event Flow:**
1. CronWorker checks every minute
2. Creates session and DispatchEvent for due jobs
3. AgentWorker executes the dispatched job
4. DispatchResultEvent emitted (ready for future agent-to-agent communication)

## What's Next

[Step 13: Post Message Back](../13-multi-layer-prompts/) - Responsive system prompt
