# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run build        # Production build (outputs to de.perdoctus.streamdeck.homeassistant.sdPlugin/)
npm run build-dev    # Watch mode with inline sourcemaps (also restarts StreamDeck plugin automatically)
npm run lint         # ESLint with auto-fix
npm run format       # Prettier formatting
npm run validate     # Validate plugin via Elgato CLI
npm run bundle       # Create distributable .sdplugin package
```

There are no test commands — this project has no automated tests.

## Architecture

This is a **StreamDeck plugin** for controlling Home Assistant entities. It has two distinct execution contexts that each run in their own browser process:

1. **Plugin** (`src/plugin/main.js` → `src/components/PluginComponent.vue`) — Runs inside the StreamDeck application. Manages the WebSocket connection to Home Assistant, handles button interactions, and renders SVG icons/labels on physical StreamDeck keys.

2. **Property Inspector (PI)** (`src/pi/main.js` → `src/components/PiComponent.vue`) — The settings UI that appears in the StreamDeck desktop app when a button is selected. Built with Vue 3 + Bootstrap.

Both contexts communicate with each other and the StreamDeck software via `src/modules/common/streamdeck.js`, which wraps the StreamDeck WebSocket protocol.

### Module Structure

**`src/modules/homeassistant/`** — Home Assistant WebSocket client:
- `homeassistant.js` — Manages auth flow, message routing, request tracking, and 2000ms debounced state sync
- `commands/` — Abstract `Command` base class with implementations: `ExecuteScriptCommand`, `GetStatesCommand`, `GetServicesCommand`, `SubscribeEventsCommand`
- `actions/` — Abstract `Action` base class with `ServiceAction` implementation

**`src/modules/plugin/`** — Rendering logic:
- `entityConfigFactoryNg.js` — Loads display config YAML (local or remote CDN), resolves domain/device-class rules, and renders SVG buttons dynamically based on entity state
- `svgUtils.js` — SVG manipulation helpers

**`src/modules/common/`** — Shared utilities:
- `streamdeck.js` — StreamDeck protocol event emitter wrapper
- `settings.js` — Settings version migration (v1→v5); preserves backward compatibility across plugin updates
- `utils.js` — General helpers

**`src/modules/pi/`** — Property Inspector models (`entity.js`, `service.js`)

### Display Configuration

Button appearance is driven by YAML config files in `public/config/`. The main config (`default-display-config.yml`) defines domain-specific icons, colors, and label templates with state-based and device-class overrides. This config can also be fetched remotely from CDN (configurable via `public/config/manifest.yml`).

### Templating

- **Nunjucks** — Used for button label templates (evaluated in the plugin context)
- **Jinja2** — Used in service call data JSON (evaluated by Home Assistant)

### Encoder / Dial Support

Encoder buttons have two contexts — the dial and the touchpad. `controllerType` in settings determines rendering path: `isEncoder()` returns true only when `controllerType === 'Encoder'`, which triggers `setFeedback()` instead of `setImage()`.

**Critical**: `controllerType` is read from `willAppear` payload (`message.payload.controller`) so it always reflects the real hardware slot, regardless of what was previously saved in settings. Do not remove this override in `willAppear`.

For encoders, `rotationPercent` (0–100) is tracked internally and synced from entity state on each HA update. `rotationAbsolute` (0–255) is derived as `rotationPercent * 2.55`. The display config YAML can set `rotationPercent` as a Nunjucks template to define the sync source (e.g., `brightness / 255 * 100` for lights).

`entityConfigFactoryNg.js` resolves `rotationPercent` from the display config, renders it as a Nunjucks template against the entity state, and returns the float value in `renderingConfig`. The plugin then syncs it to the internal dial position.

### Debugging

Plugin logs go to `~/Library/Logs/ElgatoStreamDeck/de.perdoctus.streamdeck.homeassistant0.log`. Use `$SD.value.log('message')` (not `console.log`) to write to this file — `console.log` goes to the browser devtools which are not easily accessible.

### Build Output

Vite builds both entry points into `de.perdoctus.streamdeck.homeassistant.sdPlugin/`. The `vite/RestartStreamDeck` custom plugin auto-restarts the StreamDeck plugin process on each build (both `build` and `build-dev`). Use `streamdeck link <path>` to install a dev symlink.

### Settings Migration

`settings.js` implements a versioned migration chain (v1→v5). When adding new settings fields, extend the migration chain rather than changing existing field names. The v4 migration defaults `controllerType` to `'Keypad'` — the `willAppear` handler overrides this from the actual SDK payload.
