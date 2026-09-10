# Step 13: Voice Message Processing

**Purpose**: Handle voice messages from Telegram, transcribe them using OpenAI Whisper or local Whisper, and feed the transcript into the normal text pipeline.

## Overview

```
Voice Message from Telegram
     ↓
Download .ogg file
     ↓
Transcribe via Whisper API
     ↓
Parse transcript text
     ↓
Inject into normal text chat pipeline (steps 02–11)
     ↓
Send text reply + optional voice synthesis
```

## Voice Message Detection

In Telegram, voice messages are identified by:
- `message.voice` object present in the incoming update
- File ID in `message.voice.file_id`
- Duration in `message.voice.duration` (seconds)

### Filter Condition (Add to Step 03: file_processor)

```javascript
// In file_processor step, check:
const hasVoice = steps.trigger.event.message && steps.trigger.event.message.voice;

// If true, add to processing queue:
if (hasVoice) {
  return {
    hasFile: true,
    fileType: "voice",
    fileId: message.voice.file_id,
    duration: message.voice.duration,
    mimeType: "audio/ogg"
  };
}
```

## Voice Message Download

### Telegram File Download Flow

```javascript
const axios = require('axios');

async function downloadVoiceMessage(fileId, botToken) {
  try {
    // Step 1: Get file path from Telegram
    const response = await axios.get(
      `https://api.telegram.org/bot${botToken}/getFile`,
      { params: { file_id: fileId } }
    );

    if (!response.data.ok) {
      throw new Error(`Telegram error: ${response.data.description}`);
    }

    const filePath = response.data.result.file_path;
    
    // Step 2: Download file
    const fileUrl = `https://api.telegram.org/file/bot${botToken}/${filePath}`;
    const audioData = await axios.get(fileUrl, {
      responseType: 'arraybuffer',
      timeout: 30000
    });

    return {
      buffer: audioData.data,
      format: 'ogg',
      size: audioData.data.length
    };
  } catch (error) {
    return {
      error: `Failed to download voice: ${error.message}`
    };
  }
}
```

## Transcription Options

### Option A: OpenAI Whisper API (Recommended)

**Pros**: Accurate, supports multiple languages, no local inference needed

**Cons**: Requires API key, costs per minute of audio

```javascript
const FormData = require('form-data');
const fs = require('fs');

async function transcribeWithWhisper(audioBuffer, apiKey) {
  try {
    const form = new FormData();
    form.append('file', Buffer.from(audioBuffer), 'audio.ogg');
    form.append('model', 'whisper-1');
    form.append('language', 'en');  // Optional: set language

    const response = await axios.post(
      'https://api.openai.com/v1/audio/transcriptions',
      form,
      {
        headers: {
          ...form.getHeaders(),
          'Authorization': `Bearer ${apiKey}`
        },
        timeout: 60000
      }
    );

    return {
      success: true,
      text: response.data.text,
      duration: audioBuffer.length  // Approximate
    };
  } catch (error) {
    return {
      success: false,
      error: `Whisper API error: ${error.message}`
    };
  }
}
```

### Option B: Local Whisper (Self-Hosted)

**Pros**: No API costs, fully private, runs locally on Pipedream

**Cons**: Slower, higher latency, requires model download

```bash
# Install locally:
pip install openai-whisper
```

```javascript
const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');

async function transcribeWithLocalWhisper(audioBuffer, model = 'base') {
  try {
    // Write buffer to temp file
    const tempFile = `/tmp/audio_${Date.now()}.ogg`;
    fs.writeFileSync(tempFile, audioBuffer);

    return new Promise((resolve, reject) => {
      const whisper = spawn('whisper', [
        tempFile,
        '--model', model,
        '--output_format', 'json',
        '--language', 'en'
      ]);

      let stdout = '';
      let stderr = '';

      whisper.stdout.on('data', (data) => {
        stdout += data.toString();
      });

      whisper.stderr.on('data', (data) => {
        stderr += data.toString();
      });

      whisper.on('close', (code) => {
        try {
          fs.unlinkSync(tempFile);
          
          if (code === 0) {
            const jsonFile = tempFile.replace('.ogg', '.json');
            const result = JSON.parse(fs.readFileSync(jsonFile, 'utf-8'));
            fs.unlinkSync(jsonFile);
            
            resolve({
              success: true,
              text: result.text
            });
          } else {
            reject(new Error(`Whisper exited with code ${code}: ${stderr}`));
          }
        } catch (error) {
          reject(error);
        }
      });
    });
  } catch (error) {
    return {
      success: false,
      error: `Local Whisper error: ${error.message}`
    };
  }
}
```

## Integration: New Step (Add After Step 03)

### Step 03b: Voice Transcriber

```javascript
// Triggered when file_processor detects voice message

