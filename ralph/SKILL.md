---
name: ralph
description: Autonomous development loop that implements all open issues in a milestone, one at a time, on feature branches with a test gate before merging each to a shared dev branch. Use when user wants to run ralph, start the dev loop, implement a milestone, autonomously work through a PRD's issues, or says "run ralph".
---

# Ralph — Autonomous Dev Loop

**Usage:** `/ralph <milestone-name>`

Ralph works through every open issue in a milestone, implementing each one on its own branch, running tests, then merging to `dev/<milestone>` — leaving you a clean branch to QA before merging to main.

---

## Prerequisites

### Detect Git Platform

Run this routine once per session before any operation that calls `gh` or `glab`.

#### Step 1: Check the cache

Look for a `## Claude Skills Config` section in the project's `CLAUDE.md` file (in the current working directory).

If the section exists and contains a `git-cli` line, read the values for `git-platform`, `git-cli`, and `git-remote-host` from it and **skip to Step 4** (authentication check).

#### Step 2: Detect the remote host

Run:

```
git remote -v
```

Parse the output for the fetch remote URL. Extract the hostname:

- If the URL contains `github.com` → platform is `github`, CLI is `gh`, host is `github.com`
- If the URL contains `gitlab.com` → platform is `gitlab`, CLI is `glab`, host is `gitlab.com`
- If the URL contains any other domain (self-hosted) → go to **Step 3**
- If there are no remotes → stop and tell the user: "No git remote found. Add a remote and re-run."

#### Step 3: Ask the user (unknown domain only)

If the hostname did not match `github.com` or `gitlab.com`, ask the user exactly once:

> I see remote `<host>` — is this GitHub or GitLab?

Wait for the user's answer, then set:

- GitHub → platform `github`, CLI `gh`
- GitLab → platform `gitlab`, CLI `glab`

#### Step 4: Write the config block

Write or append the following block to `CLAUDE.md` in the current working directory:

```
## Claude Skills Config
- **git-platform**: github | gitlab
- **git-cli**: gh | glab
- **git-remote-host**: <detected host>
```

Fill in the detected values (use exactly one of the two options shown for each field).

