# Java

## Install Syntax Highlighting

```vim
:TSInstall java
```

## Supported language servers

- jdtls
- nvim-jdtls

## Supported formatters

- astyle
- clang_format
- google_java_format
- npm_groovy_lint
- uncrustify

The Java language server (jdtls) also supports formatting, and it is enabled by default. It is possible to fine-tune its formatting rules, but it is also possible to use a different formatter from the above list. When such a formatter is used, jdtls formatting will be disabled to avoid conflict.

### jdtls

jdtls is installed automatically once you open a `.java` file.

:::note

jdtls requires **jdk-17 or newer** to run.

:::

Neovim (by default) passes basic options (such as `vim.opt.shiftwidth` and `vim.opt.tabstop`) to the language server when formatting.

It is possible to further customize jdtls formatting by supplying an Eclipse formatter file.

To do so, type `:LspSettings jdtls`. It will create a JSON file at `.config/lvim/lsp-settings/jdtls.json` and can be treated as global settings.

Add the following content:

```json
{
  "java.format.settings.profile": "GoogleStyle",
  "java.format.settings.url": "https://raw.githubusercontent.com/google/styleguide/gh-pages/eclipse-java-google-style.xml"
}
```

To reference a local file in the url attribute, simply set its path: `"java.format.settings.url": ".config/lvim/custom-formatter.xml"`

It is also possible to specify project-specific configs. To do so, type `:LspSettings local jdtls` which will create `.nlsp-settings/jdtls.json` in the current working directory, and paste the config that we used for the global settings.

More information about Lsp commands can be found at https://github.com/tamago324/nlsp-settings.nvim

### Custom formatters

Custom formatters are CLI tools that are wrapped with null-ls plugin, which is available by default in LunarVim. They should be installed separately from LunarVim and be available on $PATH.

### nvim-jdtls

`nvim-jdtls` is an alternative setup of the jdtls lsp, which is even promotet by [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig/blob/0011c435282f043a018e23393cae06ed926c3f4a/lua/lspconfig/server_configurations/jdtls.lua#L132) itself.

Here a sample nvim-jdtls/mason configuration:

- Export a $MASON env-var to your .bashrc like `export MASON=$HOME/.local/share/nvim/mason/`.

```lua title="config.lua"
-- disable autoconfiguration
vim.list_extend(lvim.lsp.automatic_configuration.skipped_servers, { "jdtls" })

--- install the plugin
lvim.plugins = {
  { 'mfussenegger/nvim-jdtls' },
}
```

```lua title="ftplugin/java.lua"
local project_name = vim.fn.fnamemodify(vim.fn.getcwd(), ':p:h:t')
-- ❗set your global workspace folder
local workspace_dir = vim.fn.expand("$HOME/change/me/to/wherever") .. project_name

-- See `:help vim.lsp.start_client` for an overview of the supported `config` options.
local config = {
  -- The command that starts the language server
  -- See: https://github.com/eclipse/eclipse.jdt.ls#running-from-the-command-line
  cmd = {
    'java',
    '-Declipse.application=org.eclipse.jdt.ls.core.id1',
    '-Dosgi.bundles.defaultStartLevel=4',
    '-Declipse.product=org.eclipse.jdt.ls.core.product',
    '-Dlog.protocol=true',
    '-Dlog.level=ALL',
    '-Xmx1g',
    '--add-modules=ALL-SYSTEM',
    '--add-opens', 'java.base/java.util=ALL-UNNAMED',
    '--add-opens', 'java.base/java.lang=ALL-UNNAMED',
    '-javaagent:' .. vim.fn.expand("$MASON/share/jdtls/lombok.jar"), -- LOMBOK ❤️
    '-jar', vim.fn.expand("$MASON/share/jdtls/plugins/org.eclipse.equinox.launcher.jar"),
    '-configuration', vim.fn.expand("$MASON/share/jdtls/config"),
    '-data', workspace_dir
  },

  -- ❗add all the lunarvim mappings! very important
  on_attach = require("lvim.lsp").common_on_attach,
  on_init = require("lvim.lsp").common_on_init,
  on_exit = require("lvim.lsp").common_on_exit,
  capabilities = require("lvim.lsp").common_capabilities(),
  root_dir = require('jdtls.setup').find_root({ '.git', 'mvnw', 'gradlew' }),

  -- Here you can configure eclipse.jdt.ls specific settings
  -- See https://github.com/eclipse/eclipse.jdt.ls/wiki/Running-the-JAVA-LS-server-from-the-command-line#initialize-request
  -- for a list of options
  settings = {
    java = {
    }
  },

  -- Language server `initializationOptions`
  -- You need to extend the `bundles` with paths to jar files
  -- if you want to use additional eclipse.jdt.ls plugins.
  --
  -- See https://github.com/mfussenegger/nvim-jdtls#java-debug-installation
  --
  -- If you don't plan on using the debugger or other eclipse.jdt.ls plugins you can remove this
  init_options = {
    bundles = {
    }
  },
}
-- This starts a new client & server,
-- or attaches to an existing client & server depending on the `root_dir`.
require('jdtls').start_or_attach(config)
```

#### clang-format

clang-format is traditionally used for formatting C/C++ code but can also be used for Java code formatting.

Prerequisites:
clang-format should be on the $PATH

Enable formatter in `~/.config/lvim/config.lua`:

```lua
local formatters = require "lvim.lsp.null-ls.formatters"
formatters.setup {
  {
    command = "clang-format",
    filetypes = { "java" },
  }
}
```

With the above configuration, the default settings will be used. To see the defaults, type `clang-format --dump-config` in the terminal.

clang-format supports multiple predefined styles. For the list of values see: https://clang.llvm.org/docs/ClangFormatStyleOptions.html#configurable-format-style-options

To specify such style you need to set extra args in `config.lua`:

```lua
  {
    command = "clang-format",
    filetypes = { "java" },
    extra_args = { "--style", "Google" },
  }
```

It is also possible to use a format file. For that, you will need a valid clang-format file. You can create one from an existing style that can be used as a base: `clang-format --style=Google --dump-config > .clang-format`

`config.lua`:

```lua
  {
    command = "clang-format",
    filetypes = { "java" },
    extra_args = { "--style", "file:<format_file_path>" },
  }
```

#### google-java-format

google-java-format is a program that reformats Java source code to comply with Google Java Style.

Prerequisites:
google-java-format should be on the $PATH

Enable formatter in `~/.config/lvim/config.lua`:

```lua
formatters = require "lvim.lsp.null-ls.formatters"
formatters.setup {
  {
    command = "google-java-format",
    filetypes = { "java" },
  }
}
```

#### uncrustify

uncrustify works similarly to clang-format.

Enable formatter in `~/.config/lvim/config.lua`:

```lua
formatters = require "lvim.lsp.null-ls.formatters"
formatters.setup {
  {
    command = "uncrustify",
    filetypes = { "java" },
    extra_args = { "-c", "path/to/your.cfg" },
  }
}
```

## Supported linters

```lua
{ "checkstyle", "pmd", "semgrep" }
```

## Advanced configuration

It is also possible to fully customize the language server. See https://github.com/LunarVim/starter.lvim to get ideas on how to proceed with that.
