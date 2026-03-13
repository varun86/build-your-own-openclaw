# Step 17: Memory - Long-Term Knowledge System

A specialized memory agent (Cookie) that manages persistent knowledge via agent dispatch.

## Prerequisites

Same as previous steps - copy the config file and add your API key:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Edit config.user.yaml to add your API key
```

## What We Built

### Architecture

```
User ↔ Pickle (Main Agent)
         ↓ (dispatch)
      Cookie (Memory Agent)
         ↓ (tools: read, write, bash)
      memories/
        ├── topics/      (preferences, identity)
        ├── projects/    (project-specific context)
        └── daily-notes/ (YYYY-MM-DD.md)
```

## Key Components

- **Memory agent**: Specialized agent for memory management



### `default_workspace/agents/cookie/AGENT.md` - Memory agent definition

Already exists from picklebot. Defines Cookie's role:
- Store memories using `write` tool
- Retrieve memories using `read` tool
- Organize by topics (timeless), projects, and daily notes
- Only accessible via dispatch from Pickle

## How to Use

The memory system is now configured. Pickle can dispatch to Cookie:

```
User: Remember that I prefer TypeScript for new projects

Pickle: (dispatches to Cookie)
Cookie: (stores preference in memories/topics/preferences.md)
Pickle: Got it! I've saved that preference.
```

Later:

```
User: What language should I use for my new API?

Pickle: (dispatches to Cookie to retrieve preferences)
Cookie: (reads memories/topics/preferences.md)
Pickle: Based on your preferences, I recommend TypeScript.
```

## Discussion: Memory System Approaches

This is **one approach** to implementing long-term memory. There are many alternatives:

1. **Specialized Agent** (this implementation)
2. **Direct Tools in Main Agent**
3. **Skill Based Approach, using command line tool like grep**
3. **Vector Database**

## How to Run

```bash
cd 17-memory
uv run my-bot chat

# Memory path is now available in config
# Cookie agent is ready for dispatch
# Try: "Ask Cookie to remember something for me"
```

## What's Next

This completes the tutorial! You now have a production-ready agent with:
- ✅ Event-driven architecture
- ✅ Multi-platform support
- ✅ Multi-agent collaboration
- ✅ Scheduled tasks
- ✅ Concurrency control
- ✅ Long-term memory

Next steps: Deploy, extend, and customize for your specific use case!
