---
name: write-a-prd
description: Create a PRD through user interview, codebase exploration, and module design, then create a milestone with the PRD as its description and break it into feature issues. Use when user wants to write a PRD, create a product requirements document, or plan a new feature.
---

This skill will be invoked when the user wants to create a PRD. You may skip steps if you don't consider them necessary.

## Step 0: Detect Git Platform

Run this routine once per session before any operation that calls `gh` or `glab`.

### Step 0.1: Check the cache

Look for a `## Claude Skills Config` section in the project's `CLAUDE.md` file (in the current working directory).

If the section exists and contains a `git-cli` line, read the values for `git-platform`, `git-cli`, and `git-remote-host` from it and **skip to Step 0.4** (authentication check).

### Step 0.2: Detect the remote host

Run:

```
git remote -v
```

Parse the output for the fetch remote URL. Extract the hostname:

- If the URL contains `github.com` → platform is `github`, CLI is `gh`, host is `github.com`
- If the URL contains `gitlab.com` → platform is `gitlab`, CLI is `glab`, host is `gitlab.com`
- If the URL contains any other domain (self-hosted) → go to **Step 0.3**
- If there are no remotes → stop and tell the user: "No git remote found. Add a remote and re-run."

### Step 0.3: Ask the user (unknown domain only)

If the hostname did not match `github.com` or `gitlab.com`, ask the user exactly once:

> I see remote `<host>` — is this GitHub or GitLab?

Wait for the user's answer, then set:

- GitHub → platform `github`, CLI `gh`
- GitLab → platform `gitlab`, CLI `glab`

### Step 0.4: Write the config block

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
- If `CLAUDE.md` already has a `## Claude Skills Config` section (i.e. you reached this step via Step 0.1's cache hit): do not modify the file.

### Step 0.5: Verify authentication

Run the appropriate auth check:

- GitHub: `gh auth status`
- GitLab: `glab auth status`

If the command exits with a non-zero status or prints an error indicating the user is not authenticated, stop immediately and tell the user:

> `<cli>` is not authenticated. Run `<cli> auth login` and re-run this skill.

If authentication is confirmed, continue with the calling skill.

---

1. Check the current conversation for a prior `/grill-me` session. If one exists, extract the problem description, decisions, constraints, risks, and open questions from it — do not re-ask anything already answered. Summarize what you found and confirm with the user before continuing. If no grill-me session exists, ask the user for a long, detailed description of the problem they want to solve and any potential ideas for solutions.

2. Explore the repo to verify their assertions and understand the current state of the codebase.

3. Interview the user about any aspects of the plan not already resolved in the grill-me session. Focus only on gaps: unanswered questions, unresolved trade-offs, or areas the grill-me didn't cover.

4. Sketch out the major modules you will need to build or modify to complete the implementation. Actively look for opportunities to extract deep modules that can be tested in isolation.

A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes.

Check with the user that these modules match their expectations. Check with the user which modules they want tests written for.

5. Once you have a complete understanding of the problem and solution, write the PRD using the template below and create a milestone with the PRD as its description.

   **GitHub:**
   ```
   gh api repos/:owner/:repo/milestones --method POST \
     --field title="<PRD title>" \
     --field description="<full PRD content>"
   ```

   **GitLab:** First derive the URL-encoded project path from `git remote -v` (e.g. `mygroup/myrepo` → `mygroup%2Fmyrepo`), then run:
   ```
   glab api projects/:fullpath/milestones --method POST \
     --field title="<PRD title>" \
     --field description="<full PRD content>"
   ```
   Save the `id` field from the response — you will need it when creating issues.

   The milestone IS the PRD. There is no separate PRD issue. Progress is tracked by open/closed issue count.

6. Break the PRD into **tracer bullet** issues — thin vertical slices that each cut through ALL integration layers end-to-end, not horizontal slices of one layer.

   Slices may be 'HITL' (requires human interaction) or 'AFK' (can be implemented and merged without human interaction). Prefer AFK over HITL where possible.

   Present the proposed breakdown to the user:
   - **Title**: short descriptive name
   - **Type**: HITL / AFK
   - **Blocked by**: which slices must complete first
   - **User stories covered**: which user stories this addresses

   Iterate until the user approves the breakdown.

7. Create an issue for each approved slice, in dependency order (blockers first) so you can reference real issue numbers. Assign every issue to the milestone.

   **GitHub:**
   ```
   gh issue create \
     --title "<issue title>" \
     --body "<issue body>" \
     --milestone "<PRD title>"
   ```

   **GitLab:**
   ```
   glab issue create \
     --title "<issue title>" \
     --description "<issue body>" \
     --milestone-id <milestone id>
   ```

<issue-template>
## What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Blocked by

- Blocked by #<issue-number> (if any)

Or "None - can start immediately" if no blockers.

## User stories addressed

Reference by number from the milestone description (PRD):

- User story 3
- User story 7

</issue-template>

8. Tell the user the milestone name — they'll pass it to `/ralph` when ready to implement.

<prd-template>
## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

This list should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

## Testing Decisions

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (i.e. similar types of tests in the codebase)

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature.

</prd-template>
