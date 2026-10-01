---
tep: 0000
title: Short descriptive title
status: draft
authors:
  - "@your-github-handle"
approvers:
  - "@approver-github-handle"
tracking-issue: https://github.com/tidegate-dev/enhancements/issues/0
created: 2026-01-01
last-updated: 2026-01-01
replaces: []
superseded-by: []
---

# TEP-0000: Short descriptive title

<!--
How to use this template:

1. Copy this directory to teps/NNNN-short-title/ using the next free number.
2. Fill in the front matter above. Keep `status: draft` until the PR is merged.
3. Replace the guidance comments below with your content. Delete comments once a
   section is written. Sections marked "required for provisional" must have
   content before the first merge. The rest must be complete before the TEP
   moves to `implementable`.
4. If a section does not apply, keep the heading and write one sentence saying
   why. Reviewers need to see that it was considered.

Supporting files such as diagrams can live next to this README in the same
directory.
-->

## Summary

<!--
Required for provisional.

One or two paragraphs that a reader can understand without reading the rest of
the document. Say what changes and who it affects.
-->

## Motivation

<!--
Required for provisional.

What problem does this solve, and for whom? Link to issues, incidents, or user
requests that show the problem is real.
-->

### Goals

<!--
Required for provisional.

A short list of outcomes this TEP commits to. Each goal should be specific
enough that someone can later check whether it was met.
-->

### Non-goals

<!--
Required for provisional.

What is explicitly out of scope. This keeps discussion focused and records
which related problems are left for later.
-->

## Proposal

<!--
Required for provisional.

The proposed change at the level of behavior: what users, operators, and other
components will see. Leave implementation details for the next section.
-->

### User stories

<!--
Concrete scenarios, for example: "An on-call engineer requests 30 minutes of
read-only access to the production database and is approved by a second
engineer." Include operator and auditor stories where relevant.
-->

### Risks and mitigations

<!--
What could go wrong, including how the change could be misused, and what the
design does about it. Security risks get detailed treatment in the security
considerations section below.
-->

## Design details

<!--
Required for implementable.

Enough detail for reviewers to judge the design and for someone other than the
author to implement it: APIs, data models, configuration, wire formats, state
machines, and interactions between components. Diagrams help.
-->

## Security considerations

<!--
Required for provisional (a first pass), complete for implementable.

Tidegate brokers privileged access, so every TEP must answer these questions,
even if the answer is "not affected":

- Trust boundaries: which components, users, or external systems does this
  change trust, and does it move any trust boundary?
- Privilege: does the change grant, extend, or escalate access? How is the
  scope and duration of access bounded?
- Credentials and secrets: what secrets are created, stored, transmitted, or
  logged, and how are they protected and rotated?
- Audit: which actions are recorded, where, and can the record be tampered
  with or bypassed?
- Failure behavior: when a dependency is unavailable or returns an error, does
  the system fail closed (deny access)? If not, why is that acceptable?
- Threat model: which attackers are in scope (malicious requester, compromised
  approver, compromised Tidegate component, network attacker) and how does the
  design hold up against each?
-->

## Compatibility and upgrade

<!--
Required for implementable.

Effects on existing deployments: API or configuration changes, data
migrations, mixed-version behavior during a rolling upgrade, and how to
downgrade if the change has to be reverted.
-->

## Test plan

<!--
Required for implementable.

How the change will be tested: unit, integration, and end-to-end coverage, and
any security-specific testing such as fuzzing or negative authorization tests.
-->

## Rollout

<!--
Required for implementable.

How the change reaches users. For example: behind a feature flag in one
release, enabled by default in the next. List the criteria for each step.
-->

## Drawbacks

<!--
Reasons not to do this: added complexity, maintenance load, operational
burden, or effects on other planned work.
-->

## Alternatives

<!--
Other designs that were considered and why they were not chosen. Include
"do nothing" when it is a real option.
-->

## Unresolved questions

<!--
Open questions that need answers before the TEP can move to the next status.
Remove this section, or leave it empty, once everything is resolved.
-->

## Implementation history

<!--
Major milestones, with dates and links. For example:

- 2026-01-01: TEP merged as provisional (#12)
- 2026-02-15: Moved to implementable (#20)
- 2026-04-01: Shipped in Tidegate v0.3.0
-->
