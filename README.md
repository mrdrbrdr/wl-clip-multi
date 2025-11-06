# wl-clip-multi

**Multi-MIME Type Wayland Clipboard Utility**

A clipboard tool for Wayland that offers multiple MIME types simultaneously, solving the limitation of standard `wl-clipboard` tools. This allows different applications to receive clipboard content in their preferred format from a single copy operation.

## The Problem

Standard Wayland clipboard tools (`wl-copy`, `wl-paste`) can only offer **one MIME type at a time**. This creates issues when you want to:

- Copy a screenshot and paste it as an **image** in Discord/Obsidian
- Copy the **same screenshot** and paste it as a **file path** in terminal applications like Claude Code or VS Code terminal
- Copy files from a file manager with proper compatibility across all applications

Applications that use the Wayland clipboard protocol directly (like GNOME Nautilus) can offer multiple MIME types, but command-line tools cannot.

## The Solution

`wl-clip-multi` uses the Wayland protocol directly (via `pywayland`) to offer **6 MIME types simultaneously**:

1. `x-special/gnome-copied-files` - GNOME file manager format (for terminals)
2. `text/plain;charset=utf-8` - Plain text file path
3. `text/uri-list` - Standard URI format
4. `application/vnd.portal.filetransfer` - XDG portal file transfer
5. `application/vnd.portal.files` - XDG portal files
6. `image/png` (or appropriate type) - Raw file content

Applications automatically choose their preferred MIME type when you paste.

## Features

- ✅ **Multi-MIME clipboard** - Offer 6 MIME types simultaneously
- ✅ **Universal compatibility** - Works with terminals, visual apps, browsers
- ✅ **GNOME Nautilus format** - Byte-perfect compatibility with file manager clipboard
- ✅ **Works with `wl-clip-persist`** - Clipboard persists after daemon exits
- ✅ **Simple Python script** - Easy to modify and extend
- ✅ **Direct Wayland protocol** - No external dependencies besides pywayland

## Installation

### From Source

```bash
# Clone the repository
git clone https://github.com/ozandemirezen/wl-clip-multi.git
cd wl-clip-multi

# Install dependencies
pip install pywayland

# Copy to your PATH
sudo cp bin/wl-clip-multi /usr/local/bin/
sudo chmod +x /usr/local/bin/wl-clip-multi
```

### Dependencies

- Python 3.6+
- `pywayland` - Python bindings for Wayland protocol
- A Wayland compositor (Hyprland, Sway, GNOME Wayland, etc.)

Optional but recommended:
- `wl-clip-persist` - Keeps clipboard alive after wl-clip-multi exits

## Usage

### Basic Usage

```bash
# Copy a file to clipboard
wl-clip-multi ~/Pictures/screenshot.png
```

The daemon will run in the foreground, serving clipboard requests. Press Ctrl+C to stop.

### Run in Background

```bash
# Run as background daemon
wl-clip-multi ~/Pictures/screenshot.png &
```

### Debug Mode

```bash
# See what MIME types applications request
wl-clip-multi --debug ~/Pictures/screenshot.png
```

### Integration with Screenshot Tools

See `examples/screenshot-multi-clipboard` for a complete screenshot workflow:

```bash
# Take screenshot, save to file, copy to clipboard with multi-MIME
./examples/screenshot-multi-clipboard
```

This script:
1. Uses `grim` + `slurp` to capture a region
2. Saves screenshot to `~/Pictures/`
3. Copies to clipboard with all MIME types
4. Works with `wl-clip-persist` for persistence

## Use Cases

### Screenshots in Claude Code Terminal

**Problem:** Copying screenshots with `wl-copy` only offers image data. Terminal applications like Claude Code need file paths.

**Solution:**
```bash
grim -g "$(slurp)" screenshot.png
wl-clip-multi screenshot.png &
# Now paste in Claude Code terminal - it receives the file path!
```

### Universal Screenshot Clipboard

```bash
# One screenshot, multiple paste behaviors:
wl-clip-multi screenshot.png &

# Paste in terminal → Gets file path
# Paste in Discord → Gets image data
# Paste in Obsidian → Gets image data
# Paste in VS Code → Gets file path or image based on context
```

### File Manager Compatibility

