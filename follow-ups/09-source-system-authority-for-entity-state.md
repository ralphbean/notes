# ADR 09: Let source systems govern entity state and approvals

**Status:** Proposed  
**Date:** 2026-10-02

## Context

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) gives different systems different records. GitHub and Jira hold source entities and enforce their permissions. Fullsend retains observations, selections, execution attempts, and outcomes so dispatch can recover after a restart. Kubernetes reports Job and Pod status. Calling any one of these a single “source of truth” obscures what each record can establish.

A feature may be approved and then edited in Jira. The edit changes the feature that a harness will see next. Whether that edit preserves the approval, invalidates it, or is prohibited is a rule of the source system. If Fullsend keeps a separate authoritative approval or process position, it can disagree with the feature's current state and make a different decision about what a station may do.

Harnesses select work from source entities and their context. [ADR 02](02-harness-applicability-from-entity-and-context.md) places applicability in each harness's domain predicate, and [ADR 04](04-required-station-activity.md) distinguishes a harness run from acceptance of its work product. A successful Job tells us what happened during execution; it does not approve a feature. Conversely, a source approval does not prove that a particular Fullsend Job ran.

An entity can also change while a harness is working on it. The review agent already records the PR state it started with and checks the PR again before emitting its result. That reduces the chance of publishing a review against a changed PR, though source changes and writes cannot generally be made atomic across Fullsend and the source system.

## Decision

For each entity, its source system governs current content, approval state, change rules, and permissions. Fullsend refetches source state when evaluating a harness's predicate. If a source system permits an approved feature to be edited without withdrawing approval, Fullsend treats the edited feature according to the state and approval that source system now exposes. If the source system prohibits the edit or invalidates approval, Fullsend respects that outcome. Approval evidence needed by a predicate must be available from the source system. Fullsend will not impose a separate approval-invalidation rule or maintain an authoritative process position for the entity.

A harness acts only while the entity matches its applicability predicate in the relevant context. Before a consequential result or source change, it should check that the source state it relies on is still applicable, as the review agent does for a PR. Where a source API supports a revision precondition, the harness can use it to narrow the remaining race. The exact consistency and recovery contract for changes across multiple systems remains a separate decision.

Fullsend's durable records remain authoritative for Fullsend's own dispatch history: which observation it accepted, which harness it selected, which attempts it launched, and what outcomes it recorded. Those records support replay, recovery, and audit; they do not change a source entity's approval or assert that a source transition was valid. Kubernetes remains the authority for live Job and Pod status, which Fullsend observes and records for its own recovery. A dashboard presents these records without becoming another authority over entity state.

## Consequences

- Source owners must express approval and edit rules in their systems of record. Fullsend cannot infer that an approval was invalidated by an edit if the source system continues to present the feature as approved.
- A later edit may change which harness predicate matches. Fullsend evaluates the current entity rather than treating an earlier observation, approval, or selected run as a permanent description of the entity.
- Fullsend keeps execution history even when the source entity changes. An old run outcome remains an account of that run, not a claim about the entity's current approval.
- State checks before output reduce stale results, but a change can still occur between a check and an external write. Source preconditions can narrow that window; the broader cross-system race remains to be addressed separately.
