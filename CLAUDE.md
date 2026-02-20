# CLAUDE.md

## Project Overview

Cloudflare Workers + Sandbox 上で [OpenClaw](https://github.com/openclaw/openclaw) パーソナルAIアシスタントを実行するプロジェクト。

## Tech Stack

- **Runtime**: Cloudflare Workers + Cloudflare Sandbox (container)
- **Framework**: Hono (HTTP routing)
- **Language**: TypeScript (strict mode)
- **Build**: Vite + wrangler
- **Test**: Vitest (colocated test files `*.test.ts`)
- **Lint/Format**: oxlint + oxfmt
- **VCS**: Jujutsu (jj) - Git の代わりに jj コマンドを使うこと

## Project Structure

```
moltworker/
├── src/
│   ├── index.ts          # Main Hono app, route mounting
│   ├── types.ts          # TypeScript type definitions (MoltbotEnv etc.)
│   ├── config.ts         # Constants (ports, timeouts)
│   ├── auth/             # Cloudflare Access JWT 認証
│   ├── gateway/          # OpenClaw gateway 管理 (process, env, r2, sync)
│   ├── routes/           # API route handlers (api, admin-ui, debug, cdp, public)
│   └── client/           # React admin UI (Vite)
├── start-openclaw.sh     # Container startup script
├── wrangler.jsonc        # Cloudflare Worker config
└── package.json
```

## Commands

```bash
cd moltworker
npm test              # Run tests (vitest)
npm run test:watch    # Watch mode
npm run typecheck     # TypeScript check
npm run lint          # oxlint
npm run format:check  # oxfmt check
npm run build         # Build worker + client
npm run deploy        # Build and deploy
npm run dev           # Vite dev server
npm run start         # wrangler dev (local)
```

## Key Patterns

### Environment Variables

- `MoltbotEnv` interface in `src/types.ts` が全環境変数の定義
- Container に渡す変数は `src/gateway/env.ts` の `buildEnvVars()` で管理
- 新しい環境変数を追加する場合: types.ts -> env.ts -> .dev.vars.example -> README.md

### OpenClaw CLI

Container 内の CLI 呼び出しには `--url ws://localhost:18789` が必須:
```typescript
sandbox.startProcess('openclaw devices list --json --url ws://localhost:18789')
```
CLI は WebSocket 接続のオーバーヘッドで 10-15 秒かかる。`waitForProcess()` を使用。

### Chat Channels

Discord/Telegram/Slack は環境変数設定のみで有効化:
- `DISCORD_BOT_TOKEN` -> `start-openclaw.sh` が `channels.discord` を自動設定
- DM policy: `pairing` (default, デバイスペアリング必要) or `open`

### Testing

テストファイルはソースと同じディレクトリに配置 (`*.test.ts`)。新機能追加時は対応するテストを追加すること。

### Code Style

- TypeScript strict mode
- 関数シグネチャには明示的な型を使う
- Route handler は薄く保ち、ロジックは別モジュールに抽出
- Hono の `c.json()`, `c.html()` を使う

## Architecture

```
Browser/Discord/Telegram
   │
   ▼
┌─────────────────────────────────────┐
│     Cloudflare Worker (index.ts)    │
│  - Starts OpenClaw in sandbox       │
│  - Proxies HTTP/WebSocket requests  │
│  - Passes secrets as env vars       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│     Cloudflare Sandbox Container    │
│  ┌───────────────────────────────┐  │
│  │     OpenClaw Gateway          │  │
│  │  - Control UI on port 18789   │  │
│  │  - WebSocket RPC protocol     │  │
│  │  - Agent runtime              │  │
│  │  - Chat channel connections   │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```
