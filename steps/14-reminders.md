# Step 14: Scheduled Reminders

**Purpose**: Enable users to set natural-language reminders that are delivered at the scheduled time via Telegram.

## Overview

```
User: "remind me in 30 minutes to check email"
     ↓
Intent Router classifies as "reminder"
     ↓
Reminder Parser extracts: { deltaMinutes: 30, text: "check email" }
     ↓
Store reminder in Data Store: { userId, remindAt, message, status }
     ↓
Background scheduler checks every minute
     ↓
If remindAt <= now, send reminder to user
     ↓
Update status to "delivered" or "failed"
```

## Reminder Detection (Modify Step 05: intent_router)

Add pattern matching for reminder intents:

```javascript
const reminderPatterns = [
  /remind\s+me\s+(?:in|after)\s+(\d+)\s+(minute|hour|day)s?(?:\s+to\s+(.+))?/i,
  /set\s+(?:a\s+)?reminder(?:\s+in|\s+for)?\s+(\d+)\s+(minute|hour|day)s?(?:\s+to\s+(.+))?/i,
  /(?:don't let me forget|remember)\s+(?:to\s+)?(.+)(?:\s+in\s+(\d+)\s+(minute|hour|day)s?)?/i,
  /remind\s+me\s+(?:to\s+)?(.+)\s+(?:in|at)\s+(\d+)\s+(minute|hour|day)s?/i,
  /what's?\s+(?:my|the)\s+(?:reminder|reminders)\?/i,
  /(?:list|show)\s+(?:my\s+)?reminders/i,
  /cancel\s+reminder\s+(\d+)/i,
  /(?:clear|delete)\s+(?:all\s+)?reminders/i
];

function detectReminderIntent(userMessage) {
  for (const pattern of reminderPatterns) {
    const match = userMessage.match(pattern);
    if (match) {
      return {
        intent: "reminder",
        type: detectReminderType(userMessage, match),
        confidence: 0.95
      };
    }
  }
  return null;
}

function detectReminderType(message, match) {
  if (/what's|list|show/.test(message)) return "list";
  if (/cancel/.test(message)) return "cancel";
  if (/clear|delete.*all/.test(message)) return "clear_all";
  return "set";
}
```

## Reminder Parser

Extract structured data from natural language:

```javascript
function parseReminderRequest(userMessage) {
  // Match: "remind me in 30 minutes to check email"
  const timeMatch = userMessage.match(
    /remind\s+me\s+(?:in|after)\s+(\d+)\s+(minute|hour|day)s?/i
  );

  if (!timeMatch) {
    return { error: "Could not parse reminder time" };
  }

  const delta = parseInt(timeMatch[1]);
  const unit = timeMatch[2].toLowerCase();

  // Convert to minutes
  const minutesDelta = {
    minute: delta,
    hour: delta * 60,
    day: delta * 24 * 60
  }[unit];

  // Extract reminder text
  const textMatch = userMessage.match(/(?:to|reminder:?)\s+(.+?)(?:\s*\.)?$/i);
  const reminderText = textMatch ? textMatch[1].trim() : "No details provided";

  return {
    success: true,
    deltaMinutes: minutesDelta,
    reminderText,
    scheduledFor: new Date(Date.now() + minutesDelta * 60000)
  };
}
```

## Data Schema

Store reminders in Pipedream Data Store:

```javascript
// Key: `reminder_${reminderId}` (unique ID)
{
  id: "reminder_1234567890",
  userId: 6271263631,        // Telegram chat ID
  message: "Check email",
  remindAt: "2026-09-10T15:30:00Z",  // ISO 8601
  createdAt: "2026-09-10T15:00:00Z",
  status: "pending" | "delivered" | "failed",
  deliveryAttempts: 0
}

// Key: `reminders_list_${userId}`
// Value: [list of reminder IDs for this user, max 100]
```

## Step: Reminder Handler

Intercepts reminder intents and processes them:

