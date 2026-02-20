# Discord Integration Design

## Date: 2026-02-20

## Overview

Enable Discord communication with Moltbot by configuring the OpenClaw gateway's built-in Discord channel support.

## Context

OpenClaw already has native Discord support. The `start-openclaw.sh` startup script reads `DISCORD_BOT_TOKEN` and `DISCORD_DM_POLICY` environment variables and automatically configures the `channels.discord` section in OpenClaw's config.

No new code is required.

## Design

### What's needed

1. Set `DISCORD_BOT_TOKEN` as a Cloudflare Workers secret
2. DM policy: `pairing` (default) - requires device pairing approval via admin UI
3. Deploy the worker

### How it works

1. Worker starts OpenClaw gateway in Cloudflare Sandbox
2. `start-openclaw.sh` detects `DISCORD_BOT_TOKEN` in env
3. Script patches OpenClaw config with `channels.discord` section:
   - `token`: Bot token
   - `enabled`: true
   - `dm.policy`: "pairing"
4. OpenClaw gateway connects to Discord via WebSocket as a bot
5. Users DM the bot -> pairing request created -> admin approves -> conversation begins

### Security

- `pairing` policy ensures only approved users can chat
- Bot token stored as Cloudflare Workers secret (encrypted at rest)
- Admin approval required for each new Discord user

### Existing code paths

- `src/types.ts`: `DISCORD_BOT_TOKEN` and `DISCORD_DM_POLICY` in `MoltbotEnv`
- `src/gateway/env.ts`: `buildEnvVars()` passes both vars to container
- `start-openclaw.sh`: Config patching logic at lines 246-259

## Steps

1. `npx wrangler secret put DISCORD_BOT_TOKEN`
2. `npm run deploy`
3. DM the bot on Discord
4. Approve the device via admin UI at `/_admin/`
