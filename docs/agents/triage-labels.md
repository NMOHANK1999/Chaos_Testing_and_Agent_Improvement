# Triage Labels

The skills speak in terms of five canonical triage roles. Since issues live as local markdown (see `issue-tracker.md`), these roles are recorded as a `Status:` line near the top of each issue file rather than as real tracker labels.

| Role in mattpocock/skills | Status: value in this repo | Meaning                                  |
| -------------------------- | -------------------- | ---------------------------------------- |
| `needs-triage`             | `needs-triage`       | Maintainer needs to evaluate this issue  |
| `needs-info`               | `needs-info`         | Waiting on reporter for more information |
| `ready-for-agent`          | `ready-for-agent`    | Fully specified, ready for an AFK agent  |
| `ready-for-human`          | `ready-for-human`    | Requires human implementation            |
| `wontfix`                  | `wontfix`            | Will not be actioned                     |

When a skill mentions a role (e.g. "apply the AFK-ready triage label"), use the corresponding `Status:` value from this table.
