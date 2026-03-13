# Build Your Own OpenClaw

A step-by-step tutorial to build your own AI agent, from a simple chat loop to a lightweight version of OpenClaw.

## Overview

**18 progressive steps** that teach you how to build an minimal version of OpenClaw. Each step includes:

- A `README.md` going through key components and design decision.
- A Runnable codebase.

**Example Project:** [pickle-bot](../pickle-bot/) - our reference implementation

## Tutorial Structure

### Phase 1: Capable Single Agent (Steps 1-7)
Build a fully-functional agent that can chat, use tools, learn skills, remember conversations, and access the internet.

- [**00-chat-loop**](./00-chat-loop/) - Your first agent
- [**01-tools**](./01-tools/) - Agent can take actions
- [**02-skills**](./02-skills/) - Dynamic capability loading
- [**03-persistence**](./03-persistence/) - Remember conversations
- [**04-slash-commands**](./04-slash-commands/) - User control
- [**05-compaction**](./05-compaction/) - Handle long conversations
- [**06-web-tools**](./06-web-tools/) - Search and read the web

### Phase 2: Event-Driven Architecture (Steps 8-11)
Refactor to event-driven architecture for scalability and multi-platform support.

- [**07-event-driven**](./07-event-driven/) - The Great Refactor
- [**08-config-hot-reload**](./08-config-hot-reload/) - Edit without restart
- [**09-channels**](./09-channels/) - Multi-platform support
- [**10-websocket**](./10-websocket/) - Real-time Websocket Connection

### Phase 3: Autonomous & Multi-Agent (Steps 12-16)
Add scheduled tasks, agent collaboration, and intelligent routing.

- [**11-multi-agent-routing**](./11-multi-agent-routing/) - Multiple agent & Right agent for right job
- [**12-cron-heartbeat**](./12-cron-heartbeat/) - Autonomous scheduled tasks
- [**13-multi-layer-prompts**](./13-multi-layer-prompts/) - Responsive system prompt
- [**14-post-message-back**](./14-post-message-back/) - Agent-initiated communication
- [**15-agent-dispatch**](./15-agent-dispatch/) - Agent collaboration

### Phase 4: Production & Scale (Steps 17-18)
Production features for reliability and long-term memory.

- [**16-concurrency-control**](./16-concurrency-control/) - Rate limiting and concurrency
- [**17-memory**](./17-memory/) - Long-term knowledge system

## How to Use This Tutorial

### Configure API Keys

Before running any step, you need to configure your API keys:

1. **Copy the example config:**
   ```bash
   cp default_workspace/config.example.yaml default_workspace/config.user.yaml
   ```

2. **Edit `config.user.yaml`** with your API keys:
   - See [LiteLLM providers](https://docs.litellm.ai/docs/providers)

3. Just follow each steps, read and try it out.

## Contributing

Each step is implemented in a separate session. Feel free to suggest improvements!
