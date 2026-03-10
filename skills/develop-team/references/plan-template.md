# Plan Template

Save to `.develop-team/plan.md`. The orchestrator synthesizes all research into this structure.

```markdown
# Implementation Plan: {Feature Name}

## Context
- **Ticket**: {key} - {summary} | **Branch**: {branch} | **Type**: {featureType}
- **Past context**: {summary or "None"}

## Phase {N}: {Name}
### Goal: {what this achieves}
### Tasks
#### Task {N}.1: {Title}
- **File(s)**: {paths} | **Action**: create|modify | **Effort**: trivial|small|medium|large
- **Details**: {what to do, referencing research patterns/utilities}
- **Convention notes**: {rules from Compliance Mapper's rule card for this file type}
### Verification: build, lint, {phase-specific checks}

## Design Specification (if ui-heavy|full-stack)
### Component Hierarchy — {from Design System Integrator}
### Design Tokens — {exact colors, spacing, typography from theme files}
### Responsive Behavior — {breakpoint-specific changes}
### Interaction States — {loading, empty, error per component}
### Anti-Slop Checklist
- [ ] No invented colors | No decorative escalation | Typography matches | Animations from config only

## Test Plan (unless skip-tests=true)
| Phase | Test ID | Description | Type | Target File |
{from Requirements Mapper testPlan}

## Migration Plan (unless skip-migrations=true)
| Order | Name | SQL | Post-migration updates |
{from DB Migration Analyzer or Requirements Mapper migrationPlan}

## Acceptance Criteria Coverage
| Criterion | Tasks | Status |

## Files Summary
| Action | Path | Phase |
```
