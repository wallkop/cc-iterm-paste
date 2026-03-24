# cc-iterm-paste

A Claude Code skill that makes iTerm2 smart about pasting images.

**Cmd+V behavior after install:**
- Clipboard has image → saves to `/tmp/clipboard_XXXXX.png`, types the file path
- Clipboard has text → normal paste

## Install

```bash
claude plugin install github:wusilei/cc-iterm-paste
```

Then run `/cc-iterm-paste install` in Claude Code.

## Commands

| Command | Description |
|---------|-------------|
| `/cc-iterm-paste` | Install everything |
| `/cc-iterm-paste status` | Check installation status |
| `/cc-iterm-paste uninstall` | Remove all installed files |

## Requirements

- macOS
- iTerm2
- iTerm2 Python API (`Scripts → Manage → Install Python Runtime`)
