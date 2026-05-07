---
name: scan-carryover-escalation
description: Protocol for escalating persistent unresolved findings from recurring scans (Opportunity Radar, Knowledge Gap Detector, or any monitoring routine) into first-class Paperclip issues. Use this skill whenever you complete a scan cycle and are about to log carry-over findings — invoke it before writing your completion comment to decide whether to flag, escalate, or skip. Prevents high-value findings from being indefinitely deferred by turning repeated observations into accountable action items.
---

# Scan Carry-Over Escalation Protocol

When you run a recurring scan (Opportunity Radar, Knowledge Gap Detector, or any monitoring routine), some findings carry over from previous cycles without being acted on. This protocol tells you exactly what to do with each carry-over finding so it never disappears into an endless log.

The core insight: repeated logging of the same finding without escalation means the scan is working but the feedback loop is broken. Your job is to break that pattern.

## Carry-Over Counter: The Source of Truth

Every scan issue must maintain a `carryover-log` document. This is how the counter survives heartbeat boundaries.

**On each cycle, before writing your completion comment:**

1. Read the existing `carryover-log` document: `GET /api/issues/{scanIssueId}/documents/carryover-log`
2. If it doesn't exist yet, treat all current findings as cycle 1
3. Update the log with the current cycle's findings and increment counters
4. Use the updated counters to apply the escalation rules below

### Document structure

```json
{
  "findings": {
    "<finding-key>": {
      "description": "short human-readable description",
      "weekFirstIdentified": "YYYY-MM-DD",
      "cycleCount": 3,
      "escalatedIssueId": null
    }
  },
  "lastUpdated": "YYYY-MM-DD"
}
```

Use a stable, lowercase, hyphenated string as the finding key (e.g., `meeting-action-item-bridge`, `onboarding-doc-gap`). The key must be consistent across cycles so counters accumulate correctly.

**Write the updated document back:** `PUT /api/issues/{scanIssueId}/documents/carryover-log`

---

## Escalation Rules by Cycle Count

### Cycles 1–2: Log only

Include the finding in your scan report. Note the carry-over count and the date first identified. No special action needed.

```
- Meeting Action Item Execution Bridge (carry-over: week 1, 2 cycles) — no implementation observed
```

### Cycle 3: Flag as escalation candidate

Add a `[ESCALATION CANDIDATE]` marker to the finding in your report. In your completion comment, warn that if this finding is still unresolved next cycle, it will automatically become a Paperclip issue.

```
- [ESCALATION CANDIDATE] Meeting Action Item Execution Bridge (3 cycles unresolved, first seen YYYY-MM-DD)
  → Will auto-create a Paperclip issue if unresolved in cycle 4
```

### Cycle 4+: Auto-create a Paperclip issue (if one doesn't already exist)

Before creating an issue, check whether one was already created for this finding:

1. If `escalatedIssueId` in the `carryover-log` is non-null, fetch that issue: `GET /api/issues/{escalatedIssueId}`
2. If the issue exists and its status is `in_progress` or `blocked`, **skip creation** — work is already underway
3. If the issue doesn't exist yet (or is `done`/`cancelled`), create a new one:

```bash
POST /api/companies/{companyId}/issues
{
  "title": "[Persistent Finding] <description> (N cycles unresolved)",
  "description": "This finding has appeared in recurring scans for N consecutive cycles without resolution.\n\n**First identified:** YYYY-MM-DD\n**Source scan:** [<scan-issue-identifier>](/<prefix>/issues/<scan-issue-identifier>)\n\n## Finding\n\n<full finding description from the scan report>\n\n## Why this matters\n\n<rationale from the scan>",
  "goalId": "<same goalId as the parent scan issue>",
  "assigneeAgentId": "<your manager's agent ID from chainOfCommand>",
  "parentId": "<scan issue ID>",
  "priority": "medium",
  "status": "todo"
}
```

After creating the issue, update `escalatedIssueId` in the `carryover-log` document.

**What to include in your scan completion comment:**

```
- [AUTO-ESCALATED] Meeting Action Item Execution Bridge (4 cycles unresolved) → created [TEC-XXXX](/TEC/issues/TEC-XXXX)
```

---

## Getting Your Manager's Agent ID

Use `GET /api/agents/me` — the `chainOfCommand` array lists your reporting chain. The first entry is your direct manager. Use that agent's ID as `assigneeAgentId` for escalated issues.

---

## Removing a Finding from the Counter

When a finding has been resolved (implemented, accepted as won't-fix, or superseded), remove it from the `carryover-log` document. Don't leave stale entries accumulating forever — a resolved finding should not re-trigger escalation in a future cycle.

---

## Quick Reference

| Cycle count | Action |
|-------------|--------|
| 1–2 | Log in report with carry-over count and date |
| 3 | Add `[ESCALATION CANDIDATE]` flag + warn in completion comment |
| 4+ | Auto-create Paperclip issue (unless one is already `in_progress`/`blocked`) |
| Any | Remove from log once resolved |
