# Step 12: Tools Execution

**Purpose**: Enable the AI to execute actions in external systems (GitHub, Gmail, Google Calendar, system commands) with user confirmation for destructive actions.

## Overview

This step intercepts structured tool requests from the AI, validates them, optionally prompts for user confirmation, and executes them safely.

### Tool Categories

#### 1. GitHub
- **Create Issue**: `POST /repos/{owner}/{repo}/issues`
- **List Repos**: `GET /user/repos`
- **Read File**: `GET /repos/{owner}/{repo}/contents/{path}`
- **Create PR**: Create branch + file changes + open PR

#### 2. Gmail
- **Read Inbox**: `GET /gmail/v1/users/me/messages` (read-only for now)
- **Draft Email**: `POST /gmail/v1/users/me/drafts` (creates draft only, requires confirmation to send)

#### 3. Google Calendar
- **List Events**: `GET /calendar/v3/calendars/{calendarId}/events`
- **Create Event**: `POST /calendar/v3/calendars/{calendarId}/events` (with confirmation)

#### 4. System
- **Restart Bot**: Restarts the Pipedream workflow
- **Check Status**: Returns workflow health metrics
- **View Logs**: Last N log entries (whitelisted only)

#### 5. Web Search
- Integrated in Step 7 (web_search)

## Entity Schema

```javascript
{
  tool: "github" | "gmail" | "calendar" | "system" | "web_search",
  action: "create_issue" | "list_repos" | "read_file" | ... ,
  params: {
    // Tool-specific params
  },
  requires_confirmation: boolean,
  reason: string  // Why this action is needed
}
```

## Confirmation Flow

```
AI generates tool request
     ↓
Tool is marked requires_confirmation: true?
     ↓ YES
Send to user: "Allow this? Reply 'confirm <action_id>' to proceed"
     ↓
User replies "confirm <action_id>"?
     ↓ YES
Execute tool, return result to AI
     ↓ NO or timeout
Return "Action cancelled by user" to AI
     ↓
AI generates revised response
```

### Destructive Actions (Always Require Confirmation)
- GitHub: Create/delete issues, push code
- Gmail: Send email (draft-only is safe)
- Calendar: Create/modify events
- System: Restart, kill process

### Safe Actions (No Confirmation)
- GitHub: Read repos, read files, list issues
- Gmail: Read inbox
- Calendar: List events

## Implementation

### Input Format (From AI)

The AI includes structured tool requests in its response:

```markdown
I can help with that. Let me create a GitHub issue:

<tool_request>
{
  "tool": "github",
  "action": "create_issue",
  "params": {
    "owner": "mastermindrna-hub",
    "repo": "mega-bot",
    "title": "Add voice message support",
    "body": "Users should be able to send voice messages for transcription"
  }
}
</tool_request>

Once confirmed, this issue will be created.
```

### Output Format (To AI)

After tool execution, inject result back into AI context:

```javascript
{
  tool: "github",
  action: "create_issue",
  success: true,
  result: {
    issue_number: 42,
    url: "https://github.com/mastermindrna-hub/mega-bot/issues/42"
  },
  executed_at: "2026-09-10T14:30:00Z"
}
```

Or on failure:

```javascript
{
  tool: "github",
  action: "create_issue",
  success: false,
  error: "Invalid credentials: GitHub token expired",
  executed_at: "2026-09-10T14:30:00Z"
}
```

## Tool-Specific Details

### GitHub

**Create Issue**
```javascript
{
  "tool": "github",
  "action": "create_issue",
  "params": {
    "owner": "mastermindrna-hub",
    "repo": "mega-bot",
    "title": "string",
    "body": "string",
    "labels": ["bug", "feature"],
    "assignees": ["username"]
  },
  "requires_confirmation": true
}
```

**List Repos**
```javascript
{
  "tool": "github",
  "action": "list_repos",
  "params": {
    "owner": "mastermindrna-hub",
    "type": "all" | "owner" | "public" | "private"
  },
  "requires_confirmation": false
}
```

**Read File**
```javascript
{
  "tool": "github",
  "action": "read_file",
  "params": {
    "owner": "mastermindrna-hub",
    "repo": "mega-bot",
    "path": "steps/01-trigger.md",
    "ref": "main"
  },
  "requires_confirmation": false
}
```

### Gmail

**Read Inbox** (No confirmation)
```javascript
{
  "tool": "gmail",
  "action": "read_inbox",
  "params": {
    "max_results": 10,
    "query": "is:unread"  // Optional Gmail query
  },
  "requires_confirmation": false
}
```

**Draft Email** (No send; requires confirmation to send)
```javascript
{
  "tool": "gmail",
  "action": "draft_email",
  "params": {
    "to": "recipient@example.com",
    "subject": "string",
    "body": "string"
  },
  "requires_confirmation": false  // Draft only, safe
}
```

### Google Calendar

**List Events** (No confirmation)
```javascript
{
  "tool": "calendar",
  "action": "list_events",
  "params": {
    "calendar_id": "primary",
    "time_min": "2026-09-10T00:00:00Z",
    "time_max": "2026-09-11T00:00:00Z",
    "max_results": 10
  },
  "requires_confirmation": false
}
```

**Create Event** (Requires confirmation)
```javascript
{
  "tool": "calendar",
  "action": "create_event",
  "params": {
    "calendar_id": "primary",
    "title": "string",
    "start": "2026-09-10T14:00:00Z",
    "end": "2026-09-10T15:00:00Z",
    "description": "string"
  },
  "requires_confirmation": true
}
```

### System

**Restart Bot** (Requires confirmation)
```javascript
{
  "tool": "system",
  "action": "restart_workflow",
  "params": {},
  "requires_confirmation": true
}
```

**Check Status** (No confirmation)
```javascript
{
  "tool": "system",
  "action": "check_status",
  "params": {},
  "requires_confirmation": false
}
```

Result:
```javascript
{
  "workflow_active": true,
  "last_message": "2026-09-10T14:29:00Z",
  "memory_store_size_bytes": 51200,
  "api_calls_today": 127,
  "cost_used_today": 0.34
}
```

## Error Handling

### Validation Errors
- Missing required params → Return "Invalid tool request" to AI
- Unknown tool/action → Return "Tool not found"
- Expired credentials → Return "Authentication failed"

### Execution Errors
- API rate limit → Return "Rate limit exceeded, retry in 60s"
- Network timeout → Return "API request timed out"
- Permission denied → Return "Insufficient permissions"

### Fallback
If a critical tool fails, log it and inform the AI:
```javascript
{
  "success": false,
  "error": "Tool execution failed",
  "reason": "GitHub API rate limit exceeded (60/60)",
  "retry_after_seconds": 3600
}
```

## Security Considerations

1. **API Key Rotation**: Store in Pipedream environment variables, rotate quarterly
2. **Whitelisting**: Only execute whitelisted tool/action pairs
3. **Scope Limiting**: GitHub tokens should have minimal required scopes
4. **Audit Trail**: Log all tool executions with user, action, timestamp, result
5. **Rate Limiting**: Throttle tool calls per user (max 5 tool calls per minute)

## Testing

- Test each tool with valid credentials in a safe repo
- Mock API responses for CI/CD testing
- Verify confirmation flow with test messages
- Test failure scenarios (expired token, rate limit, timeout)

## Future Enhancements

- Tool chaining (result of one tool → input to next)
- Custom tool registration by users
- Tool usage analytics and cost tracking
- Scheduled tool execution (e.g., daily email digest)
