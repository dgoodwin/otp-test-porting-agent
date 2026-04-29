---
description: Port tests from openshift-tests-private to openshift/origin
argument-hint: <count> [--from <subdir>]
---

## Name

port-tests

## Synopsis

```
/port-tests <count> [--from <subdir>]
```

## Description

Ports e2e tests from `dgoodwin/openshift-tests-private` (branch: `porting-prep`) to `openshift/origin`. Selects tests marked with `// port=yes` comments, adapts them to origin's framework and conventions, and ensures the project compiles.

This command:

- Scans the specified subdirectory in `openshift-tests-private` for tests marked `// port=yes`
- Selects the requested number of tests (preferring higher pass rates)
- Determines the appropriate destination package in `origin/test/extended/`
- Ports each test, adapting imports, utilities, and framework calls
- Ensures the destination project compiles
- Updates the port comment from `port=yes` to `port=complete` in the source repo
- Creates PRs in both repos

## Arguments

- `$1` (required): Integer — number of tests to port
- `--from <subdir>` (optional): Subdirectory under `openshift-tests-private/test/extended/` to select tests from (e.g., `networking`, `storage`). If omitted, scan all subdirectories.

## Implementation

### Step 1: Resolve Paths

- **Source repo**: `/workspace/openshift-tests-private` (cloned by Ambient)
- **Destination repo**: `/workspace/origin` (cloned by Ambient)
- **Source test root**: `/workspace/openshift-tests-private/test/extended/`
- **Destination test root**: `/workspace/origin/test/extended/`
- If `--from` is specified, narrow the source scan to `<source-test-root>/<subdir>/`

Verify both repos are cloned and accessible before proceeding.

### Step 2: Discover Candidate Tests

Search source files for `// port=yes` comments. Each candidate is a `g.It(...)` block immediately following a `// port=yes` comment line.

**Only consider tests where the comment value is exactly `port=yes`.** Do not port tests marked `port=no`, `port=maybe`, `port=complete`, `port=yes-but-difficult`, or any other value.

For each candidate, extract:
- The file path and line number
- The full `// port=yes - ...` comment (preserving pass rate metadata)
- The `g.It(...)` block name and body
- The enclosing `g.Describe(...)` context (sig tag, feature description)
- Any `g.BeforeEach` / `g.AfterEach` / `g.DeferCleanup` blocks that the test depends on
- Package-level variables and helper functions used by the test

Select up to `<count>` tests from the candidates. Prefer tests with higher pass rates.

### Step 3: Determine Destination Package

For each test, map the source subdirectory and sig tag to the appropriate destination package in `origin/test/extended/`. Use these rules:

1. **Direct match**: If a directory with the same name exists in origin's `test/extended/`, use it (e.g., `networking` -> `networking`, `storage` -> `storage`)
2. **Sig-tag mapping**: Map the `[sig-*]` tag to existing directories:
   - `[sig-network]` -> `networking/`
   - `[sig-storage]` -> `storage/`
   - `[sig-auth]` -> `authentication/` or `authorization/`
   - `[sig-apps]` -> `deployments/` or `apps/`
   - `[sig-api-machinery]` -> `apiserver/`
   - `[sig-cluster-lifecycle]` -> `cluster/`
   - `[sig-imageregistry]` -> `image_registry/`
   - `[sig-node]` -> `node/`
   - `[sig-instrumentation]` -> `prometheus/`
3. **New directory**: If no match exists, create a new package under `test/extended/` and register it in `test/extended/include.go` with a blank import

### Step 4: Port Each Test

#### 4a. Adapt Imports and Framework Calls

The source repo uses a `compat_otp` compatibility layer. The destination uses `exutil` directly:

| Source (openshift-tests-private) | Destination (origin) |
|----------------------------------|----------------------|
| `compat_otp.NewCLI(...)` | `exutil.NewCLI(...)` |
| `compat_otp.NewCLIWithoutNamespace(...)` | `exutil.NewCLIWithoutNamespace(...)` |
| `compat_otp.By(...)` | `g.By(...)` |
| `compat_otp.FixturePath(...)` | `exutil.FixturePath(...)` |
| `compat_otp.KubeConfigPath()` | Remove — `exutil.NewCLI` doesn't take this |
| `import compat_otp "github.com/openshift/origin/test/extended/util/compat_otp"` | `import exutil "github.com/openshift/origin/test/extended/util"` |
| `import "github.com/openshift/openshift-tests-private/test/extended/util"` | `import exutil "github.com/openshift/origin/test/extended/util"` |

