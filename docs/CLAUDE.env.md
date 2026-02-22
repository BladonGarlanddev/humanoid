# Environment Variables Reference

## Backend (`backend/.env`)

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `5000` | NestJS API server port |
| `DATABASE_URL` | — | Postgres connection string (required) |
| `REDIS_HOST` | `localhost` | Bull queue / ioredis host |
| `REDIS_PORT` | `6379` | Bull queue / ioredis port |
| `OPENAI_API_KEY` | — | LLM calls via LangChain |
| `ANTHROPIC_API_KEY` | — | Anthropic models via LangChain |
| `FIREWORKS_API_KEY` | — | Fireworks models via LangChain |
| `COHERE_API_KEY` | — | Cohere models via LangChain |
| `TELEGRAM_BOT_TOKEN` | — | Telegram bot token for one-way notifications |
| `TELEGRAM_CHAT_ID` | — | Target group/chat ID for Telegram notifications |

## Orchestrator Electron (`orchestrator/orchestrator-electron/.env`)

| Variable | Default | Purpose |
|---|---|---|
| `VITE_API_URL` | `http://localhost:5000` | Backend URL used by renderer |

## Visualizer (`visualizer/.env` or shell)

| Variable | Default | Purpose |
|---|---|---|
| `BACKEND_BASE_URL` | `http://localhost:5000` | Backend API for fetching corpora/nodes |

## AI training scripts (`AI/.env`)

| Variable | Default | Purpose |
|---|---|---|
| `BACKEND_URL` | `http://localhost:5000` | Backend API for training data |

## Rules
- Never rename env vars — they are public contracts referenced across processes.
- Add new vars with sensible defaults where possible; document them here.
- `.env` files are gitignored; never commit secrets.
