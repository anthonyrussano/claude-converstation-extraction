# Extracting Claude Shared Conversation Content

Last updated: 2026-05-20

This documents the procedure that successfully extracted a Claude shared-chat transcript from a URL like:

```text
https://claude.ai/share/fbe3711d-104f-4478-b1da-d5875282a971
```

Use this as a future-agent runbook when a user provides a public Claude share URL and wants the conversation content extracted into local files.

## Summary

The Claude share page itself is a JavaScript app shell. The actual transcript is loaded from this API endpoint:

```text
https://claude.ai/api/chat_snapshots/<share_id>?rendering_mode=messages&render_all_tools=true
```

Direct `curl` requests to that API endpoint were blocked by Cloudflare with HTTP `403`. Headless Playwright was also blocked. The successful approach was to run Chromium in headed mode under `xvfb-run`, open the share URL, capture the `chat_snapshots` network response, and save its JSON body.

## What Worked

Successful method:

```text
Playwright Chromium, headless: false, launched under xvfb-run.
```

This received HTTP `200` from:

```text
/api/chat_snapshots/<share_id>?rendering_mode=messages&render_all_tools=true
```

and saved the JSON response containing `chat_messages`.

## What Did Not Work

Direct page fetch:

```bash
curl -L https://claude.ai/share/fbe3711d-104f-4478-b1da-d5875282a971
```

This returned only the app shell HTML, not the rendered transcript.

Direct API fetch:

```bash
curl -L -i 'https://claude.ai/api/chat_snapshots/fbe3711d-104f-4478-b1da-d5875282a971?rendering_mode=messages&render_all_tools=true'
```

This returned Cloudflare challenge HTML with HTTP `403`.

Headless Playwright:

```js
await chromium.launch({ headless: true })
```

This also hit Cloudflare's "Just a moment..." verification page.

## Prerequisites

The successful extraction used:

- `node`
- `npm`
- `xvfb-run`
- Playwright
- Network access

Check for the main tools:

```bash
node --version
npm --version
which xvfb-run
```

## Step 1: Extract the Share ID

From a URL like:

```text
https://claude.ai/share/fbe3711d-104f-4478-b1da-d5875282a971
```

the share ID is:

```text
fbe3711d-104f-4478-b1da-d5875282a971
```

The API endpoint is:

```text
https://claude.ai/api/chat_snapshots/fbe3711d-104f-4478-b1da-d5875282a971?rendering_mode=messages&render_all_tools=true
```

## Step 2: Set Up Playwright in a Temporary Directory

Use a temp directory so the project repo is not modified:

```bash
mkdir -p /tmp/claude-playwright
cd /tmp/claude-playwright
npm install playwright@1.58.2
```

If browsers are not already installed:

```bash
npx playwright@1.58.2 install chromium
```

In the successful extraction, this was run from the project folder first:

```bash
npx -y playwright@1.58.2 install chromium
```

Then Playwright itself was installed into `/tmp/claude-playwright`:

```bash
npm install playwright@1.58.2
```

## Step 3: Capture the JSON Response

Run this from `/tmp/claude-playwright`, replacing `SHARE_ID` and output path as needed:

```bash
xvfb-run -a node -e "
const { chromium } = require('playwright');
const fs = require('fs');

const shareId = 'SHARE_ID';
const shareUrl = \`https://claude.ai/share/\${shareId}\`;
const outputPath = \`/tmp/claude_snapshot_\${shareId}.json\`;

(async () => {
  const browser = await chromium.launch({ headless: false });
  const page = await browser.newPage();
  let saved = false;

  page.on('response', async response => {
    const url = response.url();
    if (url.includes('/api/chat_snapshots/')) {
      const text = await response.text();
      fs.writeFileSync(outputPath, text);
      console.log('saved', response.status(), text.length, outputPath);
      saved = true;
    }
  });

  await page.goto(shareUrl, { waitUntil: 'domcontentloaded', timeout: 60000 });

  for (let i = 0; i < 60 && !saved; i++) {
    await page.waitForTimeout(1000);
  }

  await browser.close();

  if (!saved) {
    throw new Error('snapshot response was not captured');
  }
})().catch(e => {
  console.error(e);
  process.exit(1);
});
"
```

