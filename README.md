# OTP Test Porting Agent

Automated agent that ports Ginkgo e2e tests from [openshift-tests-private](https://github.com/openshift/openshift-tests-private) to [openshift/origin](https://github.com/openshift/origin) using the [Ambient Code Platform](https://github.com/ambient-code).

## How It Works

```
You create an issue → label it "port-tests" → GitHub Actions triggers →
Ambient session runs Claude Code with the /port-tests skill →
PR created on dgoodwin/origin → source annotations updated →
Cron checks CI every 3h → fixes failures automatically
```

## Setup

### 1. GitHub Repo Secrets

Add these secrets to this repo (**Settings → Secrets and variables → Actions**):

| Secret | Description | Required |
|--------|-------------|----------|
| `AMBIENT_API_URL` | Ambient platform API endpoint | Yes |
| `AMBIENT_BOT_TOKEN` | Ambient service account token | Yes |
| `AMBIENT_PROJECT` | Ambient project namespace | Yes |
| `GH_PAT` | GitHub PAT with read access to openshift-tests-private and write access to dgoodwin/origin | Yes |
| `SLACK_WEBHOOK_URL` | Slack incoming webhook for notifications | No |
| `PLATFORM_HOST` | Ambient platform UI base URL (for session links) | No |

> **Why `GH_PAT` instead of `GITHUB_TOKEN`?** The default `GITHUB_TOKEN` is scoped to this repo only. The agent needs cross-repo access to read openshift-tests-private and push to dgoodwin/origin.

### 2. GitHub Labels

Create these labels on this repo:

| Label | Purpose |
|-------|---------|
| `port-tests` | Triggers a porting session when added to an issue |

Create these labels on **dgoodwin/origin**:

| Label | Purpose |
|-------|---------|
| `ambient-code:managed` | Marks PRs the agent created and maintains |
| `ambient-code:needs-human` | Agent gave up after 3 retries |

### 3. Verify PAT Permissions

Your `GH_PAT` needs these scopes:
- `repo` — full access to dgoodwin/origin (push branches, create PRs)
- `repo` — read access to openshift/openshift-tests-private
- `read:org` — if openshift-tests-private requires org membership

Test it:
```bash
# Should succeed
gh api repos/openshift/openshift-tests-private --jq '.full_name'
gh api repos/dgoodwin/origin --jq '.permissions'
```

## Usage

### Port Tests via Issue

1. Create an issue on this repo describing what to port:

   **Title**: Port 5 networking tests

   **Body**:
   ```
   Port 5 tests from the networking subdirectory.
   Focus on tests with highest pass rates.
   ```

2. Add the `port-tests` label to the issue.

3. The agent will:
   - Acknowledge with 👀 and a comment
   - Create an Ambient session
   - Port the tests, verify compilation
   - Create a PR on dgoodwin/origin
   - Update annotations on openshift-tests-private
   - Comment on the issue with results

### Follow Up

Comment `@ambient-code` on the issue to give the agent follow-up instructions:

```
@ambient-code the PR has a compilation error in networking/egressfirewall.go, please fix it
```

```
@ambient-code port 3 more tests from storage
```

### Manual Batch Check

Go to **Actions → OTP Test Porting Agent → Run workflow** to trigger a batch check of all managed PRs on dgoodwin/origin.

## Architecture

```
otp-test-porting-agent (this repo)
├── .github/workflows/amber-handler.yml  ← GitHub Actions event router
├── .claude/commands/port-tests.md       ← Porting skill (runs inside Ambient session)
├── CLAUDE.md                            ← Agent conventions and context
└── README.md                            ← This file

dgoodwin/origin (fork)
└── PRs created here targeting openshift/origin

openshift/openshift-tests-private (porting-prep branch)
└── Tests annotated with // port=yes|no|maybe|complete
```

## Monitoring

- **Issue comments**: The agent posts status updates on the tracking issue
- **Slack** (optional): Notifications when sessions start/complete/need help
- **Ambient UI**: Watch agent work in real-time via session links
- **GitHub Actions**: Check workflow runs for errors
