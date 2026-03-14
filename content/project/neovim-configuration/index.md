+++
draft = false
date = '2026-03-14T09:21:13+07:00'
title = 'Neovim Configuration'
type = 'project'
description = 'A technical walkthrough of building a modular, Lua-based Neovim configuration — covering the migration from Vimscript, lazy-loading strategies, and applying software design principles to developer tooling.'
image = ''
repository = 'https://github.com/mnabila/nvimrc'
languages = ['lua']
tools = ['neovim']
+++

Most developers have a text editor. Some developers have a development environment they have deliberately engineered over time. This project falls into the second category — a Neovim configuration built from scratch in Lua, designed with the same principles I apply to production software: modularity, performance, and maintainability.

It is the tool I use every single day, across every project I work on.

## Problem Background

Neovim configuration tends to follow a predictable lifecycle. It starts as a small `init.vim` with a handful of settings. Over months, plugins accumulate, keybindings pile up, and the whole thing becomes a monolithic file that is difficult to navigate, debug, or extend.

I hit that wall. My Vimscript-based config had grown into something I was afraid to modify. Adding a new plugin meant scrolling through hundreds of lines. Debugging a conflict required understanding the entire file. The configuration had become a liability rather than an asset.

I needed to start fresh with a proper architecture.

## Solution Overview

The solution was a complete rewrite in Lua — Neovim's first-class configuration language. Rather than simply porting the old Vimscript line by line, I redesigned the entire structure around separation of concerns: every configuration aspect lives in its own module, plugins are lazy-loaded for performance, and the whole setup is organized the way you would organize a real codebase.

**Tech stack:** Lua, Neovim, lazy.nvim

**My role:** Sole developer and daily user

## System Architecture

The architecture follows a modular pattern where each file has a single responsibility:

```
init.lua              -- Entry point, bootstraps everything
lua/config/           -- Core configuration modules
  colorscheme.lua     -- Theme and highlight customization
  plugins.lua         -- Plugin declarations and lazy-load rules
  options.lua         -- Editor-level settings
  keymaps.lua         -- Key binding definitions
lsp/                  -- Per-language LSP server configurations
colors/               -- Custom colorscheme definitions
snippets/             -- Reusable code snippets
after/ftplugin/       -- Filetype-specific overrides
```

The key design decision was treating `init.lua` purely as an entry point that delegates to specialized modules. This means any configuration change — whether it is a new keybinding, a plugin addition, or a language server setup — has exactly one place where it should go. There is no ambiguity about where things live.

Plugin management is handled by [lazy.nvim](https://github.com/folke/lazy.nvim), which loads plugins on demand based on events, commands, or filetypes. This keeps startup time consistently fast regardless of how many plugins are installed.

## Key Features

- **Gruvbox theme** with custom highlight group overrides tuned for code readability across multiple languages
- **Native LSP integration** configured per-language in the `lsp/` directory, providing code completion, inline diagnostics, go-to-definition, and refactoring support out of the box
- **Telescope** for fuzzy finding across files, buffers, grep results, LSP symbols, and Git history
- **sfm.nvim** as a lightweight, distraction-free file explorer
- **gitsigns.nvim and neogit** for inline Git diff indicators and a full Git workflow without leaving the editor
- **Custom code snippets** for patterns I use frequently across different filetypes
- **Filetype-specific settings** via `after/ftplugin/` so each language gets its own indentation, formatting, and behavior rules

## Technical Challenges and Solutions

**Migrating from Vimscript to Lua.** The biggest challenge was not the rewrite itself but resisting the temptation to replicate the old structure. Vimscript encourages a flat, imperative style. Lua enables a modular, table-driven approach. The key was embracing Lua idioms rather than translating Vimscript line by line.

**Balancing plugin count with startup performance.** More plugins means more capability, but also slower startup — unless you are intentional about it. By configuring lazy-load triggers for every plugin (event-based, command-based, or filetype-based), the config maintains sub-100ms startup times despite having a substantial plugin set.

**LSP configuration per language.** Each language server has its own quirks — different initialization options, different capabilities, different formatting behaviors. Moving LSP configurations into individual files under `lsp/` made it possible to tune each one independently without creating a giant conditional block.

## Lessons Learned

**Treat your tools like production code.** The same principles that make software maintainable — single responsibility, clear module boundaries, explicit dependencies — apply to configuration. A well-structured config is one you are not afraid to change.

**Lazy loading is a design decision, not an optimization.** Choosing when and why each plugin loads forces you to understand your own workflow. It revealed plugins I had installed but never actually used.

**Modularity enables experimentation.** With each concern isolated in its own file, swapping out a colorscheme or trying a new file explorer is a one-file change. That low cost of experimentation led to better choices over time.

## Conclusion

This configuration is open source and designed to be forked. Getting started takes one command:

```bash
git clone https://github.com/mnabila/nvimrc ~/.config/nvim
```

On first launch, lazy.nvim bootstraps itself and installs all dependencies automatically. From there, each module can be customized independently:

| What to change         | Where to look                  |
|------------------------|--------------------------------|
| Theme and appearance   | `lua/config/colorscheme.lua`   |
| Plugins                | `lua/config/plugins.lua`       |
| Editor settings        | `lua/config/options.lua`       |
| Key mappings           | `lua/config/keymaps.lua`       |
| Language servers       | `lsp/` directory               |
| Per-filetype behavior  | `after/ftplugin/`              |

Building your own editor configuration is more than a convenience — it is an exercise in software design. The same discipline that produces clean production code produces a development environment that genuinely makes you more productive.
