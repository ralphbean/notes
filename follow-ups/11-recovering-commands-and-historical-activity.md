# ADR 11: Recover commands and historical activity from source evidence

**Status:** Proposed  
**Date:** 2026-10-02

## Context

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) lets a poller discover source activity and lets the dispatcher refetch an entity before selecting work. [ADR 10](10-periodic-eligibility-reevaluation.md) also calls for periodic visits to in-scope entities. Those visits can reevaluate a condition that is still true in the current entity state. They cannot necessarily recover an action that happened earlier.

For example, a current PR snapshot may show that a label is present, but it may not show when the label was added or who added it. A `/fs-*` comment is a separate command even if another comment has the same text. An approval may have been changed or withdrawn. If Fullsend misses a notification, it needs the source's activity record to recover these actions and the identity of the actor. If that record is unavailable, the current snapshot is insufficient evidence that the action occurred.

The source system governs entity content, approval state, and permissions under [ADR 09](09-source-system-authority-for-entity-state.md). Fullsend's observation records preserve what it accepted for dispatch; they do not create an independent approval authority.

## Decision

Source adapters distinguish eligibility that can be reevaluated from current entity state from commands and historical decisions that require source activity evidence. For the latter, an adapter must be able to rediscover the activity within a declared retention window, identify it by a stable provider activity ID, and supply the actor and action evidence needed for authorization. Fullsend commits the accepted observation and its covered checkpoint together. A webhook may prompt earlier discovery, but losing its delivery is recoverable only while the adapter can still retrieve the activity.

An adapter must not claim support for command- or historical-decision-triggered work if it cannot provide that recovery contract. A predicate over the source's current approval state can still be reevaluated from current state. If activity falls outside the recoverable window or required evidence is missing, Fullsend records a visible coverage gap for intervention. It does not infer the missing command or historical decision from the current entity snapshot.

This is a source-adapter and entity-input contract. Provider-specific queries, retention periods, and storage fields are implementation choices.

## Consequences

- Periodic entity sweeps can recover state-based eligibility, while activity discovery recovers distinct commands and historical decisions.
- Each adapter that supports historical actions must state how far back it can discover them. Outages longer than that window may require intervention.
- Polling and webhooks can deliver the same activity without creating two observations when they use the same provider activity ID.
- Fullsend retains dispatch evidence after the source entity changes, without treating that evidence as the source's current approval state.
