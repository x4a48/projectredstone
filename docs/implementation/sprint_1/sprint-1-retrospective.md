# Sprint 1 Retrospective

## Sprint Outcome

Sprint 1 successfully established HYP01 as the initial virtualisation platform for Northwind Outdoor Ltd.

The planned implementation work was completed within the available sprint capacity and potentially ahead of the original estimate.

The sprint delivered a stable foundation for future Northwind infrastructure while also providing significant practical Linux, Proxmox, storage and troubleshooting experience.

---

## What Went Well

- Sprint 1 was completed within the planned capacity and potentially ahead of schedule.
- The overall outcome met expectations and provides a strong platform for subsequent Northwind infrastructure.
- Collaboration and technical discussion throughout implementation supported the learning process.
- Implementation was not treated as simply completing a sequence of commands.
- Time was deliberately spent understanding why configurations were required and how the underlying technologies worked.
- Problems encountered during implementation were investigated rather than simply worked around.
- End-of-NIM summaries provided useful records of changes, validation, findings and lessons learned.
- Estimating NIMs helped establish realistic expectations for implementation effort.
- Prioritising NIMs helped maintain a logical implementation sequence.

---

## What Could Be Improved

### Documentation Timing

A significant amount of the formal GitHub implementation documentation was produced retrospectively at the end of Sprint 1.

Although implementation notes and evidence were captured during the work, the final `README.md` and `commands.md` records were not consistently maintained as part of each NIM.

This created additional documentation work at sprint closure and increased the risk of details being forgotten.

Sprint 2 will integrate repository documentation directly into the NIM lifecycle.

---

## Keep Doing

- Estimate NIM effort during sprint planning.
- Prioritise NIMs based on dependencies and implementation sequence.
- Take time to understand technologies rather than simply following implementation steps.
- Troubleshoot unexpected behaviour methodically.
- Capture implementation evidence while work is being performed.
- Summarise changes, validation results, findings and lessons learned at the end of each NIM.
- Keep implementation scope aligned with the relevant NIM rather than introducing future functionality prematurely.

---

## Stop Doing

No specific practice has been identified for removal following Sprint 1.

The primary process weakness identified relates to when documentation is completed rather than documentation itself.

This will therefore be addressed through the new Sprint 2 workflow rather than by removing an existing practice.

---

## Start Doing

Beginning with Sprint 2, each NIM will follow a consistent Git and documentation workflow.

### NIM Workflow

```text
Select NIM
    ↓
Create Git branch
    ↓
Create NIM directory
    ↓
Create README.md + commands.md
    ↓
Implement / Troubleshoot
    ↓
Capture evidence and notes
    ↓
Validate acceptance criteria
    ↓
Update README.md + commands.md
    ↓
Update Jira
    ↓
Review Git changes
    ↓
Commit
    ↓
Push
    ↓
Pull Request
    ↓
Review
    ↓
Merge to main
    ↓
Delete completed branch
    ↓
NIM Done
