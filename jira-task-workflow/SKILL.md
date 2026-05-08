---
name: jira-task-workflow
description: >-
  Guides end-to-end delivery starting from an assigned Jira issue: load issue
  context via Atlassian MCP, clarify requirements when the ticket is ambiguous,
  determine target repository from Jira labels, align scope and acceptance
  criteria, branch and implement in that repo, verify, open PR, update Jira
  status and comments. If labels do not identify exactly one repo, the agent
  must ask the user every time. Use when building or fixing work that begins
  from a Jira ticket, issue key (e.g. PROJ-123), Jira URL, or phrases like
  "จาก Jira", "งาน SD-", "เริ่มจาก task บน Jira".
disable-model-invocation: false
---

# Jira-driven development workflow

## Prerequisites

- Atlassian MCP enabled and authenticated for the target site.
- Repo cloned locally; git remotes and CI expectations match team practice.

## Phase 0 — Resolve site and issue

1. Call `getAccessibleAtlassianResources`. If the user gave a hostname (e.g. `origin-prop.atlassian.net`), use it as `cloudId` when resolving that site; otherwise use the resource `id` UUID from the response.
2. Determine the issue identifier:
   - If the user pasted a Jira URL: extract the issue key (`/browse/KEY`).
   - If only a fuzzy title: use `search` (Rovo) with a short natural-language query **or** use `searchJiraIssuesUsingJql` with explicit JQL once the user confirms project or key pattern.
3. Load full context with `getJiraIssue` (`cloudId`, `issueIdOrKey`). Request fields the team relies on (minimum: `summary`, `description`, `status`, `issuetype`, `priority`, `assignee`, **`labels`**, `components`, `fixVersions`, parent/epic link fields if used, and any custom “acceptance criteria” field your site has—add by key if known).

## Phase 1 — Intake and alignment

1. Summarize back to the user: goal, current status, assignment, blockers from description/comments if present.
2. **Requirement clarity gate:** Decide whether the ticket is **clear enough to implement without guessing** (verifiable acceptance, scope boundaries, obvious edge cases or explicit deferrals). If **not**, run **Requirement gathering (when unclear)** below **before** repository selection or any code. Do **not** open Phase 2 or Phase 3 until requirements are settled **or** the user explicitly approves a written restatement of scope (“proceed as described below”) acknowledging remaining risk.
3. Resolve the **target repository** using **Labels** only (see “Repository selection”). Never infer repo from summary, title, or description alone.
4. List **assumptions**, **open questions**, and a **Definition of Done** (tests, docs, feature flags, rollout) before writing code. For topics other than repo selection, stop and ask only when delivery is blocked by missing product or access decisions.
5. If the issue is a parent with sub-tasks, confirm whether this session covers the parent only, one sub-task, or the full breakdown.

## Requirement gathering (when unclear)

Use this flow whenever Phase 1 step 2 finds missing, vague, or conflicting requirements.

**Typical triggers:** empty or placeholder description; summary-only ticket; **no** testable acceptance criteria / Definition of Done; conflicting details between fields, Epic, or comments; undisclosed dependencies (flags, migrations, deadlines); ambiguous UX or API contracts.

**Workflow:**

1. **Mine existing sources first** — Re-check `summary`, **description**, custom AC fields, **labels**, parent/epic links, fix version. Re-fetch `getJiraIssue` after any teammate update. Follow links to specs in the description **or** use `search` (Rovo) / Confluence tools to find linked pages when a URL or page title appears.
2. **Compile gaps** — List what is unknown in **short bullets** (not prose essays). Separate **blocking** uncertainties (would change architecture or rework) from **minor** polish.
3. **Request answers in one shot** — Ask the **user** (acting as assignee/Product owner liaison) **numbered questions** covering all blocking gaps **in a single message** when possible; offer multiple-choice **only** when options are obvious.
4. **Restate and lock scope** — After replies, summarize **accepted scope**, **explicit out-of-scope** items, and **assumptions** in the chat (and optionally add a concise `addCommentToJiraIssue` with the agreement so reviewers see it later).
5. **Exit gate** — Continue to **Repository selection** only when blocking gaps are resolved **or** the user explicitly accepts a documented “best-effort” scope in writing. If the ticket still cannot be implemented without guesswork, stop and ask the user to update Jira (description/AC) before coding.