- If `CLAUDE.md` does not exist: create it containing only this section.
- If `CLAUDE.md` exists but does not have a `## Claude Skills Config` section: append this section after the existing content, preceded by a blank line.
- If `CLAUDE.md` already has a `## Claude Skills Config` section (i.e. you reached this step via Step 1's cache hit): do not modify the file.

#### Step 5: Verify authentication

Run the appropriate auth check:

- GitHub: `gh auth status`
- GitLab: `glab auth status`

If the command exits with a non-zero status or prints an error indicating the user is not authenticated, stop immediately and tell the user:

> `<cli>` is not authenticated. Run `<cli> auth login` and re-run this skill.

If authentication is confirmed, continue with the calling skill.

### Verify milestone exists

Confirm the milestone `<milestone-name>` exists and has open issues:

- GitHub: `gh issue list --milestone "<milestone-name>" --state open`
- GitLab: `glab issue list --milestone "<milestone-name>" --state opened`

If any check fails, stop and tell the user what's missing.

---

## Step 1: Setup

1. Check if branch `dev/<milestone-name>` exists locally or remotely.
   - If not: `git checkout main && git pull && git checkout -b dev/<milestone-name> && git push -u origin dev/<milestone-name>`
   - If yes: `git checkout dev/<milestone-name> && git pull`
2. Confirm the branch is tracking remote before continuing.

---

## Step 2: Pick the Next Issue

1. List all open issues in the milestone:
   - GitHub: `gh issue list --milestone "<milestone-name>" --state open --json number,title,body,labels`
   - GitLab: `glab issue list --milestone "<milestone-name>" --state opened`
2. If the list is empty → go to **Step 5: Summary**.
3. Select the highest-priority issue using your judgment:
   - Prefer foundational issues that others depend on
   - Prefer simpler issues that unblock complex ones
   - Avoid issues labeled `needs-human`
4. State which issue you picked and why before proceeding.

---

## Step 3: Implement via Subagent

Delegate each issue to a **fresh subagent** using the Agent tool. This gives every issue clean context and prevents the main loop from accumulating implementation detail.

Spawn a general-purpose subagent with this prompt (fill in the placeholders):

```
You are implementing issue #<number>: "<title>"

Issue body:
<full issue body>

Platform: <github|gitlab>
CLI: <gh|glab>
Repo: <absolute path to working directory>
Base branch: dev/<milestone-name>
Feature branch to create: feature/<number>-<kebab-title-slug>

Steps:
1. git checkout dev/<milestone-name> && git pull
2. git checkout -b feature/<number>-<kebab-title-slug>
3. Read the issue acceptance criteria carefully.
4. Explore relevant code (Grep/Glob/Read) before making changes.
5. Implement the feature. Write or update tests as needed.
6. Run the test suite. Detect the project type and use the appropriate command:
   - If `package.json` exists: check the `test` script (`npm test`, `yarn test`, or `pnpm test`)
   - If `pyproject.toml` or `setup.py` exists: `pytest`
   - If `Makefile` has a `test` target: `make test`
   - If `Cargo.toml` exists: `cargo test`
   - If `go.mod` exists: `go test ./...`
   - Otherwise: look for any test runner script in the repo root and run it
   If no test suite can be found, skip this step and note it in your report.
7. If tests pass:
   - git commit -m "feat: <title> (#<number>)"
   - git push -u origin feature/<number>-<slug>
   - If GitHub: `gh pr create --base dev/<milestone-name> --title "<title>" --body "Closes #<number>" --milestone "<milestone-name>"`
   - If GitLab: `glab mr create --target-branch dev/<milestone-name> --title "<title>" --description "Closes #<number>"`
   - If GitHub: `gh pr merge --squash --delete-branch`
   - If GitLab: `glab mr merge --squash --remove-source-branch`
   - If GitHub: `gh issue close <number>`
   - If GitLab: `glab issue close <number>`
   - Report: SUCCESS
8. If tests fail after three fix attempts:
   - Do NOT merge. Leave the branch as-is.
   - Report: FAILED — <brief explanation of what's failing and why>
```

Wait for the subagent to return before proceeding.

- **If SUCCESS**: return to **Step 2**
- **If FAILED**:
  - If GitHub: `gh issue comment <number> --body "needs human: <subagent failure explanation>"`
  - If GitLab: `glab issue note <number> --message "needs human: <subagent failure explanation>"`
  - If GitHub: `gh issue edit <number> --add-label "needs-human"`
  - If GitLab: `glab issue update <number> --label "needs-human"`
  - Return to **Step 2**

---

## Step 5: Summary

When no open issues remain (excluding `needs-human`), generate a QA test plan before opening the PR/MR:

- Run `git diff main...dev/<milestone-name> --name-only` to get the full list of changed files.
- For each changed file, derive one or more manual QA test steps that verify the externally observable behavior it affects (not implementation details).
- Group steps by feature/issue for readability.
- Format each step as a markdown checkbox.

Check if a QA PR/MR already exists:

- GitHub: `gh pr list --base main --head dev/<milestone-name> --state open --json number,url`
- GitLab: `glab mr list --target-branch main --source-branch dev/<milestone-name> --state opened`

If one exists, leave the PR/MR description untouched (preserving any checked QA boxes) and post a comment with only what's new in this run:
- List the newly implemented issues
- Run `git diff dev/<milestone-name>@{<timestamp of previous run>}...dev/<milestone-name> --name-only` to find files changed since the last run. If the previous run timestamp is unavailable, diff against the merge-base of the last feature branch merged.
- Generate a QA test plan addendum covering only those newly changed files

If GitHub:
```
gh pr comment <number> --body "## Re-run complete\n\n### Newly implemented\n<bulleted list of issues fixed in this run>\n\n### Needs human\n<updated needs-human list, or 'No change'>\n\n### QA test plan addendum\n\n<checkbox list for newly changed files only>"
```

If GitLab:
```
glab mr note <number> --message "## Re-run complete\n\n### Newly implemented\n<bulleted list of issues fixed in this run>\n\n### Needs human\n<updated needs-human list, or 'No change'>\n\n### QA test plan addendum\n\n<checkbox list for newly changed files only>"
```

If no PR/MR exists, open one:

If GitHub:
```
gh pr create \
  --base main \
  --head dev/<milestone-name> \
  --milestone "<milestone-name>" \
  --title "QA: <milestone-name>" \
  --body "## Summary\n\nThis branch contains all implemented issues for milestone **<milestone-name>**.\n\n### Implemented\n<bulleted list of ✓ issues>\n\n### Needs human\n<bulleted list of ✗ issues, or 'None'>\n\n## QA test plan\n\n<generated checkbox list of test steps grouped by feature>"
```

If GitLab:
```
glab mr create \
  --target-branch main \
  --source-branch dev/<milestone-name> \
  --title "QA: <milestone-name>" \
  --description "## Summary\n\nThis branch contains all implemented issues for milestone **<milestone-name>**.\n\n### Implemented\n<bulleted list of ✓ issues>\n\n### Needs human\n<bulleted list of ✗ issues, or 'None'>\n\n## QA test plan\n\n<generated checkbox list of test steps grouped by feature>"
```

Then print:

```
Ralph complete for milestone: <milestone-name>

Implemented:
  ✓ #<number> — <title>
  ...

Needs human:
  ✗ #<number> — <title>
  ...

QA PR/MR opened: <pr-mr-url>
Merge to main after QA is complete.
```

---

## Idempotency

If ralph is re-run on the same milestone:
- It skips already-closed issues automatically (they won't appear in `--state open` / `--state opened`)
- It picks up the dev branch where it left off
- Issues labeled `needs-human` are skipped in issue selection
