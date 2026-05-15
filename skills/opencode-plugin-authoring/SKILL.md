---
name: opencode-plugin-authoring
description: Build, package, and ship opencode plugins. Use when creating a new opencode plugin, hooking session/message/tool events, registering custom slash commands or TUI dialogs, adding a custom auth provider or model loader, exposing config overrides, packaging a plugin for npm distribution, choosing between server-side `Plugin` and `TuiPlugin` surfaces, or installing a plugin globally vs per-project. Covers the `@opencode-ai/plugin` and `@opencode-ai/plugin/tui` APIs, common patterns from real plugins (opencode-copilot-enhanced, opencode-gaslight, opencode-content-filter, opencode-keystone), and pairs with the `npm-publish` skill for releases.
---

# opencode Plugin Authoring

Patterns for building opencode plugins — distilled from real shipped plugins. For releasing the resulting npm package, defer to the **`npm-publish`** skill.

## When to Use

- Designing a new opencode plugin from scratch
- Choosing between the **server plugin** (`Plugin` from `@opencode-ai/plugin`) and the **TUI plugin** (`TuiPlugin` from `@opencode-ai/plugin/tui`) — or shipping both
- Adding a custom auth provider (OAuth device flow, token exchange)
- Mutating opencode config at startup (model variants, provider overrides)
- Hooking events (`message.updated`, `message.part.updated`, `session.error`, etc.)
- Registering slash commands or TUI dialogs
- Bootstrapping prompt files / commands into `~/.config/opencode/` on first session
- Packaging the plugin for `opencode plugin <name> -g` install

## Two plugin surfaces

opencode plugins come in two flavors. **They are different entry points and you can ship both from the same package.**

| Surface | Import | Runs in | Use for |
|--------|--------|---------|---------|
| Server | `import type { Plugin, Hooks, PluginModule } from "@opencode-ai/plugin"` | opencode server / worker process | auth providers, config rewriting, event hooks, model/provider loaders, CLI-time side effects |
| TUI    | `import type { TuiPlugin, TuiPluginApi, TuiPluginModule } from "@opencode-ai/plugin/tui"` | TUI process (separate runtime, JSX via `@opentui/solid`) | slash commands, dialogs, toasts, reading session state, mutating message parts |

Real-world split, from `opencode-content-filter`:

```jsonc
// package.json
{
  "exports": {
    ".":        { "import": "./tui.tsx" },     // TUI side
    "./tui":    { "import": "./tui.tsx" },
    "./server": { "import": "./server.ts" }    // server side
  },
  "files": ["server.ts", "tui.tsx", "README.md"]
}
```

opencode loads each side from the appropriate subpath when it discovers the package.

## Project layout

Two viable shapes:

### Source-shipping (no build)

Used by `opencode-gaslight`, `opencode-content-filter`. Ship `.ts`/`.tsx` directly; opencode's runtime (Bun) executes them.

```
my-plugin/
├── package.json
├── README.md
├── tui.tsx           # default export: TuiPluginModule
└── server.ts         # default export: PluginModule (optional)
```

Pros: no build step, fastest iteration, easiest debugging.
Cons: ships TS source, peer-deps load at runtime.

### Compiled (bun build → dist/)

Used by `opencode-copilot-enhanced`. Compile to `dist/index.js`, ship that.

```
my-plugin/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts      # exports plugin: Plugin
│   ├── config.ts
│   └── …
└── dist/             # build output, in `files`
    └── index.js
```

```jsonc
// package.json (compiled)
{
  "type": "module",
  "main": "./dist/index.js",
  "exports": { ".": { "import": "./dist/index.js" } },
  "files": ["dist"],
  "scripts": {
    "build": "bun build src/index.ts --outdir dist --target node",
    "typecheck": "tsc --noEmit",
    "test": "bun test",
    "prepublishOnly": "bun run build"
  },
  "peerDependencies": { "@opencode-ai/plugin": ">=1.1.41 <1.2.0" },
  "devDependencies": { "@opencode-ai/plugin": "1.1.41", "@types/bun": "^1.3.1", "typescript": "^5.9.3" }
}
```

Pick **source-shipping** for TUI/UI plugins (need JSX runtime hint anyway). Pick **compiled** for non-trivial server plugins with internal modules and tests.

## package.json conventions

Required fields for any opencode plugin:

```jsonc
{
  "name": "opencode-<thing>",          // convention: opencode- prefix (or @scope/opencode-...)
  "version": "0.1.0",
  "type": "module",                     // ESM only
  "engines": { "opencode": ">=1.14.22" },
  "keywords": ["opencode", "opencode-plugin", "..."],  // include both for discovery
  "exports": { ".": { "import": "./<entry>" } },
  "files": ["..."]                      // explicit allowlist
}
```

Auth/model loader plugins (server-side) — use **`peerDependencies`** for `@opencode-ai/plugin` to avoid duplicating the runtime types:

