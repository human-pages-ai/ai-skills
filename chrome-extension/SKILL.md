---
name: chrome-extension
description: |
  Drive the Claude Chrome extension from the CLI via Puppeteer and Chrome DevTools Protocol.
  Gives Claude Code browser agent capabilities — browse the web, check social media,
  fill forms, read pages, anything the Chrome extension can do, triggered from the terminal.
  Requires one-time setup: bind mount + Chrome with remote debugging.
allowed-tools:
  - Bash
  - Read
  - Write
---

# Chrome Extension Browser Agent

Drive the Claude Chrome extension from Claude Code to perform browser tasks.
Use when you need to browse the web, check social media, fill forms, read pages,
or do anything that requires a real browser session with the user's logged-in accounts.

## One-time setup

Requires `puppeteer-core`:

```bash
cd /tmp && npm install puppeteer-core
```

Chrome needs to run with remote debugging enabled on your real profile. A bind mount
is required because Chrome refuses to enable debugging on the default profile path:

```bash
mkdir -p /tmp/chrome-bind-profile
sudo mount --bind ~/.config/google-chrome /tmp/chrome-bind-profile
```

Then close Chrome and relaunch:

```bash
pkill -f google-chrome; sleep 2
rm -f /tmp/chrome-bind-profile/SingletonLock /tmp/chrome-bind-profile/SingletonSocket /tmp/chrome-bind-profile/SingletonCookie
google-chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-bind-profile --restore-last-session &
```

Verify it's working:

```bash
curl -s http://localhost:9222/json/version | head -1
```

## Usage

Check if Chrome is ready before running:

```bash
curl -s http://localhost:9222/json/version | head -1
```

If it's not running, tell the user to run the setup commands above.

### Send a task to the Chrome extension

```javascript
const puppeteer = require('puppeteer-core');
const EXT_ID = 'fcoeoabgfenejglbffodgkkbkcdhcgfn';

const browser = await puppeteer.connect({ browserURL: 'http://localhost:9222', defaultViewport: null });

// Get a valid tabId (required for API communication)
const sw = (await browser.targets()).find(t => t.url().includes(EXT_ID) && t.type() === 'service_worker');
const worker = await sw.worker();
const tabs = await worker.evaluate(async () => {
  const all = await chrome.tabs.query({});
  return all.map(t => ({ id: t.id, url: t.url }));
});
const tabId = tabs.find(t => !t.url.includes('sidepanel'))?.id;

// Open side panel (or reuse existing)
const pages = await browser.pages();
let panel = pages.find(p => p.url().includes('sidepanel'));
if (!panel) {
  panel = await browser.newPage();
  await panel.goto(`chrome-extension://${EXT_ID}/sidepanel.html?tabId=${tabId}`, { waitUntil: 'networkidle0' });
  await new Promise(r => setTimeout(r, 3000));
}

// Send a task
await panel.click('.tiptap.ProseMirror');
await panel.evaluate((task) => {
  const editor = document.querySelector('.tiptap.ProseMirror');
  editor.focus();
  document.execCommand('insertText', false, task);
}, 'YOUR TASK HERE');
await panel.evaluate(() => {
  document.querySelector('button[aria-label="Send message"]')?.click();
});

// Wait for response (poll until stable)
let prev = '';
for (let i = 0; i < 40; i++) {
  await new Promise(r => setTimeout(r, 3000));
  const text = await panel.evaluate(() => document.body?.innerText);
  const running = await panel.evaluate(() => !!document.querySelector('button[aria-label="Stop"]'));
  if (!running && text.length > 300 && text === prev) break;
  prev = text;
}

const response = await panel.evaluate(() => document.body?.innerText);
console.log(response);
await browser.disconnect();
```

## Key details

- **Extension ID**: `fcoeoabgfenejglbffodgkkbkcdhcgfn`
- **Text input**: Must use `document.execCommand('insertText')`. Puppeteer's `keyboard.type()` doesn't work with ProseMirror/TipTap contenteditable.
- **Send button**: `button[aria-label="Send message"]`
- **Stop button**: `button[aria-label="Stop"]` — present while the extension is working.
- **tabId is required**: Without it in the URL, the extension loads but can't call the API.
- **Reuse the panel**: Check for an existing sidepanel page before opening a new one.
- **The extension browses in real tabs**: It opens and navigates Chrome tabs. The user's browser is actively used.
- **Response polling**: Check for the Stop button disappearing + text stabilizing to know when it's done.

## Why bind mount?

Chrome requires `--user-data-dir` to be a non-default path for remote debugging. But changing
the profile path breaks Chrome's Secure Preferences HMAC, which silently disables extensions.
A symlink doesn't work either. A bind mount makes the same directory (same inodes) available
at a second path, passing both Chrome's path check and its HMAC validation.
