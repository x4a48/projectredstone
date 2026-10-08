# Northwind Implementation Workflow

**Organisation:** Northwind Outdoor Ltd.  
**Delivery team:** Redstone Consulting  
**Document type:** Standard Operating Procedure (SOP)  
**Version:** 1.0  
**Status:** Initial standard — adopted after Sprint 1  
**Applies to:** Technical implementation tasks (NIM) in Project Redstone

## 1. Purpose

Provide a repeatable, evidence-based process for planning, implementing, validating, documenting and closing Northwind infrastructure changes. Scale the amount of documentation to the risk and complexity of the task; do not create paperwork without operational value.

**Workflow:** Plan → Prepare → Implement → Validate & Reconcile → Close

## 2. Implementation checklist

### Stage 1 — Plan

- [ ] Review the Jira NIM task, intended business outcome, scope and acceptance criteria.
- [ ] Review applicable BRD, RTM, HLD, LLD, approved design decisions and existing system documentation.
- [ ] Confirm dependencies and whether the work is appropriate for the current sprint/implementation stage.
- [ ] Define the implementation approach, key risks, safety precautions, recovery/rollback method and maintenance needs as appropriate.
- [ ] Decide **how success will be demonstrated** before making changes; identify required evidence.
- [ ] Review the RTM for relevant requirements and traceability. Do not assign or mark requirements complete without justification.

**Gate:** Scope, design basis, dependencies and validation approach are understood.

### Stage 2 — Prepare

- [ ] Check the working tree (`git status`) and synchronise with the current remote baseline before starting work.
- [ ] Create a task branch using the agreed NIM naming convention (for example, `nim/015-document-hyp01`).
- [ ] Create `implementation/sprint-XX/NIM-XXX-short-description/README.md` and `commands.md` as needed.
- [ ] Set up an optional `evidence/` directory for sanitised screenshots, diagrams or logs.
- [ ] Draft an ordered implementation plan or Build Guide outline where appropriate, including prerequisites and validation checkpoints.
- [ ] Confirm the current configuration and that recovery access is available before potentially disruptive changes.

**Gate:** Work can begin safely and evidence has an agreed home.

### Stage 3 — Implement & capture

- [ ] Follow the agreed implementation sequence; check prerequisites before each dependent step.
- [ ] Record meaningful commands, configuration changes, decisions, errors and troubleshooting in the task `README.md` or working notes **as the work happens**.
- [ ] Capture useful screenshots or screen recordings at meaningful checkpoints; avoid unnecessary duplication.
- [ ] Update the relevant **Build/Rebuild Guide** alongside implementation when a reusable procedure is established or changed.
- [ ] Update the affected **System Record** when a server's current configuration materially changes.
- [ ] Record departures from the approved design, temporary configurations, risks and follow-up actions as they arise.
- [ ] Keep secrets, personal data, full serial numbers and sensitive network details out of material intended for public GitHub publication; sanitise evidence before committing.

**Gate:** Changes and significant decisions are traceable, not reconstructed from memory.

### Stage 4 — Validate & reconcile

- [ ] Execute the planned acceptance tests and capture **observed results**, not just commands or intended outcomes.
- [ ] Verify relevant service health, connectivity, storage, permissions, security controls, persistence across reboot and recovery behaviour **as applicable**.
- [ ] Compare the as-built configuration with the approved LLD and design decisions; classify differences as conforming, approved temporary, deviation, not yet due or requiring verification.
- [ ] Resolve or explicitly record each deviation, including impact, decision and owner/future action.
- [ ] Update the **asset register** for new, retired or materially changed assets; maintain a sanitised public copy where applicable.
- [ ] Revisit the **RTM** and update supported evidence, status and sprint links where justified; do not overstate business-requirement completion.
- [ ] Consolidate `commands.md`: remove duplicates, add short explanations, and distinguish general Linux/RHCSA commands from Proxmox-specific commands.
- [ ] Ensure the task implementation record links to authoritative System Records, Build Guides, evidence and design references instead of copying them wholesale.

**Gate:** Acceptance criteria are evidenced; differences and outstanding work are visible.

### Stage 5 — Review, publish & close

- [ ] Review the **project-root `README.md` displayed on GitHub**. Update its current phase/sprint, delivered infrastructure, progress, roadmap and status statements **where affected by this work**. Do not leave stale claims of what is planned versus operational.
- [ ] Check that System Records, Build Guides, implementation notes, asset register and RTM are mutually consistent.
- [ ] Review repository changes (`git status`, `git diff`) for accuracy, accidental files, credentials, personal details and full hardware serial numbers.
- [ ] Commit coherent changes with descriptive messages; push the NIM branch.
- [ ] Open/review the pull request, confirm acceptance criteria and merge into `main` under the agreed Git workflow.
- [ ] Confirm the merged state and mark the Jira NIM task Done, linking the PR and relevant evidence.
- [ ] At sprint close, update the sprint retrospective and project status/roadmap, and carry unresolved work into future planning **without automatically creating unnecessary tickets**.

**Gate:** Repository, Jira and project-facing status agree on what was actually delivered.

## 3. Documentation responsibilities

| Artefact | Primary question answered | Update trigger |
|---|---|---|
| Task implementation `README.md` | What changed, why, how was it tested, and what remains? | During every implementation task |
| Task `commands.md` | Which commands were used or learned, and what do they do? | When meaningful commands are encountered |
| System Record (`servers/<HOST>/README.md`) | What is this system's **current** configuration? | Material configuration change |
| Build/Rebuild Guide | How can an administrator reproduce or recover the system? | Procedure created, changed or validated |
| Asset register | What assets exist and what are their stable inventory attributes? | Asset added, retired or materially changed |
| RTM | Which requirements are planned, progressed, evidenced or satisfied? | Sprint planning and task/sprint closure |
| Project-root GitHub `README.md` | What is Northwind's **current project status** and where is it heading? | Status, milestones, infrastructure or roadmap changes |
| Sprint retrospective | What should the team repeat or improve next sprint? | Sprint closure |

**Documentation principle:** Maintain one authoritative location per fact. Link to it from other documents. System Records are concise living references; implementation records retain history and evidence.

## 4. Git quick reference — two-machine workflow

```bash
# Before beginning or switching work
git status
git fetch origin

# Create a new NIM branch from an up-to-date main (only when working tree is clean)
git switch main
git pull --ff-only
git switch -c nim/XXX-short-description

# Commit and publish work
git add <paths>
git commit -m "docs(nim-XXX): describe change"
git push -u origin nim/XXX-short-description

# On a second machine: fetch a branch not yet present locally
git fetch origin
git switch --track origin/nim/XXX-short-description

# If that branch already exists locally
git switch nim/XXX-short-description
git pull --ff-only

# Subsequent commits on a tracking branch
git push
```

**Safety:** Check `git status` before branch switches and pulls. Commit or stash local work first. Do not merge a task branch into `main` until reviewed and accepted.

## 5. Definition of Done — technical NIM

A task is Done when its agreed acceptance criteria have been met and evidenced; implementation and deviations are documented; applicable living records, inventory, RTM and project-root README have been reviewed/updated; repository changes have been reviewed and merged; and Jira reflects the outcome. Outstanding future-stage work may remain open if it is clearly recorded and does not block the task's acceptance.

## 6. Continuous improvement

This SOP originated from Sprint 1 / HYP01 lessons. Review its usefulness during Sprint 2 retrospective and adjust only where the change makes delivery safer, clearer or more efficient.
