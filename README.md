# MEGA_BOT 🤖

An advanced AI assistant built on **Pipedream**, using **Telegram** as the interface. Multi-provider AI backends, persistent memory, cost controls, web search integration, and image processing capabilities.

## Features

- **Multi-Provider AI**: Intelligent fallback chain (OpenRouter → Groq → fallback)
- **Persistent Memory**: Conversation history stored per user in Pipedream Data Store
- **Cost Controls**: Daily message limits and cost caps to prevent budget overruns
- **Web Search**: Tavily API integration for real-time information retrieval
- **Image Processing**: Support for photo and document uploads (in progress)
- **Intent Recognition**: Automatic classification of user requests (chat, search, image, tool, reminder, system)
- **Deduplication**: Message ID tracking to prevent duplicate processing
- **Debug Mode**: Optional verbose logging for troubleshooting

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      TELEGRAM USER                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  01: TELEGRAM TRIGGER                                            │
│  (New Message Updates - Instant)                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  02: DEDUP_CHECK                                                 │
│  (Check if message already processed)                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  03: FILE_PROCESSOR                                              │
│  (Extract photos/documents from message)                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  04: MEMORY_LOADER                                               │
│  (Load conversation history & user preferences)                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  05: INTENT_ROUTER                                               │
│  (Classify intent: chat/search/image/tool/reminder/system)       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  06: COST_CHECK                                                  │
│  (Verify daily limits & cost caps)                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  07: WEB_SEARCH (conditional)                                    │
│  (Tavily API if intent=search)                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  08: AI_ROUTER                                                   │
│  (OpenRouter → Groq → Fallback)                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  09: MEMORY_SAVER                                                │
│  (Save conversation history & update usage metrics)              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  10: TELEGRAM_REPLY                                              │
│  (Send reply back to user)                                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  11: DEDUP_MARK                                                  │
│  (Mark message as processed)                                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
                    ✅ Workflow Complete
```

## Setup Instructions

### 1. Prerequisites

- Pipedream account (free tier works)
- Telegram Bot Token from [@BotFather](https://t.me/botfather)
- API keys for:
  - OpenRouter (primary AI provider)
  - Groq (fallback AI provider)
  - Tavily (web search)

### 2. Environment Variables

Create a `.env` file or set these in Pipedream:

```env
# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token_here

# AI Providers
OPENROUTER_API_KEY=your_openrouter_key
GROQ_API_KEY=your_groq_key

# Search
TAVILY_API_KEY=your_tavily_key

# Optional
DEBUG_MODE=false
```

### 3. Pipedream Setup

1. Create a new workflow in Pipedream
2. Add a **Telegram** trigger ("New Message Updates - Instant")
3. Paste your bot token in the trigger config
4. Add a **Data Store** for persistent memory
5. Copy each step from the `/steps/` directory into your workflow
6. Configure environment variables in Pipedream settings
7. Deploy and test!

### 4. Telegram Bot Setup

1. Message [@BotFather](https://t.me/botfather) on Telegram
2. Send `/newbot`
3. Give your bot a name and username
4. Copy the bot token
5. Set webhook URL in Pipedream (or use polling)

## Provider Chain Logic

The AI router uses an intelligent fallback mechanism:

```
Try OpenRouter (meta-llama/llama-3.3-70b-instruct)
    ↓
    └─ on failure → Try Groq (qwen/qwen3.6-27b)
         ↓
         └─ on failure → Return fallback message
```

Each provider call has a 15-second timeout. If both fail, the user receives a friendly error message.

## Memory Schema

The Pipedream Data Store uses these keys per user:

- `{userId}_history`: Array of conversation messages (max 50 stored)
- `{userId}_prefs`: User preferences (name, language)
- `{userId}_day_YYYY-MM-DD`: Daily usage stats (count, cost)
- `processed_{messageId}`: Deduplication flag

## Current Status

- ✅ **Text chat**: Fully working
- ✅ **Memory system**: Conversation history + preferences
- ✅ **Cost caps**: Daily limits enforced
- ✅ **Multi-provider fallback**: OpenRouter → Groq
- 🚧 **Web search**: Tavily integration in progress
- 🚧 **Image processing**: Document/photo upload framework ready, vision model integration pending

## Known Issues

See [ISSUES.md](./ISSUES.md) for:
- Telegram "chat not found" errors
- Groq model decommissioning
- Vision model single point of failure
- Cost cap validation gaps

## License

MIT License - See LICENSE file for details

---

**Questions?** Check [architecture.md](./architecture.md) for detailed workflow documentation or [CHANGELOG.md](./CHANGELOG.md) for version history.
