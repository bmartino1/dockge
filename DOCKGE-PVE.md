# Dockge-PVE fork notes

This fork keeps Dockge's Compose-management UI but replaces the restricted main web console behavior with a full xterm.js PTY path intended for Proxmox VE administration.

## What changed

- `TERM` is now a real terminal type (`xterm-256color`) instead of the Dockge session name (`console`).
- The main Console sends raw xterm.js input to `node-pty`; TAB, arrow keys, Ctrl keys, escape sequences and TUI/ncurses applications are no longer intentionally filtered.
- PTY dimensions are synchronized immediately and on element resize with `ResizeObserver`, not only on browser-window resize.
- Clipboard handling uses native browser `paste`/`copy` events first, which is much more useful on Chromium-based browsers when Dockge is opened over plain HTTP on a trusted LAN.
- The browser context menu is no longer suppressed. Right-click **Paste** can feed the terminal through a real `ClipboardEvent`.
- `Ctrl+Shift+C` copies an xterm selection while plain `Ctrl+C` remains SIGINT.
- The image includes Bash completion and common host/network administration tools.
- The Console can optionally expose a second target that SSHes directly to the PVE host.

## Console targets

`Dockge Shell` always opens a shell inside the Dockge container.

`PVE Host Shell` is enabled with:

```yaml
environment:
  DOCKGE_ENABLE_CONSOLE: "true"
  DOCKGE_HOST_SHELL_ENABLED: "true"
  DOCKGE_CONSOLE_DEFAULT_TARGET: host
  DOCKGE_HOST_SSH_HOST: host.docker.internal
  DOCKGE_HOST_SSH_PORT: "2222"
  DOCKGE_HOST_SSH_USER: root
  DOCKGE_HOST_SSH_STRICT_HOST_KEY_CHECKING: accept-new
```

The container also needs:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
volumes:
  - ./ssh:/root/.ssh
```

Use an SSH key rather than embedding the PVE root password in Compose or source code.

## Terminal environment variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `DOCKGE_TERMINAL_TYPE` | `xterm-256color` | PTY terminal/terminfo type |
| `DOCKGE_UPDATE_CHECK_URL` | unset | Optional fork-compatible version endpoint; upstream update checks are disabled by default |
| `DOCKGE_HOST_SHELL_ENABLED` | `false` | Enable the host SSH console target |
| `DOCKGE_CONSOLE_DEFAULT_TARGET` | `local` | `local` or `host` |
| `DOCKGE_HOST_SSH_HOST` | `host.docker.internal` | SSH host |
| `DOCKGE_HOST_SSH_PORT` | `22` | SSH port |
| `DOCKGE_HOST_SSH_USER` | `root` | SSH username |
| `DOCKGE_HOST_SSH_IDENTITY` | unset | Optional private-key path |
| `DOCKGE_HOST_SSH_STRICT_HOST_KEY_CHECKING` | `accept-new` | `yes`, `no`, or `accept-new` |

## Clipboard behavior

For Chromium/Chrome/Edge/Brave:

- **Paste:** `Ctrl+V`, `Ctrl+Shift+V`, or right-click → Paste.
- **Copy:** select terminal text and use `Ctrl+Shift+C`, or the browser's Copy command.
- Plain `Ctrl+C` is intentionally sent to the PTY as SIGINT.

Native `ClipboardEvent` handling is preferred because `navigator.clipboard` is commonly blocked on a plain `http://192.168.x.x` origin.

For browser/PTY validation after deployment, follow **[TERMINAL-TESTS.md](./TERMINAL-TESTS.md)**.

## Build locally

```bash
npm ci
npm run lint
npm run check-ts
npm run build:frontend
npm run build:docker
```

The local image is:

```text
dockge-pve:local
```

The Dockerfile is self-contained and builds the frontend itself, so a direct build also works:

```bash
docker build --target release -t dockge-pve:local -f docker/Dockerfile .
```

## Docker Hub publishing

The included `.github/workflows/docker-publish.yml` publishes:

```text
<DOCKERHUB_USERNAME>/dockge-pve:latest
<DOCKERHUB_USERNAME>/dockge-pve:<git-tag>
<DOCKERHUB_USERNAME>/dockge-pve:sha-<commit>
```

Create these GitHub repository secrets:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`

Push to the default branch or run the workflow manually.

## Recommended PVE deployment

Start from `compose.pve.yaml`, replace the image namespace, and ensure the mounted `./ssh` directory contains the host key/known-host data required by OpenSSH.

The host-shell feature is intentionally powerful: a Dockge login can become a PVE root shell. Keep Dockge behind your trusted LAN/VPN and use strong authentication.