For the actual conversation extracted on 2026-05-20, the command was:

```bash
xvfb-run -a node -e "const { chromium } = require('playwright'); const fs = require('fs'); (async () => { const browser = await chromium.launch({ headless: false }); const page = await browser.newPage(); let saved = false; page.on('response', async response => { const url = response.url(); if (url.includes('/api/chat_snapshots/')) { const text = await response.text(); fs.writeFileSync('/tmp/claude_snapshot.json', text); console.log('saved', response.status(), text.length); saved = true; } }); await page.goto('https://claude.ai/share/fbe3711d-104f-4478-b1da-d5875282a971', { waitUntil: 'domcontentloaded', timeout: 60000 }); for (let i = 0; i < 60 && !saved; i++) await page.waitForTimeout(1000); await browser.close(); if (!saved) throw new Error('snapshot response was not captured'); })().catch(e => { console.error(e); process.exit(1); });"
```

Successful output:

```text
saved 200 154719
```

The saved file was:

```text
/tmp/claude_snapshot.json
```

## Step 4: Inspect the JSON Shape

Pretty-print the top of the JSON:

```bash
python -m json.tool /tmp/claude_snapshot.json | sed -n '1,220p'
```

Summarize keys and messages:

```bash
python - <<'PY'
import json

p = '/tmp/claude_snapshot.json'
data = json.load(open(p))

print(type(data).__name__)
print(data.keys())
print('snapshot_name:', data.get('snapshot_name'))
print('created_by:', data.get('created_by'))

messages = data.get('chat_messages') or []
print('messages:', len(messages))

for i, m in enumerate(messages[:5], 1):
    print()
    print('MSG', i, m.keys())
    print('sender:', m.get('sender'), 'created:', m.get('created_at'))
    print('content blocks:', len(m.get('content') or []))
PY
```

Observed top-level keys:

```text
uuid
conversation_uuid
created_at
updated_at
snapshot_name
created_by
creator
project_uuid
chat_messages
up_to_date
is_public
```

The useful field is:

```text
chat_messages
```

Each message has:

```text
sender
created_at
content
attachments
files
file_count
image_count
```

The `content` array contains blocks such as:

- `text`
- `tool_use`
- `tool_result`

## Step 5: Convert the JSON to Markdown

Use this parser as a baseline:

```bash
python - <<'PY'
import json
from pathlib import Path

input_path = Path('/tmp/claude_snapshot.json')
output_path = Path('/tmp/claude_transcript.md')

data = json.loads(input_path.read_text())

def message_text(message):
    parts = []
    for block in message.get('content') or []:
        if block.get('type') == 'text':
            parts.append(block.get('text') or '')
    return '\n\n'.join(p for p in parts if p)

def tool_activity(message):
    rows = []
    for block in message.get('content') or []:
        typ = block.get('type')
        if typ == 'tool_use':
            rows.append(f"- Tool use: `{block.get('name')}` - {block.get('message') or ''}")
        elif typ == 'tool_result':
            rows.append(f"- Tool result: `{block.get('name')}` - {block.get('message') or ''}")
        elif typ and typ != 'text':
            rows.append(f"- Non-text block: `{typ}`")
    return '\n'.join(rows)

lines = []
lines.append(f"# {data.get('snapshot_name') or 'Claude Shared Conversation'}")
lines.append("")
lines.append(f"Share UUID: `{data.get('uuid')}`")
lines.append(f"Conversation UUID: `{data.get('conversation_uuid')}`")
lines.append(f"Created by: `{data.get('created_by')}`")
lines.append("")

for idx, msg in enumerate(data.get('chat_messages') or [], 1):
    sender = 'You' if msg.get('sender') == 'human' else 'Claude'
    created = msg.get('created_at', '')
    lines.append(f"## Turn {idx}: {sender} ({created})")
    lines.append("")

    tools = tool_activity(msg)
    if tools:
        lines.append("Tool activity captured in the share snapshot:")
        lines.append("")
        lines.append(tools)
        lines.append("")

    text = message_text(msg).strip()
    if text:
        lines.append("~~~~text")
        lines.append(text)
        lines.append("~~~~")
    else:
        lines.append("_No visible text content in this message._")
    lines.append("")

output_path.write_text('\n'.join(lines))
print(output_path)
PY
```

