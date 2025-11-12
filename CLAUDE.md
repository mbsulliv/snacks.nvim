# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

snacks.nvim is a collection of small QoL (Quality of Life) plugins for Neovim. It's a modular plugin system where each "snack" is an independent module that can be enabled/disabled individually.

## Development Commands

### Testing
```bash
./scripts/test
# Or directly:
nvim -l tests/minit.lua --minitest
```

### Documentation Generation
```bash
./scripts/docs
```
This script:
- Generates markdown docs using `snacks.meta.docs.build()`
- Converts markdown to vim help files using panvimdoc (local setup only, not in CI)
- Fixes help file titles using `snacks.meta.docs.fix_titles()`

### Code Formatting
```bash
# Format Lua code with stylua (2 spaces, 120 column width)
stylua .
```

### Linting
The project uses selene for Lua linting with `std="vim"` configured in `selene.toml`.

## Architecture

### Module System

The core architecture uses lazy module loading:

- **Entry point**: `lua/snacks/init.lua`
  - Uses a metatable to lazy-load modules on first access
  - `Snacks.<module>` automatically requires `snacks.<module>` when accessed
  - Global `_G.Snacks` provides access to all snack modules

- **Module structure**: Each snack lives in `lua/snacks/<name>.lua` or `lua/snacks/<name>/init.lua`
  - Simple snacks are single files (e.g., `bigfile.lua`, `toggle.lua`, `rename.lua`, `bufdelete.lua`, `keymap.lua`)
  - Complex snacks use directories with submodules (e.g., `picker/`, `image/`, `explorer/`, `gh/`, `profiler/`)

- **Configuration system**:
  - Centralized config in `M.config` table
  - `M.config.get(snack, defaults, ...)` merges defaults with user config
  - `M.config.merge()` deep merges configs, handling both tables and primitives
  - Supports `example` field to load pre-built configs from `docs/examples/<snack>.lua`
  - Each snack can have a `config()` function hook for post-processing options

### Setup and Initialization

The setup process (in `lua/snacks/init.lua`) is event-driven:

- `M.setup(opts)` is called early in Neovim startup (before VimEnter)
- Modules are loaded lazily based on autocommands:
  - `BufReadPre`: bigfile, image
  - `BufReadPost`: quickfile, indent
  - `BufEnter`: explorer
  - `LspAttach`: words
  - `UIEnter`: dashboard, scroll, input, scope, picker
- Special handling for image files via `BufReadCmd` autocmd for image formats

### Core Libraries

Several modules are libraries used by other snacks:

- **win** (`lua/snacks/win.lua`): Window management system
  - Creates and manages floating windows and splits
  - Handles positioning (float, top, bottom, left, right)
  - Manages borders, backdrops, dimensions
  - Key bindings and event handling per window
  - Used extensively by picker, notifier, input, dashboard, etc.

- **util** (`lua/snacks/util/`): Utility functions
  - Highlight group management
  - Treesitter language detection
  - Spawn process management (`util/spawn.lua`)
  - Job management (`util/job.lua`) for async job handling
  - LSP utilities (`util/lsp.lua`)
  - Generic helper functions

- **animate** (`lua/snacks/animate/`): Animation library
  - 45+ easing functions
  - Single-timer architecture (one timer for all animations)
  - Used by scroll, indent, dim, etc.
  - Can be disabled via `vim.g.snacks_animate` or `vim.b.snacks_animate`

- **layout** (`lua/snacks/layout.lua`): Window layout system
  - Box model for flexible layouts
  - Used by complex UIs like dashboard
  - Z-index management for proper window stacking

- **meta** (`lua/snacks/meta/`): Documentation and type metadata
  - Documentation generation (`meta/docs.lua`)
  - Type definitions (`meta/types.lua`)
  - Automated markdown and vim help file generation

- **debug** (`lua/snacks/debug.lua`): Debugging utilities
  - Pretty inspection of Lua values
  - Backtrace generation
  - Integration with notifier for debugging output

### Complex Modules

- **picker** (`lua/snacks/picker/`):
  - Sources in `source/` directory (files, grep, git, LSP, GitHub, etc.)
  - Core logic in `core/` (finder, filter, matcher, etc.)
  - Formatters, previewers, actions, sorting
  - Configuration in `config/` directory
  - Utilities in `util/` directory (async, db, diff, highlight, markdown, etc.)
  - Supports line metadata via `vim.b.snacks_meta` for interactive features