module.exports = defineComponent({
  async run({ steps }) {
    const { OPENAI_API_KEY, TRANSCRIBE_METHOD } = process.env;
    const fileInfo = steps.file_processor;

    if (!fileInfo.hasFile || fileInfo.fileType !== 'voice') {
      return { skipped: true };
    }

    // Download voice from Telegram
    const download = await downloadVoiceMessage(
      fileInfo.fileId,
      process.env.TELEGRAM_BOT_TOKEN
    );

    if (download.error) {
      return { error: download.error };
    }

    // Transcribe
    let transcript;
    if (TRANSCRIBE_METHOD === 'openai') {
      transcript = await transcribeWithWhisper(download.buffer, OPENAI_API_KEY);
    } else {
      transcript = await transcribeWithLocalWhisper(download.buffer);
    }

    if (!transcript.success) {
      return { error: transcript.error };
    }

    return {
      success: true,
      transcript: transcript.text,
      duration: fileInfo.duration,
      method: TRANSCRIBE_METHOD,
      originalMessageId: steps.trigger.event.message.message_id
    };
  }
});
```

## Modified Step 04: Memory Loader

When a voice message is detected, modify the user message:

```javascript
let userMessage = steps.trigger.event.message.text;

// If voice transcription succeeded, use transcript
if (steps.voice_transcriber?.success) {
  userMessage = `[Voice message] ${steps.voice_transcriber.transcript}`;
}

// Continue with rest of pipeline...
```

## Voice Reply (Optional)

After generating text reply, optionally synthesize voice and send:

### Text-to-Speech Options

**Option A: Google Text-to-Speech**
```javascript
async function synthesizeVoice(text, apiKey) {
  const response = await axios.post(
    'https://texttospeech.googleapis.com/v1/text:synthesize',
    {
      input: { text },
      voice: {
        languageCode: 'en-US',
        name: 'en-US-Neural2-A'
      },
      audioConfig: {
        audioEncoding: 'OGG_OPUS'
      }
    },
    {
      params: { key: apiKey },
      timeout: 30000
    }
  );

  return Buffer.from(response.data.audioContent, 'base64');
}
```

**Option B: Telegram Built-in Audio (Recommended)**

Just send text reply — Telegram can read it aloud with `/settings`.

**Option C: ElevenLabs TTS**
```javascript
async function synthesizeVoiceElevenLabs(text, voiceId, apiKey) {
  const response = await axios.post(
    `https://api.elevenlabs.io/v1/text-to-speech/${voiceId}`,
    { text },
    {
      headers: { 'xi-api-key': apiKey },
      responseType: 'arraybuffer'
    }
  );

  return response.data;
}
```

## Step 11b: Voice Reply (Conditional)

```javascript
// After telegram_reply sends text, optionally send voice

if (process.env.SEND_VOICE_REPLIES === 'true') {
  const audioBuffer = await synthesizeVoice(
    steps.ai_router.reply,
    process.env.GOOGLE_TTS_API_KEY
  );

  await axios.post(
    `https://api.telegram.org/bot${process.env.TELEGRAM_BOT_TOKEN}/sendVoice`,
    {
      chat_id: steps.trigger.event.message.chat.id,
      voice: audioBuffer,
      reply_to_message_id: steps.trigger.event.message.message_id
    }
  );
}
```

## Environment Variables

Add to `.env.example`:

```bash
# Voice Message Processing
TRANSCRIBE_METHOD=openai  # openai | local
OPENAI_API_KEY=sk-...     # For OpenAI Whisper
GOOGLE_TTS_API_KEY=...    # For Google Text-to-Speech (optional)

# Voice Reply
SEND_VOICE_REPLIES=false  # true to send voice replies
```

## Error Handling

- **Download fails**: Return error, skip voice processing
- **Transcription times out (>60s audio)**: Return "Audio too long" to user
- **Whisper API rate limit**: Queue for retry, inform user
- **Synthesis fails**: Send text reply only, log error

## Testing

1. Send a voice message to the bot
2. Check Pipedream logs for:
   - `voice_transcriber` step success
   - Transcript text appears in `memory_loader`
3. Verify AI reply references the voice content
4. Test with different audio durations (10s, 60s, 300s)

## Performance Notes

- Voice download: ~2s for 10s audio
- OpenAI Whisper: ~5–10s for 10s audio
- Local Whisper (base): ~15–30s for 10s audio
- TTS synthesis: ~2–5s
- **Total latency**: 10–45s depending on audio length and synthesis

## Security

- Delete temporary audio files after transcription
- Never log full audio content (log filename and duration only)
- Validate audio size (max 25 MB for OpenAI Whisper)
