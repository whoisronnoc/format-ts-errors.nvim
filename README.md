# format-ts-errors.nvim

Make `ts_ls` (the TypeScript LSP) errors a little nicer looking by formatting
objects.

This plugin favors composability and direct API access over doing it for you.

## screenshots

![screenshot 1][screenshot]
![screenshot 2][screenshot2]

## Installation

**Lazy.nvim** add:

```lua
{
    "davidosomething/format-ts-errors.nvim"
}
```

You can configure the output with a setup function, e.g.:

````lua
{
    "davidosomething/format-ts-errors.nvim",
    config = function()
      require("format-ts-errors").setup({
        add_markdown = true, -- wrap types with markdown ```typescript ``` markers for syntax highlighting (more configuration required, see below)
        start_indent_level = 0, -- initial indent
      })
    end,
}
````

For markdown syntax highlighting to work correctly you must,

- 1. set the diagnostic float buffer filetype to `markdown`
- 2. disable the extmarks for `DiagnosticFloatError`

e.g.

```lua
vim.api.nvim_create_autocmd("CursorHold", {
  buffer = event.buf,
  callback = function()
  local b = vim.diagnostic.open_float(nil, { focus = false, scope = "cursor" })
  if b then
    vim.api.nvim_buf_set_option(b, "filetype", "markdown") -- set the filetype of the diagnostic float to "markdown"
    vim.cmd("hi! clear DiagnosticError") --- clear the DiagnosticError highlight group to disable extmark overrides
    vim.cmd("hi! link @markup.link.label.markdown_inline Normal")
    end
  end,
})
```

Then in the lsp setup:

```lua
local lspconfig = require("lspconfig")
lspconfig.tsserver.setup({
  handlers = {
    ["textDocument/publishDiagnostics"] = function(
      _,
      result,
      ctx,
      config
    )
      if result.diagnostics == nil then
        return
      end

      local idx = 1
      while idx <= #result.diagnostics do
        local entry = result.diagnostics[idx]

        local formatter = require('format-ts-errors')[entry.code]
        entry.message = formatter and formatter(entry.message) or entry.message


        -- ignore specific diagnostic messages (optional)
        -- codes: https://github.com/microsoft/TypeScript/blob/main/src/compiler/diagnosticMessages.json
        if entry.code == 80001 then
          -- { message = "File is a CommonJS module; it may be converted to an ES module.", }
          table.remove(result.diagnostics, idx)
        else
          idx = idx + 1
        end
      end

      vim.lsp.diagnostic.on_publish_diagnostics(
        _,
        result,
        ctx,
        config
      )
    end,
  },
})
```

Or use it in vim.diagnostic.config({ float = { format = function... } })
An example can be found in my own dotfiles:
[https://github.com/davidosomething/dotfiles/commit/ea55d6eb3ba90784f09f9f8652ae3e20a9bdefd7#diff-62fa333ae509823f7ed9ffb0e95acfba597ad6e5644c2a3b72551a6ccc05667dR162-R181]

### Config options

#### start_indent_level

```lua
start_indent_level = 0
```

Will yield:

````markdown
```typescript
some code (not indented)
```
````

Whereas 1:

```lua
start_indent_level = 1
```

Will yield:

````markdown
```typescript
  some code (indented)
```
````

[screenshot]: https://raw.githubusercontent.com/davidosomething/format-ts-errors.nvim/meta/screenshot.png
[screenshot2]: https://raw.githubusercontent.com/davidosomething/format-ts-errors.nvim/meta/screenshot-2.png
