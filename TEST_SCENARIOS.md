# Test Scenarios — Document Classification App

## 1. Document Status

Each document derives its status from `doc.lastTest`:

| Scenario | Condition | Status displayed |
|---|---|---|
| SC-DOC-1 | No test run yet (`lastTest = null`) | `Pending` (gray) |
| SC-DOC-2 | Tested, `match = true` | `Tested` (green) |
| SC-DOC-3 | Tested, `match = false` | `Failed` (red) |
| SC-DOC-4 | Tested, `match = null` (no expected label set) | `No Label` (blue) |

> `match = null` means the document was classified but cannot be validated — no expected label was set. It is NOT a failure.

---

## 2. Run Test Bar — Button States

All cases are in `LobPromptSection` (Global/L1 mode) and `Workspace` (L2 mode).

### 2a. No files selected

| Scenario | Condition | Behavior |
|---|---|---|
| SC-RTB-1 | No files in file selector | "Run test" button disabled; `FileSelectorAnnotation` annotation shown pointing to "Select file" |

### 2b. Files selected, no test run yet

| Scenario | Condition | Behavior |
|---|---|---|
| SC-RTB-2 | Files selected, no lastTest | "Run test" enabled; no accuracy shown; no approve/deploy button |

### 2c. Test running

| Scenario | Condition | Behavior |
|---|---|---|
| SC-RTB-3 | `runningIds.size > 0` | "Run test" button shows "Running…" and is disabled; approve/deploy buttons hidden |

### 2d. After test — accuracy results

| Scenario | Accuracy | Button shown |
|---|---|---|
| SC-RTB-4 | `accuracy = null` (all docs have no expected label) | No approve button shown |
| SC-RTB-5 | `accuracy = 0` (all validated docs failed) | No approve button; no deploy button — only "Re-run" |
| SC-RTB-6 | `accuracy > 0`, prompt not stale, not yet approved | "Approve for Production" (orange) |
| SC-RTB-7 | `accuracy > 0`, prompt stale (edited after test) | No approve button; stale warning shown |

### 2e. After approval

| Scenario | Condition | Button shown |
|---|---|---|
| SC-RTB-8 | `approvedSnap.text === currentPrompt`, not yet deployed | "Deploy to Production" (green) |
| SC-RTB-9 | Prompt edited after approval (stale) | "Approve for Production" again (orange) — re-approve required |

### 2f. After deploy (Active)

| Scenario | Condition | Shown |
|---|---|---|
| SC-RTB-10 | `publishedSnap.text === currentPrompt` | "✓ Active · {accuracy}% · {date}" badge (no action buttons) |
| SC-RTB-11 | Prompt edited after active | "Approve for Production" shown again (re-approval flow resets) |

---

## 3. Prompt Workflow — Full State Machine

```
[Draft]
  → run test (accuracy > 0, not stale)
    → click "Approve for Production" → modal
      → confirm → [Approved]
        → click "Deploy to Production"
          → [Deploying] (1.5s animation)
            → [Active]
```

| Scenario | Step | `tabMeta.status` | Badge color |
|---|---|---|---|
| SC-WF-1 | Initial / before test | `Draft` | Gray |
| SC-WF-2 | After approve | `Approved` | Orange |
| SC-WF-3 | During deploy (1.5s) | `Deploying` | Blue (pulsing) |
| SC-WF-4 | After deploy complete | `Active` | Green |

### Header badge updates (LobPromptSection / Workspace header)

- `Approved` orange badge appears next to version pill when `approvedSnap.text === currentPrompt && publishedSnap.text !== currentPrompt`
- `tabMeta.status` drives the `workspace__status` badge class — all four states are covered

---

## 4. Approve Modal

| Scenario | Accuracy | Modal warning | Confirm button label |
|---|---|---|---|
| SC-MOD-1 | >= 80% | None | "Approve →" |
| SC-MOD-2 | 60–79% | "Accuracy below 80% — consider refining…" | "Approve anyway →" |
| SC-MOD-3 | < 60% | Same warning | "Approve anyway →" |
| SC-MOD-4 | = 0% | Modal never opens (no button shown) | — |

---


---

## 5. Expected Label & Match Recalculation

| Scenario | Action | Result |
|---|---|---|
| SC-EL-1 | Set expected label on a tested doc | `match` recalculated live: `pred === label` |
| SC-EL-2 | Clear expected label on a tested doc | `match` set to `null` (no-label state) |
| SC-EL-3 | Set label matching prediction | Doc goes from Failed → Tested (green) |
| SC-EL-4 | Bulk set expected for entire dataset | All docs in dataset get label; match recalculated for already-tested ones |

---

## 6. Stale Prompt Warning

| Scenario | Condition | Shown |
|---|---|---|
| SC-SP-1 | Prompt text edited after test ran | "⚠ Prompt edited — results may be outdated" banner |
| SC-SP-2 | Prompt reverted to tested text | Warning disappears, accuracy shown again |
| SC-SP-3 | Stale + accuracy > 0 | No approve button shown (must re-run) |

---

## 7. LOB Overview Sync (after Deploy)

| Scenario | Action | LOB Overview state |
|---|---|---|
| SC-LOB-1 | Deploy prompt | LOB card status → `Active` |
| SC-LOB-2 | Deploy prompt | LOB card confidence → snapshot accuracy |
| SC-LOB-3 | Multiple deploys | Previous version entries → `Archived`; new entry → `Active` |
| SC-LOB-4 | Open Manage Version modal after deploy | Version list shows new entry with V.N, date, score, status=Active |

---

## 8. Dev Scenario Coverage (mock data)

| Scenario | Dev preset | Expected state |
|---|---|---|
| SC-DEV-1 | `ov-default` | LOB overview, default state |
| SC-DEV-2 | `det-l1` — L1 mode, no docs | Pending badges, Run test available after select file |
| SC-DEV-3 | `det-docs-matched` — all match=true | 100% accuracy shown in run-test-bar, Approve button shown |
| SC-DEV-4 | `det-docs-lowconf` — mixed match + conf | Partial accuracy, Approve shown |
| SC-DEV-5 | `det-docs-miss` — all match=false | 0% accuracy, no Approve button shown |
| SC-DEV-6 | `doc-pending` — docs, no tests | All Pending, Run test button enabled after select |
