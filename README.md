# playwrightmcp-claude

A working copy of the [Playwright](https://github.com/microsoft/playwright) monorepo (browser automation engine, test runner, and the **Playwright MCP server / CLI** for AI agents), set up for use with Claude Code.

This guide covers:

1. [What is in this repository](#1-what-is-in-this-repository)
2. [Prerequisites](#2-prerequisites)
3. [Cloning the repository](#3-cloning-the-repository)
4. [Installing and building](#4-installing-and-building)
5. [Using the system](#5-using-the-system)
   - [Run the MCP server from source](#51-run-the-mcp-server-from-source)
   - [Connect it to Claude Code or another MCP client](#52-connect-it-to-claude-code-or-another-mcp-client)
   - [Common MCP options](#53-common-mcp-options)
   - [Use Playwright as a library or test runner](#54-use-playwright-as-a-library-or-test-runner)
6. [Running the tests](#6-running-the-tests)
7. [Daily development workflow](#7-daily-development-workflow)
8. [Committing and pushing](#8-committing-and-pushing)
9. [Troubleshooting](#9-troubleshooting)
10. [Further reading](#10-further-reading)

---

## 1. What is in this repository

| Path | Purpose |
|------|---------|
| `packages/playwright-core` | Browser automation engine: client, server, protocol, MCP/CLI tools |
| `packages/playwright` | Test runner and the public `playwright` package |
| `packages/playwright-test` | `@playwright/test` entry point |
| `packages/playwright-core/src/tools/` | MCP server (`mcp/`), tool implementations (`backend/`), CLI (`cli-client/`, `cli-daemon/`) |
| `packages/trace-viewer`, `html-reporter`, `recorder`, `dashboard` | Web UIs |
| `tests/` | All test suites (`page`, `library`, `playwright-test`, `mcp`, ...) |
| `docs/src/` | API documentation, the source of truth for public TypeScript types |
| `utils/` | Build scripts, code generation, linting |
| `CLAUDE.md` | Project rules that Claude Code follows (commit format, test commands, ...) |

---

## 2. Prerequisites

| Tool | Version | Check with |
|------|---------|-----------|
| Git | any recent | `git --version` |
| Node.js | **20 or newer** (required by `package.json`) | `node --version` |
| npm | ships with Node.js | `npm --version` |
| Disk space | a few GB (dependencies, build output, browser binaries) | |

Optional:

- [GitHub CLI](https://cli.github.com/) (`gh`) for pull requests.
- [Claude Code](https://claude.com/claude-code) to use the repo with an AI agent.

On Windows, use PowerShell or Git Bash. The commands below work in both unless noted.

---

## 3. Cloning the repository

### 3.1 Clone your copy

```bash
git clone https://github.com/maharehanqsols-create/playwrightmcp-claude.git
cd playwrightmcp-claude
```

The history is large (thousands of upstream commits), so the first clone can take several minutes. If you only need the latest code and not the history:

```bash
git clone --depth 1 https://github.com/maharehanqsols-create/playwrightmcp-claude.git
```

A shallow clone cannot fetch older upstream history later without `git fetch --unshallow`.

### 3.2 Check your git identity

```bash
git config user.name
git config user.email
```

Set them if empty:

```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

### 3.3 (Optional) Track the upstream Playwright repository

This lets you pull in new upstream changes later.

```bash
git remote add upstream https://github.com/microsoft/playwright.git
git remote -v
```

To bring upstream changes into your `main`:

```bash
git fetch upstream
git merge upstream/main
```

Do not push to `upstream`. It is the Microsoft repository, and you almost certainly do not have write access.

---

## 4. Installing and building

From the repository root:

```bash
npm ci
npm run build
npx playwright install
```

What each step does:

1. `npm ci` installs the exact dependency versions from `package-lock.json` for all workspace packages.
2. `npm run build` compiles all packages and generates derived files (types, protocol channels, validators).
3. `npx playwright install` downloads the browser binaries (Chromium, Firefox, WebKit) that Playwright drives. To download only one browser: `npx playwright install chromium`.

For active development, use watch mode instead of a one-off build:

```bash
npm run watch
```

Leave it running in its own terminal. It rebuilds on every change, and it also regenerates generated files, so do not hand-edit those.

---

## 5. Using the system

### 5.1 Run the MCP server from source

After building, start the MCP server from your clone:

```bash
node packages/playwright-core/cli.js mcp
```

This runs the server over stdio, which is what MCP clients expect. For a standalone HTTP/SSE server on a port:

```bash
node packages/playwright-core/cli.js mcp --port 8931
```

To see every option:

```bash
node packages/playwright-core/cli.js mcp --help
```

### 5.2 Connect it to Claude Code or another MCP client

**Claude Code, using your local build:**

```bash
claude mcp add playwright-local -- node /absolute/path/to/playwrightmcp-claude/packages/playwright-core/cli.js mcp
```

On Windows, use an absolute path such as `D:\path\to\playwrightmcp-claude\packages\playwright-core\cli.js`.

**Claude Code, using the published package instead of your build:**

```bash
claude mcp add playwright -- npx @playwright/mcp@latest
```

**Other MCP clients (VS Code, Cursor, Claude Desktop, Windsurf, ...)** take a JSON entry like this:

```json
{
  "mcpServers": {
    "playwright": {
      "command": "node",
      "args": [
        "/absolute/path/to/playwrightmcp-claude/packages/playwright-core/cli.js",
        "mcp"
      ]
    }
  }
}
```

Once connected, ask your agent to do browser work in plain language, for example:

- "Open https://example.com and tell me the page heading."
- "Go to the login page, fill in the form, and take a screenshot."
- "List all links on this page."

The agent reads pages through structured accessibility snapshots, so it does not need screenshots to understand a page. If you change the code, rebuild (or keep `npm run watch` running) and restart the MCP client so it picks up the new build.

### 5.3 Common MCP options

Pass these after `mcp` (in the `args` list for JSON configs).

| Option | Effect |
|--------|--------|
| `--headless` | Run the browser without a window (default is headed) |
| `--browser <name>` | `chrome`, `firefox`, `webkit`, or `msedge` |
| `--isolated` | Keep the browser profile in memory only, nothing saved to disk |
| `--caps <list>` | Enable extra capabilities: `vision`, `pdf`, `devtools` |
| `--device "iPhone 15"` | Emulate a device |
| `--viewport-size` / `--mobile` | Control the viewport or emulate a generic mobile device |
| `--port <port>` / `--host <host>` | Serve over HTTP instead of stdio |
| `--config <path>` | Load options from a configuration file |
| `--storage-state <path>` | Start with saved cookies/local storage (isolated sessions) |
| `--cdp-endpoint <url>` | Attach to an already running Chromium via CDP |
| `--extension` | Connect to a running Chrome/Edge through the Playwright extension |
| `--output-dir <path>` | Where screenshots and other output files are written |
| `--proxy-server <url>` | Route traffic through a proxy |

Example (headless Firefox, in-memory profile):

```bash
node packages/playwright-core/cli.js mcp --browser firefox --headless --isolated
```

### 5.4 Use Playwright as a library or test runner

You can also use the built packages directly.

**Library script** (`example.js`):

```js
const { chromium } = require('playwright-core');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('https://example.com');
  console.log(await page.title());
  await browser.close();
})();
```

**Test runner:** create a `*.spec.ts` file and run it with the repo's runner:

```ts
import { test, expect } from '@playwright/test';

test('has title', async ({ page }) => {
  await page.goto('https://example.com');
  await expect(page).toHaveTitle(/Example/);
});
```

```bash
npx playwright test
npx playwright test --headed
npx playwright show-report
```

---

## 6. Running the tests

Run these from the repo root. Keep `npm run watch` running (or run `npm run build` first) so tests use up-to-date code.

| Command | What it runs |
|---------|--------------|
| `npm run ctest <filter>` | Library tests, Chromium only (best during development) |
| `npm run test <filter> -- --project=<chromium,firefox,webkit>` | Library tests for the chosen browsers |
| `npm run ttest <filter>` | Test-runner tests (`tests/playwright-test/`) |
| `npm run ctest-mcp <filter>` | MCP tool tests, Chromium only (`tests/mcp/`) |
| `npm run test-mcp <filter> -- --project=<chromium,firefox,webkit>` | MCP tool tests per browser |

Filtering examples:

```bash
npm run ctest tests/page/locator-click.spec.ts       # one file
npm run ctest tests/page/locator-click.spec.ts:12    # one test by line
npm run ctest -- --grep "should click"               # by test name
npm run ctest-mcp snapshot                           # MCP tests whose file name contains "snapshot"
```

---

## 7. Daily development workflow

1. Update your copy: `git pull origin main`.
2. Start watch mode: `npm run watch`.
3. Create a branch for your change: `git checkout -b my-change` (use `fix-<issue-number>` when fixing an issue).
4. Edit code. Public API changes start in `docs/src/` (see `.claude/skills/playwright-dev/api.md`); MCP tools live in `packages/playwright-core/src/tools/backend/` (see `.claude/skills/playwright-dev/tools.md`).
5. Add or update tests and run them (section 6).
6. Run the full lint and type check before committing:

   ```bash
   npm run flint
   ```

   Use `flint` rather than running `tsc` or eslint separately.

Coding conventions and the import rules between packages (`DEPS.list` files) are described in [CLAUDE.md](CLAUDE.md).

---

## 8. Committing and pushing

Use semantic commit messages in the form `label(scope): description`, where the label is one of `fix`, `feat`, `chore`, `docs`, `test`, `devops`:

```bash
git add <changed-files>
git commit -m "fix(proxy): handle SOCKS proxy authentication"
```

Create new commits for follow-up changes rather than amending existing ones.

Push to **your** repository:

```bash
git push origin main
```

If you cloned from `https://github.com/maharehanqsols-create/playwrightmcp-claude.git`, then `origin` is your repo and this is the correct target. Check with `git remote -v`.

If `git push` is rejected with "fetch first", the remote has commits you do not have. Run `git pull` (or `git fetch` and `git merge origin/main`), resolve any conflicts, and push again. Avoid `git push --force` unless you are certain you want to overwrite the remote history.

---

## 9. Troubleshooting

| Problem | Fix |
|---------|-----|
| `npm ci` fails with an engine or version error | Install Node.js 20 or newer and retry |
| Tests or the MCP server say a browser is missing | Run `npx playwright install` |
| Changes do not show up | Make sure `npm run watch` is running, or re-run `npm run build`; restart the MCP client |
| `cli.js` not found when starting the MCP server | The build has not run yet; run `npm run build` |
| Generated files keep reverting | They are generated by the build; edit the source (for example `docs/src/` or `protocol.yml`) instead |
| `flint` reports import violations | Update the relevant `DEPS.list` to declare the allowed import |
| First clone is very slow | Use `git clone --depth 1` |
| Windows path problems in MCP client config | Use absolute paths, and escape backslashes in JSON (`D:\\path\\to\\...`) |
| Push rejected with "fetch first" | See section 8 |

---

## 10. Further reading

- Playwright documentation: https://playwright.dev
- Playwright MCP server: https://github.com/microsoft/playwright-mcp
- Model Context Protocol: https://modelcontextprotocol.io
- Upstream contributing guide: [CONTRIBUTING.md](CONTRIBUTING.md)
- Project rules for Claude Code: [CLAUDE.md](CLAUDE.md)
- Architecture and development guides: `.claude/skills/playwright-dev/`

The original upstream Playwright README is preserved in git history (`git show bd1abdfaa:README.md`).

## License

Apache 2.0, the same as upstream Playwright. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
