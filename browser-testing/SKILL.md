---
name: browser-testing
description: "Full browser automation via Agent Browser Protocol (ABP). Navigate, click, type, scroll, drag, screenshot, extract text, handle dialogs/downloads/file pickers, manage tabs, control JS execution. Single CLI tool."
---

# Browser Automation — ABP

Single tool: `{baseDir}/browser.js <command> [args] [--flags]`

ABP is a Chromium fork with a REST API baked into the engine. Every action is deterministic — JS freezes between steps, no race conditions, no manual waits.

## Setup

```bash
{baseDir}/browser.js start           # Launch ABP on :8222
```

## Core Commands

```bash
B={baseDir}/browser.js

# Navigate
$B nav https://example.com           # Navigate active tab
$B nav https://other.com --new       # New tab
$B back                              # History back
$B forward                           # History forward
$B reload                            # Reload

# Mouse
$B click 450 320                     # Left click
$B click 450 320 --right             # Right click
$B click 450 320 --double            # Double click
$B click 450 320 --mod CTRL          # Ctrl+click
$B hover 300 200                     # Mouse move (trigger tooltips/menus)
$B scroll 640 400 --dy 500           # Scroll down 500px
$B scroll 640 400 --dy -300          # Scroll up
$B scroll 640 400 --dx 200           # Scroll right
$B drag 100 200 500 200              # Drag from→to
$B drag 100 200 500 200 --steps 20   # Smooth drag

# Keyboard
$B type hello world                  # Type text
$B key ENTER                         # Press key
$B key TAB                           # Tab
$B key ESCAPE                        # Escape
$B key a --mod CTRL                  # Ctrl+A (select all)
$B key c --mod CTRL                  # Ctrl+C (copy)
$B key ARROWDOWN                     # Arrow keys
$B key BACKSPACE
$B key a --mod CTRL --action down    # Key down only (hold)
$B key a --action up                 # Key up (release)

# Input helpers
$B slider 400 300 75                 # Set range input to 75
$B clear 400 300                     # Clear text field (click + select all + delete)
$B pick "Select the login button"    # Interactive: user clicks element in browser

# Screenshot
$B screenshot                        # Viewport with interactive markup
$B screenshot --markup clickable     # Only clickable elements
$B screenshot --markup typeable      # Only input fields
$B screenshot --markup clickable,typeable,scrollable,grid
$B screenshot --markup none          # Clean, no overlays
$B screenshot --format png           # PNG instead of WebP

# Extract content
$B text                              # All visible text (fast, API-native)
$B text "h1.title"                   # Text within CSS selector
$B eval 'document.title'             # Execute JavaScript
$B eval '({links: document.querySelectorAll("a").length})'
$B content                           # Current page as Markdown (Readability)
$B content https://example.com       # Navigate + extract as Markdown
$B cookies                           # Non-HttpOnly cookies
```

## Tabs

```bash
$B tabs                              # List all tabs
$B tabs new https://google.com       # New tab with URL
$B tabs activate <id>                # Switch to tab
$B tabs close <id>                   # Close tab
$B tabs info <id>                    # Tab details
$B tabs stop <id>                    # Stop loading
```

## Browser Events

ABP surfaces events that normally require polling — dialogs, file pickers, downloads, select dropdowns, permission prompts. They appear in the output of any action.

```bash
# Dialogs (alert, confirm, prompt)
$B dialog                            # Check for pending dialog
$B dialog accept                     # Accept
$B dialog accept "response text"     # Accept prompt with text
$B dialog dismiss                    # Dismiss/cancel

# Downloads
$B download                          # List all
$B download status <id>              # Check progress
$B download cancel <id>              # Cancel
$B download get <id>                 # Get content (base64)

# File chooser (triggered by file input click)
$B file <chooser_id> /path/to/file.pdf       # Upload file
$B file <chooser_id> file1.jpg file2.jpg     # Multiple files
$B file <chooser_id> --cancel                # Cancel picker
$B file <chooser_id> --save /path/out.pdf    # Save dialog

# Native <select> dropdown
$B select <select_id> 2              # Choose option at index

# Permissions (geolocation, camera, etc.)
$B permission                         # List pending
$B permission grant <id>              # Grant
$B permission grant <id> --lat 42.36 --lng -71.06  # Grant geo with coords
$B permission deny <id>               # Deny
```

## Execution Control

ABP freezes JS between actions by default. You can control this:

```bash
$B execution                          # Current state
$B execution pause                    # Freeze JS & virtual time
$B execution resume                   # Unfreeze
```

## Advanced

```bash
# Batch: multiple actions, one screenshot
$B batch '[{"type":"mouse_click","x":350,"y":200},{"type":"keyboard_type","text":"hello"},{"type":"keyboard_press","key":"ENTER"}]'

# Session history (SQLite-backed, for training data)
$B history                            # List sessions
$B history current                    # Current session
$B history actions                    # Action log
$B history clear                      # Delete all

# Lifecycle
$B status                             # Browser readiness
$B shutdown                           # Graceful shutdown
```

## Global Flags

| Flag | Description |
|---|---|
| `--tab <id>` | Target specific tab (default: active) |
| `--shot` | Save screenshot after action (prints path) |
| `--markup <types>` | Screenshot markup: `interactive`, `clickable,typeable,scrollable,grid,selected`, or `none` |
| `--format <fmt>` | Screenshot format: `webp` (default), `png`, `jpeg` |
| `--json` | Output raw API response as JSON |

## Event Indicators

When events occur during any action, they're printed automatically:

```
  → https://new-page.com              # Navigation happened
  ⚠ dialog (confirm): Delete item?    # Dialog appeared
  📁 file chooser id=fc_1             # File picker opened
  ⬇ download: report.pdf              # Download started
  ▾ select id=s_1 (5 options)         # Native select opened
  🔐 permission id=p_1 geolocation    # Permission requested
  ↗ popup: https://popup.com          # Popup window
```

## Speed Rules

**The fast pattern**: navigate → eval to extract. Skip screenshots unless you're lost.

1. **Start ABP first**: `browser.js start`
2. **Don't screenshot every step**: Skip `--shot` during form-filling. Only screenshot when you need to see layout.
3. **Observe the URL after search**: Most SPAs encode filters in URL params. Copy it, modify it, `nav` directly next time — skip the form entirely.
4. **Extract data via `eval`, not vision**: One JS query extracts 10 results faster than scrolling + screenshotting.
5. **Batch related inputs**: Click + type + Enter = one `batch` call instead of three.
6. **Use `text` for simple data**: `text` is faster than `eval` for plain text extraction.
7. **Use pick for ambiguity**: When coordinates are unclear, let the user click.

**Anti-pattern**: click → screenshot → read image → decide → click → screenshot → ... (each step: ~3s for screenshot + LLM vision round-trip)

**Fast pattern**: nav → click click click (no shots) → eval to extract all data → screenshot once to verify