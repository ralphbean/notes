# ADR 10: Revisit in-scope entities to reevaluate eligibility

**Status:** Proposed  
**Date:** 2026-10-02

## Context

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) proposes a poller that discovers source activity, records accepted observations, and advances a source checkpoint. That lets Fullsend recover work associated with a change. It does not ensure that Fullsend checks an entity again when nothing about that entity changes.

An entity can become eligible because time passes. For example, a harness might apply to a reviewed PR with passing checks once the PR has been idle for 48 hours. A rule like `now - updated > 48h` is false at hour 47 and true at hour 48, but an activity-only poller will not evaluate it again at hour 48. A configuration change or a change to related work can likewise alter applicability without new activity on the original entity.

Revisiting an entity creates another problem: a predicate may still match after its work has run. A periodic visit must not itself mean a new obligation to execute. Fullsend needs to distinguish checking current eligibility from creating another run for work already satisfied or underway.

## Decision

The centralized poller periodically revisits every entity in each registered source scope that can be eligible for a configured harness, including entities with no recent source activity. Each visit refetches the entity and evaluates the applicable harness predicates against current source state and an explicit evaluation time. The entity model and evaluation context expose the source timestamp and current time needed for conditions such as `now - updated > 48h`. Source activity can prompt an earlier evaluation; it is not required for reevaluation.

The sweep is bounded and resumable. Each source adapter must be able to enumerate its complete in-scope set, page through it, and retain progress across poller invocations. Sweep progress is separate from the source-activity checkpoint. A sweep is complete only after every entity in its scope has been covered; failures and incomplete coverage remain visible and are retried. The sweep interval and capacity must be chosen so a full pass finishes within the intended reevaluation delay.

A sweep visit is an evaluation request, not a new work generation. Before creating a selected run, Fullsend checks whether the same logical work is already pending, running, or satisfied for the relevant entity state. A later state or policy change can warrant new work only when the harness's applicability and work identity make that change meaningful. The detailed identity and effect-reconciliation contract belongs to a separate decision.

## Consequences

- Time-dependent predicates can become actionable without a new webhook or source update. The actual delay depends on full-sweep coverage, dispatch backlog, and run admission.
- Source adapters must provide complete, authorized enumeration of their registered scopes. An entity omitted from that enumeration has no sweep-based reevaluation guarantee.
- Periodic scans add source API and evaluation load. Bounded pages and persisted progress keep one poller invocation finite, but operators need to see sweep age and incomplete coverage.
- Activity observations and sweep visits have different identities and checkpoints.
- Repeated visits must not repeat already satisfied work. This ADR requires that distinction without deciding the general identity and external-effect recovery design.
