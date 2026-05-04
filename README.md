# opencode config

My global [opencode](https://github.com/anomalyco/opencode) configuration. Secrets are externalized via `{file:~/.secrets/...}` references.

## Setup

```bash
git clone --recursive git@github.com:Adamkadaban/opencode-config.git ~/.config/opencode
```

## Structure

```
├── opencode.json        # Global config (MCP servers, plugins, provider, skill paths)
├── tui.json             # TUI-specific plugin config
├── plugins/             # Local plugins
├── skills/
│   ├── _trailofbits/    # submodule → trailofbits/skills
│   ├── _iothackbot/     # submodule → BrownFineSecurity/iothackbot
│   ├── copilot-second-opinion-repo/  # submodule → Adamkadaban/copilot-second-opinion
│   ├── research-fetch/  # submodule → Adamkadaban/research-fetch-skill
│   └── ...              # Local skills (burp-suite, conventional-commits, etc.)
```

## Updating skills

```bash
git submodule update --remote
```

## Secrets

Store keys in `~/.secrets/` (not tracked):

```bash
mkdir -p ~/.secrets && chmod 700 ~/.secrets
echo -n "your-key" > ~/.secrets/context7-key && chmod 600 ~/.secrets/context7-key
```

Config references them with `{file:~/.secrets/context7-key}`.
