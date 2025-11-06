# wl-clip-multi

**Multi-MIME Type Wayland Clipboard Utility**

If you use Wayland and have ever tried to paste a screenshot into a terminal application (VS Code terminal, Claude Code, etc.) and nothing happens (or even worse, binary bullshit) instead of what you expected, this tool is for you.

## The Problem

**On X11:** `xclip` could offer multiple clipboard formats. Apps generally chose what format they want.

**On Wayland:** `wl-clipboard` (the standard tool on wl) will only offer **one MIME type at a time**.

### Real-World Impact

```bash
# Take a screenshot
grim screenshot.png

# Copy with wl-copy
wl-copy < screenshot.png

# Paste in Discord → works fine
# Paste in terminal applications like Claude code → literally nothing happens
```

Ie you can't paste it as an image in Discord AND as a file path in your terminal apps. You have to choose ONE format, which is fucking annoying. 
For example, if i wanna debug complicated web development, i'd usually take a screenshot and show it to my LLM CLI. But when i take the damn picture, i cant just paste it in the terminal. I literally have to 1) take screenshot, manually save it, go to my file explorer, find the damn picture, copy it manually (bc ctrl+c doesnt cut it), and THEN paste it into the LLM CLI in the terminal. 
<img width="340" height="423" alt="image" src="https://github.com/user-attachments/assets/02c075e6-39a5-48a5-9c45-1be97225b426" />


## So why is no one fixing this?

The `wl-clipboard` maintainer [explicitly won't add this](https://github.com/bugaevc/wl-clipboard/issues/71):
> "It's unclear how this would look in the command-line interface... this is getting too complicated for the command-line tool."

The problem: CLI tools use stdin for data. With multiple MIME types, where does each type's data come from? The design doesn't have a clean answer.

His recommendation: **Use the Wayland protocol directly.**
So thats what i did.

## My Solution
<img width="220" height="165" alt="image" src="https://github.com/user-attachments/assets/9acd7caa-a142-48d6-9aa4-5e63b2760484" />

`wl-clip-multi` uses the Wayland protocol directly (via `pywayland`) to offer **6 MIME types simultaneously**:

1. `x-special/gnome-copied-files` - File path (for terminals)
2. `text/plain;charset=utf-8` - Plain text path
3. `text/uri-list` - Standard URI format
4. `application/vnd.portal.filetransfer` - XDG portal
5. `application/vnd.portal.files` - XDG portal files
6. `image/png` (or appropriate) - Raw content (for visual apps)

Apps automatically choose their preferred format when you paste.

### The Result

```bash
grim screenshot.png
wl-clip-multi screenshot.png &

# Paste in Discord → Gets image data
# Paste in terminal → Gets /home/user/screenshot.png
# Same clipboard, different behavior per app!
```

## Installation

### 1. Install Dependencies

```bash
# Python and pywayland
pip install pywayland

# wl-clip-persist (CRITICAL - see below why)
sudo pacman -S wl-clip-persist  # Arch/Manjaro
# or
yay -S wl-clip-persist  # AUR
```

### 2. Install wl-clip-multi

```bash
git clone https://github.com/mrdrbrdr/wl-clip-multi.git
cd wl-clip-multi
sudo cp bin/wl-clip-multi /usr/local/bin/
sudo chmod +x /usr/local/bin/wl-clip-multi
```

### 3. Setup wl-clip-persist (Required)

Without this, your clipboard only works for a split second. Here's why:

Many Wayland systems run clipboard managers (like `elephant-clipboard`, `cliphist`) that use `wl-paste --watch` to monitor the clipboard. When they detect our multi-MIME clipboard, they read it and re-copy it - **but only preserve 1-2 MIME types**, destroying our carefully crafted 6-type clipboard.

`wl-clip-persist` solves this by reading **ALL MIME types** and preserving them.

**Add to autostart:**

<details>
<summary><b>Hyprland</b></summary>

Add to `~/.config/hypr/autostart.conf`:
```bash
exec-once = wl-clip-persist --clipboard regular
```
</details>

<details>
<summary><b>Sway</b></summary>

Add to `~/.config/sway/config`:
```bash
exec wl-clip-persist --clipboard regular
```
</details>

<details>
<summary><b>GNOME (Wayland)</b></summary>

Create `~/.config/autostart/wl-clip-persist.desktop`:
```ini
[Desktop Entry]
Type=Application
Name=wl-clip-persist
Exec=wl-clip-persist --clipboard regular
X-GNOME-Autostart-enabled=true
```
</details>

<details>
<summary><b>KDE Plasma (Wayland)</b></summary>

Create `~/.config/autostart/wl-clip-persist.sh`:
```bash
#!/bin/bash
wl-clip-persist --clipboard regular
```
Make it executable: `chmod +x ~/.config/autostart/wl-clip-persist.sh`
</details>

**Then start it now (or reboot):**
```bash
wl-clip-persist --clipboard regular &
```

## Usage

```bash
# Copy a file to clipboard with multi-MIME
wl-clip-multi ~/Pictures/screenshot.png
```

That's it. The daemon runs in foreground. Press Ctrl+C to stop, or run in background with `&`.

### Integration Example: Screenshots

See `examples/screenshot-multi-clipboard` for a complete screenshot workflow that:
1. Captures region with `grim` + `slurp`
2. Saves to `~/Pictures/`
3. Copies to clipboard with all MIME types

You can bind this to your Print key for universal screenshot clipboard.

## How It Works

1. Connects directly to Wayland compositor
2. Creates a clipboard data source
3. Offers 6 MIME types simultaneously
4. Serves the appropriate type when apps request clipboard content
5. Mimics GNOME Nautilus clipboard format for maximum compatibility

## Technical Details

See [TECHNICAL.md](TECHNICAL.md) for:
- MIME type specifications and byte-level formats
- Wayland protocol implementation details
- Comparison with other clipboard tools
- Debugging and development info

## Why This Works Universally

This isn't an Omarchy-specific solution or a Hyprland hack. It's a **Wayland protocol solution** that works on:

- ✅ Hyprland
- ✅ Sway
- ✅ GNOME (Wayland session)
- ✅ KDE Plasma (Wayland session)
- ✅ Any Wayland compositor

The problem (wl-clipboard limitation) affects everyone on Wayland. The solution (direct protocol access) works everywhere.

## License

GPL-3.0

## Author

Ozan Demirezen

## Links

- [wl-clipboard issue #71](https://github.com/bugaevc/wl-clipboard/issues/71) - Multi-MIME discussion
- [wl-clip-persist](https://github.com/Linus789/wl-clip-persist) - Essential companion tool
- [pywayland](https://github.com/flacjacket/pywayland) - Python Wayland bindings