Also update:
- `e2e "k8s.io/kubernetes/test/e2e/framework"` — keep as-is, origin uses the same import
- `g "github.com/onsi/ginkgo/v2"` and `o "github.com/onsi/gomega"` — keep as-is

#### 4b. Adapt Test Names

- Remove the `Author:xxx-` prefix from test names — origin does not use this convention
- Add an `[OTP]` tag to the test name to indicate the test was ported from openshift-tests-private
- Keep sig tags like `[sig-networking]` and feature tags like `[Feature:X]`
- Keep severity and variant tags if present (e.g., `ConnectedOnly`)
- Add `[apigroup:xxx]` tags if the test uses specific API groups and origin requires them

Example: `Author:huirwang-High-53223-Verify ACL audit logs work correctly` becomes `[OTP] Verify ACL audit logs work correctly`

#### 4c. Port Helper Functions and Utilities

When the test calls helper functions defined in the source package:

1. **First, look for an equivalent in origin's existing utilities** (`test/extended/util/`, or the destination package). Use the origin equivalent if one exists.
2. **If no equivalent exists and the helper is small** (< 50 lines), copy it to the destination package.
3. **If the helper is large or pulls in significant dependencies**, mark the test as `port=yes-but-difficult` in the source and pick another candidate test instead. Do not prompt interactively — the agent should make the call autonomously.

#### 4d. Port Test Data / Fixtures

If the test references fixture files (YAML manifests, JSON files, etc.):
- Copy them to the corresponding `testdata/` directory in the destination
- Update paths in the test code accordingly

#### 4e. Write the Test File

- If the destination package already has test files, add the new test to an existing file that covers the same feature area, or create a new file if the test covers a distinct feature
- Use the destination repo's package name (e.g., `package networking`)
- Include the ported `g.Describe` / `g.It` blocks with all necessary setup

### Step 5: Register New Packages

If you created any new packages under `test/extended/`, add blank imports to `test/extended/include.go`:

```go
_ "github.com/openshift/origin/test/extended/<new-package>"
```

### Step 6: Verify Compilation

Run in the destination repo:

```bash
cd /workspace/origin && go build ./test/extended/...
```

- If compilation fails, resolve missing imports or utility functions
- Iterate until compilation succeeds
- If stuck after 3 attempts on a specific test, mark it `port=yes-but-difficult` and move on

### Step 7: Create Origin PR

Create a branch and PR on `dgoodwin/origin`:

```bash
cd /workspace/origin
BRANCH="port-tests-$(date +%Y%m%d-%H%M%S)"
git checkout -b "$BRANCH"
git add -A
git commit -m "Port $COUNT tests from openshift-tests-private [OTP]

Ported tests:
- <list each test name and source file>

Co-Authored-By: Ambient Code Agent <noreply@ambient-code.com>"
git push origin "$BRANCH"
gh pr create --repo dgoodwin/origin \
  --title "Port $COUNT e2e tests from openshift-tests-private [OTP]" \
  --body "$(cat <<'EOF'
<!-- acp:session_id=$AGENTIC_SESSION_NAME source=#ISSUE_NUMBER last_action=$(date -u +%Y-%m-%dT%H:%M:%SZ) retry_count=0 -->

## Summary
- Ported $COUNT tests from openshift-tests-private
- Source branch: porting-prep
- All tests compile successfully

## Ported Tests
| Test | Source | Destination |
|------|--------|-------------|
| ... | ... | ... |

## Skipped Tests
| Test | Reason |
|------|--------|
| ... | ... |

---
Labels: ambient-code:managed
EOF
)"
```

Add the `ambient-code:managed` label to the PR.

### Step 8: Update Source Repo

For each successfully ported test, update the comment in `openshift-tests-private` from `port=yes` to `port=complete`.

Commit and push directly to the `porting-prep` branch on `dgoodwin/openshift-tests-private`:

```bash
cd /workspace/openshift-tests-private
git add -A
git commit -m "Mark ported tests as port=complete

Tests ported to openshift/origin PR: <PR_URL>"
git push origin porting-prep
```

Since `porting-prep` is the working branch on the fork, push directly — no PR needed for source annotation updates.

### Step 9: Report Results

Display a summary:

| Status | Test | Source | Destination |
|--------|------|--------|-------------|
| Ported | test description | `networking/egressfirewall.go:42` | `networking/egressfirewall_ported.go:15` |
| Skipped | test description | `storage/general_csi.go:100` | -- (large utility dependency) |

Include:
- Total tests ported vs skipped
- Any new packages created
- Compilation status
- Origin PR URL
- Source annotation PR URL
