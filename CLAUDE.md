# OTP Test Porting Agent

Automated agent for porting Ginkgo e2e tests from `openshift/openshift-tests-private` to `openshift/origin`.

## Repos

- **Source**: `openshift/openshift-tests-private` (branch: `porting-prep`) — contains tests annotated with `// port=yes|no|maybe|complete`
- **Destination**: `dgoodwin/origin` (fork of `openshift/origin`) — PRs created here against `openshift/origin`
- **This repo**: `dgoodwin/otp-test-porting-agent` — orchestration only (workflows, skills, CLAUDE.md)

## Session Environment

When running as an Ambient session, repos are cloned under `/workspace/`:
- `/workspace/openshift-tests-private/` — source tests
- `/workspace/origin/` — destination for ported tests

The `gh` CLI is authenticated via the `GITHUB_TOKEN` environment variable.

## Porting Workflow

```
Issue created with label "port-tests" → GitHub Actions → Ambient session created →
Agent runs /port-tests skill → PR created on dgoodwin/origin →
Source updated on porting-prep branch → Agent monitors CI on origin PR
```

## Conventions

- Only port tests marked `// port=yes` — never `port=no`, `port=maybe`, or `port=complete`
- Prefer tests with higher pass rates when selecting candidates
- Add `[OTP]` tag to ported test names, remove `Author:xxx-` prefixes
- Replace `compat_otp` imports with `exutil` equivalents
- Verify compilation with `go build ./test/extended/...` after porting
- Update source annotations to `port=complete` after successful port
- Mark tests as `port=yes-but-difficult` if skipped due to complexity

## PR Management

- All PRs created by this agent get the `ambient-code:managed` label
- PR body includes frontmatter: `<!-- acp:session_id=... source=#N last_action=... retry_count=0 -->`
- Circuit breaker: after 3 failed fix attempts, label `ambient-code:needs-human` and stop
- Never merge PRs — leave open for human review
- Never force-push

## Commands

- `/port-tests` — main porting skill (see `.claude/commands/port-tests.md`)
