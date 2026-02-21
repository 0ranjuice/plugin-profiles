# plugin-profiles

> Claude Code plugin: save and restore named plugin combinations for different workflows.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## Installation

```bash
# Clone into your local plugins directory
git clone https://github.com/0ranjuice/plugin-profiles.git ~/.claude/local-plugins/plugin-profiles
```

Then enable the plugin in Claude Code settings.

## Quick Start

```
/profile-save my-setup       # Save current enabled plugins as "my-setup"
/profile-list                 # See all saved profiles
/profile-load my-setup        # Restore that plugin set (restart required)
```

## Commands

| Command | Description |
|---------|-------------|
| `/profile-save <name>` | Save current enabled plugins as a named profile |
| `/profile-load <name>` | Load a saved profile, replacing current plugins |
| `/profile-list` | List all saved profiles |
| `/profile-delete <name>` | Delete a saved profile |

## Features

- **Auto-backup** — Every load auto-saves your current config as `_previous`, so you can always undo
- **Self-preservation** — The plugin keeps itself enabled after every load
- **Safe validation** — Skips uninstalled or blocked plugins with clear warnings
- **Read-only save** — Saving never modifies your settings, only reads them

## Documentation

See [USER-MANUAL.md](USER-MANUAL.md) for full documentation including workflows, data storage schema, safety features, and troubleshooting.

## License

[MIT](LICENSE)
