# .github

Reusable GitHub Actions workflows shared by my repositories: an issue-driven delivery pipeline in
which an agent can implement an issue and open the pull request, while I review and merge.

## The flow

Every change starts as an issue on the [portfolio board](https://github.com/users/NicollasFrazao/projects/3).

| Status | How an issue gets there |
| --- | --- |
| **Todo** | Opened in a repository; `board-sync` adds it, the board's "Item added" workflow sets Todo |
| **Ready** | Labelled `agent`: specified and released to the agent (`board-sync`) |
| **In Progress** | The agent starts (`agent`), or a person starts by hand; an umbrella issue follows its first sub-issue (`board-sync`) |
| **In review** | A pull request with `Closes #N` opens (the board's "Pull request linked to issue" workflow) |
| **Done** | The pull request merges and closes the issue (the board's "Item closed" workflow) |

Merging is always a person's call. The agent never merges, never pushes to the default branch and
never edits workflows, rulesets or secrets. Each repository's ruleset still requires every CI check.

## Workflows

| Workflow | Kind | What it does |
| --- | --- | --- |
| [`board-sync.yml`](.github/workflows/board-sync.yml) | reusable | Adds issues to the board, moves `agent`-labelled issues to Ready, moves an umbrella issue to In Progress when work on a sub-issue starts |
| [`agent.yml`](.github/workflows/agent.yml) | reusable | Runs [Claude Code](https://github.com/anthropics/claude-code-action) on an `agent`-labelled issue: reads the repository's context base, implements, runs the local gate and opens the pull request. One agent at a time per repository |
| [`ci.yml`](.github/workflows/ci.yml), [`codeql.yml`](.github/workflows/codeql.yml) | this repository | actionlint and CodeQL on the workflows themselves |

## Opting a repository in

Add `.github/workflows/pipeline.yml`:

```yaml
name: Pipeline

on:
  issues:
    types: [opened, reopened, labeled]
  pull_request:
    types: [opened, reopened, ready_for_review]

permissions: {}

jobs:
  board:
    if: vars.BOARD_SYNC_ENABLED == 'true' && github.actor != 'dependabot[bot]'
    uses: NicollasFrazao/.github/.github/workflows/board-sync.yml@master
    secrets:
      BOARD_TOKEN: ${{ secrets.BOARD_TOKEN }}

  agent:
    if: >-
      vars.AGENT_ENABLED == 'true'
      && github.event_name == 'issues'
      && github.event.action == 'labeled'
      && github.event.label.name == 'agent'
    uses: NicollasFrazao/.github/.github/workflows/agent.yml@master
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
    with:
      sync-board: ${{ vars.BOARD_SYNC_ENABLED == 'true' }}
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      AGENT_GITHUB_TOKEN: ${{ secrets.AGENT_GITHUB_TOKEN }}
      BOARD_TOKEN: ${{ secrets.BOARD_TOKEN }}
```

Both jobs are **off until switched on** with repository variables, so the file is safe to add first.

| Switch (repository variable) | Needs (repository secrets) |
| --- | --- |
| `BOARD_SYNC_ENABLED=true` | `BOARD_TOKEN`: a classic token with the `project` scope (the default token cannot reach a user project) |
| `AGENT_ENABLED=true` | `CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token`; `AGENT_GITHUB_TOKEN` (fine-grained: contents, pull requests and issues read/write), because pull requests opened with the default token do not trigger CI. Without it, install the Claude GitHub App |

## Safety

- Only someone with triage access can add the `agent` label, so an issue from a stranger never
  starts the agent on its own; read the issue before labelling it, since its text becomes the
  agent's instructions.
- `max-turns` (default 80) caps how much one issue can spend; the concurrency group keeps one
  agent per repository.
- Third-party actions are pinned to a commit; Dependabot keeps the pins current.

## License

[MIT](LICENSE)
