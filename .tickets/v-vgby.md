---
id: v-vgby
status: open
deps: []
links: []
created: 2026-04-07T01:38:37Z
type: task
priority: 3
assignee: Jibles
---
# Vulti Agent App — UI Polish Pass

Collection of small UI polish fixes for the Vulti Agent mobile app.

## 1. Credits badge (star + 500) is non-interactive
**Location:** `vultiagent-app/src/features/agent/components/AgentHeader.tsx:37-54`
**Issue:** The sparkle icon + "500" credits badge in the top-right header is wrapped in a plain `View` — clicking it does nothing. The value appears hardcoded. Users will tap it expecting something to happen.
**Fix:** Either wire it to a credits/usage screen or remove it until that feature is ready.

## 2. New chat button placement is confusing
**Location:** `vultiagent-app/src/features/agent/components/SidePanel.tsx:208-214`
**Issue:** The blue `+` AddButton sits tight against the vault selector chip (gap: 10px). It reads as "add vault" rather than "new chat." Compare to Claude app where the new-chat icon (pencil/compose) is visually separated from navigation controls.
**Fix:** Change icon from `+` to a compose/pencil icon (matching Claude app convention). Increase spacing between vault selector and new-chat button so they don't look related.

## 3. No-vault empty state is unclear
**Location:** `vultiagent-app/src/features/agent/components/AgentEmptyState.tsx`
**Issue:** When no vault is attached, the screen shows "Welcome to Vulti Agent" which is generic. There's no clear indicator that the user needs to connect a vault — just a subtitle buried below.
**Fix:** Show a distinct empty state with a vault icon, clear "No vault connected" message, and a prominent "Connect Vault" / "Create Vault" button.