## Repository selection (Labels)

**Convention:** Jira **Labels** are the source of truth for which repository (or service/repo slug) the issue belongs to. The team defines label values that map 1:1 to a clone path or Git remote (e.g. one label per repo name).

1. Read `fields.labels` from the issue. Map those labels to the single intended repo using the team’s agreed label→repo mapping.
2. **Ask the user every time** (do not guess or default) when any of the following is true:
   - There are **no** labels, or none that map to a repository in the agreed set.
   - **More than one** label maps to a different repo, and it is unclear which is in scope for this task.
   - The current Cursor workspace / open folder is **not** the confirmed repo: ask whether to switch roots, open another folder, or paste the path—**do not** run `git` in the wrong tree.
3. After the user confirms the repo (either unambiguously via labels or explicitly in reply), repeat the confirmed choice in one short line and proceed. If the user fixes labels in Jira, re-read the issue before Phase 2.

## Phase 2 — Workspace and branch

1. Operate only in the **repository confirmed in Phase 1**. Ensure the working tree is clean or intentionally stashed.
2. Sync default branch (`git fetch`, checkout, pull as per project convention).
3. Name the branch **only** the issue key (e.g. `SD-1234`), with no prefix or summary slug unless the user or repository conventions explicitly override this.

## Phase 3 — Implement and verify

1. **Plan before coding:** Produce a short, explicit plan first (approach, files or modules to touch, tests to add or extend, risks and edge cases). Share it with the user when the change is non-trivial or when multiple approaches exist; otherwise keep the plan internal but still complete this step before edits.
2. **Implement:** Work in small commits. Prefer commit messages that reference the key, e.g. `SD-3251: add resident list filter`.
3. **Verify after the work is done (required every time):** Before opening a PR, run the project’s appropriate checks—**always** run at least one of: automated **tests** (unit/integration/e2e as the repo uses) **or** a **build/compile** (e.g. `npm run build`, `tsc --noEmit`, framework build). Use the narrowest command that still proves the change; if both tests and build are standard for the repo, prefer running what CI enforces. Fix failures before Phase 4.
4. Follow project rules (TypeScript strictness, lint, formatting). Do not broaden scope beyond the issue without explicit user agreement; file or link follow-up tickets for discovered extra work.

## Phase 4 — Pull request

1. Push the branch and open a PR against the correct target branch.
2. PR **title**: keep it short and aligned with the Jira summary (often `KEY: summary` if the team uses that pattern).
3. PR **body**: use **exactly** these top-level sections, in order, with clear content under each:

```markdown
## Summary

[Brief purpose and outcome of the PR — what problem this solves or what capability it adds]

## Ticket/Issue Link

[Jira browse URL or issue key line, e.g. `SD-1234`; include parent/epic link only if relevant]

## Changes

[Bullet list or short paragraphs — files/areas touched, behavioral changes, config/flags, migrations]

## How to Test

[Steps for reviewers or QA — commands, URLs, credentials scope (no secrets), data setup, edge cases checked]
```

4. Paste the **PR URL** into the Phase 5 Jira comment.

## Phase 5 — Update Jira

1. Add a comment on the issue with `addCommentToJiraIssue`: concise summary of the change, PR link, deployment notes if any, and what is **not** included.
2. Transition status only if it matches team rules: call `getTransitionsForJiraIssue`, pick the allowed transition (e.g. *In Progress* when starting, *In Review* or *Done* when merging per policy), then `transitionJiraIssue`. Do not force a “Done” transition before merge if the team waits on main.

## Phase 6 — Close the loop

After merge (or when pipeline is green, per team practice), either add a closing comment with merge commit or revision, or transition to the terminal state the team uses.

## Escalation checklist

- Issue not found or 403: verify `cloudId`, site, and that the logged-in account matches the user's Jira identity.
- Missing or ambiguous **labels** for repo: **always** ask the user which repository to use before branching; do not infer from other fields.
- **Unclear requirements:** follow **Requirement gathering (when unclear)**; do not implement from title alone or fill gaps with silent assumptions.
- Empty description: treat as “unclear” unless comments or linked specs fully define scope; still run the requirement flow as needed.
- Conflicting fields (version, component): ask one focused question, default to leaving fields unchanged over guessing.
