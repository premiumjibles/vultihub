---
name: discord-digest
description: >
  Generates a daily engineering activity digest across all Vultisig repos
  and posts it to Discord via webhook. Surfaces per-engineer commits, PRs,
  issue activity, review bottlenecks, CI failures, and security-sensitive changes.
argument-hint: "[discord-webhook-url]"
---

## Step 1: Gather activity from each repo

Repos live under `/home/sean/Repos/vultisig/`. If a directory doesn't exist locally, skip it and note at the bottom.

Repo map (local-name=github):
- agent-backend=vultisig/agent-backend
- mcp=vultisig/mcp
- station=stationmoney/station
- vultiagent-app=vultisig/vultiagent-app
- vultiagent-cli=premiumjibles/vultiagent-cli
- vultiserver=vultisig/vultiserver
- vultisig-android=vultisig/vultisig-android
- vultisig-ios=vultisig/vultisig-ios
- vultisig-sdk=vultisig/vultisig-sdk
- vultisig-windows=vultisig/vultisig-windows

**Tracked engineers:** Jibles, NeOMakinG, gomes, Apotheosis. Only include activity from these engineers. Other contributors' work appears only if it surfaces in Needs Attention (e.g., stale PR assigned to a tracked engineer for review).

For each repo, run in parallel:

1. `git -C <path> fetch --all --quiet`
2. `git -C <path> log --all --since="24 hours ago" --format="%h|%an|%ae|%s|%ar" --no-merges`
3. `gh pr list --repo <github> --state all --json number,title,author,state,createdAt,mergedAt,updatedAt,reviewRequests,reviews --limit 20` — filter to PRs updated in last 24h
4. `gh run list --repo <github> --limit 3 --json status,conclusion,headBranch,event` — CI on default branch. If no runs returned or auth fails, omit CI from Needs Attention silently
5. `gh issue list --repo <github> --state all --json number,title,author,assignees,state,createdAt,updatedAt,labels --limit 20` — filter to issues created, updated, assigned, or closed in last 24h

If `gh` returns an auth error, skip API data for that repo and rely on git-only. If `git fetch` fails, use local state and note at the bottom.

**Name normalization:** Deduplicate engineers by email (`%ae`) and GitHub username. Match against the tracked engineers list — some engineers use different display names across git and GitHub (e.g., "JP" / "gomes" / "Jean-Pierre"). Use the canonical name from the tracked list.

**Timezone:** UTC for the date header and the 24-hour window. Abbreviate relative dates: 5d, 2mo, 1y.

## Step 2: Build the digest

Four sections: engineer activity, issue activity, needs attention, quiet repos.

**Character budget:** Target 1800 chars (Discord webhook max is 2000).

**Discord formatting:** Use `**bold**` and `•` bullets. Discord webhook `content` does NOT render markdown links — use bare URLs wrapped in `<>` for auto-linking: `PR #174 <https://github.com/...>`.

<example>
📊 **Vultisig Daily Digest — 2026-04-02 (UTC)**

👤 **gomes** — 12 commits across agent-backend, mcp
• SUI/TON tool labels, deposit scanner migration
• Scheduler MVP merged
PRs: 3 merged, 2 open

👤 **NeOMakinG** — 8 commits in vultisig-sdk
• Pluggable MpcEngine for React Native, audit refactors
PRs: 2 merged

🎫 **Issues**
• #11 Team Infra & Process Setup — updated (vultihub, Jibles)
• #15 MPC Native Integration — opened, assigned to NeOMakinG (vultisig-sdk)
• #8 Deposit Scanner v2 — closed (agent-backend, gomes)

⚠️ **Needs Attention**
• 🔐 PR #174 vultisig-sdk (Ehsan) — PSBT signing flow <https://github.com/vultisig/vultisig-sdk/pull/174>
• PR #42 agent-backend — open 3d, no reviewers <https://github.com/vultisig/agent-backend/pull/42>

💤 No activity: station (last: 2mo), vultiserver (last: 5d)
</example>

<example>
📊 **Vultisig Daily Digest — 2026-03-28 (UTC)**

👤 **Jibles** — 5 commits in vultiagent-cli
• Biome linter setup, CI workflow
PRs: 2 open

🎫 **Issues**
• #3 Model Research — updated (vultihub, Jibles)

⚠️ **Needs Attention**
• ❌ CI failing on main: vultisig-sdk (2 runs failed)
• 🔐 PR #176 vultisig-sdk — MpcEngine, touches mpc/dkls <https://github.com/vultisig/vultisig-sdk/pull/176>

💤 No activity: station (last: 2mo), mcp (last: 1d), vultiagent-cli (last: 3d)
</example>

### Engineer activity (👤)

- Only tracked engineers. Group by engineer, not repo. 2-4 lines each
- Summarize related commits into one line
- Header: commit count + repo spread. PR line: merged/open counts
- Only name specific PRs if notable

### Issue activity (🎫)

- Issues created, closed, reopened, or assigned to a tracked engineer in the last 24h
- One line per issue: number, title, action (opened/closed/updated/assigned), repo, assignee
- Skip issues with only bot/automated updates

### Needs Attention (⚠️)

Surface up to 3 items, prioritized by risk then staleness. Always include all security-flagged PRs even if this exceeds 3.

1. **Security-sensitive PRs** 🔐 — Flag if PR diff touches files matching `**/sign*`, `**/keygen*`, `**/mpc*`, `**/derivation*`, `**/crypto*`
2. **CI failures** ❌ — Failed runs on default branch from `gh run list`
3. **Stale PRs** — Open >2 days AND no review activity in the last 24h

Include a bare URL for each flagged PR: `<https://github.com/<org>/<repo>/pull/N>`

### Quiet repos (💤)

Single line listing repos with zero activity. For each, include last commit date via `git -C <path> log -1 --format="%ar"`.

## Step 3: Post to Discord

If `$ARGUMENTS` contains a webhook URL, POST the digest as `{"content":"<digest>"}`. Report non-204 status codes. If no URL, print the digest to stdout.

Generate the daily digest now and post it to `$ARGUMENTS` (or stdout if no URL).