```javascript
module.exports = defineComponent({
  async run({ steps }) {
    const intentRouter = steps.intent_router;

    if (intentRouter.intent !== "reminder") {
      return { skipped: true };
    }

    const userMessage = steps.trigger.event.message.text;
    const userId = steps.trigger.event.message.chat.id;
    const $db = pd.data;

    let response;

    if (intentRouter.type === "set") {
      // Parse the reminder
      const parsed = parseReminderRequest(userMessage);

      if (parsed.error) {
        response = {
          intent: "reminder",
          type: "set",
          success: false,
          message: `I couldn't parse that. Try: "remind me in 30 minutes to check email"`,
          pendingReminders: 0
        };
      } else {
        // Store in Data Store
        const reminderId = `reminder_${userId}_${Date.now()}`;
        await $db.set(reminderId, {
          id: reminderId,
          userId,
          message: parsed.reminderText,
          remindAt: parsed.scheduledFor.toISOString(),
          createdAt: new Date().toISOString(),
          status: "pending",
          deliveryAttempts: 0
        });

        // Add to user's reminder list
        const listKey = `reminders_list_${userId}`;
        const existing = (await $db.get(listKey)) || [];
        existing.push(reminderId);
        await $db.set(listKey, existing.slice(-100));  // Keep last 100

        response = {
          intent: "reminder",
          type: "set",
          success: true,
          message: `✅ Reminder set: "${parsed.reminderText}" in ${parsed.deltaMinutes} minute${parsed.deltaMinutes > 1 ? 's' : ''}`,
          reminderId,
          remindAt: parsed.scheduledFor.toISOString(),
          pendingReminders: existing.length + 1
        };
      }
    } else if (intentRouter.type === "list") {
      // List all reminders for this user
      const listKey = `reminders_list_${userId}`;
      const reminderIds = (await $db.get(listKey)) || [];
      const reminders = [];

      for (const id of reminderIds) {
        const reminder = await $db.get(id);
        if (reminder && reminder.status === "pending") {
          reminders.push(reminder);
        }
      }

      if (reminders.length === 0) {
        response = {
          intent: "reminder",
          type: "list",
          message: "You have no pending reminders.",
          reminders: []
        };
      } else {
        let list = "📋 Your reminders:\n\n";
        reminders.forEach((r, i) => {
          const time = new Date(r.remindAt).toLocaleString();
          list += `${i + 1}. "${r.message}" at ${time}\n`;
        });

        response = {
          intent: "reminder",
          type: "list",
          message: list,
          reminders,
          count: reminders.length
        };
      }
    } else if (intentRouter.type === "cancel") {
      // Cancel a specific reminder
      const match = userMessage.match(/cancel\s+reminder\s+(\d+)/i);
      if (!match) {
        response = {
          intent: "reminder",
          type: "cancel",
          success: false,
          message: "Please specify which reminder to cancel. Reply with: 'cancel reminder <number>'"
        };
      } else {
        // Simplified: cancel by user's reminder index
        const index = parseInt(match[1]) - 1;
        const listKey = `reminders_list_${userId}`;
        const reminderIds = (await $db.get(listKey)) || [];

        if (index < 0 || index >= reminderIds.length) {
          response = {
            intent: "reminder",
            type: "cancel",
            success: false,
            message: `Reminder #${index + 1} not found.`
          };
        } else {
          const reminderId = reminderIds[index];
          const reminder = await $db.get(reminderId);

          await $db.set(reminderId, {
            ...reminder,
            status: "cancelled"
          });

          response = {
            intent: "reminder",
            type: "cancel",
            success: true,
            message: `✅ Cancelled reminder: "${reminder.message}"`,
            reminderId
          };
        }
      }
    } else if (intentRouter.type === "clear_all") {
      // Clear all reminders
      const listKey = `reminders_list_${userId}`;
      const reminderIds = (await $db.get(listKey)) || [];

      for (const id of reminderIds) {
        const reminder = await $db.get(id);
        if (reminder) {
          await $db.set(id, { ...reminder, status: "cancelled" });
        }
      }

      await $db.set(listKey, []);

      response = {
        intent: "reminder",
        type: "clear_all",
        success: true,
        message: `✅ Cleared ${reminderIds.length} reminder${reminderIds.length !== 1 ? 's' : ''}.`
      };
    }

    return response;
  }
});
```

## Background Scheduler

### Pipedream Scheduled Trigger

Create a separate workflow with a **Scheduled Trigger** (every 1 minute):

```javascript
// Scheduled Workflow: Check Reminders (runs every 1 minute)