- **image** (`lua/snacks/image/`):
  - Kitty Graphics Protocol implementation
  - Format conversion (`convert.lua`) using ImageMagick/ffmpeg
  - Document rendering (`doc.lua`) for markdown images, PDF files
  - Inline image support (`inline.lua`)
  - Terminal interaction (`terminal.lua`)
  - Placement and rendering (`placement.lua`)

- **explorer** (`lua/snacks/explorer/`):
  - File tree explorer built on top of picker
  - Tree rendering and navigation

- **gh** (`lua/snacks/gh/`):
  - GitHub integration module
  - GitHub API wrapper (`api.lua`)
  - Actions for PRs, issues, diffs (`actions.lua`)
  - Buffer rendering (`buf.lua`)
  - Item formatting (`item.lua`)
  - Render utilities in `render/` directory
  - Features:
    - View and interact with GitHub PRs
    - Reply to review comments in diffs
    - Inline review comment annotations
    - GitHub diff viewer with fancy diff style

- **profiler** (`lua/snacks/profiler/`):
  - Performance profiling module
  - Core profiling functionality (`core.lua`)
  - Location tracking (`loc.lua`)
  - Tracer implementation (`tracer.lua`)
  - Picker integration (`picker.lua`)
  - UI for viewing profiles (`ui.lua`)

### Type Definitions

The project uses LuaLS annotations extensively:

- Main config types defined in module files as `@class` annotations
- Type aggregation in `lua/snacks/meta/types.lua` for documentation
- Each module typically has its config class named `snacks.<module>.Config`

## Recent Features & Changes

### GitHub Integration (gh module)
Recent additions include:
- Reply to review comments in diffs with `a` action
- Inline review comment annotations in diff viewer
- Force `fancy` diff style for proper review comment rendering
- Improved handling of pending requests
- Lua-based date parsing for performance in fast contexts

### Picker Enhancements
- Generalized line metadata support via `vim.b.snacks_meta`
- GitHub diff source improvements
- Enhanced diff utilities in `picker/util/diff.lua`
- Horizontal rule rendering support

### Layout Improvements
- Z-index calculation now ignores very high z-index windows to stay below notifications

### LSP Integration
- Proper buffer detachment on LspDetach events
- Fixed nil handling in LSP config picker

### Image Module
- Terminal capability detection now runs synchronously when needed for reliability

## Important Patterns

### Module Metadata
Each module should define:
```lua
M.meta = {
  desc = "Description of the module",
}
```

### Config Merging
When adding config options:
- Use `M.config.get(snack, defaults, user_opts)` to merge configs
- Deep merging preserves nested tables
- Later values override earlier ones

### Styles System
Windows can use predefined styles via `Snacks.config.styles`:
- Register styles with `M.config.style(name, defaults)`
- Reference in win configs via `style = "name"`
- Styles are merged with per-window config

### Compatibility Layer
`lua/snacks/compat.lua` provides compatibility for Neovim < 0.11:
- Global `svim` is either `vim` (0.11+) or the compat layer
- Use `svim.islist()` instead of `vim.islist()` for cross-version support

### Buffer Metadata
Some modules use buffer-local variables for metadata:
- `vim.b.snacks_meta`: Line metadata for picker items (used for interactive features like replying to comments)
- `vim.b.snacks_animate`: Per-buffer animation disable flag
- `vim.g.snacks_animate`: Global animation disable flag

### Async Operations
The project uses various async patterns:
- `util/spawn.lua`: Spawn external processes
- `util/job.lua`: Async job management
- `picker/util/async.lua`: Picker-specific async operations
- Proper error handling with backtraces for async errors

## Documentation

- Markdown docs in `docs/` are the source of truth
- Generated vim help files in `doc/`
- Module documentation uses special markers like `<!-- docgen -->` for auto-generation
- Examples can be embedded in docs and loaded via `example = "name"` in config
- Run `:checkhealth snacks` to verify setup

## Testing

- Tests use mini.test framework
- Test setup in `tests/minit.lua` downloads lazy.nvim bootstrap
- Tests use `.tests` directory as stdpath
- Individual test files in `tests/*_spec.lua`
- Complex modules have test subdirectories (e.g., `tests/image/`, `tests/picker/`)
