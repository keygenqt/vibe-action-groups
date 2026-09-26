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

| Group     | Actions                       | About                           |
| --------- | ----------------------------- | ------------------------------- |
| `code`    | comment, explain, review      | Work with code                  |
| `data`    | extract, fetch, find          | Fetch and extract external data |
| `gen`     | mock, naming, regex, synonyms | Generate patterns and data      |
| `project` | commit, scan                  | Whole-project operations        |
| `text`    | spellcheck, tone, translate   | Work with text                  |
| `vision`  | describe, whois               | Work with images                |

## How groups are loaded

Groups are declared in `~/.vibe-action/config.yaml`:

```yaml
groups:
  - git: https://gitcode.com/keygenqt_vz/vibe-action-groups.git
    path: /code
    name: code
    about: Work with code
```

| Field   | Description                                           |
| ------- | ----------------------------------------------------- |
| `git`   | Repository URL, cloned once into the cache            |
| `path`  | Local dir, or subfolder inside the clone (`/` = root) |
| `ref`   | Branch, tag, or commit pin (git only)                 |
| `name`  | CLI group name (`vibe-action <name> <action>`)        |
| `about` | Short description for help                            |

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

| Field     | Type   | Required | Description                                       |
| --------- | ------ | -------- | ------------------------------------------------- |
| `version` | string | yes      | Must match `PIPELINE_VERSION` (currently `0.0.2`) |
| `name`    | string | yes      | Action name; becomes the CLI subcommand           |
| `about`   | string | yes      | Short description for `--help`                    |
| `notify`  | bool   | no       | Desktop notification on completion                |
| `check`   | string | no       | Regex validating the final pipeline result        |
| `args`    | list   | no       | CLI argument definitions                          |
| `api`     | object | no       | IDE plugin metadata (ignored by CLI runtime)      |
| `actions` | list   | yes      | Pipeline steps                                    |

### Args

| Field     | Type   | Required | Description                                        |
| --------- | ------ | -------- | -------------------------------------------------- |
| `name`    | string | yes      | Convention: `arg_` prefix; becomes `--<name>` flag |
| `short`   | char   | no       | Short flag alias                                   |
| `input`   | string | yes      | `string`, `bool`, `number`, `path`                 |
| `default` | string | no       | Default value; makes the arg optional              |
| `help`    | string | no       | Help text                                          |

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

### Actions (steps)

| Field    | Type   | Required | Description                                         |
| -------- | ------ | -------- | --------------------------------------------------- |
| `tag`    | string | yes      | Unique id, referenced by other steps via `data`     |
| `run`    | string | yes      | Engine (below)                                      |
| `val`    | list   | no       | Val candidates                                      |
| `when`   | string | no       | Action-level guard; skip whole step if false        |
| `reg`    | string | no       | Regex validating step output; mismatch = hard error |
| `ask`    | bool   | no       | Confirm before execution (default false)            |
| `action` | string | yes      | Template with `{name}` placeholders                 |

Run types:

| Value    | Engine                                     |
| -------- | ------------------------------------------ |
| `value`  | Literal passthrough, no execution          |
| `cmd`    | Shell via `sh -c`; values are shell-quoted |
| `tiny`   | LLM — tiny model (1-3b)                    |
| `small`  | LLM — small model (3-7b)                   |
| `medium` | LLM — medium model (7-14b)                 |
| `large`  | LLM — large model (14b+)                   |
| `vision` | LLM with extracted base64 images           |

Execution order is resolved by data dependencies (topological sort), not list position. A step that references `tag_x` runs after the step producing `tag_x`.

### Val candidates

| Field  | Type          | Description                                   |
| ------ | ------------- | --------------------------------------------- |
| `name` | string        | Placeholder `{name}` in the `action` template |
| `data` | string        | Source tag or literal value                   |
| `mods` | string        | Operator pipe, applied left to right          |
| `when` | string        | Pre-mods soft guard; skip candidate if false  |
| `fail` | string        | Post-mods hard guard; abort if false          |
| `each` | bool / object | Fan-out over list items                       |

- Candidates resolve in order; the first whose `when` passes (or has none) wins.
- Dead-tag rule: if a step declares multiple names and any one of them has no winning candidate, the whole step is skipped.

### Data sources

| Prefix     | Source                |
| ---------- | --------------------- |
| `query_*`  | User input            |
| `system_*` | Runtime environment   |
| `arg_*`    | CLI argument          |
| `tag_*`    | Another step's result |
| _(none)_   | Mods-only candidate   |

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

### Example 4: dead-tag guard (dry-run)

```yaml
- tag: tag_commit_exec
  run: cmd
  ask: true
  val:
    - name: dry
      data: arg_dry_run
      when: 'contains:false'
    - name: path
      data: tag_project_path
    - name: msg
      data: tag_commit_message
      mods: 'lower'
  action: cd {path} && git add . && git commit -m {msg}
```

