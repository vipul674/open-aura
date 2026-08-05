# Design — Per-project skill chips on "What they build" cards

Status: approved (post-brainstorm). Restores per-project aggregated skill chips on
project cards, as a session-scoped correction follow-up. Skills stay per-session
in `telemetry.inferred_skills`; the project card folds only that project's own
sessions' skills into a top-4 chip row. No DB schema change.

## Session-wise context (already shipped)

- Per-session `telemetry.inferred_skills` is the source of truth (scoring LLM).
- Session-view hero + toolkit show THAT session's skills (`Inferred from this
  session`); overall-profile toolkit + hero show the cross-session aggregate.
- Project cards currently render no skills (a prior fix removed the folded row).
  This restores it with the same per-project weighted logic.

## Decision (locked)

Project-card skills = weighted aggregation across **that project's** sessions,
top-4, measured-skills deduped — same engine as the toolkit.

## Data flow

```
session.telemetry.inferred_skills ──┐  per-project bucket
                                    ├─> _top_skills(bucket, measured, 4)
                                    │   -> project.skills [{name, count}]
                            weight  ┘   (conf × SOURCE_WEIGHTS["inferred"] 0.7 × recency)
-> aura-projects.tsx green chip row
```

## Contract / changes

- `aura_profile_facts.py`: each project re-accumulates its sessions' inferred
  skills into per-project `skill_weights`; emit `project["skills"] =
  _top_skills(…, 4)` (top-4 by weight, measured-wins dedup). Reuses existing
  `_accumulate_inferred` + `_top_skills`.
- `types.ts`: restore `AuraProject.skills?: ToolkitEntry[]`.
- `aura-projects.tsx`: restore the "Skills" green chip row (reused token
  `border-[rgba(0,230,118,.28)] bg-[rgba(0,230,118,.08)]`).

## Behavior

- Project cards render their aggregated per-project skills in BOTH main-profile
  and session-view (the `isLocalOwnerProfile` gate is unchanged — "always show
  with skills").

## Tests

- Replace `test_project_cards_have_no_folded_skills` with a positive per-project
  test: multi-session weighting (top-4), measured-wins dedup, projects with no
  sessions' skills carry no `skills` key.

## Out of scope

- No change to session-view hero/toolkit scoping.
- No change to public `/u/{handle}` route (still pops `toolkit`/`projects`).