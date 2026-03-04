# IDE Support: What Actually Works in 2026

For most of Q's history, the IDE situation was: use a text editor, run your code in the q REPL, repeat. This was not considered a problem because the alternative — the q IDE that shipped with older KX products — was not obviously better.

The landscape has genuinely improved. There are now real options with real features: syntax highlighting, autocomplete, code navigation, debugging integration, and linters that tell you things you didn't know you were doing wrong. This chapter covers what actually works, what sounds good in a README but doesn't in practice, and the one setup that's worth the configuration time.

## VS Code with the q Extension

The `kdb` extension from KX (marketplace ID: `KxSystems.kdb`) is the current best-in-class option. It's actively maintained, it's free, and it works.

### Installation

Install from the VS Code marketplace:

```
ext install KxSystems.kdb
```

Or from the command palette: `Extensions: Install Extensions` → search "kdb".

### What You Get

**Syntax highlighting**: Accurate Q syntax highlighting including functional forms, system commands, iterators. Not the "we colored the keywords" approach — it correctly handles Q's context-dependent syntax.

**Code completion**: Autocomplete for built-in Q functions and operators. Less impressive for user-defined namespaces unless you're connected to a running process (see below).

**Live connection**: The extension can connect to a running kdb+ process. With a live connection, you get:
- Execute selected code against the connected process (Cmd/Ctrl+Return)
- Q console output in VS Code's output panel
- Variable completion from the connected process's namespace

**q scratch pad**: Open a `.q` file, connect to a process, and evaluate expressions interactively. This is the workflow that replaces the REPL loop.

### Configuration

In your VS Code `settings.json`:

```json
{
  "kdb.servers": [
    {
      "serverName": "dev",
      "host": "localhost",
      "port": 5001,
      "auth": false,
      "tls": false
    },
    {
      "serverName": "prod",
      "host": "prod-kdb.internal",
      "port": 5001,
      "auth": true,
      "tls": true
    }
  ],
  "kdb.defaultServer": "dev",
  "kdb.hideSubscriptionWarning": false
}
```

The server list appears in the KDB panel in the activity bar. Click to connect, right-click to set as default.

### The Workflow That Actually Works

The productive VS Code + Q workflow:

1. Keep your `.q` source files open in the editor
2. Connect to a local development q process
3. Highlight a block and hit Cmd+Return to execute it
4. Check results in the output panel or the Q console
5. Iterate

For namespace navigation: `Go to Definition` works for functions defined in the connected process. For functions defined in files you haven't loaded yet, it falls back to text search.

### Debugging

The extension doesn't yet support the Q debugger interactively (breakpoints, step-through). For debugging, you're still using:

```q
/ Q's built-in debug mode when a function errors
\e 1   / enable debug mode
/ Then when a function errors, you're dropped into its context
/ .z.ex contains the failing expression
/ .z.ey contains the argument
```

Or the increasingly popular approach of extracting functions to test them in isolation rather than debugging in-place.

## IntelliJ / JetBrains with the Q Plugin

The `q4intellij` plugin provides Q support for IntelliJ IDEA and the JetBrains family. If your team is already JetBrains-based, this is worth considering.

Install via JetBrains Marketplace: search "q language support" or "kdb+".

What you get:
- Syntax highlighting and formatting
- Structure view (namespace/function tree)
- Go to Definition for local code
- Remote REPL integration

What you don't get: the same depth of kdb+ integration as the KX VS Code extension. The KX extension benefits from being written by the language vendor.

## Emacs and Vim: The Long Game

Emacs has `q-mode`, maintained with the same level of enthusiasm you'd expect from Emacs maintainers (which is to say: it works, has been working for years, and the documentation is a README in a GitHub repo from 2014 that still applies).

```elisp
;; q-mode for Emacs
;; Install via MELPA:
(use-package q-mode
  :ensure t
  :mode "\\.q\\'"
  :config
  (setq q-host "localhost")
  (setq q-port 5001))
```

With `q-mode`:
- `C-c C-l`: Load current buffer into Q
- `C-c C-r`: Send region to Q
- `C-c C-z`: Open Q REPL

Vim users have `vim-q`, which provides syntax highlighting and some REPL integration. If you're already fluent in Vim, this keeps you in Vim. If you're not, the VS Code extension is less friction.

## Notepad++ and Sublime Text

Both have syntax highlighting plugins that make Q code readable. Neither has REPL integration or anything beyond colorization. Useful for reading Q code on a machine where you haven't installed a proper IDE; not a development workflow.

