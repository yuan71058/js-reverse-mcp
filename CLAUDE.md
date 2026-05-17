# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Development Commands

```bash
# Install dependencies
npm ci

# Build the project (TypeScript compilation + post-build scripts)
npm run build

# Run all tests
npm test

# Run a single test file (after building)
node --require ./build/tests/setup.js --no-warnings=ExperimentalWarning --test-reporter spec --test-force-exit --test "build/tests/path/to/test.test.js"

# Run only tests marked with { only: true }
npm run test:only

# Update test snapshots
npm run test:update-snapshots

# Type checking without emitting
npm run typecheck

# Lint and format
npm run format

# Check formatting without fixing
npm run check-format

# Start the MCP server (builds first)
npm run start

# Start with debug logging
npm run start-debug

# Generate documentation (after changing tools)
npm run docs
```

## Testing with MCP Inspector

```bash
npx @modelcontextprotocol/inspector node build/src/index.js
```

## Architecture Overview

This is an MCP (Model Context Protocol) server that exposes Chrome DevTools functionality to AI coding assistants via Patchright.

### Core Components

- **`src/main.ts`**: Entry point. Creates the MCP server, registers all tools, and handles tool execution with mutex-based serialization.

- **`src/McpContext.ts`**: Central state management class. Maintains:
  - Browser and page state (selected page, page list)
  - Accessibility tree snapshots for element interaction
  - Network and console collectors
  - Debugger context for script/breakpoint management
  - Performance trace state

- **`src/McpResponse.ts`**: Response builder that tools use to construct their output (text, images, snapshots, network data, console data).

- **`src/browser.ts`**: Browser lifecycle management. Handles launching new Chrome instances or connecting to existing ones via browserURL/wsEndpoint.

### Tool System

Tools are defined in `src/tools/` using the `defineTool()` helper from `src/tools/ToolDefinition.ts`. Each tool has:

- `name`: Tool identifier
- `description`: For MCP clients
- `annotations`: Category and read-only hint
- `schema`: Zod schema for input validation
- `handler`: Implementation that receives request, response builder, and context

Tool categories are defined in `src/tools/categories.ts` and can be disabled via CLI flags (e.g., `--categoryNetwork=false`).

### Data Collection

- **`src/PageCollector.ts`**: Base class for collecting page events. Extended by `NetworkCollector` and `ConsoleCollector` to track requests and console messages across pages.

- **`src/formatters/`**: Convert collected data to text format for MCP responses.

### Third-Party Integration

- **`src/third_party/index.ts`**: Re-exports from bundled dependencies (patchright, @modelcontextprotocol/sdk, zod).

- **`chrome-devtools-frontend`**: Used for performance trace analysis and issue descriptions. Included via tsconfig.json.

### Testing

Tests use Node.js built-in test runner. Test files mirror source structure under `tests/`. Snapshots are stored alongside test files with `.snapshot` extension.

- `tests/setup.ts`: Configures snapshot paths and serializers
- `tests/server.ts`: Test utilities for MCP server testing
- `tests/utils.ts`: Shared test helpers

## Development Philosophy

**第一性原理 — the simplest strategy that achieves the goal.** This is the first element of software development in this project. Before adding or retaining any code, ask:

1. **Does it actually do something?** Verify against the upstream library source. Don't keep "safety net" code that's a no-op. (Example: a 137-line `stealth-args.ts` was deleted in 2026-05 after auditing Patchright's `chromiumSwitches.js` revealed almost every flag in it was either duplicated by Patchright's own defaults or never added by Patchright at all.)
2. **Is it the minimum needed?** Reject "filter / merge / partial retention" of legacy patterns. When old code and new architecture belong to different paradigms, pick the architecturally correct path and delete the other completely.
3. **Is it a backward-compat shim?** Don't add `--legacyX` flags or deprecation paths unless explicitly asked. A clean cut is preferred.

Stealth/anti-detection in this project is layered cleanly:
- **Protocol layer** → Patchright (CDP automation signal suppression, zero JS injection)
- **Source layer** → CloakBrowser binary (optional, via `--cloak`; 49–57 C++ fingerprint patches)
- **Wrapper layer (this MCP)** → zero stealth responsibility

Do **not** reintroduce config-level fingerprint flags (`--disable-blink-features=AutomationControlled`, canvas noise hacks, `--lang=en-US` spoofing, etc.) — they belong to the deprecated `playwright-stealth` paradigm and conflict with the layers above.

### Product scope: reverse-engineering tool, not an automation framework

The user of this MCP is a human analyst who **sees** the browser and uses it as a live debugger. They are not impersonating an end user, not scraping at scale, not running cron jobs. Several "automation features" that look superficially relevant are **out of scope and must not be added**:

| Anti-feature | Why it does not belong |
|---|---|
| Headless mode | Analyst needs to see the page. `headless: false` is hardcoded; do not reintroduce a `--headless` flag |
| Humanize (mouse Bézier / keyboard timing / scroll curves) | Analyst is not a "user" being simulated; humanize would slow down every tool call |
| Proxy server flag (`--proxy-server` passthrough) | System-level proxies (Clash / Surge / mitmproxy / VPN) cover this. Browser-level proxy without GeoIP/timezone alignment is a half-measure that leaks bot signals |
| GeoIP timezone/locale auto-resolution | Requires `mmdb-lib` + 70MB GeoLite2 DB; only matters for "scrape from another country" scenarios that aren't ours |
| WebRTC IP spoofing (`--fingerprint-webrtc-ip`) | Only relevant when proxying; we don't proxy |
| Auto-granted permissions (`geolocation`, `notifications`) | Real users see a prompt; silent grant is itself a bot signal |
| Hardcoded `colorScheme` / `deviceScaleFactor` / `isMobile` / `hasTouch` | Static fingerprint assumptions in the wrapper layer — pick a value and every user looks the same way. Let Playwright defaults or the real OS reflect through |

The wrapper layer's job is **plumbing only**: connect/launch Chrome, thread CLI flags through, expose the `BrowserContext` to MCP tools. Anything that "decides" what the browser should pretend to be belongs in the binary (`--cloak`) or not at all.

## Conventions

- Follow [conventional commits](https://www.conventionalcommits.org/) for PR and commit titles
- Run `npm run docs` after adding or modifying tools to update documentation
- Node version: v22 (see .nvmrc)

## Repository & Publishing

- **GitHub**: https://github.com/zhizhuodemao/js-reverse-mcp
- **npm**: https://www.npmjs.com/package/js-reverse-mcp
- **Install**: `npx js-reverse-mcp`
- **MCP config**:
  ```json
  "js-reverse": {
    "command": "npx",
    "args": ["js-reverse-mcp"]
  }
  ```
- Publish to npm: `npm publish` (requires npm login)
- Create GitHub release: `gh release create v<version>`
