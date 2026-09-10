# MEGA_BOT Workflow Architecture

Complete step-by-step documentation of the Pipedream workflow, data flow, and system design.

## Workflow Steps

### Step 1: Telegram Trigger
**Type**: Telegram - "New Message Updates (Instant)"

**Input**:
- Telegram webhook/polling event
- Contains: `message`, `from`, `chat`, `update_id`, etc.

**Output** (`steps.trigger.event`):
```javascript
{
  update_id: 12345,
  message: {
    message_id: 67890,
    from: { id: 111, first_name: "John", ... },
    chat: { id: 111, type: "private", ... },
    date: 1234567890,
    text: "Hello bot",
    photo: [...],      // optional
    document: {...}    // optional
  }
}
```

---

### Step 2: Dedup Check
**File**: `steps/02-dedup_check.js`

**Purpose**: Prevent processing the same message twice (Telegram can send duplicates)

**Data Flow**:
- **Input**: `steps.trigger.event.update_id`
- **Check**: Look up `processed_{messageId}` in Data Store
- **Output**: `{ isDuplicate: false, messageId }`
- **Action**: If duplicate found, exit workflow with `$.flow.exit()`

**Data Store Keys Used**:
- `processed_{messageId}`: Boolean flag (true = already processed)

---

### Step 3: File Processor
**File**: `steps/03-file_processor.js`

**Purpose**: Extract and encode files (photos, documents) from messages

**Data Flow**:
- **Input**: `steps.trigger.event.message` (photo or document)
- **Process**:
  1. Extract `file_id` from photo or document
  2. Call Telegram API: `getFile` endpoint
  3. Download file from CDN URL
  4. Encode to base64
- **Output**: 
```javascript
{
  hasFile: true/false,
  text: "message text",
  fileData: {
    base64: "...",
    mimeType: "image/jpeg",
    fileName: "photo.jpg"
  }
}
```

---

### Step 4: Memory Loader
**File**: `steps/04-memory_loader.js`

**Purpose**: Load conversation history and user preferences

**Data Flow**:
- **Input**: `steps.trigger.event.message.from.id` (user ID)
- **Lookup**:
  - `{userId}_history`: Conversation history (array)
  - `{userId}_prefs`: User preferences (object)
- **Output**:
```javascript
{
  history: [
    { role: "user", content: "..." },
    { role: "assistant", content: "..." },
    ...
  ],
  prefs: { name: "John", language: "en" },
  userId: "111"
}
```

**Default Values**:
- History: `[]` (empty array)
- Prefs: `{ name: null, language: "en" }`

---

### Step 5: Intent Router
**File**: `steps/05-intent_router.js`

**Purpose**: Classify user intent and extract entities

**Classification Rules**:

| Intent | Trigger Keywords | Confidence | Use Case |
|--------|-----------------|------------|----------|
| `image` | Has photo/document | 0.95 | Image analysis/processing |
| `search` | "search", "find", "look up" | 0.8 | Web searches |
| `tool` | "github", "create issue" | 0.85 | External integrations |
| `tool` | "email", "mail", "send" | 0.85 | Gmail/email actions |
| `reminder` | "remind", "schedule" | 0.8 | Future tasks |
| `system` | "status", "help" | 0.9 | Bot info |
| `chat` | *default* | 0.5 | General conversation |

**Output**:
```javascript
{
  intent: "search",
  confidence: 0.8,
  entities: { query: "machine learning" },
  hasFile: false
}
```

**Fallback**: If confidence < 0.6, default to `chat`

---

### Step 6: Cost Check
**File**: `steps/06-cost_check.js`

**Purpose**: Enforce daily message limits and cost caps

**Limits**:
- **Daily Message Limit**: 50 messages per user per day
- **Daily Cost Limit**: $0.50 USD per user per day

**Data Flow**:
- **Input**: `steps.trigger.event.message.from.id` (user ID)
- **Lookup**: `{userId}_day_YYYY-MM-DD` from Data Store
- **Check**: 
  - If `usage.count >= 50` → Exit with "Daily message limit reached"
  - If `usage.cost >= 0.50` → Exit with "Daily cost limit reached"
- **Output**:
```javascript
{
  allowed: true,
  usage: { count: 5, cost: 0.0007 },
  dailyLimit: 50,
  costLimit: 0.50
}
```

**Data Store Schema**:
```javascript
{
  count: 10,        // messages sent today
  cost: 0.00140     // $ spent today
}
```

---

### Step 7: Web Search (Conditional)
**File**: `steps/07-web_search.js`

**Purpose**: Execute web search via Tavily API if intent is `search`

**Condition**: Only runs if `steps.intent_router.intent === "search"`

**Data Flow**:
- **Input**: 
  - Query: extracted from `steps.intent_router.entities.query`
  - API Key: `process.env.TAVILY_API_KEY`
- **Tavily API Call**:
  ```
  POST https://api.tavily.com/search
  {
    query: "machine learning",
    search_depth: "basic",
    max_results: 5
  }
  ```
- **Output**:
```javascript
{
  enabled: true,
  query: "machine learning",
  results: "1. Result Title\n   Content preview\n   Source: url\n\n2. ..."
}
```

**Fallback**: If API key missing or request fails, return `{ enabled: false }`

---

### Step 8: AI Router (Multi-Provider)
**File**: `steps/08-ai_router.js`

**Purpose**: Generate AI response with intelligent provider fallback

**Provider Chain**:

#### Primary: OpenRouter
```
POST https://openrouter.ai/api/v1/chat/completions
Model: meta-llama/llama-3.3-70b-instruct
Timeout: 15 seconds
```

#### Fallback: Groq
```
POST https://api.groq.com/openai/v1/chat/completions
Model: qwen/qwen3.6-27b
Timeout: 15 seconds
```

#### Final Fallback
```
Response: "I'm having trouble connecting. Please try again."
```

**Prompt Construction**:
```
{Recent conversation history (last 6 messages)}
User: {current message}