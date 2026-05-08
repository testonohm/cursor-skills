---
name: jira-task-workflow
description: >-
  Guides end-to-end delivery starting from an assigned Jira issue: load issue
  context via Atlassian MCP, clarify ambiguous requirements, establish root
  cause for bugs before coding, resolve repo from Labels, implement locally
  without committing until the user finishes manual testing, then—only on
  explicit user request—assist with git actions if asked.
  The user typically commits, pushes, and opens the PR themselves; once a PR
  URL exists, the agent must update Jira automatically (status transitions and
  activity via comments). If labels do not identify exactly one repo, the agent
  must ask the user every time. Use for Jira-linked work, keys (e.g. PROJ-123),
  URLs, or phrases like "จาก Jira", "งาน SD-", "เริ่มจาก task บน Jira".
disable-model-invocation: false
---

# Jira-driven development workflow

## Prerequisites

- Atlassian MCP enabled and authenticated for the target site.
- Repo cloned locally; git remotes and CI expectations match team practice.

## Git and PR ownership (mandatory)

1. **No commit or push from the agent** until the user states that **their own manual testing** is done and **nothing is blocking**. Treat “ready to commit” as an **explicit user signal**—not an assumption after automated tests pass.
2. **The user owns committing, pushing, and opening the PR.** Default: the user runs `git commit`, `git push`, and creates the PR on the host (GitHub/GitLab/Bitbucket). The agent runs those commands **only** if the user **explicitly asks** the agent to run them, and only **after** manual testing is confirmed—never proactively.
3. **Handoff for QA:** After implementation and automated verify (Phase 3), stop and let the user run through their manual checks. Provide a short checklist of what changed and how they can exercise it; do not commit while waiting.
4. **After a PR exists**, as soon as the user provides the **PR URL** (or confirms the PR is open and shares the link), **immediately** run **Phase 5 — Jira sync after PR** without waiting for extra prompts: update **status** and **activity** (see Phase 5).

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
6. **Root cause before implementation (bugs, flaky UI, platform-specific weirdness):** Before editing code for bugs or regressions, **separate symptom from root cause** and document a short hypothesis in the plan (or in chat for trivial tickets).
   - **Map the system:** List subsystems that touch the failing flow (e.g. two modal libraries on one screen, nested navigators, multiple providers, native vs JS bridges). Read library docs or source for **lifecycle/stack** behavior—not only “this prop fixed a demo once.”
   - **Avoid tactical whack-a-mole:** If the first patch does not stick or QA reports the same class of bug after a second patch (stacking, wrong z-order, duplicate overlays, “works until step X”), **stop adding dismiss/timing/stack tweaks** until you either have evidence for the root cause or a deliberate minimal repro. Repeated surface fixes often mask an **incompatible combination** (e.g. Gorhom `BottomSheetModal` + `react-native-modal` on one screen).
   - **Prefer eliminating the class of bug** when the root cause is architectural: unify on one modal/stack contract, one navigator pattern, or one provider boundary, instead of chaining behavioral workarounds that break the next edge case.
   - **Optional handoff:** After you land the fix, a one-line Jira comment with **root cause** (not only “dismissAll + delay”) helps the next person avoid the same loop.

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

1. **Plan before coding:** Produce a short, explicit plan first (approach, files or modules to touch, tests to add or extend, risks and edge cases). For **bugs / UI regressions**, the plan must include the **root-cause hypothesis** (Phase 1 step 6)—not only the symptom or a list of props to try. Share it with the user when the change is non-trivial or when multiple approaches exist; otherwise keep the plan internal but still complete this step before edits.
2. **Implement (local only until user approves):** Apply code changes on the working branch. **Do not** `git commit` or `git push` per **Git and PR ownership**—keep work uncommitted unless the user explicitly orders the agent to commit after manual testing passes.
3. **Automated verify (required every time before handoff):** Run the project’s appropriate checks—**always** run at least one of: automated **tests** (unit/integration/e2e as the repo uses) **or** a **build/compile** (e.g. `npm run build`, `tsc --noEmit`, framework build). Use the narrowest command that still proves the change; if both are standard, prefer what CI enforces. Fix failures before asking the user to test manually.
4. **Manual testing gate:** Give the user a concise **what to try** list. **Stop** until they confirm manual testing passed with no blockers. Only then may they (or the agent on direct request) commit/push/open PR.
5. Follow project rules (TypeScript strictness, lint, formatting). Do not broaden scope beyond the issue without explicit user agreement; file or link follow-up tickets for discovered extra work.

## Phase 4 — Pull request (user-led)

1. **User-driven git and PR:** The user runs `git commit` / `git push` / open PR—or explicitly asks the agent to run specific git commands **after** the manual-testing gate (Phase 3). The agent **does not** open the PR unless the user explicitly asks the agent to do so.
2. **Draft PR copy for the user:** Provide suggested **PR title** (short; align with Jira summary, often `KEY: summary`) and **PR body** the user can paste into the forge. Body must use **exactly** these top-level sections, in order:

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

3. **Trigger for Jira:** When the user shares the **PR URL**, treat that as the handoff to **Phase 5** and run it in the same turn without asking for extra confirmation.

## Phase 5 — Jira sync after PR (automatic)

Run **immediately** when the **PR URL** is known (user pasted it or confirmed the PR is open).

1. **Activity:** Add a Jira **comment** via `addCommentToJiraIssue`: PR link, one-line summary of the change, target branch if useful, what is **out of scope**, and anything reviewers should know. This is the primary **activity** feed update unless the site also expects a worklog (only add `addWorklogToJiraIssue` if team policy requires time on the ticket).
2. **Status:** Call `getTransitionsForJiraIssue`, then `transitionJiraIssue` to the appropriate post-PR state for this team (commonly *In Review* / *Code Review* when the PR exists). **Do not** move to a terminal *Done* state before merge if team policy ties *Done* to main. If the only valid next status is already the current one, skip transition and mention that in the comment.
3. If transition fails (permission or workflow), still post the comment and report the error to the user with the available transitions.

## Phase 6 — Close the loop

When the **user** reports merge or asks for a post-merge Jira update: add a brief comment (merge SHA or release note) and transition per team rules (e.g. *Done* after main). Do not commit, push, or merge on the user’s behalf unless explicitly asked.

## Escalation checklist

- Issue not found or 403: verify `cloudId`, site, and that the logged-in account matches the user's Jira identity.
- Missing or ambiguous **labels** for repo: **always** ask the user which repository to use before branching; do not infer from other fields.
- **Unclear requirements:** follow **Requirement gathering (when unclear)**; do not implement from title alone or fill gaps with silent assumptions.
- Empty description: treat as “unclear” unless comments or linked specs fully define scope; still run the requirement flow as needed.
- Conflicting fields (version, component): ask one focused question, default to leaving fields unchanged over guessing.
- **Git / PR:** Do not `git commit`, `git push`, or open/create a PR unless manual testing is confirmed and the user **explicitly asks the agent** to run those steps. If the user opens the PR themselves, run **Phase 5** as soon as they provide the PR URL.
- **Same bug class after multiple tactical fixes:** Re-run Phase 1 step 6; assume an **architectural or lifecycle** conflict until disproven. Do not stack timing/dismiss/ordering patches without a stated root-cause theory.
