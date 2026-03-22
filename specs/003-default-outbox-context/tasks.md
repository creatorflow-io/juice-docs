# Tasks: DefaultOutboxContext — Full-Route IMessageService Outside Transactions

**Input**: Design documents from `/specs/003-default-outbox-context/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, contracts/ ✅

**Organization**: All tasks edit a single file (`content/core/messaging/local-transport/_index.md`).
Because they share one file, tasks within a story are sequential (no `[P]` within a phase).
Different stories are independently testable increments.

---

## Phase 1: Setup

**Purpose**: Confirm working state before any edits.

- [X] T001 Verify branch `003-default-outbox-context` is checked out in juice-docs and read the full current content of `content/core/messaging/local-transport/_index.md`

---

## Phase 2: User Story 1 — IMessageService Table + Full-Route Registration (Priority: P1) 🎯 MVP

**Goal**: Fix the outdated claim that `IMessageService` only supports `"local-channel"`. Add `AddDefaultMessageService()` registration path with delivery warning and NuGet entry.

**Independent Test**: Navigate to the local-transport page and verify: (1) the `IMessageService` table has three rows — two for `IMessageService` (one per registration path) and one for `IMessageService<TContext>`; (2) a `### Full-route outside transactions` subsection exists with a `AddDefaultMessageService()` code snippet and a delivery callout; (3) `Juice.Messaging.Outbox.EF` appears in the NuGet table.

- [X] T002 [US1] Replace the `IMessageService` table with the 3-row version per `contracts/imessageservice-table.md` in `content/core/messaging/local-transport/_index.md`
- [X] T003 [US1] Add `Juice.Messaging.Outbox.EF` row to the NuGet packages table in `content/core/messaging/local-transport/_index.md`
- [X] T004 [US1] Add `### Full-route outside transactions (DefaultOutboxContext)` subsection (including `AddDefaultMessageService()` snippet, delivery callout, and `TryAddScoped` warning) per `contracts/di-registration.md` in `content/core/messaging/local-transport/_index.md`

**Checkpoint**: US1 complete — `IMessageService` table is accurate, `AddDefaultMessageService()` is documented, delivery and first-wins warnings are present.

---

## Phase 3: User Story 2 — DefaultOutboxContext Concept & Schema (Priority: P1)

**Goal**: Explain what `DefaultOutboxContext` is, its schema (no new migrations), `IsManaged = false` behavior, and isolation from `OutboxContext`.

**Independent Test**: The page answers three questions without ambiguity: (1) What tables? Same as `OutboxContext`. (2) What migrations? None new — run existing. (3) Will it collide with `OutboxContext`? No — separate EF registrations.

- [X] T005 [US2] Insert `## DefaultOutboxContext` section (with Schema, IsManaged, and Isolation subsections) after the `## DI Registration` section per `contracts/default-outbox-context-section.md` in `content/core/messaging/local-transport/_index.md`

**Checkpoint**: US2 complete — `DefaultOutboxContext` is documented as a standalone concept with schema and transaction behavior fully explained.

---

## Phase 4: User Story 3 — Coexistence with IMessageService\<TContext\> (Priority: P2)

**Goal**: Show both registrations coexisting in DI and provide a clear when-to-inject table so developers know which interface to use in each context.

**Independent Test**: The coexistence section contains: (1) a DI snippet registering both `AddMessageService<AppDbContext>()` and `AddDefaultMessageService()`; (2) a when-to-inject table with at least 5 rows; (3) the `TryAddScoped` first-wins note.

- [X] T006 [US3] Insert `## Coexistence — IMessageService and IMessageService<TContext>` section (DI snippet + when-to-inject table + first-wins note) after the `## DefaultOutboxContext` section per `contracts/coexistence-section.md` in `content/core/messaging/local-transport/_index.md`

**Checkpoint**: US3 complete — coexistence is documented and developers can determine which service to inject without ambiguity.

---

## Phase 5: User Story 4 — Delivery Configuration Clarification (Priority: P2)

**Goal**: Ensure no reader of the local-transport page is left thinking delivery is auto-wired when using `AddDefaultMessageService()`.

**Independent Test**: The page contains an explicit statement that `AddDefaultMessageService()` does NOT auto-register `DeliveryHostedService<DefaultOutboxContext>`, and shows the explicit delivery configuration snippet `AddDeliveryProcessor<DefaultOutboxContext>(...)`. (Note: the delivery callout from T004 already covers this within the DI registration subsection — this task verifies coverage and adds a forward reference note in the Route Behavior section if absent.)

- [X] T007 [US4] Verify the `"local"` route behavior section in `content/core/messaging/local-transport/_index.md` either already references the manual delivery requirement for `DefaultOutboxContext` (via the T004 callout) or add a one-line note: "If using `DefaultOutboxContext`, configure `AddDeliveryProcessor<DefaultOutboxContext>()` explicitly — delivery is not auto-registered."

**Checkpoint**: US4 complete — no path through the page leaves delivery ambiguous for `DefaultOutboxContext` users.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verify rendering, links, and accuracy before merging.

- [X] T008 [P] Verify all `{{< ref >}}` cross-references in `content/core/messaging/local-transport/_index.md` still resolve (check for broken links to outbox, delivery, full-setup, message-context pages)
- [X] T009 [P] Confirm all C# code snippets in the updated sections are complete and syntactically correct (no truncated lines, balanced braces)
- [ ] T010 Run `hugo server` and load `/core/messaging/local-transport/` to verify the page renders without errors, tables display correctly, and Mermaid diagrams (if any) are intact

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **US1 (Phase 2)**: Depends on Phase 1 only — blocks nothing else
- **US2 (Phase 3)**: Depends on US1 (edits same file sequentially)
- **US3 (Phase 4)**: Depends on US2 (inserts after `## DefaultOutboxContext`)
- **US4 (Phase 5)**: Depends on US1 (verifies T004 callout coverage)
- **Polish (Phase 6)**: Depends on all US phases complete

### Within Each Story

All tasks edit `content/core/messaging/local-transport/_index.md` — sequential only, no parallel within a phase.

### Parallel Opportunities

- T008 and T009 (Polish) can run in parallel — both are read-only verification tasks on the same file
- US2, US3, US4 could theoretically be drafted in parallel then applied sequentially to the file

---

## Parallel Example: Polish Phase

```bash
# T008 and T009 can run simultaneously (both read-only checks):
Task: "Verify {{< ref >}} cross-references resolve"
Task: "Confirm C# code snippets are complete and correct"
```

---

## Implementation Strategy

### MVP (US1 + US2 — P1 stories only)

1. Complete Phase 1: Setup (T001)
2. Complete Phase 2: US1 — fix `IMessageService` table, add `AddDefaultMessageService()` (T002–T004)
3. Complete Phase 3: US2 — add `DefaultOutboxContext` concept section (T005)
4. **STOP and VALIDATE**: Check that the two highest-risk misconceptions are corrected
5. Merge or demo if ready

### Full Delivery

1. MVP above
2. Phase 4: US3 — coexistence section (T006)
3. Phase 5: US4 — delivery verification (T007)
4. Phase 6: Polish (T008–T010)

---

## Notes

- All tasks target a single file: `content/core/messaging/local-transport/_index.md`
- Each task has an exact section anchor (heading text) to locate insertion points
- Contracts in `specs/003-default-outbox-context/contracts/` contain the exact Markdown to insert
- `[P]` appears only in Polish phase where tasks are read-only and non-conflicting
- No new Hugo pages are required (plan.md decision: content fits in existing page)
