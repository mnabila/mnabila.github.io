+++
draft = false
date = '2026-03-14T09:21:13+07:00'
title = 'Neovim Configuration'
type = 'project'
description = 'A modern, Lua-based Neovim setup designed for efficiency and customization. Features lazy-loaded plugins, Gruvbox theme, LSP support, and modular configuration structure.'
image = ''
repository = 'https://github.com/mnabila/nvimrc'
languages = ['lua']
tools = ['neovim']
+++

Over the years, I've spent a lot of time tweaking and refining my Neovim setup. What started as a simple `init.vim` has evolved into a fully modular, Lua-based configuration that I use daily for all my development work. This project is the result of that journey — a Neovim config that's fast, clean, and easy to extend.

## Why Lua?

Neovim has supported Lua as a first-class configuration language for a while now, and honestly, once I made the switch from Vimscript, there was no going back. Lua is faster, more readable, and makes it much easier to organize configurations into separate modules. Everything in this setup is written in pure Lua.

## How It's Organized

One thing I really care about is keeping the config modular. Instead of dumping everything into a single file, each concern lives in its own place:

```
init.lua              -- Main entry point
lua/config/           -- Modular config files
  colorscheme.lua     -- Theme customization
  plugins.lua         -- Plugin management
  options.lua         -- General settings
  keymaps.lua         -- Keyboard shortcuts
lsp/                  -- Custom LSP setups
colors/               -- Custom colorscheme definitions
snippets/             -- Custom code snippets
after/ftplugin/       -- Filetype-specific settings
```

This structure makes it easy to find what I'm looking for. Need to change a keybinding? Go to `keymaps.lua`. Want to add a new plugin? Open `plugins.lua`. It keeps things manageable even as the config grows.

## Plugin Management with lazy.nvim

I use [lazy.nvim](https://github.com/folke/lazy.nvim) to manage plugins. The biggest advantage of lazy.nvim is that it lazy-loads plugins by default — they only get loaded when they're actually needed, whether that's triggered by an event, a command, or a specific filetype. This keeps Neovim's startup time fast, even with a decent number of plugins installed.

## What's Included

Here's a quick rundown of the main things this config ships with:

- **Gruvbox theme** with custom highlight tweaks to match my personal taste
- **LSP support** configured per-language inside the `lsp/` directory, giving me code completion, diagnostics, go-to-definition, and refactoring out of the box
- **Telescope** for fuzzy finding pretty much everything — files, buffers, grep results, you name it
- **sfm.nvim** as a lightweight file explorer
- **gitsigns.nvim and neogit** for Git integration directly inside the editor
- **Custom snippets** for code patterns I use frequently
- **Filetype-specific settings** through `after/ftplugin/` so each language gets its own treatment

## Getting Started

If you want to try it out, just clone the repo into your Neovim config directory:

```bash
git clone https://github.com/mnabila/nvimrc ~/.config/nvim
```

On the first launch, lazy.nvim will automatically bootstrap itself and install all the plugins. After that, you're good to go.

## Making It Your Own

The whole point of this config is that it's easy to customize. Here are the files you'd want to touch depending on what you're changing:

- **Theme** — edit `lua/config/colorscheme.lua`
- **Plugins** — add or remove in `lua/config/plugins.lua`
- **General settings** — tweak `lua/config/options.lua`
- **Key mappings** — modify `lua/config/keymaps.lua`
- **Language servers** — configure in the `lsp/` directory
- **Per-filetype behavior** — add files under `after/ftplugin/`

Feel free to fork it, break it apart, or just steal the parts you like. That's what dotfiles are for.
