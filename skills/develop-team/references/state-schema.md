# State File Schema

Full schema for `.develop-team/state.json`.

```json
{
  "input": "PROJ-123", "inputType": "ticket",
  "parameters": {
    "type": "auto", "autoCommit": true, "skipReview": false,
    "skipTests": false, "skipPr": false, "skipMigrations": false, "planOnly": false
  },
  "featureType": null, "currentPhase": 0, "status": "in_progress",
  "startedAt": "2026-02-21T10:00:00Z", "branchName": null,
  "ticket": null, "specFile": null, "pastTicketContext": null,
  "research": {
    "codebaseExplorer": null, "complianceMapper": null, "requirementsMapper": null,
    "designSystemIntegrator": null, "asyncFlowAnalyzer": null, "dbMigrationAnalyzer": null
  },
  "plan": null, "designSpec": null,
  "implementation": { "phases": [], "commits": [] },
  "tests": { "generated": 0, "passed": 0, "failed": 0, "skipped": false, "perPhase": [] },
  "migrations": { "planned": [], "applied": [], "skipped": false, "localFiles": [] },
  "refactor": { "delegated": false, "fixedItems": 0, "strategicItems": 0, "commit": null },
  "review": { "delegated": false, "iterations": 0, "quickFixResolved": 0, "strategicItems": 0, "commits": [] },
  "pullRequest": { "created": false, "url": null, "number": null, "skipped": false }
}
```

## Field Notes

| Field | Description |
|-------|-------------|
| `input` | Original ticket key or freeform task description |
| `inputType` | `"ticket"` or `"freeform"` |
| `featureType` | `"ui-heavy"`, `"full-stack"`, `"backend-only"`, or `"async-heavy"` |
| `currentPhase` | Last completed phase number (0-8) |
| `status` | `"in_progress"`, `"completed"`, `"failed"`, `"paused"` |
| `specFile` | Path to spec/backlog file if found |
| `pastTicketContext` | Summary of related past PRs and commits |
| `research.*` | JSON output from each research agent (null if not run) |
| `designSpec` | Extracted design specification for UI phases |
| `implementation.phases` | Per-phase status: `{number, status, agent, filesChanged[], errors[]}` |
| `implementation.commits` | Commit hashes with phase labels |
| `tests.perPhase` | `{phase, generated, passed, failed, testFiles[]}` |
| `migrations.*` | Planned SQL, applied status, local file paths |

## Backward Compatibility

Old state files missing new fields (`pastTicketContext`, `designSpec`, `migrations`, `tests`, `pullRequest`) still load — missing fields default to null/empty.
