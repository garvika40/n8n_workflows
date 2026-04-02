# Granola → n8n → Gmail Workflow

## Overview

This n8n workflow collects meeting notes and transcripts from the Granola app, processes them with OpenAI to extract actionable items, and sends a daily summary via email.

---

## Workflow Nodes

### 1. Schedule Trigger

| Property | Value |
|---|---|
| Type | Trigger |
| Frequency | Daily at midnight |
| Cron expression | `0 0 * * *` |

**Purpose:** Initiates the workflow daily to ensure all of the day's meetings are processed automatically.

---

### 2. Code Node — Date Range

| Property | Value |
|---|---|
| Type | Code execution |
| Mode | Run once for all items |
| Language | JavaScript |

**Purpose:** Generates the start and end timestamps for the current day in ISO 8601 format.

**Output:**
```json
{
  "start": "2026-04-02T00:00:00.000Z",
  "end":   "2026-04-02T23:59:59.999Z"
}
```

**Script:**
```javascript
const today = new Date();

const start = new Date(
  today.getFullYear(),
  today.getMonth(),
  today.getDate(),
  0, 0, 0, 0
).toISOString();

const end = new Date(
  today.getFullYear(),
  today.getMonth(),
  today.getDate(),
  23, 59, 59, 999
).toISOString();

return [{ json: { start, end } }];
```

> To process yesterday instead, add `today.setDate(today.getDate() - 1);` before the `start` line.

---

### 3. HTTP Request — List Notes

| Property | Value |
|---|---|
| Type | HTTP Request |
| Method | GET |
| URL | `https://public-api.granola.ai/v1/notes` |
| Authentication | Bearer Token (`grn_YOUR_API_KEY`) |
| Response format | JSON |

**Query parameters:**

| Name | Value |
|---|---|
| `created_after` | `{{ $json.start }}` |
| `created_before` | `{{ $json.end }}` |

**Purpose:** Retrieves all meeting notes created within today's date range from the Granola REST API.

**Response shape:**
```json
{
  "notes": [
    {
      "id":         "not_1d3tmYTlCICgjy",
      "title":      "Standup",
      "summary":    "Team synced on sprint goals...",
      "created_at": "2026-04-02T09:00:00Z"
    }
  ],
  "hasMore": false,
  "cursor":  null
}
```

> **Note:** The API only returns notes that have a completed AI summary. Notes still being processed are excluded automatically.

---

### 4. Loop Over Items

| Property | Value |
|---|---|
| Type | Batch processing (SplitInBatches) |
| Field to split on | `notes` |
| Batch size | `1` |

**Purpose:** Processes each meeting note individually, feeding one note at a time through the transcript fetch and OpenAI steps.

**Output ports:**

| Port | Wired to | When it fires |
|---|---|---|
| Loop (Output 1) | HTTP Request — get transcript | Once per meeting |
| Done (Output 2) | Merge node | After all meetings processed |

---

### 5. HTTP Request — Get Transcript

| Property | Value |
|---|---|
| Type | HTTP Request |
| Method | GET |
| URL | `https://public-api.granola.ai/v1/notes/{{ $json.id }}?include=transcript` |
| Authentication | Bearer Token (reuse from Node 3) |
| Response format | JSON |
| Continue on fail | ON |
| Timeout | 10000 ms |

**Purpose:** Fetches the full transcript for a single meeting using its ID from the current batch iteration.

> **Important:** `?include=transcript` is required — without it Granola omits the transcript array entirely.

---

### 6. Code Node — Flatten Transcript

| Property | Value |
|---|---|
| Type | Code execution |
| Mode | Run for each item |
| Language | JavaScript |

**Purpose:** Converts the Granola transcript array of objects into a plain text string for OpenAI.

**Script:**
```javascript
const transcriptArray = $json.transcript || [];

const transcript_text = transcriptArray
  .map(t => {
    const speaker = t.speaker?.source === 'microphone' ? 'You' : 'Them';
    return speaker + ': ' + t.text;
  })
  .join('\n');

return [{
  json: {
    id:              $json.id,
    title:           $json.title,
    summary:         $json.summary,
    created_at:      $json.created_at,
    output:          $json.output,
    transcript_text: transcript_text
  }
}];
```

> Without this node, the transcript array renders as `[object Object],[object Object]...` in the OpenAI prompt.

---

### 7. Message a Model (OpenAI)

| Property | Value |
|---|---|
| Type | OpenAI — Message a Model |
| Model | `gpt-4o-mini` |
| Temperature | `0` |
| Max tokens | `1000` |

**Purpose:** Processes each meeting transcript to extract action items and tasks.

**System message:**
```
You are a meeting assistant. Your only job is to extract action items and tasks from meeting transcripts.

Always return valid JSON. Never include markdown, backticks, or explanation — raw JSON only.

If no tasks are found, return an empty tasks array.
```

**User message:**
```
Meeting title: {{ $json.title }}
Date: {{ $json.created_at }}

Transcript:
{{ $json.transcript_text }}

Extract every action item mentioned. Return ONLY this JSON shape:

{
  "meeting_title": "...",
  "tasks": [
    {
      "task":  "clear description of what needs to be done",
      "owner": "full name of person responsible, or Unassigned",
      "due":   "due date if mentioned, or null"
    }
  ]
}
```

---

### 8. Code Node — Parse OpenAI Response

| Property | Value |
|---|---|
| Type | Code execution |
| Mode | Run for each item |
| Language | JavaScript |