The `~~~~text` fence is intentional. Claude responses often contain Markdown code fences. Using tilde fences reduces the chance that embedded triple backticks will break the generated transcript Markdown.

## Step 6: Know What the Share Snapshot Does Not Include

The shared Claude page can hide uploaded files. In the extracted conversation, the page showed:

```text
Files hidden in shared chats
```

The JSON indicated `file_count: 1` on the first user message, but did not include the PDF contents. The PDF had to be extracted separately from the local project directory.

If the user's requested output depends on uploaded files, ask for the files or inspect local attachments the user says are present.

## Step 7: Optional PDF Extraction Used in This Project

For the Workday Snowflake POC, the user also had the PDF locally:

```text
/home/russanoa/workday_snowflake_poc/WDC EA Assignment5 (1).pdf
```

The environment did not have `pdftotext`, so this worked:

```bash
uv run --with pypdf python - <<'PY'
from pypdf import PdfReader

reader = PdfReader('WDC EA Assignment5 (1).pdf')
for i, page in enumerate(reader.pages, 1):
    text = page.extract_text() or ''
    print(f'\n\n--- PAGE {i} ---\n')
    print(text)
PY
```

## Troubleshooting

If direct API fetch returns `403`, use the Playwright/Xvfb method above.

If Playwright headless returns "Just a moment...", use:

```js
chromium.launch({ headless: false })
```

and run it through:

```bash
xvfb-run -a node ...
```

If `require('playwright')` fails when using `npx -p playwright`, install Playwright into a temp directory instead:

```bash
mkdir -p /tmp/claude-playwright
cd /tmp/claude-playwright
npm install playwright@1.58.2
```

If the network response is not captured, increase the wait loop from 60 seconds or log responses:

```js
page.on('response', r => {
  const u = r.url();
  if (u.includes('chat_snapshots') || u.includes('/api/')) {
    console.log('RESPONSE', r.status(), u);
  }
});
```

## Security and Cleanup Notes

- Treat extracted chat JSON as user data. It may contain sensitive content even if the share URL is public.
- Prefer saving raw JSON under `/tmp` unless the user asks to keep it.
- Do not commit extracted conversations unless the user explicitly requests it.
- Temporary Playwright install directory can be removed after extraction:

```bash
rm -rf /tmp/claude-playwright
```

## Minimal Reusable Command

After setting `SHARE_ID`, this is the shortest reusable extraction pattern:

```bash
export SHARE_ID='fbe3711d-104f-4478-b1da-d5875282a971'
mkdir -p /tmp/claude-playwright
cd /tmp/claude-playwright
npm install playwright@1.58.2
xvfb-run -a node -e "
const { chromium } = require('playwright');
const fs = require('fs');
const shareId = process.env.SHARE_ID;
const outputPath = \`/tmp/claude_snapshot_\${shareId}.json\`;
(async () => {
  const browser = await chromium.launch({ headless: false });
  const page = await browser.newPage();
  let saved = false;
  page.on('response', async response => {
    if (response.url().includes('/api/chat_snapshots/')) {
      fs.writeFileSync(outputPath, await response.text());
      console.log(outputPath);
      saved = true;
    }
  });
  await page.goto(\`https://claude.ai/share/\${shareId}\`, { waitUntil: 'domcontentloaded', timeout: 60000 });
  for (let i = 0; i < 60 && !saved; i++) await page.waitForTimeout(1000);
  await browser.close();
  if (!saved) throw new Error('snapshot response was not captured');
})().catch(e => { console.error(e); process.exit(1); });
"
```

Important: the command above reads `process.env.SHARE_ID`, so `SHARE_ID` must be exported or provided inline:

```bash
export SHARE_ID='fbe3711d-104f-4478-b1da-d5875282a971'
```

or:

```bash
SHARE_ID='fbe3711d-104f-4478-b1da-d5875282a971' xvfb-run -a node -e "..."
```
