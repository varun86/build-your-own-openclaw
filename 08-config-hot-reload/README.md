# Step 08: Config Hot Reload

Automatically reload configuration when files change without restarting the agent.

## What We Will Build

```
┌─────────────────────────────────────────────────────────────┐
│                        ChatLoop                             │
│                                                             │
│  ┌─────────────┐    ┌──────────────────┐                   │
│  │ Config      │    │ ConfigReloader   │                   │
│  │             │◀───│ (watchdog)       │                   │
│  │ - reload()  │    │                  │                   │
│  └─────────────┘    └──────────────────┘                   │
│         │                      ▲                            │
│         │                      │                            │
│         ▼                      │ file change                │
│  ┌─────────────────────────────────────────┐               │
│  │ config.user.yaml / config.runtime.yaml  │               │
│  └─────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

**Key Components:**
- **ConfigReloader** - Watches workspace for config file changes using watchdog
- **ConfigHandler** - FileSystemEventHandler that triggers config.reload()
- **Config.reload()** - Re-reads and merges config files, updates in-place
- **Config Merging** - Runtime config overrides user config via deep merge

## Key Changes

### 1. Config with Merging and Reload ([src/mybot/utils/config.py](src/mybot/utils/config.py))

```python
class Config(BaseModel):
    """Configuration with hot reload support."""

    @classmethod
    def _load_merged_configs(cls, workspace_dir: Path) -> dict[str, Any]:
        """Load and merge user and runtime config files."""
        config_data: dict[str, Any] = {}

        user_config = workspace_dir / "config.user.yaml"
        runtime_config = workspace_dir / "config.runtime.yaml"

        if user_config.exists():
            with open(user_config) as f:
                config_data = cls._deep_merge(config_data, yaml.safe_load(f) or {})

        if runtime_config.exists():
            with open(runtime_config) as f:
                config_data = cls._deep_merge(config_data, yaml.safe_load(f) or {})

        return config_data

    def reload(self) -> bool:
        """Re-read config files and update in-place."""
        try:
            config_data = self._load_merged_configs(self.workspace)
            new_config = Config.model_validate(config_data)

            for field_name in Config.model_fields:
                setattr(self, field_name, getattr(new_config, field_name))

            return True
        except Exception as e:
            logging.debug("Config reload failed: %s", e)
            return False
```

### 2. ConfigHandler ([src/mybot/utils/config.py](src/mybot/utils/config.py))

```python
class ConfigHandler(FileSystemEventHandler):
    """Handles config file modification events."""

    def __init__(self, config: Config):
        self._config = config

    def on_modified(self, event):
        """Reload config when config.user.yaml changes."""
        if not event.is_directory and event.src_path.endswith("config.user.yaml"):
            self._config.reload()
```

### 3. ConfigReloader ([src/mybot/utils/config.py](src/mybot/utils/config.py))

```python
class ConfigReloader:
    """Manages watchdog observer for config hot reload."""

    def __init__(self, config: Config):
        self._config = config
        self._observer = Observer()

    def start(self) -> None:
        """Start watching config file for changes."""
        handler = ConfigHandler(self._config)
        self._observer.schedule(handler, str(self._config.workspace), recursive=False)
        self._observer.start()

    def stop(self) -> None:
        """Stop watching."""
        self._observer.stop()
        self._observer.join()
```

### 4. ChatLoop Integration ([src/mybot/cli/chat.py](src/mybot/cli/chat.py))

```python
class ChatLoop:
    def __init__(self, config: Config, agent_id: str | None = None):
        self.config_reloader = ConfigReloader(config)
        # ... rest of init

    async def run(self) -> None:
        self.config_reloader.start()

        for worker in self.workers:
            worker.start()

        try:
            # ... chat loop
        finally:
            for worker in self.workers:
                await worker.stop()
            self.config_reloader.stop()
```

## How to Run

```bash
cd 08-config-hot-reload
uv run my-bot chat
```

Test hot reload:
1. Start the chat
2. Edit `config.user.yaml` (e.g., change `llm.model`)
3. Config automatically reloads without restart

## What's Next

Step 09 will add **Channels** - multi-platform support for CLI, Telegram, and other interfaces.
