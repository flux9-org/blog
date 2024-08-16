---
title: "Neovim and Tmux Developer Setup for Platformio C/C++"
date: 2024-08-16
draft: false
summary: "Neovim Developer Setup for Platformio C/C++"
tags: ["developer_env"]
---

## Neovim and Tmux Setup
This page goes over a Neovim and Tmux setup with some specifics for microcontroller development with platformio. It's a personalised setup that I've found works for me for fast code editing and file movement. This setup can also be used for a wide variety of languages and frameworks in web development etc.

### Neovim with Clangd LSP (With ESP32 Support)
I'm using [AstroNvim](https://github.com/AstroNvim/AstroNvim) as a base config.

In the Neovim community there is some argument over using a distro like AstroNvim or not. My view on it is that I don't overly care about how to make my editor look nice, or how to setup gitsigns, or which Lua plugins are popular and effective currently. For me AstroNvim handles all of that by default. I remove plugins I don't need like the landing screen `{ "goolord/alpha-nvim", enabled = false }`.

Running your own configuration can clearly give you much better understanding and control over your editor and that is fair if you want or need that. 

Starting on relevant parts of the config. Some overall vim options, no mouse, and relativenumbers so you can do operations like `14k` to jump up to a specific line 14 above current:

```lua
---@type LazySpec
return {
	"AstroNvim/astrocore",
	---@type AstroCoreOpts
		options = {
			opt = {
				relativenumber = true,
				number = true,
				spell = false,
				signcolumn = "auto",
				wrap = true,
				mouse = "",
			},
		},
	  ...
}
```

So onto the actual specific configuration for Platformio C/C++. With Mason specify the `clangd` language server:

```lua
return {
  {
    "williamboman/mason-lspconfig.nvim",
    opts = function(_, opts)
      opts.ensure_installed = require("astrocore").list_insert_unique(opts.ensure_installed, {
        ...
        "clangd",
        ...
      })
    end,
  },
}
```

If your distro has a `clangd` installation binary in its package manager(On Fedora `sudo dnf install clang-extra-tools -y`) then you you can add it to a `cmd` configuration with a permissive `--query-driver` setting:

```lua
return {
	"AstroNvim/astrolsp",
		clangd = {
			cmd = {
				"clangd",
				"--query-driver=**",
			},
			capabilities = { offsetEncoding = "utf-8" },
		},
}
```

Now in each Platformio project you need to run `platformio run -t compiledb` everytime you add a new dependency and the LSP should function fine. Without doing this there is no `compile_commands.json` file generated and the Clangd LSP doesn't work correctly as it doesn't have the information it needs to compile the microcontroller code.

Specific for ESP32, If this approach of a default clangd binary and permissive query-driver doesn't work for you, can also try building the `xtensa_clangd` and add that to your LSP `cmd`: https://github.com/espressif/esp-idf/issues/6721#issuecomment-997150632
In the same github issue there is people who have as good LSP support just with the permissive query-driver: https://github.com/espressif/esp-idf/issues/6721#issuecomment-1593710089

#### Opening files: Using Neotree with Vim-keybind Pane Movements and Telescope
There are more advanced methods of moving around in files like ThePrimeagens harpoon: https://github.com/ThePrimeagen/harpoon. 

I've found that a combination of Neotree with Telescope to find files can also work well with little mental overhead required. If you don't know exactly the name of the file, you just navigate to it using Neotree, and if you do know the name you use Telescope.

Vim-keybind movements between panes by setting your mappings as follows with `s` as a prefix. `s` is an inbuilt vim operation for substituion but it's not that useful as a keybind(`r` and `R` work fine) so can be overridden:

```lua
mappings = {
	["<SPACE>"] = { "<Nop>" },
	["s"] = { "<Nop>" },

	-- astro defaults overrides these to gj, just use gj for moving in-between wrapped lines
	["j"] = { "j" },
	["k"] = { "k" },

	-- vertical movement
	["<C-d>"] = { "<C-d>zz" },
	["<C-u>"] = { "<C-u>zz" },

	-- pane movement
	["sh"] = { "<C-w>h" },
	["sl"] = { "<C-w>l" },
	["sk"] = { "<C-w>k" },
	["sj"] = { "<C-w>j" },

	-- pane splits
	["sy"] = { ":vsplit<CR>" },
	["su"] = { ":split<CR>" },

	...

	-- PLUGINS
	-- telescope
	["<Leader>a"] = { ":Telescope find_files<CR>" },
	["<Leader>s"] = { ":Telescope live_grep<CR>" },
}
```

Neotree configuration to open automatically:

```lua
-- open neo-tree on startup
local neo_tree_group = vim.api.nvim_create_augroup("neo_tree_group", { clear = true })
vim.api.nvim_create_autocmd("VimEnter", {
	command = "lua vim.defer_fn(function() vim.cmd('Neotree show left') end, 10)",
	group = neo_tree_group,
})
```

Neotree configuration:

```lua
---@type LazySpec
return {
  {
    -- NOTE: in future might try to enable horizontal scrolling because names get cut-off, see: https://github.com/nvim-neo-tree/neo-tree.nvim/discussions/267
    -- see: https://github.com/nvim-neo-tree/neo-tree.nvim/issues/348
    "neo-tree.nvim",
    lazy = false,
    opts = {
      window = {
        position = "left",
        width = 32,
        mappings = {
          -- unbind 's' as its used for movement
          ["s"] = false,
          ["o"] = { "open", nowait = true },
          ["oc"] = "noop",
          ["od"] = "noop",
          ["og"] = "noop",
          ["om"] = "noop",
          ["on"] = "noop",
          ["os"] = "noop",
          ["ot"] = "noop",
          ["f"] = "open_vsplit",
          ["/"] = "filter_on_submit",
          ["T"] = "clear_filter",
        },
      },
      filesystem = {
        filtered_items = {
          -- show hidden files and filtered items
          visible = true,
        },
      },
      event_handlers = {
        {
          event = "neo_tree_buffer_enter",
          handler = function()
            vim.cmd([[
                setlocal number
                setlocal relativenumber
              ]])
          end,
        },
      },
    },
  },
}
```