## qStudio: The Dedicated Option

qStudio (from timestored.com) is a dedicated kdb+/q IDE. It's been around for a long time and has features the VS Code extension doesn't:

- **Table viewer**: Paginated, sortable display of Q tables — much better than reading raw console output
- **SQL-style query editor**: Write Q queries with a GUI for table results
- **Charts**: Built-in charting of Q result sets
- **Schema browser**: Tree view of tables, functions, variables across connected processes

Free version available with most features. Pro version adds more chart types and advanced features.

The honest comparison: qStudio is better for *querying and exploring* kdb+ data interactively. VS Code with the KX extension is better for *developing and maintaining* Q code. Many teams use both — qStudio for ad hoc analysis, VS Code for the actual codebase.

## Jupyter with Q

If your team uses Jupyter heavily, there's a kernel for Q:

```bash
pip install jupyterq
# or via conda
conda install -c kx jupyterq
```

This gives you Q code cells in Jupyter notebooks. Useful for:
- Exploratory analysis with Q + Python mixed in the same notebook
- Sharing Q analyses with colleagues who want a notebook interface
- Teaching Q in an interactive environment

```python
# In a Jupyter notebook cell — magic syntax after installing jupyterq
%%q
/ This is a Q cell
select vwap: size wavg price by sym from trade where date=.z.d
```

The limitation: Jupyter's model of discrete cell execution doesn't map perfectly onto Q's stateful, incremental development style. It works, but Q developers tend to find the Q REPL more natural for iteration.

## The Recommended Setup

For a developer who writes Q professionally:

1. **VS Code + KX extension**: Primary development environment. Connected to a local q process. Use it for all code editing, namespace navigation, and interactive execution.

2. **qStudio**: For data exploration and ad hoc queries, especially when looking at large table results you want to sort/filter interactively.

3. **Terminal Q REPL**: Still useful for quick experiments and for running long-lived processes. The VS Code extension doesn't replace the terminal q session; it complements it.

4. **Git**: Not an IDE feature, but: version control your Q code. `.q` and `.k` files belong in version control. The Q shop that doesn't do this is the Q shop that loses code.

## Linting and Formatting

The VS Code KX extension includes a linter (via `kdb-lsp`). It catches:

- Type errors it can detect statically
- Unused variables
- Shadowed names
- Style issues (inconsistent spacing, missing semicolons)

For CI integration, `qls` (Q Language Server) can be run headlessly:

```bash
# Run linter in CI
qls --check src/*.q
```

We'll cover linting and tooling in depth in the [Developer Tooling](./devtools.md) chapter.

## A Note on Color Themes

Q's syntax is unusual enough that generic dark themes don't highlight it well. The KX extension comes with Q-specific token colors; use them, or use a theme that supports custom token colors and add:

```json
// VS Code settings.json
{
  "editor.tokenColorCustomizations": {
    "[One Dark Pro]": {
      "textMateRules": [
        {
          "scope": "keyword.operator.q",
          "settings": { "foreground": "#56B6C2" }
        },
        {
          "scope": "entity.name.function.q",
          "settings": { "foreground": "#61AFEF" }
        }
      ]
    }
  }
}
```

This is more optional than the preceding advice, but staring at poorly highlighted Q code for eight hours is an unforced error.

## Keybindings for Q Development

The VS Code KX extension's most useful keybindings:

| Action | macOS | Linux/Windows |
|--------|-------|---------------|
| Execute selected code | `Cmd+Return` | `Ctrl+Return` |
| Execute current line | `Cmd+Return` (no selection) | `Ctrl+Return` |
| Execute entire file | `Cmd+Shift+Return` | `Ctrl+Shift+Return` |
| Connect to server | via KDB panel | via KDB panel |
| Toggle Q terminal | `Cmd+J` | `Ctrl+J` |

The execute-selection shortcut is the one you'll use constantly. The workflow is: write a function definition, select it, hit `Cmd+Return`, see it load into the connected q process. Adjust, re-select, repeat. Much faster than copy-pasting into a terminal REPL.