```jsonc
"peerDependencies": { "@opencode-ai/plugin": ">=1.1.41 <1.2.0" }
```

UI plugins that need OpenTUI — declare **`dependencies`** so they install:

```jsonc
"dependencies": {
  "@opencode-ai/plugin": "latest",
  "@opentui/core": "latest",
  "@opentui/solid": "latest",
  "solid-js": "^1.9.0"
}
```

Add a sanity check script so CI catches import-time errors without booting opencode:

```jsonc
"scripts": {
  "check": "bun -e \"await import('./tui.tsx'); await import('./server.ts'); console.log('plugin import ok')\""
}
```

## Server plugin (`Plugin`) — anatomy

```ts
import type { Plugin, Hooks, PluginModule } from "@opencode-ai/plugin"

export const plugin: Plugin = async (input) => {
  // input.client: opencode SDK client (logs, app, sessions, parts, etc.)
  // input.app, input.directory, input.$: shell helper

  return {
    // Mutate user config at startup. Runs once.
    async config(config) {
      // e.g. inject model variants, register providers
    },

    // Per-event hook. Fires for every server event.
    async event({ event }) {
      if (event.type === "message.part.updated") { /* … */ }
    },

    // Custom auth provider
    auth: {
      provider: "github-copilot",
      async loader(getAuth, provider) {
        const info = await getAuth()
        return { baseURL, apiKey: "", fetch: customFetch }
      },
      methods: [{ type: "oauth", label: "Login with …", prompts: […], authorize, callback }],
    },

    // Model/provider hook (extension)
    provider: {
      id: "github-copilot",
      async models(provider, ctx) { /* return mutated provider.models */ },
    },
  }
}

// If installed via `opencode plugin <name> -g`, opencode imports the default export:
const mod: PluginModule = { plugin }   // or just `export default plugin`
export default mod
```

Returning an object from the async factory registers the hooks. Anything you don't need, omit. Real example: `opencode-copilot-enhanced/src/index.ts` returns `{ config, auth }`.

### Server-side caveats from real plugins

- **Never `console.log` in a server plugin** — it corrupts the TUI's stdio. Use `client.app.log({ body: { service, level, message } })` for diagnostics. Pattern from `opencode-keystone`:

  ```ts
  await client?.app?.log?.({ body: { service: "my-plugin", level: "info", message: "…" } })
  ```

- **Detect `opencode run` (CLI) vs interactive TUI** when you want to write to stderr only in non-interactive mode (pattern from `opencode-content-filter/server.ts`):

  ```ts
  function isRunCommand() {
    if (process.env.OPENCODE_PROCESS_ROLE === "worker") return false
    return process.argv.includes("run")
  }
  ```

- **Worker vs main process**: server plugins load in both. Guard expensive setup with the env var above when you only want it once.

## TUI plugin (`TuiPlugin`) — anatomy

Single async factory receiving `api`. JSX uses Solid via `@opentui/solid`.

```tsx
/** @jsxImportSource @opentui/solid */
import type { TuiPlugin, TuiPluginApi, TuiPluginModule } from "@opencode-ai/plugin/tui"

const tui: TuiPlugin = async (api) => {
  // Register a slash command
  api.command.register(() => [{
    title: "My Command",
    value: "plugin.mything",
    description: "Does the thing",
    category: "Session",
    slash: { name: "mything" },                            // /mything
    enabled: api.route.current.name === "session",
    onSelect: async () => { /* open dialog, mutate state */ },
  }])

  // Subscribe to events; return value is unsubscribe fn
  const off = api.event.on("message.updated", (event) => { /* … */ })
  api.lifecycle.onDispose(off)
}

const plugin: TuiPluginModule & { id: string } = { id: "opencode-mything", tui }
export default plugin
```

### TUI API surface (commonly used)

| `api.*` | Purpose |
|---------|---------|
| `command.register(() => Command[])` | Register slash + palette commands. Re-evaluated on route change. |
| `event.on(type, handler)` | Subscribe to TUI events: `message.updated`, `message.part.updated`, `session.error`, `session.idle`, etc. Returns unsubscribe. |
| `route.current` | Current route, e.g. `{ name: "session", params: { sessionID } }`. |
| `state.session.messages(sessionID)` | All messages in a session. |
| `state.part(messageID)` | All parts of a message. |
| `client.part.update({ sessionID, messageID, partID, part })` | Mutate a part (used by gaslight to rewrite history). |
| `ui.toast({ variant, title, message, duration })` | Notification. |
| `ui.dialog.replace(render, onClose)` / `setSize("large"\|"medium")` / `clear()` | Modal dialogs. |
| `ui.DialogSelect`, `ui.DialogPrompt` | Built-in dialog components. |
| `lifecycle.onDispose(fn)` | Cleanup on unload. |

### Building UI

`@opentui/solid` provides Solid-flavored JSX into terminal primitives (`<box>`, `<text>`, `<textarea>`). Hook into keyboard:

