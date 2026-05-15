# A Practical User Reference

The official zathura documentation covers launch flags, keybindings, and configuration options thoroughly. What it does not cover is how zathura actually works as a tool — what the mode system means, which settings are worth changing, how to configure it for real use, and why things sometimes look wrong.

This reference fills that gap.

---

## What Is Zathura and Who Is It For

Zathura is a minimalist document viewer for Linux. It opens PDFs, EPUBs, DjVu files, PostScript documents, and other formats depending on which plugins are installed. The interface has no toolbar, no menu bar, and no sidebar by default. Navigation, search, zoom, and everything else is handled through the keyboard.

This makes zathura fast and distraction-free, but it requires learning a small set of commands before it becomes comfortable. The learning curve is short — most users are productive within a session or two.

Zathura is a good fit for:

- Reading academic papers and technical documentation
- LaTeX users who want tight editor integration via SyncTeX
- Users who prefer keyboard-driven workflows
- Anyone who finds heavier PDF viewers (Okular, Evince) slower than they need

It is not a good fit for filling PDF forms, annotating documents extensively, or viewing complex interactive PDFs.

---

## Installation

### Arch Linux

```bash
sudo pacman -S zathura zathura-pdf-mupdf
```

The `zathura-pdf-mupdf` plugin is the most commonly used PDF backend. Alternatively, `zathura-pdf-poppler` is available if you prefer the poppler rendering engine.

For additional format support:

```bash
sudo pacman -S zathura-djvu       # DjVu support
sudo pacman -S zathura-ps         # PostScript support
sudo pacman -S zathura-cb         # Comic book archives
```

### Debian / Ubuntu

```bash
sudo apt install zathura
```

### Fedora

```bash
sudo dnf install zathura zathura-pdf-mupdf
```

### macOS (via Homebrew)

```bash
brew install zathura
brew install zathura-pdf-mupdf
```

---

## The Mode System

This is the most important thing to understand about zathura before using it.

Zathura operates in four distinct modes. What your keyboard does depends entirely on which mode you are currently in. If zathura behaves unexpectedly, the first question to ask is: which mode am I in?

|Mode|Purpose|How to Enter|How to Exit|
|---|---|---|---|
|**Normal**|Navigation, zoom, search, most commands|Default on launch|Already here; press `Escape` if unsure|
|**Index**|Browse the document's table of contents|Press `Tab` from Normal|Press `Tab` or `Escape`|
|**Fullscreen**|Distraction-free reading, limited controls|Press `F11` from Normal|Press `F11`|
|**Presentation**|Slide-style view, minimal controls|Press `F5` from Normal|Press `F5`|

**The most important habit:** if zathura stops responding to expected keys, press `Escape`. This returns you to Normal mode from any other state.

---

## Your First Session

### Opening a file

From the terminal:

```bash
zathura document.pdf
```

Open at a specific page:

```bash
zathura -P 42 document.pdf
```

Open and search for a term immediately:

```bash
zathura -f "search term" document.pdf
```

From inside zathura, open a new document with `o`. This opens a file picker in the inputbar at the bottom of the screen. Type a path or filename and press `Enter`. Tab completion works.

### Navigating and searching

Use `j`/`k` to scroll down and up, `J`/`K` to move between pages, `gg` to go to the first page, and `G` to go to the last. Press `nG` to jump to a specific page number. See the full keybinding reference below for all navigation and zoom options.

Press `/` to search forward or `?` to search backward. Results highlight as you type. Use `n`/`N` to move between matches, and `Escape` to clear highlights.

### Quitting

Press `q` to quit.

---

## Keybindings by Workflow

### Page and document navigation

|Key|Action|
|---|---|
|`J` / `K`|Next / previous page|
|`gg`|First page|
|`G`|Last page|
|`nG`|Go to page n|
|`P`|Snap to current page (re-centres view)|
|`H`|Go to top of current page|
|`L`|Go to bottom of current page|
|`Ctrl+o`|Jump backward in history|
|`Ctrl+i`|Jump forward in history|

### Scrolling

|Key|Action|
|---|---|
|`h` / `l`|Scroll left / right|
|`j` / `k`|Scroll down / up|
|`Space`|Scroll full page down|
|`Shift+Space`|Scroll full page up|
|`Ctrl+d`|Scroll half page down|
|`Ctrl+u`|Scroll half page up|
|`Ctrl+f`|Scroll full page down (alternative)|
|`Ctrl+b`|Scroll full page up (alternative)|

