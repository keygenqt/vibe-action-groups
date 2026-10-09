# AGENTS.md

Instructions for AI coding agents working in this repository.

## What this repo is

External action groups for [Vibe Action](https://vibe-action.keygenqt.com) — a command router for shell and LLM tasks via YAML pipelines.

Each folder is a **group**; each `.yaml` file is one **action**. The file name (without `.yaml`) is the action name. Actions run as:

```text
vibe-action <group> <action> [query] [args...]
```

## Project structure

```text
.
├── AGENTS.md
├── LICENSE
├── README.md
├── ci
│   ├── spdx.yaml
│   ├── summary.yaml
│   └── validate.yaml
├── code
│   ├── comment.yaml
│   ├── explain.yaml
│   └── review.yaml
├── data
│   ├── extract.yaml
│   ├── fetch.yaml
│   └── find.yaml
├── gen
│   ├── mock.yaml
│   ├── naming.yaml
│   ├── regex.yaml
│   └── synonyms.yaml
├── project
│   ├── commit.yaml
│   └── scan.yaml
├── text
│   ├── spellcheck.yaml
│   ├── tone.yaml
│   └── translate.yaml
└── vision
    ├── describe.yaml
    └── whois.yaml
```

## Groups

- `ci` — spdx, summary, validate — pre-review and CI checks
- `code` — comment, explain, review — work with code
- `data` — extract, fetch, find — fetch and extract external data
- `gen` — mock, naming, regex, synonyms — generate patterns and data
- `project` — commit, scan — whole-project operations
- `text` — spellcheck, tone, translate — work with text
- `vision` — describe, whois — work with images

## How groups are loaded

Groups are declared in `~/.vibe-action/config.yaml`:

```yaml
groups:
  - git: https://gitcode.com/keygenqt_vz/vibe-action-groups.git
    path: /code
    name: code
    about: Work with code
```

- `git` — repository URL, cloned once into the cache
- `path` — local dir, or subfolder inside the clone (`/` = root)
- `ref` — branch, tag, or commit pin (git only)
- `name` — CLI group name (`vibe-action <name> <action>`)
- `about` — short description for help

Either `git` or `path` is required. A git source is cloned once and re-pulled after `vibe-action clean`.

## How to add an action

1. Pick the group folder that matches the action's purpose, or create a new folder.
2. Create `<action>.yaml` in that folder. File name = action name.
3. Follow the schema below and the examples. Keep prompts in English.

## Writing a pipeline

### Minimal action

```yaml
version: '0.0.2'
name: hello
about: Say hello

actions:
  - tag: tag_greeting
    run: value
    val:
      - name: input
        data: query_raw
    action: 'Hello, {input}!'
```

### Top-level fields

- `version` (string, required) — must match `PIPELINE_VERSION` (currently `0.0.2`)
- `name` (string, required) — action name; becomes the CLI subcommand
- `about` (string, required) — short description for `--help`
- `notify` (bool, optional) — desktop notification on completion
- `args` (list, optional) — CLI argument definitions
- `api` (object, optional) — IDE plugin metadata (ignored by CLI runtime)
- `actions` (list, required) — pipeline steps

### Args

- `name` (string, required) — convention: `arg_` prefix; becomes `--<name>` flag
- `short` (char, optional) — short flag alias; must be an ASCII letter
- `input` (string, required) — `string`, `bool`, `number`, `path`
- `default` (string, optional) — default value; makes the arg optional
- `help` (string, optional) — help text

```yaml
args:
  - name: arg_dry_run
    short: 'd'
    input: bool
    default: false
    help: 'Print message without executing'
```

### `api` block

```yaml
api:
  output: replace # replace | clipboard | dialog
  input: query_raw # main input query tag
  args: # extra inputs (optional)
    arg_file: query_file_path
```

- `output` — `replace`, `clipboard`, or `dialog`
- `input` — main input query tag; must be a known `query_*` key
- `args` — extra inputs; values must be known `query_*` keys

### Actions (steps)

- `tag` (string, required) — unique id, referenced by other steps via `data`
- `run` (string, required) — engine (below)
- `val` (list, optional) — val candidates
- `off` (object, optional) — skip condition (`data` + `when`); when it passes, the step is skipped (dead tag, resolves to empty)
- `reg` (string, optional) — regex validating step output; mismatch = hard error
- `ask` (bool, optional) — confirm before execution (default false)
- `action` (string, required) — template with `{name}` placeholders

Run types:

- `value` — literal passthrough, no execution
- `cmd` — shell via `sh -c`; values are shell-quoted
- `tiny` — LLM, tiny model (1-3b)
- `small` — LLM, small model (3-7b)
- `medium` — LLM, medium model (7-14b)
- `large` — LLM, large model (14b+)
- `vision` — LLM with extracted base64 images

Execution order is resolved by data dependencies (topological sort), not list position. A step that references `tag_x` runs after the step producing `tag_x`.

### Val candidates

- `name` (string) — placeholder `{name}` in the `action` template
- `data` (string) — source tag or literal value
- `mods` (string) — operator pipe, applied left to right
- `when` (string) — pre-mods soft guard; skip candidate if false
- `fail` (string) — post-mods hard guard; abort if false
- `each` (bool / object) — fan-out over list items

- Candidates resolve in order; the first whose `when` passes (or has none) wins.
- If any name has no winning candidate, the pipeline aborts with an error —
  add a fallback candidate or declare the skip with `off`.

### Data sources

- `query_*` — user input (reserved prefix)
- `system_*` — runtime environment (reserved prefix)
- CLI argument — the argument's own name
- another step's result — the referenced `tag`
- _(none)_ — mods-only candidate

Only `query_*` and `system_*` are reserved prefixes; `arg_` and `tag_` are conventions, not rules.

### Example 1: file path or inline text → LLM

The most common pattern. First candidate handles a path, second handles raw text.

```yaml
actions:
  - tag: tag_content
    run: value
    reg: '.+'
    val:
      - name: content
        data: query_raw
        when: 'is:path'
        mods: 'fetch|text'
      - name: content
        data: query_raw
        fail: 'is:empty:not'
    action: '{content}'

  - tag: tag_review
    run: large
    reg: '.+'
    val:
      - name: lang
        data: system_language
      - name: content
        data: tag_content
    action: |
      [Task]
      Analyze the provided code for bugs.
      Write strictly in {lang}.

      [Code]
      {content}
```

### Example 2: shell command + fan-out

```yaml
actions:
  - tag: tag_files
    run: cmd
    val:
      - name: path
        data: arg_path
        fail: 'is:empty:not'
    action: find {path} -type f -name '*.rs'

  - tag: tag_summaries
    run: small
    reg: '.+'
    val:
      - name: file
        data: tag_files
        each:
          split: '\n'
          merge: '\x1F'
    action: |
      [Task]
      Summarize this file in one line.

      [File]
      {file}
```

`each` splits the file list, runs the prompt once per file, then merges results back with `\x1F` so list structure survives into the next step.

### Example 3: image source fallback

```yaml
actions:
  - tag: tag_raw
    run: vision
    reg: '.+'
    val:
      - name: image
        data: query_image
        when: 'is:empty:not'
      - name: image
        data: query_clipboard_image
        when: 'is:empty:not'
      - name: image
        mods: 'screenshot|text'
        fail: 'is:empty:not'
    action: |
      {image}

      [Task]
      Describe this screenshot.
```

The third candidate has no `data` — it is a mods-only candidate (`screenshot` is a read operator, no input needed).

### Example 4: skip guard (dry-run)

```yaml
- tag: tag_commit_exec
  run: cmd
  ask: true
  off:
    data: arg_dry_run
    when: 'equals:true'
  val:
    - name: path
      data: tag_project_path
    - name: msg
      data: tag_commit_message
      mods: 'lower'
  action: cd {path} && git add . && git commit -m {msg}
```

When `arg_dry_run` is `true`, the `off` guard passes → dead tag → the step is skipped (no execution, no `ask` confirmation). Downstream `{tag_commit_exec}` placeholders expand to an empty string.

### Example 5: operator chain and writing to a file

```yaml
# Filter, deduplicate, join
mods: 'split|filter:eq:-|uniq|join'

# Write a step's result to a file (pass-through)
- tag: tag_save
  run: value
  val:
    - name: path
      data: tag_path
    - name: json
      data: tag_json
      mods: 'file:{tag_path}'
  action: 'Saved to {path}'
```

### Fan-out (`each`)

```yaml
each: true                            # split/merge by newline
each: { split: '\n', merge: '\x1F' }  # custom separators
```

Use `merge: '\x1F'` when the result must stay a list for downstream list operators.

List values use a hidden `\x1F` separator internally; it is converted to `\n` whenever a value is substituted into an `action` template or stored as a step result.

### Guards

`when` and `fail` accept inspect operators only. Every inspect operator supports `:not`.

- `contains:<X>` — substring test
- `equals:<X>` — exact equality
- `matches:<re>` — regex test
- `compare:<mode>:<N>` — numeric compare; modes `gt` `lt` `gte` `lte`
- `is:<kind>` — `empty` `num` `int` `bool` `url` `path` `json`

### Operators (in `mods`)

Read (world → value):

- `ast[:brief|full|lang]` — parse source to JSON AST (`lang` = file extension: `rs`, `py`, `ts`, …)
- `fetch` — download URL to temp file, or resolve local path
- `resolve[:dir|file]` — resolve path to absolute
- `scan` — scan directory → list of file paths
- `screenshot` — interactive capture → image path
- `text` — extract text from HTML/PDF/image, or base64 passthrough

Transform (value → value):

- `split[:sep]` — split to list (default newline)
- `join[:sep]` — collapse list to string
- `item:<N>` — Nth element; negative counts from end
- `size` — item count / byte length
- `filter:<X>` / `filter:eq:<X>` / `filter:not:<X>` — keep/remove by substring
- `grep:<re>` / `grep:<re>:not` — keep/remove by regex
- `sort[:asc|desc]` — sort
- `uniq` — deduplicate
- `reverse` — reverse
- `take:<N>` / `tail:<N>` — first / last N
- `lower` / `upper` — case conversion
- `replace:<from>:<to>` — substring replace
- `trim[:chars]` — trim ends / drop empty items
- `strip` — remove Markdown fences
- `default:<X>` — fallback for empty
- `base64:encode` / `base64:decode` — base64
- `format:json|json5|yaml|toml` — format conversion

Write (pass-through, side effect):

- `clipboard_text` — copy to clipboard
- `clipboard_image` — copy base64 image to clipboard
- `file:<path>` / `file:<path>:append` — write / append to file

### Query tags

- `query_raw` — raw input, no validation or checks.
- `query_prompt` — interactive input: if used as a data source, the app prompts the user at runtime and the answer becomes the value.
- `query_clipboard` — combined clipboard by priority: text → paths → image (token-limited).
- `query_clipboard_text` — raw text from the clipboard.
- `query_clipboard_path` — copied file paths from the clipboard.
- `query_clipboard_image` — clipboard image as base64 PNG.
- `query_file_path` — explicit input → existing file path, else empty.
- `query_project_path` — explicit input → project root (walks up to a project marker), else empty.
- `query_line` — explicit input → first line, else empty.
- `query_image` — explicit input (URL/file/base64) → validated base64; `Err` if invalid.

### System tags

All return a value or an empty string — never an error.

Identity and runtime:

- `system_os` — operating system name (`macos`, `linux`)
- `system_arch` — CPU architecture (`aarch64`, `x86_64`)
- `system_hostname` — machine hostname
- `system_user` — current user name
- `system_uid` — current user ID
- `system_pid` — current process ID
- `system_shell` — shell name from `$SHELL` (basename)
- `system_language` — language code from `$LANG` (`en_US.UTF-8` → `en`)

Time:

- `system_date` — current date, ISO 8601 (`YYYY-MM-DD`)
- `system_time` — current time (`HH:MM:SS`)
- `system_datetime` — current date and time, ISO 8601
- `system_timestamp` — unix epoch seconds

Hardware and languages:

- `system_cpu_cores` — logical CPU core count
- `system_mem_available` — available memory in bytes
- `system_code_langs` — extensions of supported code languages, newline-separated (`rs`, `py`, `ts`, `js`, `java`, `go`, `cs`, `kt`, `swift`, `dart`, `ets`)
- `system_code_shell` — extensions of supported shell languages, newline-separated (`sh`, `bat`)

Directories (via `dirs`, missing → empty):

- `system_dir_home` — user home
- `system_dir_pwd` — current working directory
- `system_dir_config` — user config directory
- `system_dir_data` — user data directory
- `system_dir_cache` — user cache directory
- `system_dir_download` — user downloads directory
- `system_dir_temp` — temporary directory

### Placeholders & escapes

Templates (`action`):

- `{name}` — substitutes the resolved val candidate value.
- `{{X}}` — literal `{X}` (for JSON templates / LLM output).

Operator args (`mods`):

- `{X}` — escapes separators: braces are stripped, `:` and `|` inside survive as data.
  Example: `join:{|}` joins with a literal `|`.
- `{name}` — tag interpolation: if `name` is a known tag, its value is substituted
  (e.g. `file:{tag_path}`). Referencing an unresolved tag is a runtime error —
  declare it via `data:` in another step first.
- `\n`, `\t`, `\s` — expanded (also in `each` split/merge).

`run: cmd` specifics:

- Values are shell-quoted automatically — do not quote them again.
- A multi-line value arrives as one quoted argument; use `for x in $(echo {tag})`
  to split it back into lines.
- Non-zero exit fails the step; the error message is the command's stdout or stderr.

## Validation rules

A group fails to load if any of these is violated:

- `version` does not match `PIPELINE_VERSION`.
- Empty `name` or `about`.
- Empty `action` template (prompt/command).
- Empty val candidate `name`.
- Duplicate action name within the group (same name in different groups is allowed).
- Action name equals the group name.
- Action name collides with a system command (`clean`, `status`, `bench`, `stop`) — applies to grouped actions too.
- Duplicate `tag` or duplicate `arg` name across the file.
- `data` references its own `tag`, or bare `query` (use `query_raw`).
- `query_*` / `system_*` used as a tag or arg name.
- Arg name: letters, numbers, underscores only; `short` is an ASCII letter.
- `api.input` / `api.args` values must be known `query_*` keys.
- Unknown operator in `mods`; non-inspect operator in `when` / `fail` / `off.when`.
- `off.data` is empty, references its own `tag`, bare `query`, or an unknown tag.
- Empty `split` or `merge` in `each`.
- Undeclared `{name}` in `action`.
- Invalid `reg` regex.

## Conventions

- Folder name = group name; file name = action name.
- Prefix argument names with `arg_`.
- Use `when` guards to avoid unnecessary work; `fail` guards to catch bad input before the LLM.
- Use `off` to declare conditional steps (dry-run, optional stages).
- Use `ask: true` for destructive shell commands.
- `reg: '.+'` catches empty results from a loose LLM.
- Keep prompts in English; write action `name`/`about` in English.
- Tag naming: `tag_<purpose>` (e.g. `tag_content`, `tag_review`).

## See also

Full operator reference and pipeline schema: [Vibe Action docs](https://vibe-action.keygenqt.com).