module.exports = defineComponent({
  async run({ steps }) {
    const $db = pd.data;
    const now = new Date();

    // Get all reminder keys
    const allKeys = await $db.keys();
    const reminderKeys = allKeys.filter(k => k.startsWith('reminder_'));

    let sent = 0;
    let failed = 0;

    for (const key of reminderKeys) {
      const reminder = await $db.get(key);

      // Skip if not pending
      if (reminder.status !== "pending") continue;

      const remindAt = new Date(reminder.remindAt);

      // Check if it's time to send
      if (remindAt <= now) {
        // Send reminder via Telegram
        try {
          await axios.post(
            `https://api.telegram.org/bot${process.env.TELEGRAM_BOT_TOKEN}/sendMessage`,
            {
              chat_id: reminder.userId,
              text: `⏰ Reminder: ${reminder.message}`,
              parse_mode: "HTML"
            }
          );

          // Mark as delivered
          await $db.set(key, {
            ...reminder,
            status: "delivered",
            deliveredAt: now.toISOString()
          });

          sent++;
        } catch (error) {
          // Retry up to 3 times
          if (reminder.deliveryAttempts < 3) {
            await $db.set(key, {
              ...reminder,
              deliveryAttempts: reminder.deliveryAttempts + 1
            });
          } else {
            await $db.set(key, {
              ...reminder,
              status: "failed",
              error: error.message
            });
          }

          failed++;
        }
      }
    }

    return {
      checked: reminderKeys.length,
      sent,
      failed,
      timestamp: now.toISOString()
    };
  }
});
```

## Integration with Intent Router

Modify `intent_router` to classify reminder intents:

```javascript
// In step 05: intent_router

const intentClassifier = async (userMessage) => {
  // Check reminder patterns first
  const reminderDetection = detectReminderIntent(userMessage);
  if (reminderDetection) {
    return reminderDetection;
  }

  // Fall through to other intent types
  // ...
};
```

## Reminder Commands

Users can interact with reminders via these commands:

| Command | Example | Behavior |
|---------|---------|----------|
| Set | "remind me in 30 minutes to check email" | Create new reminder |
| Set | "set a reminder to call mom in 2 hours" | Create new reminder |
| List | "what are my reminders?" | Show all pending reminders |
| Cancel | "cancel reminder 2" | Cancel specific reminder by number |
| Clear | "clear all reminders" | Delete all pending reminders |

## Response Examples

**Set Reminder**:
```
✅ Reminder set: "check email" in 30 minutes
```

**List Reminders**:
```
📋 Your reminders:

1. "check email" at 3:30 PM
2. "call mom" at 5:00 PM
3. "review PRs" at 6:00 PM
```

**Reminder Triggered**:
```
⏰ Reminder: check email
```

## Environment Variables

```bash
# Reminder settings
REMINDER_CHECK_INTERVAL=1  # minutes
REMINDER_MAX_PER_USER=100
REMINDER_RETENTION_DAYS=30  # Keep delivered reminders for 30 days before cleanup
```

## Storage Efficiency

- Keep max 100 reminders per user
- Clean up "delivered" and "failed" reminders older than 30 days
- Use Pipedream Data Store (free tier: 512MB total, plenty for this)

## Limitations

- **No recurring reminders**: Each reminder is one-time only
- **Minute precision**: Scheduled to nearest minute (not second-level accuracy)
- **No timezone support**: Uses server time (UTC)
- **No persistent delivery**: If bot restarts, reminders may delay up to 1 minute

## Future Enhancements

- Recurring reminders ("remind me every Monday at 9 AM")
- Timezone support (detect user timezone from Telegram)
- Snooze feature ("snooze for 5 minutes")
- Reminder categories / tags
- Completion tracking ("mark reminder as done")