### Zoom and view

|Key|Action|
|---|---|
|`+` / `-` / `=`|Zoom in / out / reset|
|`a`|Best-fit zoom|
|`s`|Width-fit zoom|
|`n=`|Zoom to n percent|
|`r`|Rotate 90 degrees clockwise|
|`Ctrl+r`|Toggle recolor (grayscale invert)|
|`d`|Toggle dual page view|
|`D`|Cycle first column in dual page view|
|`F5`|Toggle presentation mode|
|`F11`|Toggle fullscreen mode|

### Search

|Key|Action|
|---|---|
|`/`|Search forward|
|`?`|Search backward|
|`n` / `N`|Next / previous result|
|`Escape`|Clear search and return to Normal|

### Links

|Key|Action|
|---|---|
|`f`|Follow a link (opens linked page or URL)|
|`F`|Display the link target without following|
|`c`|Copy link target to clipboard|

### Bookmarks and quickmarks

Zathura has two bookmark systems: persistent bookmarks (saved to disk) and quickmarks (single-session, single-key).

**Bookmarks** (saved between sessions):

| Command | Action |
|---|---|
| `:bmark` | Save a bookmark at the current page |
| `:blist` | List all bookmarks |
| `:bjump [name]` | Jump to a bookmark |
| `:bdelete [name]` | Delete a bookmark |


**Quickmarks** (current session only):

|Key|Action|
|---|---|
|`mX`|Set a quickmark at letter or number X|
|`'X`|Jump to quickmark X|

For example, `ma` sets a quickmark at `a`, and `'a` returns to it.

### Interface toggles

|Key|Action|
|---|---|
|`Ctrl+m`|Toggle inputbar (command line at bottom)|
|`Ctrl+n`|Toggle statusbar|
|`Tab`|Toggle index (table of contents)|

---

## Index Mode

Index mode displays the document's table of contents in a panel, allowing you to jump directly to any section.

Press `Tab` from Normal mode to enter Index mode.

|Key|Action|
|---|---|
|`j` / `k`|Move down / up through entries|
|`l`|Expand a collapsed entry|
|`L`|Expand all entries|
|`h`|Collapse an entry|
|`H`|Collapse all entries|
|`Space` or `Enter`|Jump to the selected entry|
|`gg` / `G`|Jump to first / last entry|
|`Tab` or `Escape`|Exit Index mode and return to Normal|

If the document has no table of contents, Index mode will open but display nothing.

---

## The Command Interface

Press `:` from Normal mode to open the command inputbar. Tab completion works — press `Tab` to cycle through available completions.

### Commonly used commands

