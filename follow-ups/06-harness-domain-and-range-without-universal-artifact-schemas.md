# ADR 06: Use harness domains and ranges without universal artifact schemas

**Status:** Proposed  
**Date:** 2026-10-02

## Context

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) starts with a source observation, fetches the current entity and relevant activity, selects a harness, and records its run. [ADR 02](02-harness-applicability-from-entity-and-context.md) proposes a harness `domain` for the entities it accepts and a `range` for the entities it intends to produce or advance. That gives us a way to describe how harnesses relate without assuming that every run consumes and produces a new document.

For example, a code harness might accept a work item marked ready for code and expect to create an open pull request in its target repository. A review harness might accept open pull requests. The code harness's range and the review harness's domain suggest a possible connection. The domain can help select work; the range can describe expected output and support a check of the existing output queue before a run starts. Neither declaration proves that a particular run created a pull request or that a later run should be authorized.

We could require every harness to exchange a separate, versioned artifact with a common schema. We do not yet have a Fullsend dispatch case that needs that contract. It would add a second description of work alongside the source entity and the harness declarations, before the normalized entity vocabulary and the `domain`/`range` relationship are settled.

## Decision

Fullsend will use harness `domain` and `range` declarations to describe accepted source entities and intended output entities. It will not introduce a universal artifact schema or require a separate typed artifact at each station boundary for dispatch.

As an illustration of the intended meaning, a code harness could have a domain of “work items marked ready for code” and a range of “open pull requests in the target repository.” A review harness could have a domain of “open pull requests.” These are descriptions of entity sets, not a proposed expression syntax or a guarantee that one run produced a member of the declared range.

One possible expression shape, using invented entity names and fields, is:

```text
code.domain:   entity.kind == "work_item" && "ready_for_code" in entity.labels
code.range:    entity.kind == "change_request" &&
               entity.repository == context.target_repository && entity.state == "open"
review.domain: entity.kind == "change_request" &&
               entity.repository == context.target_repository && entity.state == "open"
```

Here `change_request` stands for a pull request or merge request. The matching code range and review domain make their possible connection visible. An event such as a label being added may still determine *when* the code harness is considered; whether that remains in `trigger` or is represented with `domain` is unresolved.

The work in AISDLC-48 must settle the initial range vocabulary and its use for runner WIP checks. AISDLC-49 must settle how a domain is represented or derived from `trigger`, and how domains and ranges are compared for graphing. This ADR does not prescribe normalized entity fields, expression syntax, or a required `range` for every harness.

When Fullsend needs to rely on a particular output to admit later work, that use must identify the observed entity and the evidence that connects it to the run. A successful Job or a declared range alone cannot establish that connection. The contract for accepting such evidence belongs to the process decision that needs it.

## Consequences

- Existing harnesses can remain runnable while domain and range support evolves. Dispatch does not need a universal artifact registry or a schema for each station.
- A graph derived from domains and ranges shows possible connections. It does not certify that a run produced an expected output or that a process gate passed.
- A WIP check can use a declared range to count relevant current entities before execution. The declaration does not validate the run's eventual effects.
- If a future process depends on a produced artifact, it will need an explicit way to identify and verify that output. This decision leaves that requirement with the process that uses it.