And optionally a telescope plugin configuration for navigating in results:

```lua
return {
  {
    "telescope.nvim",
    lazy = false,
    opts = function()
      local actions = require("telescope.actions")
      return {
        defaults = {
          path_display = { "truncate" },
          sorting_strategy = "ascending",
          layout_config = {
            horizontal = {
              prompt_position = "top",
              preview_width = 0.55,
            },
            vertical = {
              mirror = false,
            },
            width = 0.87,
            height = 0.80,
            preview_cutoff = 120,
          },
          mappings = {
            i = {
              ["<C-n>"] = actions.cycle_history_next,
              ["<C-p>"] = actions.cycle_history_prev,
              ["<C-j>"] = actions.move_selection_next,
              ["<C-k>"] = actions.move_selection_previous,
            },
            n = { ["q"] = actions.close },
          },
          file_ignore_patterns = {
            ".git/",
          },

          -- custom changes to vimgrep, not in AstroNvim defaults
          vimgrep_arguments = {
            "rg",
            "--color=never",
            "--no-heading",
            "--with-filename",
            "--line-number",
            "--column",
            "--smart-case",
            "--hidden",
          },
        },
      }
    end,
  },
}
```

With all of this configured if you can type `sh` in vim normal mode(not insert) the cursor will jump to the NeoTree pane and you can move about, use `o` or `Enter` to open directories and files. Use `f` on a highlighted file to open it in a vertical split and keep the current pane open to edit 2 files or more with each visible.

Then if you have a large codebase with a deep directory structure(Java...) then using NeoTree and opening each directory can become tedious, instead use `<leader>a`(leader is space by default in AstroNvim) in normal mode and type part of the filename. Use vim-like `<Ctrl>-j` to move down in results, `Enter` to open in current pane or `<Ctrl>-v` to open in a vertical split. Alternatively use `<leader>s` to search for specific text within files instead of filenames.

### Tmux for terminal panes

With Neovim configured you have a system for editing code quickly. Very often though you need to run shell commands such as `pio run -t upload`, or ssh into a remote system, etc. There are some Neovim in-built terminal systems but I find that they just can't replicate using Tmux which naturally keeps your Neovim pane totally separate from your terminal pane.

Here is an example Tmux configuration, also with Vim-like keybinds after you use the prefix `<Ctrl-b>`, so `<Ctrl-b>j` to move to the terminal pane you've open below the Neovim pane:

```
# Set 'v' for vertical and 'h' for horizontal split
bind y split-window -h -c "#{pane_current_path}"
bind -r u split-window -v -c "#{pane_current_path}"
bind -r * split-window -v -l 30% -c "#{pane_current_path}"

# vim-like pane switching
bind -r k select-pane -U
bind -r j select-pane -D
bind -r h select-pane -L
bind -r l select-pane -R

# vim-like pane resizing
bind -r C-k resize-pane -U
bind -r C-j resize-pane -D
bind -r C-h resize-pane -L
bind -r C-l resize-pane -R

# remove default binding since replacing
unbind %
unbind Up
unbind Down
unbind Left
unbind Right

unbind C-Up
unbind C-Down
unbind C-Left
unbind C-Right

# Enter Tmux copy-mode with ctrl-b [, then hit 'v' to enter visual mode, then 'y' to yank
# Now to paste we hit ctrl-b P
bind P paste-buffer
bind-key -T copy-mode-vi v send-keys -X begin-selection
bind-key -T copy-mode-vi y send-keys -X copy-selection
bind-key -T copy-mode-vi r send-keys -X rectangle-toggle

# Fix colours in Neovim being different in Tmux
set -g default-terminal "screen-256color"
set -ga terminal-overrides ",*256col*:Tc"

# Increase scrollback
set-option -g history-limit 8000

# Plugin List
set -g @plugin 'tmux-plugins/tpm'
# set -g @plugin 'arcticicestudio/nord-tmux'
set -g @plugin 'catppuccin/tmux'
# set -g @catppuccin_flavour 'frappe'
set -g @catppuccin_flavour 'macchiato'

# trigger plugin installs with <prefix>I
run '~/.tmux/plugins/tpm/tpm'
```

Now when you want to start development in a directory, open a shell there, see the [zsh-z](https://github.com/agkozak/zsh-z) plugin which can save a ton of time instead of typing out full `cd` commands each time. 

Then use commands `tmux`, `nvim` and `<Ctrl-b>*` to open a bottom terminal pane. With this now you can edit code in the directory in the top pane, and in the smaller bottom pane jump down to it with `<Ctrl-b>j`, run a commands, and back up to Neovim with `<Ctrl-b>k`.

Then to go further with Tmux you can easily create new full windows(a set of panes) with `<Ctrl-b>c` and move between them with `<Ctrl-b>n`. This becomes very useful if you need to be ssh'd into a remote server to run some commands in one window while also editing some code in the previous window.

## Image
An example image, can see some LSP errors present in dependencies(seems near unavoidable even with `clangd_xtensa`), and a bottom tmux pane where you can run bash or zsh shell commands:

![example_image](assets/neovim_tmux_example.png)
