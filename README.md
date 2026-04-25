# OTP Test Porting Agent

Automated agent that ports Ginkgo e2e tests from [openshift-tests-private](https://github.com/openshift/openshift-tests-private) to [openshift/origin](https://github.com/openshift/origin) using the [Ambient Code Platform](https://github.com/ambient-code).

## How It Works

```
You create an issue → label it "port-tests" → GitHub Actions triggers →
Ambient session runs Claude Code with the /port-tests skill →
PR created on dgoodwin/origin → source annotations updated →
Cron checks CI every 3h → fixes failures automatically
```

### Architecture

```
otp-test-porting-agent (this repo)
├── .github/workflows/amber-handler.yml  ← GitHub Actions event router
├── .claude/commands/port-tests.md       ← Porting skill (runs inside Ambient session)
├── CLAUDE.md                            ← Agent conventions and context
└── README.md                            ← This file

dgoodwin/origin (fork)
└── PRs created here targeting openshift/origin

dgoodwin/openshift-tests-private (fork, porting-prep branch)
└── Tests annotated with // port=yes|no|maybe|complete
```

### Three-Layer Architecture

| Layer | What | Where |
|-------|------|-------|
| **GitHub Actions** | Event router — listens for GitHub events, decides when to act | `.github/workflows/amber-handler.yml` |
| **`ambient-action`** | Bridge — calls the Ambient API to create/resume sessions | Referenced as `uses: ambient-code/ambient-action@v0.0.5` |
| **Ambient Session** | Worker — runs Claude Code in a K8s pod with repo access and `gh` CLI | Managed by Ambient platform |

GitHub Actions is the cheap event router (zero cost at idle). Ambient sessions are the expensive workers (only run when there's actual work). The event router triggers workers, not the other way around.

---

## Setup Guide (Step by Step)

### Step 1: Fork the Repos

You need forks of both repos so the agent can push branches and create PRs:

1. **Fork openshift/origin** → `dgoodwin/origin`
   - The agent pushes ported-test branches here and creates PRs against `openshift/origin`

2. **Fork openshift/openshift-tests-private** → `dgoodwin/openshift-tests-private`
   - The agent reads tests marked `// port=yes` from the `porting-prep` branch
   - After porting, the agent pushes `port=complete` updates back to this branch

#### Push the porting-prep branch to your fork

If you created the `porting-prep` branch locally on the upstream clone, push it to your fork:

```bash
cd ~/go/src/github.com/openshift/openshift-tests-private
git remote add fork git@github.com:dgoodwin/openshift-tests-private.git
git push fork porting-prep
```

### Step 2: Create an Ambient Access Key (Bot Token)

The agent authenticates to the Ambient API using a Kubernetes ServiceAccount token created through the Ambient UI.

1. Go to your Ambient project settings:
   `https://<ambient-host>/projects/dgoodwin`
2. Navigate to **Access Keys** (under project settings)
3. Create a new access key:
   - **Name**: `porting-bot`
   - **Role**: `edit` (sufficient for creating sessions and sending messages)
   - **Expiration**: 90 days (or your preference)
4. Copy the generated JWT token — this is your `AMBIENT_BOT_TOKEN`

#### Identify the correct API URL

**Important**: The browser URL (e.g., `https://ambient-code.apps.rosa.vteam-uat.0ksl.p3.openshiftapps.com`) is the frontend behind an OAuth proxy. It will reject Bearer tokens with `401 User token required`.

The `AMBIENT_API_URL` must point to the **public-api** route, which accepts Bearer tokens directly. Find it with:

```bash
oc get route -n <ambient-namespace> | grep public-api
```

The public-api URL is typically something like:
```
https://public-api-route-<namespace>.apps.rosa.vteam-uat.0ksl.p3.openshiftapps.com
```

#### Verify the token works

```bash
curl -s -w "\n%{http_code}\n" \
  -H "Authorization: Bearer <your-token>" \
  "<PUBLIC_API_URL>/projects/dgoodwin/agentic-sessions"
```

You should get `200` with a JSON list. If you get `401`, either the token is invalid or you're hitting the OAuth-proxied frontend URL instead of the public-api URL.

### Step 3: Create a GitHub Personal Access Token (PAT)

The agent needs cross-repo access (the default `GITHUB_TOKEN` is scoped to this repo only).

1. Go to https://github.com/settings/tokens?type=beta (fine-grained tokens)
2. **Token name**: `otp-porting-agent`
3. **Repository access**: Select repositories → add `dgoodwin/origin` and `dgoodwin/openshift-tests-private`
4. **Permissions**:
   - **Contents**: Read and write (push branches)
   - **Pull requests**: Read and write (create PRs, add labels)
   - **Metadata**: Read (auto-included)

If fine-grained tokens don't work for org repos, use a **classic token** with `repo` scope instead.

### Step 4: Add Repository Secrets

Go to this repo's **Settings → Secrets and variables → Actions → Repository secrets** and add:

| Secret | Value | Notes |
|--------|-------|-------|
| `AMBIENT_API_URL` | Public API route URL | **Not** the browser/frontend URL (see Step 2) |
| `AMBIENT_BOT_TOKEN` | JWT from access key | Created in Step 2 |
| `AMBIENT_PROJECT` | `dgoodwin` | Your Ambient project namespace |
| `GH_PAT` | GitHub PAT | Created in Step 3 |
| `PLATFORM_HOST` | Browser URL (optional) | For session links in comments, e.g., `https://ambient-code.apps.rosa...` |
| `SLACK_WEBHOOK_URL` | Slack webhook (optional) | For notifications |

### Step 5: Create GitHub Labels

On **this repo** (`dgoodwin/otp-test-porting-agent`):

```bash
gh label create "port-tests" --repo dgoodwin/otp-test-porting-agent \
  --description "Triggers a test porting session" --color 0E8A16
```

On **dgoodwin/origin**:

```bash
gh label create "ambient-code:managed" --repo dgoodwin/origin \
  --description "PR managed by Ambient Code agent" --color 1D76DB

gh label create "ambient-code:needs-human" --repo dgoodwin/origin \
  --description "Agent gave up after 3 retries" --color D93F0B
```

### Step 6: Test It

Create an issue and apply the `port-tests` label:

```bash
gh issue create --repo dgoodwin/otp-test-porting-agent \
  --title "Port 3 router tests" \
  --body "Port 3 tests from the networking/router subdirectory. Pick tests with the highest pass rates." \
  --label "port-tests"
```

Watch the workflow run at: `https://github.com/dgoodwin/otp-test-porting-agent/actions`

---

## Usage

### Port Tests via Issue

1. Create an issue describing what to port:

   **Title**: Port 5 networking tests

   **Body**:
   ```
   Port 5 tests from the networking subdirectory.
   Focus on tests with highest pass rates.
   ```

2. Add the `port-tests` label.

3. The agent will:
   - React with 👀 and post an acknowledgement comment
   - Create an Ambient session that clones both repos
   - Run the `/port-tests` skill to port tests
   - Create a PR on `dgoodwin/origin` with the `ambient-code:managed` label
   - Push `port=complete` updates to the `porting-prep` branch on `dgoodwin/openshift-tests-private`
   - Comment on the issue with results and links

### Follow Up with Comments

Comment `@ambient-code` on the issue to give follow-up instructions:

```
@ambient-code the PR has a compilation error in networking/egressfirewall.go, please fix it
```

```
@ambient-code port 3 more tests from storage
```

Only repo members/collaborators can trigger the agent (prevents abuse on public repos).

### Batch CI Check

Every 3 hours on weekdays, the workflow automatically checks all `ambient-code:managed` PRs on `dgoodwin/origin`. If CI is failing or there are merge conflicts, it triggers a fix session.

You can also trigger this manually: **Actions → OTP Test Porting Agent → Run workflow**.

### Session Reuse

The agent embeds session metadata in PR bodies as an HTML comment:
```html
<!-- acp:session_id=abc123 source=#42 last_action=2026-04-25T10:00:00Z retry_count=0 -->
```

When subsequent triggers fire (cron, comments), the workflow extracts `session_id` and resumes the **existing session** instead of creating a new one. This preserves conversation history and avoids re-exploring the codebase.

### Circuit Breaker

The `retry_count` prevents infinite fix loops:
- Each fix attempt increments `retry_count`
- At `retry_count >= 3`, the agent adds `ambient-code:needs-human`, removes `ambient-code:managed`, and stops
- A human must intervene and reset if needed

---

## Monitoring

| Channel | What you see |
|---------|-------------|
| **Issue comments** | Status updates, PR links, summaries |
| **GitHub Actions** | Workflow run logs, errors |
| **Ambient UI** | Real-time agent session (via session links in comments) |
| **Slack** (optional) | Notifications when sessions start/complete/need help |

---

## Troubleshooting

| Problem | Check |
|---------|-------|
| Workflow doesn't trigger | Verify workflow is on the default branch. Check Actions tab. |
| `401 User token required` | You're hitting the frontend/OAuth URL, not the public-api URL. See Step 2. |
| `401 Unauthorized` on session creation | `AMBIENT_BOT_TOKEN` is expired or invalid. Recreate the access key. |
| `403 Forbidden` | The access key role doesn't have access to the project. Use `edit` role. |
| Agent can't push to repos | `GH_PAT` missing `repo` scope or doesn't have access to the target repos. |
| Agent can't create PRs on origin | Ensure `GH_PAT` has write access to `dgoodwin/origin`. |
| Bot reacts to its own comments | The `author_association` filter prevents this — bots aren't MEMBER/OWNER. |
| Session created but agent does nothing | Check the Ambient session UI for errors. The runner pod may have failed to start. |