| Command                   | Description                                                                        |
| ------------------------- | ---------------------------------------------------------------------------------- |
| `:o [path]`               | Open a document                                                                    |
| `:jumplist`               | Show recent jump history (last 5 by default)                                       |
| `:jumplist 20`            | Show last 20 jumps                                                                 |
| `:nohl`                   | Remove search highlights                                                           |
| `:hlsearch`               | Re-enable search highlights                                                        |
| `:info`                   | Show document metadata (title, author, pages, etc.)                                |
| `:print`                  | Open the print dialog                                                              |
| `:write`                  | Save the document                                                                  |
| `:write!`                 | Save and overwrite without prompting                                               |
| `:export [id] [filename]` | Export an attachment from the document                                             |
| `:offset [n]`             | Set a page offset (useful when document page numbers don't match PDF page numbers) |
| `:dump`                   | Write all current settings to a file                                               |
| `:q`                      | Quit                                                                               |

### Executing external commands

The `:exec` command (aliased as `:!`) runs an external command from within zathura. The following variables are available:

- `$FILE` — the current document path
- `$PAGE` — the current page number
- `$DBUS` — the D-Bus bus name for the current instance

Example: open the current document in another application:

```
:exec evince $FILE
```

---

## Configuration

Zathura is configured through a plain text file called `zathurarc`. Zathura reads configuration from:

1. `/etc/zathurarc` (system-wide)
2. `~/.config/zathura/zathurarc` (user-specific, takes priority)

If the file does not exist, create it:

```bash
mkdir -p ~/.config/zathura
touch ~/.config/zathura/zathurarc
```

Each line in the file is evaluated independently. Comments begin with `#`.

### Setting options

```
set <option> <value>
```

Examples:

```
set option1 5           # integer
set option2 2.0         # float
set option3 "hello"     # string with spaces
set option4 true        # boolean
set default-fg "#CCBBCC"  # color (quote the hash)
```

### A recommended starting configuration

The following is a practical starting point. It addresses the most common complaints new users have about zathura's defaults.

```
# Use sqlite for bookmark storage (the plain backend is deprecated)
set database sqlite

# Open documents in width-fit mode instead of best-fit
set adjust-open width

# Recolor mode: dark background with light text (easier on the eyes)
set recolor true
set recolor-darkcolor "#FFFFFF"
set recolor-lightcolor "#1C1C1C"

# Keep original image colors when recoloring (prevents image inversion)
set recolor-reverse-video true

# Remember last position when reopening documents
set open-first-page false

# Show page number in the window title
set window-title-page true

# Use ~ instead of full home path in title
set window-title-home-tilde true

# Slightly larger scroll step for faster navigation
set scroll-step 80

# Stop at page boundaries when scrolling by half or full pages
set scroll-page-aware true

# Use system clipboard instead of primary selection for copied text
set selection-clipboard clipboard

# Incremental search (highlights as you type)
set incremental-search true

# Font for the interface
set font "monospace normal 11"
```

### Remapping keybindings

Use `map` to add or change keybindings:

```
map [mode] <key> <function> [argument]
```

The mode is optional and defaults to Normal. Available modes are `normal`, `fullscreen`, `presentation`, and `index`.

Examples:

```
# Remap zoom in/out to Ctrl+= and Ctrl+-
map <C-=> zoom in
map <C--> zoom out

# Open a file with Ctrl+o
map <C-o> open

# In presentation mode, use arrow keys to advance slides
map [presentation] <Right> navigate next
map [presentation] <Left> navigate previous
```

To remove an existing keybinding:

```
unmap <key>
unmap [mode] <key>
```

### Including other config files

Split your configuration across multiple files using `include`:

```
include ~/.config/zathura/keybindings
include ~/.config/zathura/colors
```

---

## Useful Configuration Options Reference

This covers the options most users will actually want to change. For the full list, run `:dump` inside zathura or consult the zathurarc man page.

### Display

|Option|Default|What It Does|
|---|---|---|
|`adjust-open`|`best-fit`|How documents open: `best-fit` fits the whole page, `width` fits page width|
|`pages-per-row`|`1`|Number of pages shown side by side|
|`page-v-padding`|`1`|Vertical gap in pixels between pages|
|`page-h-padding`|`1`|Horizontal gap in pixels between pages|
|`scroll-step`|`40`|Pixels scrolled per keypress|
|`scroll-page-aware`|`false`|If true, page-scrolling stops at page boundaries|
|`scroll-wrap`|`false`|If true, wraps from last page to first|
|`zoom-step`|`10`|Percentage change per zoom keypress|
|`zoom-min`|`10`|Minimum zoom percentage|
|`zoom-max`|`1000`|Maximum zoom percentage|
|`vertical-center`|`false`|If true, centres the view vertically on the page midpoint|

### Appearance

|Option|Default|What It Does|
|---|---|---|
|`font`|`monospace normal 9`|Interface font and size|
|`default-bg`|`#000000`|Background color|
|`default-fg`|`#DDDDDD`|Default foreground color|
|`statusbar-bg`|`#000000`|Statusbar background|
|`statusbar-fg`|`#FFFFFF`|Statusbar text color|
|`recolor`|`false`|Enable recolor mode on launch|
|`recolor-darkcolor`|`#FFFFFF`|Color used for dark content in recolor mode|
|`recolor-lightcolor`|`#000000`|Color used for light content in recolor mode|
|`recolor-reverse-video`|`false`|Keep original image colors when recoloring|
|`recolor-keephue`|`false`|Preserve hue when recoloring|

### Behaviour

|Option|Default|What It Does|
|---|---|---|
|`database`|`plain`|Storage backend for bookmarks and history (`sqlite` recommended)|
|`open-first-page`|`false`|If true, always opens at page 1 instead of last remembered position|
|`incremental-search`|`true`|Highlight search results as you type|
|`abort-clear-search`|`true`|Clear search highlights when pressing Escape|
|`selection-clipboard`|`primary`|Which clipboard copied text goes to (`clipboard` for Ctrl+V paste)|
|`window-title-page`|`false`|Show current page number in window title|
|`window-title-basename`|`false`|Show only filename (not full path) in window title|
|`window-title-home-tilde`|`false`|Replace home path with ~ in window title|
|`statusbar-basename`|`false`|Show only filename in statusbar|
|`statusbar-page-percent`|`false`|Show position as percentage instead of page number|
|`continuous-hist-save`|`false`|Save position history at every page change, not only on close|
|`double-click-follow`|`true`|Whether double or single click follows a link|

---

## SyncTeX Integration

SyncTeX is a synchronisation system used with LaTeX. It allows you to jump from a position in your PDF directly to the corresponding source line in your editor (backward sync), and from a source line directly to the corresponding position in the PDF (forward sync).

Zathura supports both directions.

### Forward synchronisation (editor to PDF)

Forward sync jumps from your editor to the corresponding position in zathura. This is triggered from the editor side and requires passing position information to zathura via the command line.

```bash
zathura --synctex-forward LINE:COLUMN:SOURCEFILE document.pdf
```

For Neovim with the `vimtex` plugin, add to your configuration:

```vim
let g:vimtex_view_method = 'zathura'
```

VimTeX handles the forward sync command automatically.

For gvim, add the following to your vim configuration:

```vim
function! Synctex()
  execute "silent !zathura --synctex-forward " . line('.') . ":" . col('.') . ":" . bufname('%') . " " . g:syncpdf
  redraw!
endfunction
map <C-Enter> :call Synctex()<cr>
```

Then launch zathura with:

```bash
zathura -x "gvim --servername vim -c \"let g:syncpdf='$1'\" --remote +%{line} %{input}" document.pdf
```

### Backward synchronisation (PDF to editor)

Backward sync jumps from a position in zathura to the corresponding source line in your editor. Hold `Ctrl` and left-click on any position in the PDF to trigger it.

The editor command is set via the `synctex-editor-command` option in your zathurarc:

```
set synctex-editor-command "nvim --headless -c \"VimtexInverseSearch %{line} '%{input}'\""
```

Adjust the command for your editor. The `%{line}` and `%{input}` variables are expanded by zathura to the line number and source file path respectively.

To change the modifier key used for backward sync (default is `Ctrl`):

```
set synctex-edit-modifier shift
```

### Editors with built-in zathura support

- **LaTeXTools for Sublime Text** — zathura is a supported viewer
- **VimTeX** — full forward and backward sync support out of the box

---

## Sandbox Mode

Zathura includes an optional sandbox mode (`zathura-sandbox` binary) that applies a seccomp and landlock-based security sandbox. This restricts what zathura can do at the system level, which reduces the attack surface when opening untrusted documents.

The following features are disabled in sandbox mode:

- Saving and writing files
- Printing
- Bookmarks and history
- D-Bus integration
- SyncTeX support
- Input methods (ibus, etc.)

Use `zathura-sandbox` instead of `zathura` when opening documents from untrusted sources:

```bash
zathura-sandbox untrusted-document.pdf
```

Sandbox mode is experimental and currently tested only with glibc.

---

## Launch Flags Reference

|Flag|Description|Example|
|---|---|---|
|`-P, --page`|Open at a specific page|`zathura -P 10 doc.pdf`|
|`-f, --find`|Open and search for a string|`zathura -f "conclusion" doc.pdf`|
|`-w, --password`|Provide password for encrypted document|`zathura -w password doc.pdf`|
|`-c, --config-dir`|Use a custom config directory|`zathura -c ~/.config/zathura-alt doc.pdf`|
|`-d, --data-dir`|Use a custom data directory|`zathura -d /tmp/zathura-data doc.pdf`|
|`-l, --log-level`|Set log verbosity|`zathura -l debug doc.pdf`|
|`-x, --synctex-editor-command`|Set SyncTeX editor command|`zathura -x "nvim ..." doc.pdf`|
|`--synctex-forward`|Jump to a SyncTeX position|`zathura --synctex-forward 42:1:main.tex doc.pdf`|
|`--mode`|Start in a non-default mode|`zathura --mode presentation doc.pdf`|
|`--fork`|Fork into background|`zathura --fork doc.pdf`|

---

## Frequently Asked Questions

**Why is zathura not responding to my keypresses?**

You are likely in a mode other than Normal, or the inputbar is focused. Press `Escape` to return to Normal mode. If the inputbar is open (a colon prompt at the bottom), press `Escape` to dismiss it.

**How do I open a file from inside zathura without using the terminal?**

Press `o` in Normal mode. The inputbar opens and accepts a file path. Tab completion works. Press `Enter` to open the file.

**Why does text look blurry or pixelated?**

This is usually a rendering backend issue. If you are using `zathura-pdf-poppler`, try switching to `zathura-pdf-mupdf` — MuPDF generally produces sharper text rendering, particularly at non-standard zoom levels.

**How do I make zathura remember where I was in a document?**

Zathura saves the last position automatically when you close a document. If it is always opening at page 1, check your zathurarc for `set open-first-page true` and remove or change it. Also ensure `set database sqlite` is set, as the `plain` backend is deprecated and may not save history reliably. See Configuration → Behaviour.

**How do I copy text from a PDF?**

Drag to select text with the left mouse button. Selected text goes to the primary clipboard by default — paste with the middle mouse button or `Shift+Insert`. To copy to the system clipboard instead, set `selection-clipboard clipboard` in your zathurarc. See Configuration → Behaviour.

**Why are my bookmarks not saved between sessions?**

The default `plain` database backend is deprecated. Set `database sqlite` in your zathurarc. See Configuration → Behaviour.

**How do I view two pages side by side?**

Press `d` in Normal mode to toggle dual page view. Press `D` to change which column the first page appears in. To make this permanent, set `pages-per-row 2` in your zathurarc. See Configuration → Display.

**Why do my PDFs look inverted or have strange colors?**

Recolor mode may be enabled. Press `Ctrl+r` to toggle it off. To keep recolor mode but preserve original image colors, set `recolor-reverse-video true` in your zathurarc. See Configuration → Appearance.

**How do I set zathura as my default PDF viewer?**

```bash
xdg-mime default org.pwmt.zathura.desktop application/pdf
```

Verify with:

```bash
xdg-mime query default application/pdf
```

**Can zathura open password-protected PDFs?**

Yes. Either provide the password at launch with `-w yourpassword`, or open the document normally and zathura will prompt for it.

**Why is zathura not showing the table of contents?**

Press `Tab` to open Index mode. If the panel opens but is empty, the document does not have an embedded table of contents. This is common with scanned PDFs and some older documents.

**How do I use zathura with LaTeX?**

See the SyncTeX Integration section above. The short version: install VimTeX if you use Neovim or Vim, and add `let g:vimtex_view_method = 'zathura'` to your configuration.

---

## Troubleshooting

**Pages are not rendering or appear blank**

Ensure you have a PDF plugin installed. Zathura without a plugin cannot render any document.

```bash
# Arch
sudo pacman -S zathura-pdf-mupdf

# Debian/Ubuntu
sudo apt install zathura-pdf-mupdf
# or
sudo apt install zathura-pdf-poppler
```

**Text or graph characters are misaligned**

Zathura uses Unicode characters in its interface. If characters overlap or appear misaligned, the issue is with your terminal or display font. This does not affect document rendering — only the statusbar and inputbar appearance.

If you use Konsole or Yakuake, disable "Bi-Directional text rendering" in terminal settings.

**Colors look wrong or washed out**

If colours appear muted, your display or compositor may be applying colour correction. This is not specific to zathura. Check your compositor settings.

If recolor mode is causing unexpected results, press `Ctrl+r` to toggle it off, or adjust the recolor settings in your zathurarc. See Configuration → Appearance.

**Zathura crashes with large documents**

If `GDK_NATIVE_WINDOWS` is enabled in your environment, disable it:

```bash
unset GDK_NATIVE_WINDOWS
```

If `overlay-scrollbar` is enabled in `GTK_MODULES`, remove it:

```bash
export GTK_MODULES=""
```

**SyncTeX backward sync is not working**

Confirm that `synctex-editor-command` is set in your zathurarc and that the command is correct for your editor. Test the command manually in a terminal to verify it works independently of zathura.

Also confirm that `dbus-service` is enabled (it is by default):

```
set dbus-service true
```

SyncTeX requires D-Bus. If you are running zathura in sandbox mode, SyncTeX is not available.

---

_For bug reports, see [github.com/pwmt/zathura/issues](https://github.com/pwmt/zathura/issues). For plugin-specific issues, report to the relevant plugin repository._