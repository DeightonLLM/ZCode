# ZCode

<div align="center">
  <img src="public/logo/icons/1024x1024.png" alt="ZCode" width="128" height="128" />
</div>
<p align="center">
  <a href="https://applink.feishu.cn/client/chat/chatter/add_by_link?link_token=47ag983c-8fcb-4d6d-814b-5395193a712c&amp;qr_code=true">Feishu Community</a> ·
  <a href="https://discord.gg/z9aBcQXZQ3">Discord</a>
</p>
<p align="center">
  English
</p>

ZCode is an AI coding workspace that provides a desktop application, browser interface, and terminal Agent. This repository contains the client, backend services, shared UI, and the source code for the Agent CLI and runtime.

## Updates

- 2026-09-23: Updated to ZCode v3.14.3.

## Initialization

Prepare Git, Node.js **24.14.0**, and pnpm **10.33.2**. The required versions are defined in [mise.toml](mise.toml). All development and packaging commands below should be run from the repository root.

```bash
pnpm bootstrap
```

`pnpm bootstrap` installs workspace dependencies, prepares local desktop runtime resources, and then executes `build:bootstrap`.

The Agent CLI and runtime source code are located in [apps/zcode-cli/](apps/zcode-cli/), cloned as a normal directory inside this repository, so there is no need to pull or initialize a separate Git submodule.

Choose the appropriate initialization or build entry based on your needs:

| Command | Purpose |
| --- | --- |
| `pnpm install` | Install dependencies |
| `pnpm prepare:desktop-runtime` | Prepare desktop runtime resources; by default includes remote resource preparation |
| `pnpm prepare:remote-assets` | Prepare remote runtime resources separately |
| `pnpm bootstrap:with-remote` | Initialize dependencies, local and remote resources, and build related packages sequentially; skips the desktop app bundle |
| `pnpm build` | Recursively run the build scripts for each workspace package, including package-level resource preparation steps |

By default, `bootstrap` skips remote resource preparation and is intended for local desktop development. When working with a remote workspace or validating remote release resources, run the corresponding preparation commands.

## Development and Running

### Desktop

```bash
pnpm dev:desktop

# Use the test environment
pnpm dev:desktop:test
```

`pnpm dev:desktop` is the same as `pnpm dev:desktop:prod` by default and uses the production service configuration. The startup script prepares local runtime resources, builds the desktop Agent, and then launches Electron and source watching.

If you need an isolated development data directory, set `ZCODE_DATA_BASE_DIR`. For example, on macOS / Linux:

```bash
ZCODE_DATA_BASE_DIR="$HOME/.zcode-dev-home" pnpm dev:desktop:test
```

### Remote Features (SSH / WSL)

First run `pnpm bootstrap:with-remote` to prepare remote resources (mock-cdn), then run `pnpm dev:desktop`. When connecting to a remote project, choose the resource option: "Download locally and upload". Development resources are taken from the local `packages/...` directory.

### Web Development

When modifying Web or backend source code, use development mode:

```bash
pnpm dev:web

# Specify the backend workspace (macOS / Linux)
ZCODE_SERVER_WORKSPACE=/path/to/project pnpm dev:web
```

This command starts both the Web development server (default `http://localhost:5173`) and the backend (default `http://localhost:3030`). The browser accesses the former. Requests to `/ws` and normal `/api` are proxied to the local backend.

After modifying the Agent source code, run `pnpm --filter @zcode/cli... build` and restart the service. If you need to verify the full release package, follow the "ZCode Command Line" packaging section below and run the unpacked output.

### ZCode Command Line

The command-line distribution includes the TUI, Web, and Agent, and all are launched via `zcode`:

- With no arguments, it opens the TUI.
- If the first argument is `--web`, it starts the Web UI.
- Other arguments are passed through to the existing Agent CLI.

```bash
# Open the terminal interactive interface by default
zcode

# Start the Web interface
zcode --web

# Specify the project and port without automatically opening the browser
zcode --web --workspace /path/to/project --port 3030 --no-open

# View CLI or Web parameters
zcode --help
zcode --web --help
```

In Web mode, the default working directory is the current directory, it listens on `127.0.0.1`, does not enable an access token by default, automatically selects an available port, and opens the browser. Visit the address printed in the terminal and stop the service with `Ctrl+C`.

When directly starting the generic Web service HTTP entry, configure API/WebSocket authentication using `ZCODE_SERVER_AUTH_TOKEN`. When creating a service programmatically, use the `authToken` option.

For build instructions, see the packaging section below. `pnpm build:zcode` only generates the distribution package and does not replace an existing `zcode` already in `PATH`. If the command still points to an old installation or another source directory, macOS / Linux users can check `which zcode` and update `PATH` as needed.

### CLI Source Development

To develop the TUI or Agent directly, run the source entry:

```bash
pnpm --filter @zcode/cli dev --help
pnpm --filter @zcode/cli dev

# Build the CLI and its workspace dependencies
pnpm --filter @zcode/cli... build
node apps/zcode-cli/packages/cli/dist/zcode.cjs --help
```

