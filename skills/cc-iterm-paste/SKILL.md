---
name: cc-iterm-paste
description: iTerm2 智能图片粘贴安装工具。当用户说 "cc-iterm-paste"、"智能粘贴"、"安装图片粘贴"、"install iterm paste" 时触发。自动安装 clip2png + smart_paste.py，让 iTerm2 支持 Cmd+V 粘贴图片自动转为本地文件路径。支持 install / uninstall / status 三个子命令。
---

# cc-iterm-paste

iTerm2 智能图片粘贴工具。Cmd+V 时自动检测剪贴板内容：
- 剪贴板是图片 → 保存为 /tmp/clipboard_XXXXX.png，输出文件路径
- 剪贴板是文字 → 正常粘贴

## 使用方式

用户输入 `/cc-iterm-paste` 后，读取参数决定执行哪个操作：
- `/cc-iterm-paste` 或 `/cc-iterm-paste install` → 执行安装
- `/cc-iterm-paste uninstall` → 执行卸载
- `/cc-iterm-paste status` → 检查状态

---

## install

按以下步骤执行安装，每步完成后打印进度：

### Step 1：检查前置条件

运行以下检查，任何一项失败则提示用户修复后再安装：

```bash
# 检查 macOS
[[ "$(uname)" == "Darwin" ]] && echo "OK" || echo "FAIL: 仅支持 macOS"

# 检查 iTerm2
ls /Applications/iTerm.app 2>/dev/null && echo "OK" || echo "FAIL: 未找到 iTerm2"

# 检查 iTerm2 Python API
ls ~/Library/Application\ Support/iTerm2/iterm2env 2>/dev/null \
  && echo "OK" || echo "WARN: iTerm2 Python API 可能未启用，请先执行 Scripts → Manage → Install Python Runtime"
```

### Step 2：写入 clip2png

将以下内容写入 `~/.local/bin/clip2png`，然后 `chmod +x`：

```zsh
#!/bin/zsh
tempfile=$(mktemp /tmp/clipboard_XXXXXX.png)

osascript << APPLESCRIPT
set tempPath to "$tempfile"
try
    set theData to the clipboard as «class PNGf»
on error
    set theData to the clipboard as «class TIFF»
end try
set fileRef to open for access POSIX file tempPath with write permission
write theData to fileRef
close access fileRef
APPLESCRIPT

# 只保留最近 20 张，删除多余的
ls -t /tmp/clipboard_*.png 2>/dev/null | tail -n +21 | xargs rm -f

echo "$tempfile"
```

执行：
```bash
mkdir -p ~/.local/bin
chmod +x ~/.local/bin/clip2png
```

### Step 3：写入 smart_paste.py

将以下内容写入 `~/Library/Application Support/iTerm2/Scripts/AutoLaunch/smart_paste.py`：

```python
#!/usr/bin/env python3
import iterm2
import subprocess
import os

async def main(connection):
    app = await iterm2.async_get_app(connection)

    pattern = iterm2.KeystrokePattern()
    pattern.required_modifiers = [iterm2.Modifier.COMMAND]
    pattern.keycodes = [iterm2.Keycode.ANSI_V]

    async with iterm2.KeystrokeFilter(connection, [pattern]):
        async with iterm2.KeystrokeMonitor(connection) as mon:
            while True:
                keystroke = await mon.async_get()

                if (iterm2.Modifier.COMMAND in keystroke.modifiers and
                        keystroke.keycode == iterm2.Keycode.ANSI_V):

                    result = subprocess.run(
                        ['osascript', '-e', 'clipboard info'],
                        capture_output=True, text=True
                    )

                    has_image = 'TIFF' in result.stdout or 'PNGf' in result.stdout
                    session = app.current_terminal_window.current_tab.current_session

                    if has_image:
                        convert = subprocess.run(
                            [os.path.expanduser('~/.local/bin/clip2png')],
                            capture_output=True, text=True
                        )
                        filepath = convert.stdout.strip()
                        if filepath:
                            await session.async_send_text(filepath)
                    else:
                        text = subprocess.run(
                            ['pbpaste'], capture_output=True, text=True
                        ).stdout
                        await session.async_send_text(text)

iterm2.run_forever(main)
```

执行：
```bash
mkdir -p ~/Library/Application\ Support/iTerm2/Scripts/AutoLaunch
```

### Step 4：验证安装

```bash
test -x ~/.local/bin/clip2png && echo "✅ clip2png OK" || echo "❌ clip2png 未找到"
test -f ~/Library/Application\ Support/iTerm2/Scripts/AutoLaunch/smart_paste.py && echo "✅ smart_paste.py OK" || echo "❌ smart_paste.py 未找到"
```

### Step 5：提示用户完成配置

安装完成后输出：

```
✅ 安装完成！

还需要在 iTerm2 中确认一步：
  菜单 Scripts → AutoLaunch → smart_paste.py  ← 确认前面有 ✓

如果 Python API 未启用，请先：
  菜单 Scripts → Manage → Install Python Runtime

完成后重启 iTerm2，Cmd+V 即可智能粘贴图片。
```

---

## uninstall

删除所有安装的文件：

```bash
rm -f ~/.local/bin/clip2png
rm -f ~/Library/Application\ Support/iTerm2/Scripts/AutoLaunch/smart_paste.py
```

删除后输出：
```
✅ 已卸载 cc-iterm-paste。重启 iTerm2 生效。
```

---

## status

检查并输出当前安装状态：

```bash
echo "=== cc-iterm-paste 状态 ==="

test -x ~/.local/bin/clip2png \
  && echo "✅ clip2png        已安装" \
  || echo "❌ clip2png        未安装"

test -f ~/Library/Application\ Support/iTerm2/Scripts/AutoLaunch/smart_paste.py \
  && echo "✅ smart_paste.py  已安装" \
  || echo "❌ smart_paste.py  未安装"
```
