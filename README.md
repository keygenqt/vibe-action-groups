# Vibe Action Groups

External action groups for [Vibe Action](https://vibe-action.keygenqt.com) — a command router for shell and LLM tasks via YAML pipelines.

Each folder is a group, each YAML file is an action. Connect this repo in your Vibe Action config to get all groups at once.

## Groups

| Group     | Actions                       | About                           |
| --------- | ----------------------------- | ------------------------------- |
| `code`    | comment, explain, review      | Work with code                  |
| `data`    | extract, fetch, find          | Fetch and extract external data |
| `gen`     | mock, naming, regex, synonyms | Generate patterns and data      |
| `project` | commit, scan                  | Whole-project operations        |
| `text`    | spellcheck, tone, translate   | Work with text                  |
| `vision`  | describe, whois               | Work with images                |

## Usage

Add the repo to your Vibe Action config:

```yaml
groups:
  - name: code
    about: Work with code
    git: https://gitcode.com/keygenqt_vz/vibe-action-groups.git
    path: /code

  - name: text
    about: Work with text
    git: https://gitcode.com/keygenqt_vz/vibe-action-groups.git
    path: /text
```

`path` selects a subfolder inside the clone (`/` = repo root). Pin a
specific branch, tag, or commit with `ref`:

```yaml
groups:
  - name: code
    about: Work with code
    git: https://gitcode.com/keygenqt_vz/vibe-action-groups.git
    path: /code
    ref: v1.0.0
```

Then run:

```bash
vibe-action code review main.rs
vibe-action text translate README.md -l Chinese
vibe-action project commit
```

## Actions

### code

#### `comment`

Replace every `TODO` and `@todo` in code with a concise technical comment.

|        |                                        |
| ------ | -------------------------------------- |
| Model  | `medium`                               |
| Input  | `query_raw` (file path or inline text) |
| Output | `replace`                              |

- Keeps the original code and comment markers.
- File/module-level comments: two short sentences.
- Class/struct/field/method comments: max 7 words.
- Preserves indentation and structure, wraps output in a code block matching the input language.

#### `explain`

Add explanatory comments above each logical block in code.

|        |                                        |
| ------ | -------------------------------------- |
| Model  | `medium`                               |
| Input  | `query_raw` (file path or inline text) |
| Output | `replace`                              |

- The code itself is unchanged — only comments are inserted.
- Comment language follows the system language.

#### `review`

Critically analyze code for logic flaws, concurrency bugs, memory leaks, and performance bottlenecks.

|        |                                        |
| ------ | -------------------------------------- |
| Model  | `large`                                |
| Input  | `query_raw` (file path or inline text) |
| Output | `dialog`                               |

- Output is blunt and technical.
- Each issue has a one-line `Fix:` patch.
- Outputs `PERFECT` when no flaws are found.

### data

#### `extract`

Extract structured data or matching lines from text and logs.

|        |                                                      |
| ------ | ---------------------------------------------------- |
| Model  | `large` + value steps                                |
| Input  | `query_prompt`                                       |
| Output | `dialog`                                             |
| Args   | `-f` / `--arg_file` — path or URL to the source file |

Pipeline: each line is evaluated independently by the LLM — relevant lines are output unchanged, others become `-`; results are filtered, deduplicated, and joined.

#### `fetch`

Fetch a web page or PDF via URL and write a short summary.

|        |                |
| ------ | -------------- |
| Model  | `medium`       |
| Input  | `query_prompt` |
| Output | `dialog`       |

- Downloads the URL, extracts text, and summarizes in 2–5 sentences.
- Summary language follows the system language.

#### `find`

Semantic file finder — finds files by meaning, not just name.

|        |                                                          |
| ------ | -------------------------------------------------------- |
| Model  | `medium` + shell steps                                   |
| Input  | `query_prompt`                                           |
| Output | `dialog`                                                 |
| Args   | `-p` / `--arg_path` — directory to search (default: `.`) |

Pipeline: collect text files → build one prompt per file → LLM evaluates each against the query → non-matches are filtered out.

### gen

#### `mock`

Generate realistic mock data arrays.

|        |                                                                 |
| ------ | --------------------------------------------------------------- |
| Model  | `medium`                                                        |
| Input  | `query_prompt`                                                  |
| Output | `dialog`                                                        |
| Args   | `-f` / `--arg_format` — `json`, `yaml`, `csv` (default: `json`) |

#### `naming`

Generate 5–10 professional naming suggestions from a description.

|        |                |
| ------ | -------------- |
| Model  | `medium`       |
| Input  | `query_prompt` |
| Output | `dialog`       |

- Output is plain: one suggestion per line, no numbering or code blocks.

#### `regex`

Generate a regex pattern from a natural language description.

|        |                                                                  |
| ------ | ---------------------------------------------------------------- |
| Model  | `medium`                                                         |
| Input  | `query_prompt`                                                   |
| Output | `dialog`                                                         |
| Args   | `-e` / `--arg_example` — optional example string to test against |

- Outputs only the raw pattern, exactly one line.

#### `synonyms`

Find programming/technical synonyms for a word.

|        |                       |
| ------ | --------------------- |
| Model  | `medium` + value step |
| Input  | `query_raw`           |
| Output | `dialog`              |

- 5 single-word alternatives, lowercased and newline-joined.

### project

#### `commit`

Generate a conventional commit message from the working tree.

|        |                                                           |
| ------ | --------------------------------------------------------- |
| Model  | `medium` + `large` + shell steps                          |
| Input  | `query_project_path`                                      |
| Output | `dialog`                                                  |
| Args   | `-d` / `--arg_dry_run` — print message without committing |

Pipeline:

1. Resolve project path.
2. Shell: `git status`, `git diff --stat`, recent commits, changed files, per-file diffs.
3. LLM (`medium`): summarize each file's change purpose in one phrase.
4. LLM (`large`): synthesize all summaries into one commit message.
5. Shell (`ask: true`): run `git add . && git commit -m`.

Message format: `<type>: <message>`, max 100 chars, no scope or parentheses.

#### `scan`

Scan a project directory, parse every source file into a JSON AST, and export to a single structured JSON file in the downloads directory.

|        |                           |
| ------ | ------------------------- |
| Model  | value steps only (no LLM) |
| Input  | `query_project_path`      |
| Output | `dialog`                  |

Pipeline: `resolve:dir` → `scan` → `ast` → `format:json` → `file`.

### text

#### `spellcheck`

Check and fix spelling in text or files.

|        |                                        |
| ------ | -------------------------------------- |
| Model  | `small` + `medium` + shell steps       |
| Input  | `query_raw` (file path or inline text) |
| Output | `replace`                              |

Two-stage pipeline: a small model decides `DIRTY` / `CLEAR`, then a medium model fixes errors only when dirty. Preserves code syntax and formatting.

#### `tone`

Transform rude or aggressive text into a professional, calm equivalent.

|        |             |
| ------ | ----------- |
| Model  | `medium`    |
| Input  | `query_raw` |
| Output | `replace`   |

- Preserves the original meaning and language.
- Built-in examples for English, Chinese, and Russian.

#### `translate`

Translate text or files to a target language.

|        |                                                          |
| ------ | -------------------------------------------------------- |
| Model  | `medium`                                                 |
| Input  | `query_raw` (file path or inline text)                   |
| Output | `replace`                                                |
| Args   | `-l` / `--arg_to` — target language (default: `Russian`) |

- Preserves proper names, code, and comments inside code blocks.

### vision

#### `describe`

Describe a screenshot in rich detail for text-only LLMs.

|        |                     |
| ------ | ------------------- |
| Model  | `vision` + `medium` |
| Input  | `query_image`       |
| Output | `dialog`            |

Two-step pipeline: the vision model produces a raw description, then a small model formats it into clean Markdown. Lists colors with hex codes.

Image source fallback chain: `query_image` → `query_clipboard_image` → `screenshot|text`.

#### `whois`

Identify public tech figures in a screenshot.

|        |               |
| ------ | ------------- |
| Model  | `vision`      |
| Input  | `query_image` |
| Output | `dialog`      |

Outputs structured summaries (name, role, impact) from left to right.

Image source fallback chain: `query_image` → `query_clipboard_image` → `screenshot|text`.