When `arg_dry_run` is `true`, the `dry` candidate fails its `when`, no candidate wins for `dry`, and the whole step becomes a dead tag and is skipped.

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

### Guards

`when` and `fail` accept inspect operators only. Every inspect operator supports `:not`.

| Operator             | Meaning                                        |
| -------------------- | ---------------------------------------------- |
| `contains:<X>`       | Substring test                                 |
| `equals:<X>`         | Exact equality                                 |
| `matches:<re>`       | Regex test                                     |
| `compare:<mode>:<N>` | Numeric compare; modes `gt` `lt` `gte` `lte`   |
| `is:<kind>`          | `empty` `num` `int` `bool` `url` `path` `json` |

### Operators (in `mods`)

> Note: `\|` in tables below is an escaped pipe character — in YAML use a plain `|`.

Read (world → value):

| Operator                  | Description                                             |
| ------------------------- | ------------------------------------------------------- |
| `ast[:brief\|full\|lang]` | Parse source to JSON AST                                |
| `fetch`                   | Download URL to temp file, or resolve local path        |
| `resolve[:dir\|file]`     | Resolve path to absolute                                |
| `scan`                    | Scan directory → list of file paths                     |
| `screenshot`              | Interactive capture → image path                        |
| `text`                    | Extract text from HTML/PDF/image, or base64 passthrough |

Transform (value → value):

| Operator                                          | Description                           |
| ------------------------------------------------- | ------------------------------------- |
| `split[:sep]`                                     | Split to list (default newline)       |
| `join[:sep]`                                      | Collapse list to string               |
| `item:<N>`                                        | Nth element; negative counts from end |
| `size`                                            | Item count / byte length              |
| `filter:<X>` / `filter:eq:<X>` / `filter:not:<X>` | Keep/remove by substring              |
| `grep:<re>` / `grep:<re>:not`                     | Keep/remove by regex                  |
| `sort[:asc\|desc]`                                | Sort                                  |
| `uniq`                                            | Deduplicate                           |
| `reverse`                                         | Reverse                               |
| `take:<N>` / `tail:<N>`                           | First / last N                        |
| `lower` / `upper`                                 | Case conversion                       |
| `replace:<from>:<to>`                             | Substring replace                     |
| `trim[:chars]`                                    | Trim ends / drop empty items          |
| `strip`                                           | Remove Markdown fences                |
| `default:<X>`                                     | Fallback for empty                    |
| `base64:encode` / `base64:decode`                 | Base64                                |
| `format:json\|json5\|yaml\|toml`                  | Format conversion                     |

Write (pass-through, side effect):

| Operator                             | Description                    |
| ------------------------------------ | ------------------------------ |
| `clipboard_text`                     | Copy to clipboard              |
| `clipboard_image`                    | Copy base64 image to clipboard |
| `file:<path>` / `file:<path>:append` | Write / append to file         |

### Query tags

`query_raw`, `query_prompt`, `query_clipboard`, `query_clipboard_text`,
`query_clipboard_path`, `query_clipboard_image`, `query_file_path`,
`query_project_path`, `query_line`, `query_image`.

### System tags

`system_os`, `system_arch`, `system_hostname`, `system_user`, `system_uid`,
`system_pid`, `system_shell`, `system_language`, `system_date`,
`system_time`, `system_datetime`, `system_timestamp`, `system_cpu_cores`,
`system_mem_available`, `system_dir_home`, `system_dir_pwd`,
`system_dir_config`, `system_dir_data`, `system_dir_cache`,
`system_dir_download`, `system_dir_temp`.

### Placeholders & escapes

- `{name}` — substitutes the resolved val candidate value.
- `{{X}}` — literal `{X}` (for JSON templates / LLM output).
- In operator args and `each`: `\n`, `\t`, `\s` are expanded.
- For `run: cmd`, placeholder values are shell-quoted automatically — do not quote them again.

## Validation rules

A group fails to load if any of these is violated:

- `version` does not match `PIPELINE_VERSION`.
- Empty `name` or `about`.
- Duplicate action name within the group.
- Action name equals the group name.
- Action name collides with a system command (`clean`, `status`, `bench`, `stop`).
- Duplicate `tag` across the file.
- `data` references its own `tag`.
- Bare `query` in `data` (use `query_raw`).
- `query_*` / `system_*` used as a tag or arg name.
- Unknown operator in `mods`.
- Non-inspect operator in `when` / `fail`.
- Undeclared `{name}` in `action`.

## Conventions

- Folder name = group name; file name = action name.
- Prefix argument names with `arg_`.
- Use `when` guards to avoid unnecessary work; `fail` guards to catch bad input before the LLM.
- Use `ask: true` for destructive shell commands.
- `reg: '.+'` catches empty results from a loose LLM.
- Keep prompts in English; write action `name`/`about` in English.
- Tag naming: `tag_<purpose>` (e.g. `tag_content`, `tag_review`).

## See also

Full operator reference and pipeline schema: [Vibe Action docs](https://vibe-action.keygenqt.com).