```bash
# Copy file like Nautilus does
wl-clip-multi ~/Documents/important.pdf &

# Paste in terminal → Gets file path
# Paste in browser upload → Gets file content
```

## How It Works

1. **Direct Wayland Protocol Access**: Uses `pywayland` to communicate with the Wayland compositor
2. **Data Source Creation**: Creates a clipboard data source offering 6 MIME types
3. **On-Demand Serving**: When apps request clipboard content, serves the appropriate MIME type
4. **GNOME Format Compatibility**: Uses byte-perfect GNOME Nautilus clipboard format (`copy\nfile:///path`)

## Comparison with Other Tools

| Tool | Multi-MIME | Use Case |
|------|-----------|----------|
| `wl-copy` | ❌ | Simple clipboard, single MIME type |
| `wl-clipboard-rs` | ❌ | Rust rewrite of wl-copy, single MIME type |
| `cliphist` | ❌ | Clipboard history manager, not multi-MIME copy |
| `wl-clip-persist` | ✅ | Preserves multi-MIME, but doesn't create it |
| **`wl-clip-multi`** | ✅ | Creates and serves multi-MIME clipboard |

`wl-clip-multi` works **with** `wl-clip-persist` - use both together for best results!

## Configuration with wl-clip-persist

For clipboard persistence after the daemon exits:

```bash
# Start wl-clip-persist at boot (recommended)
# Add to ~/.config/hypr/autostart.conf or equivalent:
exec-once = wl-clip-persist --clipboard regular
```

Then `wl-clip-multi` will serve the clipboard, and `wl-clip-persist` will preserve all MIME types even after `wl-clip-multi` exits.

## Technical Details

### MIME Types Offered

The tool offers MIME types in the exact order GNOME Nautilus uses:

1. `x-special/gnome-copied-files` - Format: `copy\nfile:///absolute/path` (NO trailing newline)
2. `text/plain;charset=utf-8` - Plain text file path
3. `text/uri-list` - Format: `file:///absolute/path\n` (WITH trailing newline)
4. `application/vnd.portal.filetransfer` - XDG portal protocol
5. `application/vnd.portal.files` - XDG portal files
6. File-specific MIME type (e.g., `image/png`, `application/pdf`)

### Why This Format?

Terminal applications like Claude Code expect the `x-special/gnome-copied-files` MIME type with the exact format Nautilus uses. The byte-level format matters - even a trailing newline breaks compatibility!

## Troubleshooting

### Clipboard Gets Cleared Immediately

**Problem:** Clipboard only works for a split second

**Cause:** Another clipboard watcher (like `wl-paste --watch` from `elephant-clipboard`) is reading the clipboard and re-copying with only 1-2 MIME types

**Solution:** Install and run `wl-clip-persist`:
```bash
# Install
sudo pacman -S wl-clip-persist  # Arch/Manjaro
# or
yay -S wl-clip-persist          # AUR

# Run at boot (add to autostart)
wl-clip-persist --clipboard regular &
```

### PyWayland Not Found

```bash
# Install pywayland
pip install pywayland

# or system package:
sudo pacman -S python-pywayland  # Arch/Manjaro
```

### Permission Denied

```bash
# Make sure the script is executable
chmod +x /usr/local/bin/wl-clip-multi
```

## Contributing

Contributions are welcome! Areas for improvement:

- Support for more file types (videos, audio, etc.)
- Better MIME type detection
- Integration examples for other screenshot tools
- Support for copying multiple files at once
- Configuration file for custom MIME type mappings

## Why This Exists

The `wl-clipboard` maintainer intentionally doesn't support multi-MIME because of CLI design complexity ([see issue #71](https://github.com/bugaevc/wl-clipboard/issues/71)). His recommendation: "Use the Wayland protocol directly."

That's exactly what `wl-clip-multi` does!

## License

GPL-3.0 - See LICENSE file

## Author

Ozan Demirezen

## Links

- GitHub: https://github.com/ozandemirezen/wl-clip-multi
- Related: [wl-clipboard](https://github.com/bugaevc/wl-clipboard)
- Related: [wl-clip-persist](https://github.com/Linus789/wl-clip-persist)
- Related: [pywayland](https://github.com/flacjacket/pywayland)
