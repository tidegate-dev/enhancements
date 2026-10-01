---
tep: 0001
title: TEP process
status: implemented
authors:
  - "@trevex"
approvers:
  - "@trevex"
tracking-issue: null
created: 2026-10-01
last-updated: 2026-10-01
replaces: []
superseded-by: []
---

# TEP-0001: TEP process

## Summary

This TEP defines the Tidegate Enhancement Proposal process: what a TEP is, when one is required, how it moves from idea to shipped feature, and who decides. Changes to this process are themselves made through a TEP that amends or replaces this one.

## Motivation

Tidegate is a privileged access management system. It decides who may reach production systems, for how long, and with which permissions. Design mistakes in such a system turn into security incidents, and they are expensive to fix once deployments depend on the behavior.

Without a written process, large changes get designed in chat threads and pull request comments. The reasoning is hard to find later, security review depends on whoever happened to be watching, and new contributors cannot tell how to propose something big. A lightweight proposal process fixes that while the project is small enough for the process to stay simple.

### Goals

- Give contributors one documented path for proposing significant changes.
- Make sure every significant change gets an explicit security review before implementation starts.
- Keep a permanent, searchable record of design decisions, including rejected ones.
- Keep the overhead low enough that people use the process.

### Non-goals

- Replacing code review. A TEP approves a design; the implementation is still reviewed in the code repositories.
- Defining the project's full governance model, such as how maintainers are appointed. This TEP only uses that model.
- Tracking release planning or milestones.

## Proposal

### What a TEP is

A TEP is a Markdown document in this repository at `teps/NNNN-short-title/README.md`. It starts with YAML front matter holding its metadata, followed by the sections from the [template](../0000-template/README.md). Supporting files such as diagrams live in the same directory.

### When a TEP is required

A TEP is required for:

- New user-facing features or new components.
- Changes to the security model: authentication, authorization, credential handling, session brokering, audit logging, or trust boundaries between components.
- Breaking changes to an API, CLI, configuration format, or on-disk or wire format.
- New external dependencies that become part of the trusted computing base.
- Changes to project governance or to this process.

Bug fixes, refactors, documentation, and behavior-preserving improvements do not need a TEP. If a maintainer reviewing a pull request decides the change needs one, the pull request waits until a TEP for it reaches `implementable`.

### Roles

- Author: Writes the TEP, responds to feedback, and keeps the status current. A TEP can have several authors.
- Approver: A maintainer who signs off on the TEP. Approvers are listed in the front matter. An author cannot be the only approver of their own TEP, except for this bootstrapping TEP.
- Maintainers: Until a governance document says otherwise, the maintainers are the members of the `tidegate-dev` GitHub organization with write access to this repository.

### Lifecycle

```
draft -> provisional -> implementable -> implemented
```

A TEP starts as a `draft` pull request and normally follows the path above. It can leave that path at several points: `rejected` or `withdrawn` before it is implemented, `deferred` once it is `provisional`, and `replaced` from any merged status. The statuses are:

| Status | Meaning |
|---|---|
| `draft` | Under discussion in a pull request. Not merged yet. |
| `provisional` | The project agrees the problem is worth solving and the direction is sound. Design details may still be open. |
| `implementable` | The design is approved. Implementation can begin. |
| `implemented` | The change has shipped in a release. |
| `deferred` | Accepted in principle, but nobody is working on it. Can return to `provisional` or `implementable`. |
| `rejected` | Approvers decided against it. The TEP stays in the repository with the reasons recorded. |
| `withdrawn` | The author dropped it. |
| `replaced` | Superseded by another TEP, named in `superseded-by`. |

Each status change is a pull request that edits the front matter, updates `last-updated`, and adds an entry to the implementation history section.

### Numbering

Authors take the next free number when they open the pull request, zero-padded to four digits. If two open pull requests claim the same number, the one merged second renumbers before merging. CI rejects duplicate numbers. Number 0000 is reserved for the template.

### Approval

A TEP moves to `provisional` or `implementable` when these conditions hold:

1. At least two maintainers who are not authors approve the pull request. While the project has fewer than three maintainers, one non-author approval is enough.
2. A final comment period of seven calendar days has passed since an approver announced it on the pull request, and no maintainer has raised a blocking objection during that time. An objection is resolved by changing the TEP or by the objecting maintainer withdrawing it.
3. The security considerations section has content appropriate for the target status: a first pass for `provisional`, complete for `implementable`.

Rejection follows the same rule: two maintainers agree, with the reasons written into the TEP, and the TEP is merged as `rejected` so the reasoning stays on record.

If maintainers cannot reach agreement, the decision goes to the full set of maintainers and is settled by simple majority. That fallback is meant to be rare.

### Implementation and code repositories

Implementation pull requests in the code repositories link to the TEP. A TEP marked `implementable` is the reference for code reviewers: if implementation shows the design has to change in a way that affects behavior, security, or compatibility, the author updates the TEP first.

## Design details

The repository has this layout:

```
README.md                      Overview and generated index
teps/0000-template/README.md   Template to copy
teps/NNNN-short-title/         One directory per TEP
  README.md                    The TEP itself
  (supporting files)
scripts/validate-teps.py       Metadata checks and index generation
.github/                       Issue and PR templates, CI workflow
```

The front matter fields are:

| Field | Required | Description |
|---|---|---|
| `tep` | yes | The TEP number. Must match the directory name. |
| `title` | yes | Short title. |
| `status` | yes | One of the statuses listed above. |
| `authors` | yes | GitHub handles, each starting with `@`. |
| `approvers` | yes | GitHub handles of the approving maintainers. |
| `tracking-issue` | yes | URL of the tracking issue, or `null` before one exists. |
| `created` | yes | Date in `YYYY-MM-DD` format. |
| `last-updated` | yes | Date in `YYYY-MM-DD` format. |
| `replaces` | no | TEP numbers this one supersedes. |
| `superseded-by` | no | TEP numbers that supersede this one. Required when the status is `replaced`. |

`scripts/validate-teps.py` checks the directory names, the front matter, and number uniqueness, and verifies that the index in the top-level README matches the TEPs on disk. Running it with `--write-index` regenerates the index. CI runs the checks on every pull request.

## Security considerations

The process exists partly to put security review in a fixed place. Every TEP must fill in the security considerations section of the template, which asks about trust boundaries, privilege scope and duration, credential handling, audit, failure behavior, and threat model. A missing or empty section blocks approval.

Proposals that disclose an unfixed vulnerability must not go through this public process. Those follow the project's security policy for private disclosure, and a TEP can follow once a fix is public.

## Compatibility and upgrade

This TEP introduces the process and has no effect on deployed software.

## Test plan

CI validates TEP metadata on every pull request. The process itself is reviewed by amending this TEP when it causes friction.

## Rollout

The process takes effect when this TEP is merged. Work already in progress before then does not need a retroactive TEP.

## Drawbacks

The process adds a week or more before significant work can start, because of the final comment period. Authors can prototype while a TEP is under review, so the delay mostly affects when code merges.

## Alternatives

Design discussions in GitHub issues alone would cost less effort, but issues are hard to keep as a coherent current design and do not give a natural place for structured security review.

Adopting the Kubernetes KEP process unchanged would bring production readiness reviews, SIG ownership, and per-release tracking that a project of Tidegate's size does not need yet. This TEP keeps the KEP status model and template structure and drops the rest. Later TEPs can add pieces back as the project grows.

## Unresolved questions

- Should approvers be grouped by area (for example, security, API, operations) once the maintainer group grows?

## Implementation history

- 2026-10-01: TEP process established.
