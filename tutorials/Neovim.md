# Getting Started with Neovim: A Beginner's Guide

If you have never used a terminal text editor before, opening Neovim for the first time can feel disorienting. Nothing works the way you expect. Typing does not type. Closing the window seems impossible. This guide exists to get you past that wall as quickly as possible, and to show you why the effort is worth it.

---

## What Is Neovim and Why Use It

Neovim is a text editor that runs entirely inside a terminal. There is no menu bar, no toolbar, no mouse required. You navigate, select, edit, and save entirely through keyboard commands.

That sounds like a step backwards. In practice, for many people it becomes the opposite.

Once the commands become muscle memory, editing text in Neovim is significantly faster than reaching for a mouse. Neovim is also available on virtually every Linux and macOS system, which means if you ever work on a remote server or someone else's machine, you already know the editor that is there.

It is also, once configured, a genuinely powerful development environment. This guide covers the basics: enough to open a file, make changes, and leave without losing your work.

---

## Installation

**Arch Linux**
```bash
sudo pacman -S neovim
```

**Ubuntu / Debian**
```bash
sudo apt install neovim
```

**macOS (via Homebrew)**
```bash
brew install neovim
```

To confirm the installation worked:
```bash
nvim --version
```

To open Neovim:
```bash
nvim
```

To open a specific file:
```bash
nvim filename.txt
```

---

## The Concept of Modes

This is the most important thing to understand about Neovim, and the biggest difference from editors you may have used before.

In most editors, you open a file and start typing. In Neovim, the editor is always in one of several **modes**, and what your keyboard does depends entirely on which mode you are in.

| Mode | What it does | How to enter it |
|---|---|---|
| **Normal** | Navigate and run commands | Press `Esc` from any mode |
| **Insert** | Type text | Press `i` from Normal mode |
| **Visual** | Select text | Press `v` from Normal mode |
| **Command** | Run editor commands | Press `:` from Normal mode |

Neovim always opens in **Normal mode**. This is why pressing keys when you first open it does unexpected things — you are not typing, you are issuing commands.

The most important habit to build: **press `Esc` whenever you are unsure what mode you are in.** Esc always returns you to Normal mode, from anywhere.

---

## Surviving Your First Session

These are the absolute minimum commands you need to open a file, make a change, and leave safely.

### Opening and navigating

Open a file from the terminal:
```bash
nvim myfile.txt
```

In Normal mode, move the cursor using:

| Key | Direction |
|---|---|
| `h` | Left |
| `j` | Down |
| `k` | Up |
| `l` | Right |

Arrow keys also work, but `h j k l` are faster once learned because your hands never leave the home row.

### Entering Insert mode and typing

From Normal mode, press `i` to enter Insert mode. You will see `-- INSERT --` at the bottom of the screen. Now you can type normally.

When you are done typing, press `Esc` to return to Normal mode.

### Saving and quitting

These commands are entered in Command mode. From Normal mode, press `:` to open the command line at the bottom of the screen.

| Command | What it does |
|---|---|
| `:w` | Save (write) the file |
| `:q` | Quit |
| `:wq` | Save and quit |
| `:q!` | Quit without saving (force quit) |

Press `Enter` after each command to run it.

If you have made changes and try to quit without saving, Neovim will warn you. Use `:q!` if you want to discard your changes and leave.

---

## Essential Commands to Learn Next

Once you are comfortable with the basics above, these commands will make editing significantly faster.

| Command | What it does |
|---|---|
| `dd` | Delete the entire current line |
| `yy` | Copy (yank) the entire current line |
| `p` | Paste below the current line |
| `u` | Undo the last change |
| `Ctrl + r` | Redo |
| `/searchterm` | Search for text (press `Enter` to confirm, `n` for next result) |
| `:%s/old/new/g` | Replace all instances of "old" with "new" in the file |
| `gg` | Go to the top of the file |
| `G` | Go to the bottom of the file |

None of these need to be memorised immediately. Return to this table as you get comfortable and add one or two commands at a time.

---

## Installing LazyVim

Neovim on its own is a capable editor, but its real power comes from configuration and plugins. Setting this up from scratch takes time and expertise. **LazyVim** is a pre-configured Neovim setup that gives you a modern, full-featured editor out of the box, with sensible defaults and a plugin manager already in place.

It is the recommended starting point for anyone who wants more than a basic editor without spending days on configuration.

### Prerequisites

Before installing LazyVim, ensure the following are installed:

- `git`
- A [Nerd Font](https://www.nerdfonts.com/) set as your terminal font (for icons to display correctly)
- `ripgrep` (for file search functionality)

On Arch Linux:
```bash
sudo pacman -S git ripgrep
```

### Installation

First, back up any existing Neovim configuration:
```bash
mv ~/.config/nvim ~/.config/nvim.bak
mv ~/.local/share/nvim ~/.local/share/nvim.bak
```

Then clone the LazyVim starter configuration:
```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
```

Remove the `.git` folder so you can manage your own configuration going forward:
```bash
rm -rf ~/.config/nvim/.git
```

Open Neovim:
```bash
nvim
```

LazyVim will automatically install all plugins on first launch. Wait for this to complete, then restart Neovim.

---

## What LazyVim Gives You

Once installed, LazyVim provides several features that would otherwise require significant manual configuration.

**File tree**
Press `Space + e` to open and close a sidebar file explorer. Navigate your project directory without leaving the editor.

**Fuzzy finder**
Press `Space + Space` to open a fuzzy file finder. Start typing any part of a filename to locate it instantly across your entire project.

**Syntax highlighting**
LazyVim uses Treesitter to provide accurate, context-aware syntax highlighting for most common programming languages and file types out of the box.

**Status line**
The bar at the bottom of the screen shows your current mode, file name, cursor position, and git branch information at a glance.

**Which-key**
Press `Space` from Normal mode and wait a moment. A menu will appear showing all available keyboard shortcuts. This is particularly useful while you are still learning — you do not need to memorise everything at once.

---

## Next Steps

The built-in tutorial is the best place to continue learning core Neovim commands. Run it from inside Neovim:
```
:Tutor
```

It takes approximately thirty minutes and covers navigation, editing, and searching in a hands-on format.

For LazyVim-specific documentation, the official site covers all included plugins and key mappings:
[lazyvim.org](https://www.lazyvim.org)

For questions, troubleshooting, and community discussion:
- [r/neovim](https://www.reddit.com/r/neovim)
- [LazyVim GitHub Discussions](https://github.com/LazyVim/LazyVim/discussions)

---

*The learning curve is real but short. Most people feel comfortable within a week of daily use, and find it difficult to go back.*
