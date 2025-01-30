# :white_check_mark: :x: :moon: DOs and DON'Ts for modern Neovim Lua plugin development

> [!IMPORTANT]
>
> The code snippets in this document use the Neovim 0.10.0 API.

> [!NOTE]
>
> - For a guide to using Lua in Neovim,
>   please refer to [`:h lua-intro`](https://neovim.io/doc/user/lua.html).
>
> - For an example based on this guide, check out
    [@ColinKennedy's plugin template](https://github.com/ColinKennedy/nvim-best-practices-plugin-template).

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
- [llscheck](https://github.com/jeffzi/llscheck)
- [luacheck](https://github.com/mpeterv/luacheck) for additional linting.
- [lazydev.nvim](https://github.com/folke/lazydev.nvim)

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

> [!TIP]
>
> There exists a Lua library, [`mega.cmdparse`](https://github.com/ColinKennedy/mega.cmdparse),
> which can take care of much of the boilerplate for you
> and comes with many features.

Here's a basic example of how to implement completions manually.
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
        vim.notify("Rocks: Unknown command: " .. subcommand_key, vim.log.levels.ERROR)
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
        local subcmd_key, subcmd_arg_lead = cmdline:match("^['<,'>]*Rocks[!]*%s(%S+)%s(.*)$")
        if subcmd_key 
            and subcmd_arg_lead 
            and subcommand_tbl[subcmd_key] 
            and subcommand_tbl[subcmd_key].complete
        then
            -- The subcommand has completions. Return them.
            return subcommand_tbl[subcmd_key].complete(subcmd_arg_lead)
        end
        -- Check if cmdline is a subcommand
        if cmdline:match("^['<,'>]*Rocks[!]*%s+%w*$") then
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
  Here are 3 differing implementations:
    - [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim/tree/96610122a40f0bb4ce2e452c6d2429bf093d6700?tab=readme-ov-file#telescope-setup-structure)
    - [nvim-treesitter-textobjects (legacy)](https://github.com/nvim-treesitter/nvim-treesitter-textobjects/tree/84cc9ed772f1fee2f47c1e076f518829583d8347?tab=readme-ov-file#text-objects-select)
    - [nvim-treesitter-refactor (legacy)](https://github.com/nvim-treesitter/nvim-treesitter-refactor/tree/65ad2eca822dfaec2a3603119ec3cc8826a7859e?tab=readme-ov-file#smart-rename)
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

> [!TIP]
>
> Some benefits of `<Plug>` mappings over exposing a lua function:
> 
> - You can enforce options like `expr = true`.
> - Expose functionality only or specific modes (`:h map-modes`).
> - Expose different behaviour for different modes with
>   a single `<Plug>` mapping.

For example, in your plugin:

```lua
vim.keymap.set("n", "<Plug>(SayHello)", function() 
    print("Hello from normal mode") 
end, { noremap = true })

vim.keymap.set("v", "<Plug>(SayHello)", function() 
    print("Hello from visual mode") 
end, { noremap = true })
```

In the user's config:

```lua
vim.keymap.set({"n", "v"}, "<leader>h", "<Plug>(SayHello)")
```

### :white_check_mark: DO

...just expose a Lua API that people can use to define keymaps, if

- You have a function that takes a large options table,
  and it would require lots of `<Plug>` mappings to expose all of its uses
  (You could still create some for the most common ones).
- You don't care about people who prefer Vimscript for configuration.

Another alternative is just to expose user commands.

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

#### Is there any functionality that is specific to a filetype?

- Put your initialization logic in a `ftplugin/{filetype}.lua` script.
- See [`:h filetype`](https://neovim.io/doc/user/filetype.html#%3Afiletype).

**Example:**

```lua
-- ftplugin/rust.lua
if not vim.g.loaded_my_rust_plugin then
    -- Initialise
end
-- NOTE: Using vim.g.loaded_ prevents the plugin from initializing twice
-- and allows users to prevent plugins from loading (in both Lua and Vimscript).
vim.g.loaded_my_rust_plugin = true

local bufnr = vim.api.nvim_get_current_buf()
-- do something specific to this buffer, e.g. add a <Plug> mapping or create a command
vim.keymap.set("n", "<Plug>(MyPluginBufferAction)", function() 
    print("Hello")
end, { noremap = true, buffer = bufnr, })
```

#### Is your plugin *not* filetype-specific, but it likely won't be needed every single time a user opens a Neovim session?

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
  return ok, err and path .. "." .. err
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
[`:h vim.notify`](https://neovim.io/doc/user/lua.html#vim.notify()) and [`:h vim.health`](https://neovim.io/doc/user/pi_health.html#health-functions).

## :stethoscope: Troubleshooting

### :white_check_mark: DO

...provide health checks in `lua/{plugin}/health.lua`.

Some things to validate:

- User configuration
- Proper initialization
- Presence of lua dependencies
- Presence of external dependencies

#### :books: Further reading

- [`:h vim.health`](https://neovim.io/doc/user/pi_health.html#health-functions)

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

- [vimCATS](https://github.com/mrcjkb/vimcats)
- [panvimdoc](https://github.com/kdheepak/panvimdoc)

#### :books: Further reading

- [Diátaxis - A systematic approach to technical documentation authoring](https://diataxis.fr/)

## :test_tube: Testing

### :white_check_mark: DO

...automate testing as much as you can.

### :x: DON'T

...use plenary.nvim for testing.

Historically, `plenary.test` has been very popular for testing,
because there was no convenient way for using Neovim as a lua interpreter.
That has changed with the introduction of `nvim -l` in Neovim 0.9.

While plenary.nvim is still being maintained, much of its functionality
is [gradually being upstreamed into Neovim or moved into other libraries](https://github.com/nvim-telescope/telescope.nvim/issues/2552).

### :white_check_mark: DO

...use [busted](https://github.com/lunarmodules/busted) for testing,
which is a lot more powerful. 

> [!NOTE]
> 
> plenary.nvim bundles a *limited subset* of luassert.

We advocate for using luarocks + busted
for testing, primarily for the following reasons:

- **Familiarity:**
  It is the de facto tried and tested standard in the Lua community,
  with a familiar API modeled after [rspec](https://rspec.info/).
- **Consistency:**
  Having a consistent API makes life easier for
    - Contributors to your project.
    - Package managers, who [run test suites](https://github.com/NixOS/nixpkgs/blob/7feaafb5f1dea9eb74186d8c40c2f6c095904429/pkgs/development/lua-modules/overrides.nix#L581)
    to ensure they don't deliver a broken version of your plugin.
- **Better reproducibility:**
  By using luarocks to manage your test dependencies, you can easily
  pin them. Checking out git repositories is prone to flakes in CI
  and "it works on my machine" issues.
- **Dependency management**:
  With luarocks, you have the whole
  ecosystem of Lua modules at your
  fingertips, making it easy to manage and
  integrate dependencies for your project.

> [!TIP]
>
> For combining busted with other test frameworks,
> check out our [busted interop examples](https://github.com/nvim-neorocks/busted-interop-examples).

#### :page_facing_up: Template

- [`nvim-lua/nvim-lua-plugin-template`](https://github.com/nvim-lua/nvim-lua-plugin-template/)

#### :books: Further reading

- [Using Neovim as Lua interpreter with Luarocks](https://zignar.net/2023/01/21/using-luarocks-as-lua-interpreter-with-luarocks/)
- [Testing Neovim plugins with Busted](https://hiphish.github.io/blog/2024/01/29/testing-neovim-plugins-with-busted/)
- [Test your Neovim plugins with luarocks & busted](https://mrcjkb.dev/posts/2023-06-06-luarocks-test.html)
- [Debugging Lua in Neovim](https://zignar.net/2023/06/10/debugging-lua-in-neovim/)

#### :hammer_and_wrench: Tools

- [`nvim-busted-action`](https://github.com/marketplace/actions/nvim-busted-action)
- [`nlua`](https://github.com/mfussenegger/nlua)
- [`neorocksTest`](https://github.com/nvim-neorocks/neorocks) (for Nix users)
- [Neotest](https://github.com/nvim-neotest/neotest) adapters:
  - [`HiPhish/neotest-busted`](https://gitlab.com/HiPhish/neotest-busted)
  - [`MisanthropicBit/neotest-busted`](https://github.com/MisanthropicBit/neotest-busted)

## :electric_plug: Integrating with other plugins

### :white_check_mark: DO

...consider integrating with other plugins.

For example, it might be useful to add
a [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim) extension
or a [lualine](https://github.com/nvim-lualine/lualine.nvim) component.

> [!TIP]
>
> If you don't want to commit to maintaining compatibility with another plugin's API,
> you can expose your own API for others to hook into.