```tsx
import { useKeyboard } from "@opentui/solid"
useKeyboard((evt) => {
  if (evt.name === "escape") { evt.preventDefault(); props.onCancel() }
})
```

See `opencode-gaslight/tui.tsx` for a full tabbed editor with `<textarea>` and keyboard navigation.

## Common patterns

### Pattern 1: Filter / observer (no UI)

`opencode-content-filter` watches `message.updated` for `finish === "content-filter"` and toasts. Both sides:

```ts
// tui.tsx
const offMessage = api.event.on("message.updated", (e) => inspect(e.properties.info))
const offPart    = api.event.on("message.part.updated", (e) => inspectSession(e.properties.sessionID))
api.lifecycle.onDispose(() => { offMessage(); offPart() })
```

```ts
// server.ts — same logic but writes to stderr in `opencode run`
event: async ({ event }) => { /* set process.exitCode = 1, write to stderr */ }
```

### Pattern 2: Slash-command-only (TUI only)

`opencode-gaslight` registers `/gaslight`, picks an assistant message, opens an editor dialog, mutates parts via `api.client.part.update(...)`. No server plugin needed. Single `tui.tsx`.

### Pattern 3: Auth + model sync (server only)

`opencode-copilot-enhanced` ships a `Plugin` that:
- `config(config)` — reads `~/.local/share/opencode/auth.json`, syncs models against Copilot API, injects reasoning-effort variants into the user's config.
- `auth: { provider, loader, methods: [oauth-device-flow] }` — full OAuth device flow with custom `clientId`, supports GitHub.com and GHE deployments.

Pattern: store credentials by writing to opencode's auth store via the SDK, return a custom `fetch` from `loader()` that injects rotating tokens.

### Pattern 4: Bootstrap files into `~/.config/opencode/`

`opencode-keystone` ships a `/keystone` slash command **as a Markdown file** plus a tiny shim plugin that symlinks the file into `~/.config/opencode/commands/` on first session start. This is the simplest way to ship slash commands that are just prompt templates:

```js
// plugin/index.js
export const KeystonePlugin = async ({ client }) => {
  if (existsSync(TARGET)) return {}
  mkdirSync(TARGET_DIR, { recursive: true })
  try   { symlinkSync(SOURCE, TARGET) }
  catch { copyFileSync(SOURCE, TARGET) }
  return {}
}
```

Idempotent (`existsSync` short-circuit), fail-soft (logs via `client.app.log`, never throws).

## Installation modes

Once published, users install with:

```bash
# Global (recommended for most plugins) — installs into ~/.config/opencode/plugins
opencode plugin <pkg-name> -g

# Per-project — adds to the project's opencode config
opencode plugin <pkg-name>
```

opencode imports the package's default export and:
- if it has a `tui` field → loads it in the TUI process
- if it has a `plugin` field (or is callable) → loads it server-side
- if it exports `./server` and `./tui` subpaths → uses both

Test locally before publishing by linking:

```bash
cd my-plugin && bun link
cd ~/.config/opencode && bun link my-plugin
# then add to ~/.config/opencode/opencode.json plugins array
```

## Versioning compatibility

opencode evolves the plugin API. Pin carefully:

- `engines.opencode` — minimum opencode version required (read this from the API features you use).
- `peerDependencies["@opencode-ai/plugin"]` — narrow range like `">=1.1.41 <1.2.0"` for compiled server plugins. The `<minor` upper bound prevents silent breakage on opencode minor bumps.
- TUI plugins typically use `"@opencode-ai/plugin": "latest"` in `dependencies` since they ship source and want runtime types to match the host opencode.

When a breaking change lands in `@opencode-ai/plugin`, bump your plugin's **major** and widen/raise `engines.opencode`.

## Releasing

Use the **`npm-publish` skill** for the full release workflow. opencode-plugin-specific extras:

- Tag both `opencode` and `opencode-plugin` keywords for npm search discovery.
- Set `publishConfig.access: "public"` (most plugins are scoped or public).
- Enable `provenance: true` and publish via GitHub Actions OIDC trusted publisher — users installing from `opencode plugin` benefit from the supply-chain signal.
- After publish, smoke-test: `opencode plugin <pkg> -g` in a clean opencode config dir, start a session, exercise the command/hook.

## Quick scaffolding checklist

- [ ] Pick surface(s): server, TUI, or both
- [ ] Decide: ship source (`.tsx`) or compiled (`dist/`)
- [ ] `package.json`: `type: module`, `exports`, `files`, `engines.opencode`, `keywords: ["opencode","opencode-plugin"]`
- [ ] Default export shape: `{ id, tui }` or `{ plugin }` or both subpaths
- [ ] Add `check` script that imports each entry
- [ ] Replace any `console.log` in server plugin with `client.app.log({...})`
- [ ] Add `lifecycle.onDispose` cleanup for every `event.on` subscription (TUI)
- [ ] README: install command, what it does, screenshots/asciinema if UI
- [ ] Use the `npm-publish` skill to release
