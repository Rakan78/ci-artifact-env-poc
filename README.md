# ci-artifact-env-poc

Self-contained reproduction of a GitHub Actions **artifact-poisoning → `$GITHUB_ENV` injection →
code execution** primitive. Built by a security researcher in their own account. Every payload
here is an `echo`, a `console.log`, or one `fprintf` to stderr. Nothing in this repository
touches, references, or depends on any third party's infrastructure.

## The shape under test

Two workflows, the pattern as it appears in real projects:

1. **`Validate`** runs `on: pull_request`. For a fork pull request GitHub runs the workflow file
   **from the pull request's own merge ref** — so the fork author controls this file and every
   byte of the artifact it uploads. The job itself is unprivileged: read-only `GITHUB_TOKEN`, no
   secrets (`Secret source: None`), PR-scoped Actions cache.
2. **A consumer** runs `on: workflow_run` after `Validate` completes. That trigger runs the
   **base branch's** workflow in the **base repository's** context — repository secrets in scope
   (`Secret source: Actions`), token not forced read-only. It downloads the artifact the fork
   produced, `unzip`s it into the workspace, and appends every line of the extracted `env` file
   to `$GITHUB_ENV`.

The fork therefore chooses **environment variables** for a job running as the base repository,
and **files** in that job's workspace. Those two together are code execution.

Two consumers are included deliberately:

| file | shape | primitive it demonstrates |
|---|---|---|
| `.github/workflows/post-comment.yml` | has a `run:` step after the unzip | `BASH_ENV` |
| `.github/workflows/faithful-consumer.yml` | **no** `run:` step after the unzip; last step is a Node action | `LD_PRELOAD` |

The second shape is the one that matters: a consumer whose only post-injection step is a Node
action cannot be attacked through `BASH_ENV`, and `NODE_OPTIONS` is blocked (below). `LD_PRELOAD`
works against it.

## Results

| PR | run | what it showed |
|---|---|---|
| #1 | `44447818728` | A fork-chosen env var and a fork-chosen **file** both cross into the privileged job. `NODE_OPTIONS` is **rejected** by the runner. |
| #2 | `44448324839` | `BASH_ENV` → the fork's shell executes as the base repo on `refs/heads/main`. |
| #3 | `44448582608` | Read-only probe: the next Node action's `dist/index.js` is **writable** by the injected process. Nothing was modified. |
| #4 | `44449336989` | `LD_PRELOAD` → the fork's shared object is loaded **into the Node action's process**, in the consumer shaped like the real-world case. |

Decisive lines from the run #4 job log:

```
  LD_PRELOAD: /home/runner/work/ci-artifact-env-poc/ci-artifact-env-poc/poc.so
[poc-ldpreload] shared object loaded into this process
node action running; LD_PRELOAD=/home/runner/work/ci-artifact-env-poc/ci-artifact-env-poc/poc.so
```

## Two things that do NOT work

Both were tried here, and both failed. Anyone writing this class of finding should know before
they publish:

- **`NODE_OPTIONS=--require <file>`** — the obvious primitive, and it does not work.
  `NODE_OPTIONS` is on the runner's `$GITHUB_ENV` block list:
  `##[error]Can't store NODE_OPTIONS output parameter using '$GITHUB_ENV' command.`
- **Actions cache poisoning** — unavailable from this job. Every fork-triggered `workflow_run`
  job logs `Cache mode: read`; the same workflow on a base-branch `push` logs
  `Cache mode: write`. There is no cache-write credential here.

## The approval gate, measured

The first pull request from a brand-new account sat at `conclusion: action_required` until the
repository owner approved it (`run_attempt: 2`, `run_started_at` 19s after `created_at`). After
that pull request was **merged**, every later pull request from the same account ran immediately
(`run_attempt: 1`, `run_started_at == created_at`).

GitHub's default gate — "require approval for first-time contributors" — therefore costs an
attacker exactly one merged pull request. "Requires contributor status" is not a security
control.

## Reproducing it

Fork this repository with a second account, change the `Save PR number` step in `validate.yml`
to write your own `env` file (and whatever file it references) into `./env/`, and open a pull
request. Remember that the runner workspace path is `/home/runner/work/<repo>/<repo>`.

Read the privileged job's log with `curl`, not `gh api` — the logs contain terminal escape
sequences that `gh` refuses to print:

```bash
curl -sSL -H "Authorization: Bearer $(gh auth token)" \
  "https://api.github.com/repos/<owner>/<repo>/actions/jobs/<job id>/logs"
```

## Remediation for the pattern

A `workflow_run` job must treat a downloaded artifact as untrusted input:

- never pipe artifact contents into `$GITHUB_ENV`, `$GITHUB_PATH` or `$GITHUB_OUTPUT`;
- parse the one value you need and validate it (`PR_NUMBER` should match `^[0-9]+$`);
- extract artifacts to a scratch directory, never the workspace;
- set an explicit least-privilege `permissions:` block on the consuming workflow rather than
  inheriting the repository default.

## Scope of this repository

Researcher-owned base repository and researcher-owned fork. No third-party repository was forked,
and no pull request, comment or workflow dispatch was ever directed at anyone else's project.
