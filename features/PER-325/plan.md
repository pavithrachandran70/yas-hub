# Plan — PER-325 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Display review summary in product page header using existing review data; no API, schema or storage changes, single repo UI-only update.

## Steps

1. [x] **analyst (inline_story)** — Convert ticket into implementable story with acceptance criteria for the UI changes.
   - focus: Focus on the UI state management after review submission and the display logic for zero-review vs. multiple-review cases.
2. [x] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [x] **dev** _(auto)_ — Loop delivery.
4. [x] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [x] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Only one spec document (the inline story); no cross-artifact consistency to check.
