# :white_check_mark: :x: :moon: DOs and DON'Ts for modern Neovim Lua plugin development

> [!IMPORTANT]
>
> The code snippets in this document use the Neovim 0.10.0 API.

> [!NOTE]
>
> For a guide to using Lua in Neovim,
> please refer to [`:h lua-intro`](https://neovim.io/doc/user/lua.html).

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

- [LuaCATS documentation](https://luals.github.io/wiki/annotations/)
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

**Example:**

In your plugin:

```lua
vim.keymap.set("n", "<Plug>(MyPluginAction)", function() print("Hello") end, { noremap = true })
```

In the user's config:

```lua
vim.keymap.set("n", "<leader>h", "<Plug>(MyPluginAction)")
```

## :zap: Initialization

### :x: DON'T

...force users to call a `setup` function
in order to be able to use your plugin.

> [!WARNING]
>
> This one often sparks heated debates.
> I have written in detail about the various reasons 
> why this is an anti pattern [here](https://mrcjkb.dev/posts/2023-08-22-setup.html).
>
> - If you still disagree, feel free to open an issue.

These are the rare cases in which a `setup` function 
for initialization could be useful:

- You want your plugin to be compatible with Neovim <= 0.6.
- *And* your plugin is actually multiple plugins in a monorepo.
- *Or* you're integrating with another plugin that *forces* you to do so.

### :white_check_mark: DO

- Cleanly separate configuration and initialization.
- Automatically initialize your plugin *(smartly)*,
  with minimal impact on startup time (see the next section).

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
if not _G.did_my_rust_plugin_init then
    -- Initialise
end
_G.did_my_rust_plugin_init = true

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
> This helps for plugins that don't implement proper lazy loading,
> but it just adds overhead for those that do.
> The same applies to [autocommands](https://github.com/folke/lazy.nvim/blob/e44636a43376e8a1e851958f7e9cbe996751d59f/lua/lazy/core/handler/event.lua#L68),
> [keymaps](https://github.com/folke/lazy.nvim/blob/e44636a43376e8a1e851958f7e9cbe996751d59f/lua/lazy/core/handler/keys.lua#L112),
> etc.

## :wrench: Configuration

### :white_check_mark: DO

...use [LuaCATS annotations](https://luals.github.io/wiki/annotations/)
to make your API play nicely with lua-language-server, while
providing type safety.

One of the largest foot guns in Lua is `nil`.
You should avoid it in your internal configuration.
On the other hand, users don't want to have to set every possible field.
It is convenient for them to provide a default configuration and merge it
with an override table.

This is a common practice:

```lua
---@class myplugin.Config
---@field do_something_cool boolean
---@field strategy "random" | "periodic"

---@type myplugin.Config
local default_config = {
    do_something_cool = true,
    strategy = "random",
}

-- could also be passed in via a function. But there's no real downside to using `vim.g` or `vim.b`.
local user_config = ...
local config = vim.tbl_deep_extend("force", default_config, user_config or {})
return config
```

In this example, a user can override only individual configuration fields:

```lua
{
    strategy = "periodic"
}
```

...leaving the unset fields as their default.
However, if they have lua-language-server configured to pick up your plugin
(for example, using [neodev.nvim](https://github.com/folke/neodev.nvim)),
it will show them a warning like this:

```lua
{ -- ⚠ Missing required fields in type `myplugin.Config`: `do_something_cool`
    strategy = "periodic"
}
```

To mitigate this, you can split configuration *option* declarations
and internal configuration *values*.

This is how I like to do it:

```lua
-- config/meta.lua

---@class myplugin.Config
---@field do_something_cool? boolean (optional) Notice the `?`
---@field strategy? "random" | "periodic" (optional)

-- Side note: I prefer to use `vim.g` or `vim.b` tables (:h lua-vim-variables).
-- You can also use a lua function but there's no real downside to using `vim.g` or `vim.b`
-- and it doesn't throw an error if your plugin is not installed.
-- This annotation says that`vim.g.my_plugin` can either be a `myplugin.Config` table, or
-- a function that returns one, or `nil` (union type).
---@type myplugin.Config | fun():myplugin.Config | nil
vim.g.my_plugin = vim.g.my_plugin

--------------------------------------------------------------
-- config/internal.lua

---@class myplugin.InternalConfig
local default_config = {
    ---@type boolean
    do_something_cool = true,
    ---@type "random" | "periodic"
    strategy = "random",
}

local user_config = type(vim.g.my_plugin) == "function" and vim.g.my_plugin() or vim.g.my_plugin or {}

---@type myplugin.InternalConfig
local config = -- ...merge configs
```

> [!IMPORTANT]
>
> This does have some downsides:
>
> - You have to maintain two configuration types.
> - As this is fairly uncommon, first time contributors will often overlook one
>   of the configuration types.
>
> Since this provides increased type safety for both the plugin
> and the user's config, I believe it is well worth the slight inconvenience.

### :white_check_mark: DO

...validate configs.

Once you have merged the default configuration with the user's config,
you should validate configs.

Validations could include:

- Correct types, see `:h vim.validate`
- Unknown fields in the user config (e.g. due to typos).
  This can be tricky to implement, and may be better suited for
  a health check, to reduce overhead.

> [!WARNING]
>
> `vim.validate` will `error` if it fails a validation.

Because of this, I like to wrap it with `pcall`,
and add the path to the field in the config
table to the error message:

```lua
---@param path string The path to the field being validated
---@param tbl table The table to validate
---@see vim.validate
---@return boolean is_valid
---@return string|nil error_message
local function validate_path(path, tbl)
  local ok, err = pcall(vim.validate, tbl)
  return ok or false, path .. "." .. err
end
```

The function can be called like this:

```lua
---@param cfg myplugin.InternalConfig
---@return boolean is_valid
---@return string|nil error_message
function validate(cfg)
    return validate_path("vim.g.my_plugin", {
        do_something_cool = { cfg.do_something_cool, "boolean" },
        strategy = { cfg.strategy, "string" },
    })
end
```

And invalid config will result in an error message like
`"vim.g.my_plugin.strategy: expected string, got number"`.

By doing this, you can use the validation with both 
`:h vim.notify` and `:h vim.health`.

## :stethoscope: Troubleshooting

### :white_check_mark: DO

...provide health checks in `lua/{plugin}/health.lua`.

Some things to validate:

- User configuration
- Proper initialization
- Presence of lua dependencies
- Presence of external dependencies

#### :books: Further reading

- `:h vim.health`

## :hash: Versioning and releases

### :x: DON'T

...use [0ver](https://0ver.org/) or omit versioning completely,
e.g. because you believe doing so is a commitment to stability.

> [!TIP]
>
> Doing this won't make people any happier about breaking changes.

### :white_check_mark: DO

...use [SemVer](https://semver.org/) to properly communicate
bug fixes, new features, and breaking changes.

#### :books: Further reading

- [Just use Semantic Versioning](https://vhyrro.github.io/posts/versioning/).

### :white_check_mark: DO

...automate versioning and releases, and publish to luarocks.org.

#### :books: Further reading

- [rocks.nvim/Introduction](https://github.com/nvim-neorocks/rocks.nvim?tab=readme-ov-file#moon-introduction)
- [Luarocks :purple_heart: Neovim](https://github.com/nvim-neorocks/sample-luarocks-plugin)

#### :hammer_and_wrench: Tools

- [luarocks-tag-release](https://github.com/marketplace/actions/luarocks-tag-release)
- [release-please-action](https://github.com/googleapis/release-please-action)
- [semantic-release](https://github.com/semantic-release/semantic-release)

## :notebook: Documentation

### :white_check_mark: DO

...provide vimdoc, so that users can read your plugin's documentation in Neovim,
by entering `:h {plugin}`.

### :x: DON'T

...simply dump generated references in your `doc` directory.

#### :hammer_and_wrench: Tools

- [panvimdoc](https://github.com/kdheepak/panvimdoc)
- [lemmy-help](https://github.com/numToStr/lemmy-help)

> [!IMPORTANT]
>
> `lemmy-help` is a nice tool, but it hasn't been maintained in years.
> It is no longer fully compatible with LuaCATS, which has diverged
> from EmmyLua.

#### :books: Further reading

- [Diátaxis - A systematic approach to technical documentation authoring](https://diataxis.fr/)