**Purpose:** Extracts the text from the OpenAI Responses API structure and carries all original meeting fields forward.

**Script:**
```javascript
const original = $input.first().json;

// Handle Responses API structure: output[0].content[0].text
const output = original.output || [];
const rawText = output
  .filter(o => o.type === 'message')
  .flatMap(o => o.content || [])
  .filter(c => c.type === 'output_text')
  .map(c => c.text)
  .join('\n')
  .trim();

// Try to parse as JSON, fall back to raw text
let tasks = [];
let raw_output = '';

try {
  const clean = rawText
    .replace(/```json/g, '')
    .replace(/```/g, '')
    .trim();
  const parsed = JSON.parse(clean);
  tasks = parsed.tasks || [];
} catch(e) {
  raw_output = rawText;
}

return [{
  json: {
    id:          original.id,
    title:       original.title,
    summary:     original.summary,
    created_at:  original.created_at,
    output:      original.output,
    tasks:       tasks,
    raw_output:  raw_output
  }
}];
```

---

### 9. Merge

| Property | Value |
|---|---|
| Type | Merge |
| Mode | Collect all items |

**Purpose:** Waits for all batch iterations to complete and collects every processed meeting into a single array before the email is formatted.

> The Loop node's Done port (Output 2) connects here. The Merge node's output goes to the Format Email Code node.

---

### 10. Code Node — Format Email

| Property | Value |
|---|---|
| Type | Code execution |
| Mode | Run once for all items |
| Language | JavaScript |

**Purpose:** Formats all processed meetings into a single email body string.

**Script:**
```javascript
const meetings = $input.all().map(i => i.json).filter(m => !m.error);

const sections = meetings.map(m => {

  // Extract text from Responses API structure
  const output = m.output || [];
  const rawText = output
    .filter(o => o.type === 'message')
    .flatMap(o => o.content || [])
    .filter(c => c.type === 'output_text')
    .map(c => c.text)
    .join('\n')
    .trim();

  // Clean markdown formatting for plain text email
  const cleanText = rawText
    .replace(/\*\*(.+?)\*\*/g, '$1')
    .replace(/### /g, '')
    .replace(/## /g, '')
    .replace(/# /g, '')
    .replace(/\n   - /g, '\n  - ')
    .replace(/\n- /g, '\n  - ');

  const title   = m.title   || 'Untitled meeting';
  const summary = m.summary || '';

  return '📅 ' + title +
    (summary ? '\n' + summary : '') +
    '\n\n' + cleanText;

}).join('\n\n---\n\n');

const date    = new Date().toDateString();
const count   = meetings.length;
const subject = 'Meeting tasks — ' + date + ' (' + count + ' meetings)';
const body    = 'Daily meeting summary — ' + date + '\n\n' + sections;

return [{ json: { subject, body, count } }];
```

**Output fields:**

| Field | Used by |
|---|---|
| `subject` | Gmail node → Subject as `{{ $json.subject }}` |
| `body` | Gmail node → Message as `{{ $json.body }}` |
| `count` | IF node → skip email if `0` |

---

### 11. IF Node — Skip Empty Days

| Property | Value |
|---|---|
| Type | IF |
| Condition | `{{ $json.count }}` greater than `0` |
| True branch | Gmail — Send a Message |
| False branch | NoOp (stop) |

**Purpose:** Prevents sending an empty email on days with no meetings.

---

### 12. Send a Message (Gmail)

| Property | Value |
|---|---|
| Type | Gmail — Send a Message |
| To | `your@email.com` |
| Subject | `{{ $json.subject }}` |
| Message | `{{ $json.body }}` |
| Email type | Text |

**Purpose:** Sends the daily meeting summary email.

> Switch Subject and Message fields from **Fixed** to **Expression** mode before entering the `{{ }}` expressions.

---

## Node Connection Map

```
Schedule Trigger
  └──→ Code (date range)
         └──→ HTTP Request (list notes)
                └──→ Loop Over Items
                       ├── loop ──→ HTTP Request (get transcript)
                       │                └──→ Code (flatten transcript)
                       │                       └──→ Message a Model (OpenAI)
                       │                              └──→ Code (parse response)
                       │                                     └──→ [back to Loop]
                       └── done ──→ Merge
                                     └──→ Code (format email)
                                            └──→ IF node
                                                   ├── true  ──→ Gmail
                                                   └── false ──→ Stop
```

---

## Credentials Required

| Service | Type | Where to get it |
|---|---|---|
| Granola API | Bearer Token (`grn_...`) | Granola → Settings → API Keys |
| OpenAI | API Key | platform.openai.com → API Keys |
| Gmail | OAuth2 | n8n → Credentials → Gmail OAuth2 |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `[object Object]` in transcript | Transcript not flattened | Add flatten Code node before OpenAI |
| "Untitled meeting" in email | `title` not carried through | Spread `...original` in parse Code node |
| "No tasks identified" | `m.tasks` is undefined | Check parse node returns `tasks` array |
| Empty email body | `m.output` dropped | Explicitly carry `output` field through all Code nodes |
| `SyntaxError: Unexpected end of input` | Broken template literal | Replace backtick strings with `+` concatenation |
| 404 on transcript fetch | Note still processing | Enable "Continue on Fail" on HTTP Request node |
| Email not sending | IF node condition wrong | Confirm `{{ $json.count }}` is greater than `0` |
