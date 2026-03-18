---
name: wechat-search-pick-send

description: >
  Send a WeChat message on macOS by automating: open WeChat, fullscreen, search a contact or
  group chat name, screenshot results, visually choose the correct row (treat every result
  containing the target name/keyword as a candidate), navigate with ↓, enter the chat, paste
  the message, and send. Use when the user says things like: "请打开微信，给XXX发消息YYY",
  "微信给XXX发YYY", or any request to send a message to a WeChat contact/group.
---

# WeChat Search → Pick → Send (Mac)

Follow this workflow **exactly**. This task is fragile and order‑dependent.

## Preconditions

- WeChat for macOS is installed and logged in
- The host running OpenClaw has **Accessibility** permission
- Chinese text input must be done **via clipboard paste**, never keystroke

---

## Workflow FSM

```
OPEN_WECHAT
→ FULLSCREEN
→ SEARCH_TARGET
→ (optional) SCREENSHOT_RESULTS
→ (optional) VISUAL_DECISION
→ ENTER_CHAT
→ FOCUS_INPUT
→ PASTE_AND_SEND
→ DONE

-- For sending a local file/image:
OPEN_WECHAT
→ FULLSCREEN
→ SEARCH_TARGET
→ ENTER_CHAT
→ FOCUS_INPUT
→ FINDER_COPY_FILE
→ PASTE_AND_SEND
→ DONE
```

---

## 1. Open WeChat and **force-focus window** (critical)

WeChat may be “running in the background” with **no visible window**. Use this **robust open → activate → force-front** sequence before any UI steps.

### 1.1 Robust open (preferred)

Per the updated local best-practice: **only** use `open -a WeChat` to bring up the WeChat UI.

```bash
open -a WeChat
```

Wait briefly for the window to appear:

```applescript
delay 0.8
```

### 1.2 Maximize window (fullscreen)

After `open -a WeChat`, maximize the WeChat window before any UI steps.

```applescript
-- Ctrl+Cmd+F fullscreen toggle
try
  tell application "System Events" to tell process "WeChat" to keystroke "f" using {control down, command down}
end try

delay 0.6
```

> Note: If fullscreen is undesirable, replace this with a fixed window size/position step.

### 1.3 Enter fullscreen (retry-safe)

```applescript
tell application "System Events"
  tell process "WeChat"
    keystroke "f" using {control down, command down} -- Ctrl+Cmd+F
  end tell
end tell

delay 0.5

-- Retry once (fullscreen can fail on first attempt if focus is wrong)
tell application "System Events"
  tell process "WeChat"
    keystroke "f" using {control down, command down}
  end tell
end tell
```

**If any `System Events` step fails**, instruct the user to enable:
- System Settings → Privacy & Security → **Accessibility**
- System Settings → Privacy & Security → **Automation** (Terminal/OpenClaw → WeChat)

**Rule:** If WeChat is not visibly frontmost after this step, **stop and ask user to confirm** before continuing.

---

## 2. Search target name (contact or group)

Use this for both friends and group chats. Always search via clipboard paste to avoid IME issues.


Always use clipboard to avoid IME issues.

```applescript
set the clipboard to "{targetName}"

tell application "System Events"
  keystroke "f" using command down
  delay 0.6
  keystroke "a" using command down
  key code 51 -- delete
  delay 0.2
  keystroke "v" using command down
end tell
```

Wait for results:

```applescript
delay 0.6
```

---

## 3. Screenshot search results

Capture the visible WeChat window.

```bash
/usr/sbin/screencapture -x ~/wechat_search_results.png
```

---

## 4. Visual decision logic (core rule)

From the screenshot:

- Count **only list items whose text contains `{targetName}`**
- Treat **each matching item as one row**, regardless of section (群聊 / 搜索建议)
- Prefer exact/closest name match
- If still tied, prefer **contact (person)** over group unless the user explicitly said “群聊”
- If still tied (same match closeness), **pick the first/topmost candidate**

Determine:

```
rowIndex = 1-based index of the correct row
```

If the cursor starts on row 1:

```
downCount = rowIndex - 1
```

If ambiguous, ask the user to confirm before continuing.

---

## 5. Keyboard navigation

```applescript
repeat downCount times
  tell application "System Events" to key code 125 -- ↓
  delay 0.15
end repeat

-- Enter selected chat
tell application "System Events" to key code 36
```

---

## 6. Focus input box (robust)

**Key lesson:** If you were in search/results view, the cursor often stays in the left results panel. The most reliable way is:

1) **Enter** to open the selected chat
2) **Esc** to collapse the search panel back to normal chat layout (this usually places the caret in the input)
3) Only if needed, use **Tab** to advance focus into the input

Use this sequence:

```applescript
-- After entering the chat, collapse search panel
tell application "System Events" to key code 53 -- Esc

delay 0.2

-- Tab into input (1–2 times)
tell application "System Events" to key code 48 -- Tab

delay 0.15

tell application "System Events" to key code 48 -- Tab
```

Fallbacks (in order):
- Press Esc once more, then Tab again
- As last resort, a single mouse click inside the input area is allowed

Goal: ensure the text cursor is inside the input box before pasting.

---

## 7. Paste and send message

```applescript
set the clipboard to "{message}"

tell application "System Events"
  keystroke "v" using command down
  delay 0.2
  key code 36 -- Enter
end tell
```

---

## 8. Send an image/file (Finder-copy workflow)

When the user asks to send a local image/file (e.g. `/Users/.../xxx.jpeg`) to a WeChat chat, use this dedicated branch so text sending is unaffected.

**Exact order (do not change):**

1) **WeChat: open + fullscreen + search & enter chat** (steps 1–5)

2) **WeChat: make sure the caret is in the input box** (step 6; Enter → Esc collapse → Tab if needed)

3) **Finder: reveal & copy the file**

```bash
open -R "{filePath}"
```

```applescript
tell application "System Events" to keystroke "c" using {command down}
```

4) **Back to WeChat: paste and send**

```applescript
tell application "WeChat" to activate
```

```applescript
tell application "System Events"
  keystroke "v" using {command down}
  delay 0.4
  key code 36 -- Enter
end tell
```

Notes:
- Preparing the WeChat input caret **before** copying the file avoids focus loss.
- If the paste shows a file chip/preview, Enter sends it.
- If WeChat opens a confirmation dialog (large file), stop and ask the user before confirming.

---

## Hard Rules (Do Not Break)

- Never type Chinese text directly
- Never rely on mouse coordinates for selecting search results
- Always decide the row **from the screenshot**, not assumptions
- Always navigate with keyboard ↓ and Enter

---

## Expected Result

- Correct WeChat group chat is entered
- Message is successfully sent as a chat bubble
