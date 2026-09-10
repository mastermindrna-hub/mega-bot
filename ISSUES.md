# Known Issues & Status

## ✅ Resolved (v0.8.0)

### Bot Connection Mismatch
- **Issue**: The `telegram_reply` step was connected to old bot (`MEGABOT`, ID `8793994819`) while trigger used new bot (`mega_chief_2026_bot`, ID `8931765471`)
- **Resolved**: 10 Sept 2026
- **Fix**: Reconnected `telegram_reply` step to `mega_chief_2026_bot`

### Chat ID Confusion
- **Issue**: Passing bot's own ID (`8793994819`) as Chat ID instead of user's ID (`6271263631`)
- **Resolved**: 10 Sept 2026
- **Fix**: Updated Chat ID to user's Telegram ID

### Trigger Source Disabled
- **Issue**: Telegram trigger source was disabled, causing 0/11 steps to execute
- **Resolved**: 10 Sept 2026
- **Fix**: Clicked "Turn it on" in Pipedream warning banner

### AI Hallucinating System Metrics
- **Issue**: AI invented fake data when asked about "problems in workflow" (CPU 87%, fake error rates, etc.)
- **Resolved**: 10 Sept 2026
- **Fix**: Added system prompt guard preventing metric fabrication

### Groq Model Decommissioning
- **Issue**: Models `llama-3.3-70b-versatile`, `llama-3.1-70b-versatile`, `llama-3.2-90b-vision-preview`, `gemma2-9b-it` all removed
- **Resolved**: 10 Sept 2026
- **Fix**: Verified via Groq `/models` endpoint; switched to `openai/gpt-oss-120b`

### OpenRouter `:free` Model Pulled
- **Issue**: `meta-llama/llama-3.3-70b-instruct:free` made paid-only
- **Resolved**: 10 Sept 2026
- **Fix**: Switched to paid slug `meta-llama/llama-3.3-70b-instruct`

---

## 🚧 In Progress (v0.9.0)

### Groq Fallback Untested End-to-End
- **Impact**: High — fallback path critical for reliability
- **Status**: `provider_used` always shows `openrouter` because it never fails
- **Next Steps**: Break `OPENROUTER_API_KEY` temporarily to verify Groq path works
- **Target**: v0.9.0

### Data Store Token Exposure
- **Impact**: Medium — security risk
- **Issue**: Live JWT for `jarvis_memory` was pasted in chat logs
- **Status**: Identified, not yet rotated
- **Next Steps**: Rotate Data Store credentials
- **Target**: v0.8.1 (hotfix)

### Cost Cap Not Yet Hit
- **Impact**: Medium — feature untested in production
- **Issue**: $0.50/day limit has never been reached
- **Status**: Scaffolding in place, behavior unverified
- **Next Steps**: Integration test with real-world load
- **Target**: v0.9.0

### Image Analysis Incomplete
- **Impact**: Medium — core feature blocked
- **Issue**: Vision path scaffolded but not fully tested
- **Status**: Waiting for Gemini API integration
- **Next Steps**: Implement image download + Gemini Vision API call
- **Target**: v0.9.0

### Multi-User Support Missing
- **Impact**: Low — MVP constraint
- **Issue**: Chat ID is hardcoded (6271263631)
- **Status**: Will require significant refactor
- **Next Steps**: Parameterize user ID; test with multiple users
- **Target**: v1.0.0

---

## 📋 Planned (v1.0.0)

### Web Search End-to-End Testing
- Trigger search intent classification
- Verify Tavily API returns results
- Confirm results injected into AI prompt
- Test with various query types

### Voice Message Support
- Implement Whisper transcription (OpenAI or local)
- Handle .ogg format download
- Feed transcript into text pipeline
- Optional: Voice reply synthesis

### Tool Execution System
- Implement GitHub integration (create issues, read files)
- Implement Gmail integration (read inbox, draft emails)
- Implement Google Calendar (list events, create events)
- Add confirmation flow for destructive actions

### Scheduled Reminders
- Natural language parsing ("remind me in 30 minutes")
- Store reminders in Data Store
- Background scheduler (1-minute check interval)
- Delivery via Telegram

### Dashboard (Optional)
- View conversation history
- Check usage stats and cost
- Manage reminders and tools
- Analytics

---

## 🔮 Future (v2.0.0)

- Multi-workspace support
- Custom skills/plugins system
- Proactive daily briefings
- Tool usage analytics
- Self-healing (auto-retry, circuit breakers)
- Cost analytics dashboard
- API for third-party integrations

---

## How to Report Issues

If you encounter a bug:

1. Check this file first — it may already be known
2. If new, create a GitHub Issue with:
   - **Title**: Brief summary
   - **Description**: What happened? What did you expect?
   - **Steps to reproduce**
   - **Logs** (from Pipedream event history)
   - **Environment**: Bot name, date/time, Telegram user ID (if relevant)

3. Assign label: `bug`, `feature`, `question`, or `documentation`
