---
name: write-a-prd
description: Create a PRD through user interview, codebase exploration, and module design, then create a GitHub milestone with the PRD as its description and break it into feature issues. Use when user wants to write a PRD, create a product requirements document, or plan a new feature.
---

This skill will be invoked when the user wants to create a PRD. You may skip steps if you don't consider them necessary.

1. Check the current conversation for a prior `/grill-me` session. If one exists, extract the problem description, decisions, constraints, risks, and open questions from it — do not re-ask anything already answered. Summarize what you found and confirm with the user before continuing. If no grill-me session exists, ask the user for a long, detailed description of the problem they want to solve and any potential ideas for solutions.

2. Explore the repo to verify their assertions and understand the current state of the codebase.

3. Interview the user about any aspects of the plan not already resolved in the grill-me session. Focus only on gaps: unanswered questions, unresolved trade-offs, or areas the grill-me didn't cover.

4. Sketch out the major modules you will need to build or modify to complete the implementation. Actively look for opportunities to extract deep modules that can be tested in isolation.

A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes.

Check with the user that these modules match their expectations. Check with the user which modules they want tests written for.

5. Once you have a complete understanding of the problem and solution, write the PRD using the template below and create a GitHub milestone with the PRD as its description:
   ```
   gh api repos/:owner/:repo/milestones --method POST \
     --field title="<PRD title>" \
     --field description="<full PRD content>"
   ```
   The milestone IS the PRD. There is no separate PRD issue. Progress is tracked by open/closed issue count.

6. Break the PRD into **tracer bullet** issues — thin vertical slices that each cut through ALL integration layers end-to-end, not horizontal slices of one layer.

   Slices may be 'HITL' (requires human interaction) or 'AFK' (can be implemented and merged without human interaction). Prefer AFK over HITL where possible.

   Present the proposed breakdown to the user:
   - **Title**: short descriptive name
   - **Type**: HITL / AFK
   - **Blocked by**: which slices must complete first
   - **User stories covered**: which user stories this addresses

   Iterate until the user approves the breakdown.

7. Create a GitHub issue for each approved slice using `gh issue create`, in dependency order (blockers first) so you can reference real issue numbers. Assign every issue to the milestone.

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