This entry directly runs the Agent CLI without going through the distribution package’s `--web` dispatch. For Web development, use `pnpm dev:web`; to verify the unified `zcode` command, use the unpacked `bin/zcode.mjs` shown below.

## Configuration

The root file [.env.example](.env.example) provides service address and build configuration examples that can be copied to `.env` as needed, with local overrides placed in `.env.local`. The Desktop development environment is configured via `dev:desktop:test` / `dev:desktop`.

| Configuration | Purpose |
| --- | --- |
| `ZCODE_DATA_BASE_DIR` | Base directory for application data; writes go under `.zcode/` inside it |
| `ZCODE_SERVER_WORKSPACE` | Workspace path for the Web backend |
| `ZCODE_BUILTIN_PROVIDER_CONFIG_FILE` | Path to the local Provider configuration file; if unset, the built-in configuration is used |
| `ZCODE_DIST_BASE_URL` | Download root URL used by the command-line installer |

Runtime variables can be explicitly set in the environment used to start the application. The default runtime configuration shipped with the client is documented in [config/README.md](config/README.md).

## Packaging

Third-party declarations, release validation flow, and where these declarations appear in the distribution are described in [third-party/README.md](third-party/README.md).

### Desktop

```bash
pnpm bundle:desktop

# Specify target platform and CPU architecture
pnpm bundle:desktop -- --os win --arch x64

pnpm bundle:desktop -- --help
```

The default target is macOS arm64, and the default output directory is `packages/desktop/dist/`. `--os` supports `mac`, `win`, and `linux`; `--arch` supports `x64` and `arm64`. Actual packaging and signing require the corresponding platform toolchain and signing configuration.

Installation: Double-click the generated DMG and drag ZCode into the "Applications" folder. Local builds are unsigned. If macOS blocks the first launch, run:

```bash
sudo xattr -rd com.apple.quarantine /Applications/ZCode.app
```

### ZCode Command Line

The build entry is `pnpm build:zcode`. The script sequentially builds the CLI/TUI, backend, and Web, collects the native libraries, worker files, and runtime dependencies for the TUI, and assembles the distribution package. Running the packaged output still requires Node.js.

Before packaging, set the download root URL `ZCODE_DIST_BASE_URL` (either in `.env`, `.env.local`, or as an environment variable), or pass it using `--base-url`. The URLs below are examples; replace them with your actual release URL.

```bash
pnpm build:zcode --base-url https://downloads.example.com/zcode/

# When ZCODE_DIST_BASE_URL is already configured
pnpm build:zcode

# Repackage only, reusing existing Agent, backend, and Web build artifacts
pnpm build:zcode --skip-build

# View version, output directory, and other optional parameters
pnpm build:zcode --help
```

By default, the version is taken from the root `package.json`, and the output directory is `dist/zcode/`:

- `releases/<version>/zcode-<version>.tar.gz`: runtime package
- `releases/<version>/sha256.txt`: checksum summary
- `latest.json`, `install.sh`: version index and installation script

The full directory can be uploaded to the configured download root. The installation script downloads the runtime package from that address, installs it under `~/.zcode/runtime` by default, and creates the `zcode` command in `~/.local/bin`. The installation directory can be changed using `ZCODE_RUNTIME_DIR` or by passing a configuration option.

Older Lite users should switch to the commands above, environment variables, and the new installation script. The new installation does not remove the old Lite directory and does not migrate or delete existing session data.

To debug a packaged artifact locally, you can unpack and run it directly without uploading or installing:

```bash
zcode_version=$(node -p "require('./dist/zcode/latest.json').version")
mkdir -p dist/zcode/debug
tar -xzf "dist/zcode/releases/$zcode_version/zcode-$zcode_version.tar.gz" \
  -C dist/zcode/debug
# Start the default TUI
node dist/zcode/debug/zcode/bin/zcode.mjs

# Start the Web interface
node dist/zcode/debug/zcode/bin/zcode.mjs --web \
  --workspace "$PWD" --port 3030 --no-open
```

Open the browser to `http://127.0.0.1:3030` to verify the complete end-to-end flow from the backend hosting the Web page to the Agent. This port must be free; if `pnpm dev:web` is already running, use a different `--port`.

## Repository Structure

| Directory | Responsibility |
| --- | --- |
| `packages/desktop` | Electron Main, Host, Renderer, and desktop packaging |
| `packages/web` | Web client |
| `packages/server` | HTTP / WebSocket service and remote connections |
| `packages/zcode-server-cli` | Independent server startup and process management |
| `packages/ui` | Shared React components, hooks, and Zustand state |
| `packages/services` | Business services and persistence |
| `packages/shared`, `packages/rpc`, `packages/client` | Shared protocols and types, RPC framework, Agent client SDK |
| `packages/provider`, `packages/provider-node` | Shared Provider capabilities and Node implementation |
| `apps/zcode-cli` | Agent CLI, TUI, runtime, and tools |
| `scripts`, `config`, `third-party` | Build maintenance scripts, built-in configuration, and third-party declaration materials |

## Project Statement

Details on feature scope, maintenance rules, execution and data risks, and licensing and third-party copyright information are available in [NOTICE.md](NOTICE.md).
