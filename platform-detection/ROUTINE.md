# Platform Detection Routine

This is a shared instruction block embedded by `write-a-prd` and `ralph`. Copy it verbatim into the appropriate location in each skill file.

---

## Detect Git Platform

Run this routine once per session before any operation that calls `gh` or `glab`.

### Step 1: Check the cache

Look for a `## Claude Skills Config` section in the project's `CLAUDE.md` file (in the current working directory).

If the section exists and contains a `git-cli` line, read the values for `git-platform`, `git-cli`, and `git-remote-host` from it and **skip to Step 4** (authentication check).

### Step 2: Detect the remote host

Run:

```
git remote -v
```

Parse the output for the fetch remote URL. Extract the hostname:

- If the URL contains `github.com` → platform is `github`, CLI is `gh`, host is `github.com`
- If the URL contains `gitlab.com` → platform is `gitlab`, CLI is `glab`, host is `gitlab.com`
- If the URL contains any other domain (self-hosted) → go to **Step 3**
- If there are no remotes → stop and tell the user: "No git remote found. Add a remote and re-run."

### Step 3: Ask the user (unknown domain only)

If the hostname did not match `github.com` or `gitlab.com`, ask the user exactly once:

> I see remote `<host>` — is this GitHub or GitLab?

Wait for the user's answer, then set:

- GitHub → platform `github`, CLI `gh`
- GitLab → platform `gitlab`, CLI `glab`

### Step 4: Write the config block

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

### Step 5: Verify authentication

Run the appropriate auth check:

- GitHub: `gh auth status`
- GitLab: `glab auth status`

If the command exits with a non-zero status or prints an error indicating the user is not authenticated, stop immediately and tell the user:

> `<cli>` is not authenticated. Run `<cli> auth login` and re-run this skill.

If authentication is confirmed, continue with the calling skill.
