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

Or point to the repo root and let the default config pick up all groups.

Then run:

```bash
vibe code review
vibe text translate README.md
vibe project commit
```
