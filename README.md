# :white_check_mark: DOs and :x: DON'Ts for Neovim Lua plugin development

> [!IMPORTANT]
>
> The code snippets in this document use the Neovim 0.10.0 API.

## :safety_vest: Type safety

Lua, as a dynamically typed language, is great for configuration.
It provides virtually immediate feedback.

### :x: DON'T

...make your plugin susceptible to unexpected bugs at the wrong time.

### :white_check_mark: DO

...leverage [LuaCATS annotations](https://luals.github.io/wiki/annotations/),
along with [lua-language-server](https://github.com/LuaLS/lua-language-server) to
catch potential bugs in your CI before your plugin's users do.

#### :hammer_and_wrench: Tools

- [lua-typecheck-action](https://github.com/marketplace/actions/lua-typecheck-action)
- [luacheck](https://github.com/mpeterv/luacheck) for additional linting.
- [neodev.nvim](https://github.com/folke/neodev.nvim)

For Nix users:

- [nix-gen-luarc-json](https://github.com/mrcjkb/nix-gen-luarc-json),
  which can be used to integrate with [git-hooks.nix](https://github.com/cachix/git-hooks.nix).

#### :books: Further reading

- [Algebraic data types in Lua (Almost)](https://mrcjkb.dev/posts/2023-08-17-lua-adts.html)

## :speaking_head: User Commands

### :x: DON'T

...pollute the command namespace with a command for each action.

**Example:**

- `:RocksInstall {arg}`
- `:RocksPrune {arg}`
- `:RocksUpdate`
- `:RocksSync`

This can quickly become overwhelming when users rely on
command completion.

### :white_check_mark: DO

...gather subcommands under scoped commands
and implement completions for each subcommand.

**Example:**

- `:Rocks install {arg}`
- `:Rocks prune {arg}`
- `:Rocks update`
- `:Rocks sync`

<details>
  <summary>
    <b>Screenshots</b>
  </summary>

  Subcommand completions:

  ![](https://github.com/mrcjkb/nvim-best-practices/assets/12857160/33bf7825-b08b-427b-aa55-c51b8b10a361)

  Argument completions:

  ![](https://github.com/mrcjkb/nvim-best-practices/assets/12857160/e8c8de05-f2aa-477b-82e5-f983775d5fd3)
</details>

Here's an example of how to implement completions.
In this example, we want to 

- provide *subcommand completions* if the user has typed 
  `:Rocks ...`
- *argument completions* if they have typed `:Rocks {subcommand} ...`

First, define a type for each subcommand, which has:

- An implementation, a function which is called when executing
  the subcommand.
- Optional completions for the subcommand's arguments.

```lua
---@class MyCmdSubcommand
---@field impl fun(args:string[], opts: table) The command implementation
---@field complete? fun(subcmd_arg_lead: string): string[] (optional) Command completions callback, taking the lead of the subcommand's arguments
```

Next, we define a table mapping subcommands to their
implementations and completions:

```lua
---@type table<string, MyCmdSubcommand>
local subcommand_tbl = {
    update = {
        impl = function(args, opts)
          -- Implementation (args is a list of strings)
        end,
        -- This subcommand has no completions
    },
    install = {
        impl = function(args, opts)
            -- Implementation
        end,
        complete = function(subcmd_arg_lead)
            -- Simplified example
            local install_args = {
                "neorg",
                "rest.nvim",
                "rustaceanvim",
            }
            return vim.iter(install_args)
                :filter(function(install_arg)
                    -- If the user has typed `:Rocks install ne`,
                    -- this will match 'neorg'
                    return install_arg:find(subcmd_arg_lead) ~= nil
                end)
                :totable()
        end,
        -- ...
    },
}
```

Then, create a lua function to implement the main command:

```lua
---@param opts table :h lua-guide-commands-create
local function my_cmd(opts)
    local fargs = opts.fargs
    local subcommand_key = fargs[1]
    -- Get the subcommand's arguments, if any
    local args = #fargs > 1 and vim.list_slice(fargs, 2, #fargs) or {}
    local subcommand = subcommand_tbl[subcommand_key]
    if not subcommand then
        vim.notify("Rocks: Unknown command: " .. cmd, vim.log.levels.ERROR)
        return
    end
    -- Invoke the subcommand
    subcommand.impl(args, opts)
end
```

Finally, we register our command, along with the completions:

```lua
-- NOTE: the options will vary, based on your use case.
vim.api.nvim_create_user_command("Rocks", my_cmd, {
    nargs = "+",
    desc = "My awesome command with subcommand completions",
    complete = function(arg_lead, cmdline, _)
        -- Get the subcommand.
        local subcmd_key, subcmd_arg_lead = cmdline:match("^Rocks[!]*%s(%S+)%s(.*)$")
        if subcmd_key and subcmd_arg_lead and subcommand_tbl[subcmd_key] and subcommand_tbl[subcmd_key].complete then
            -- The subcommand has completions. Return them.
            return subcommand_tbl[subcmd].complete(subcmd_arg_lead)
        end
        -- Check if cmdline is a subcommand
        if cmdline:match("^Rocks[!]*%s+%w*$") then
            -- Filter subcommands that match
            local subcommand_keys = vim.tbl_keys(subcommand_tbl)
            return vim.iter(subcommand_keys)
                :filter(function(key)
                    return key:find(arg_lead) ~= nil
                end)
                :totable()
        end
    end,
    bang = true, -- If you want to support ! modifiers
})
```

#### :books: Further reading

- `:h lua-guide-commands-create`

## :keyboard: Keymaps

### :x: DON'T

...create any keymaps automatically (unless they are not controversial).
This can easily lead to conflicts.

### :x: DON'T

...define a fancy DSL for enabling keymaps via a `setup` function.

- You will have to implement and document it yourself.
- What if someone else comes up with a slightly different DSL?
  This will lead to inconsistencies and confusion.
- If a user has to call a `setup` function to set keymaps,
  it will cause an error if your plugin is not installed or disabled.
- Neovim already has a built-in API for this.

### :white_check_mark: DO

...provide `:h <Plug>` mappings to allow users to define their own keymaps.

- It requires one line of code in user configs.
- Even if your plugin is not installed or disabled,
  creating the keymap won't lead to an error.

***Example:**

In your plugin:

```lua
vim.keymap.set("n", "<Plug>(MyPluginAction)", function() print("Hello") end, { noremap = true })
```

In the user's config:

```lua
vim.keymap.set("n", "<leader>h", "<Plug>(MyPluginAction)")
```

## :zap: Initialization

<!-- TODO -->

## :sleeping_bed: Lazy loading

### :x: DON'T

...rely on plugin managers to take care of lazy loading for you.

- Making sure a plugin doesn't unnecessarily impact startup time
  should be the responsibility of plugin authors, not users.
- A plugin's functionality may evolve over time, potentially
  leading to breakage if it's the user who has to worry about
  lazy loading.
- A plugin that implements its own lazy initialization properly
  will likely have less overhead than the mechanisms used by a
  plugin manager or user to load that plugin lazily.

### :white_check_mark: DO

...think carefully about when which parts of your plugin need to be loaded.

**Is there any functionality that is specific to a filetype?**

- Put your initialization logic in a `ftplugin/{filetype}.lua` script.
- See `:h filetype`.

**Example:**

```lua
-- ftplugin/rust.lua
if not vim.g.did_my_rust_plugin_init then
    -- Initialise
end
vim.g.did_my_rust_plugin_init = true

local bufnr = vim.api.nvim_get_current_buf()
-- do something specific to this buffer, e.g. add a <Plug> mapping or create a command
vim.keymap.set("n", "<Plug>(MyPluginBufferAction)", function() 
    print("Hello")
end, { noremap = true, buffer = bufnr, })
```

**Is your plugin *not* filetype-specific, but it likely
won't be needed every single time a user opens a Neovim session?**

Don't eagerly `require` your lua modules.

**Example:**

Instead of:

```lua
local foo = require("foo")
vim.api.nvim_create_user_command("MyCommand", function()
    foo.do_something()
end, {
  -- ...
})
```

...which will eagerly load the `foo` module,
and any modules it eagerly imports, you can lazy load it
by moving the `require` into the command's implementation.

```lua
vim.api.nvim_create_user_command("MyCommand", function()
    local foo = require("foo")
    foo.do_something()
end, {
  -- ...
})
```

> [!TIP]
>
> For a Vimscript equivalent to `require`, see `:h autoload`.

> [!NOTE]
>
> - What about eagerly creating user commands at startup?
> - Wouldn't it be better to rely on a plugin manager to lazy load my plugin
>   via a user command and/or autocommand?
>
> No! To be able to lazy load your plugin with a user command,
> a plugin manager [has to itself create a user command](https://github.com/folke/lazy.nvim/blob/e44636a43376e8a1e851958f7e9cbe996751d59f/lua/lazy/core/handler/cmd.lua#L16).
> This works great for plugins that don't implement proper lazy loading,
> but it just adds overhead for those that do.
> The same applies to [autocommands](https://github.com/folke/lazy.nvim/blob/e44636a43376e8a1e851958f7e9cbe996751d59f/lua/lazy/core/handler/event.lua#L68)
> [keymaps](https://github.com/folke/lazy.nvim/blob/e44636a43376e8a1e851958f7e9cbe996751d59f/lua/lazy/core/handler/keys.lua#L112),
> etc.