For loading whole files, a useful VS Code task:

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Load Q file",
      "type": "shell",
      "command": "echo '\\\\l ${file}' | nc localhost 5001",
      "group": "build",
      "presentation": { "reveal": "silent" },
      "keybinding": "ctrl+shift+l"
    }
  ]
}
```

## Managing Multiple kdb+ Connections

Real Q development involves multiple environments — at minimum a local dev process and a shared test environment. The KX extension handles this cleanly.

In `settings.json`, define all your environments:

```json
{
  "kdb.servers": [
    {
      "serverName": "local-dev",
      "serverAlias": "dev",
      "host": "localhost",
      "port": 5001,
      "auth": false
    },
    {
      "serverName": "shared-test",
      "serverAlias": "test",
      "host": "test-kdb.internal",
      "port": 5001,
      "auth": true,
      "username": "dev"
    },
    {
      "serverName": "hdb-prod-readonly",
      "serverAlias": "hdb",
      "host": "hdb.internal",
      "port": 5010,
      "auth": true,
      "username": "readonly"
    }
  ]
}
```

The extension shows these in the KDB panel. Active connection is shown in the status bar. Switching connections is one click. **Never accidentally execute against prod** by always checking the status bar before running anything that writes.

For the test and prod connections that require passwords, the extension prompts on connect rather than storing passwords in settings files, which means your `settings.json` can safely be in version control.

## Code Snippets for Common Q Patterns

VS Code snippets save significant typing for boilerplate-heavy Q patterns. Add these to your Q snippets file (`Cmd+Shift+P` → "Configure User Snippets" → "q"):

```json
{
  "Q function with named args": {
    "prefix": "fn",
    "body": [
      "${1:funcName}: {[${2:args}]",
      "    ${0}",
      "    }"
    ],
    "description": "Q function with named parameters"
  },
  "Q namespace block": {
    "prefix": "ns",
    "body": [
      "\\\\d .${1:namespace}",
      "",
      "${0}",
      "",
      "\\\\d ."
    ],
    "description": "Q namespace block"
  },
  "Q select statement": {
    "prefix": "sel",
    "body": [
      "select ${2:cols}",
      "from ${1:table}",
      "where ${0}"
    ],
    "description": "Q select"
  },
  "Q select by": {
    "prefix": "selby",
    "body": [
      "select ${3:aggregates}",
      "from ${1:table}",
      "where ${2:conditions}",
      "by ${0:groups}"
    ],
    "description": "Q select with by"
  },
  "Q protected eval": {
    "prefix": "trp",
    "body": [
      "@[${1:func}; ${2:args}; {[e] ${0}}]"
    ],
    "description": "Protected evaluation"
  },
  "Q table definition": {
    "prefix": "tbl",
    "body": [
      "${1:name}: ([]",
      "    ${2:col1}: \\`${3:type}\\$();",
      "    ${0}",
      "    )"
    ],
    "description": "Q table schema"
  },
  "Q timer": {
    "prefix": "timer",
    "body": [
      ".z.ts: {[]",
      "    ${0}",
      "    }",
      "\\\\t ${1:1000}"
    ],
    "description": "Q timer setup"
  }
}
```

## The kdb+ REPL in VS Code Terminal

The terminal REPL and the VS Code extension aren't mutually exclusive. A useful setup: split pane with the source file on the left and a terminal running q on the right. The extension sends code to the connected process; the terminal shows output and lets you run ad hoc queries.

```bash
# Start a q session in VS Code's integrated terminal
# Terminal → New Terminal, then:
q -p 5001

# Now the extension can connect to this same process
# Code you execute via Cmd+Return appears in the terminal output
```

The advantage: you can see exactly what's happening in the q process — error messages, timer output, debug prints — while editing in the left pane.

## IDE Comparison Summary

| Feature | VS Code + KX | qStudio | IntelliJ + plugin | Emacs/Vim |
|---------|-------------|---------|-------------------|-----------|
| Syntax highlighting | ✓ | ✓ | ✓ | ✓ |
| Live connection | ✓ | ✓ | ✓ | ✓ |
| Autocomplete (builtins) | ✓ | limited | ✓ | limited |
| Autocomplete (connected) | ✓ | ✓ | partial | — |
| Table result viewer | basic | ✓✓ | basic | — |
| Linting (qls) | ✓ | — | — | — |
| Charts | — | ✓ | — | — |
| Schema browser | ✓ | ✓ | — | — |
| Git integration | ✓✓ | — | ✓✓ | ✓ |
| Actively maintained | ✓ | ✓ | partial | community |
| Cost | free | free/paid | free/paid | free |

The recommendation hasn't changed: **VS Code + KX extension** for development, **qStudio** for data exploration. But the right answer is whatever you'll actually use — a well-configured Emacs setup beats a poorly-configured VS Code setup every time.
