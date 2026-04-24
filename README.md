# ClawKitchen

ClawKitchen is a local-first UI companion for ClawRecipes and OpenClaw recipe workflows. It gives agent and team scaffolding a cleaner operational surface: manage recipes, inspect workspaces, run local orchestration flows, and keep file-first team systems easier to use.

## What it demonstrates

- Local-first product thinking for agent workflow management
- UI layer for recipe, team, and workspace operations
- OpenClaw plugin integration with a practical install path
- Tailscale-friendly remote access with explicit authentication requirements

---

## Prerequisites

- OpenClaw installed and available on PATH as `openclaw`
- ClawRecipes installed or linked so `openclaw recipes ...` works

---

## Run as an OpenClaw plugin

ClawKitchen can be loaded as an OpenClaw plugin so it runs locally on the orchestrator.

### 1. Install or load the plugin

Recommended for end users: install the published plugin package. It ships with a prebuilt `.next/` directory so you do not need to run npm commands.

```bash
openclaw plugins install @jiggai/kitchen

# If you use a plugin allowlist, explicitly trust it:
openclaw config get plugins.allow --json
openclaw config set plugins.allow --json '["memory-core","telegram","recipes","kitchen"]'
```

Edit `~/.openclaw/openclaw.json` and add:

```json5
{
  "plugins": {
    "allow": ["kitchen", "recipes"],
    "entries": {
      "kitchen": {
        "enabled": true,
        "config": {
          "enabled": true,
          "dev": false,
          "host": "127.0.0.1",
          "port": 7777,
          "authToken": ""
        }
      }
    }
  }
}
```

Notes:
- Plugin id is `kitchen` from `openclaw.plugin.json`.
- If `plugins.allow` is present, it must include `kitchen` or config validation will fail.

### 2. Restart the gateway

```bash
openclaw gateway restart
```

### 3. Confirm Kitchen is running

```bash
openclaw kitchen status
openclaw kitchen open
```

Then open:

```text
http://127.0.0.1:7777
```

---

## Tailscale / remote access

This is intended for Tailscale-only remote access.

### 1. Pick an auth token

Use a long random string.

```bash
openssl rand -base64 32
openssl rand -hex 32
node -e "console.log(require('crypto').randomBytes(32).toString('base64url'))"
```

### 2. Bind to your Tailscale IP

Update OpenClaw config:

```json5
{
  "plugins": {
    "entries": {
      "kitchen": {
        "enabled": true,
        "config": {
          "host": "<tailscale-ip>",
          "port": 7777,
          "authToken": "<token>",
          "dev": false
        }
      }
    }
  }
}
```

Restart:

```bash
openclaw gateway restart
```

### 3. Connect

Open in a browser:

```text
http://<tailscale-ip>:7777
```

Authentication:

- username: `kitchen`
- password: `<token>`

Safety rule: if `host` is not localhost, `authToken` is required.

---

## Goals

See [docs/GOALS.md](docs/GOALS.md).

---

## Notes

- This app shells out to `openclaw` on the same machine.
- Phase 2 will add marketplace, search, and publish flows.

---

## Troubleshooting

### Stop Kitchen

Kitchen runs in-process with the OpenClaw Gateway. The supported way to stop it is to disable the plugin and restart the gateway.

```bash
openclaw plugins disable kitchen
openclaw gateway restart
```

Re-enable later with:

```bash
openclaw plugins enable kitchen
openclaw gateway restart
```

### 500 errors for Next static chunks

If you see 500 errors for `/_next/static/chunks/*.js`, Kitchen is likely running with `dev: false` but the local `.next/` build output is missing or out of date.

Fix:

```bash
cd /home/control/clawkitchen
npm install
npm run build
openclaw gateway restart
```

Then hard refresh the browser.
