# ADR 03: Select one harness per station through deferral order

**Status:** Proposed  
**Date:** 2026-10-02

## Context

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) selects every authorized harness whose predicate matches. Its conflict reservations prevent conflicting runs from executing at the same time, but they do not decide which harness should handle a piece of work. If two matching harnesses make different changes to the same entity, running them one after the other can still produce contradictory results.

[ADR 01](01-stations-and-harness-bundles.md) defines a station as a named role and treats station claims as descriptive. [ADR 02](02-harness-applicability-from-entity-and-context.md) makes each harness's `domain` responsible for input applicability. Harnesses claiming the same station in the same tenant and context are alternative ways to handle that station's work. We need one selected owner for that work so the alternatives do not each act on it. Their declared domains can overlap, though. Requiring authors to keep every domain pair mutually exclusive would mean changing several harnesses whenever one domain changes. Discovering overlap only when dispatch selects work would make a configuration change fail during use.

## Decision

For dispatch, harnesses registered to the same **tenant, context, and station** are alternatives. At most one of them may be selected as the owner of that station's work for an evaluation. This gives station registration a routing effect beyond the descriptive claim in ADR 01.

Each such group declares deferral relationships that establish a complete order among its registered harnesses. A harness defers to every harness above it in that order, including those reached transitively. A single chain is sufficient; each harness need not name every higher-priority harness directly. For example, if `standard` defers to `specialist` and `specialist` defers to `urgent`, the order is `urgent > specialist > standard`.

Fullsend derives each harness's effective domain from its declared domain: it matches only when its own domain matches and no higher-priority harness's declared domain matches. In the example, `specialist` effectively matches `specialist AND NOT urgent`; `standard` effectively matches `standard AND NOT specialist AND NOT urgent`. Authors can change one declared domain without editing the others to restore mutual exclusion.

Before a configuration revision becomes active, Fullsend validates each affected tenant/context/station group: domains must parse and type-check; deferral references must resolve within the group; and the relationships must be acyclic and place every pair in a definite order. An invalid revision is rejected as a whole. Validation constructs the effective domains and guarantees at most one match within each group without needing to prove that arbitrary declared domains are disjoint. The change-time report shows the resulting order and which lower-priority domains may be shadowed by a changed domain.

Selection records the configuration revision, declared-domain evaluations, deferral order, chosen harness, and any harnesses suppressed by that order. A domain evaluation error is an observable selection error, not a false result. If no domain matches, dispatch records an observable no-match outcome.

Different stations are selected independently. Their domains may overlap, and this decision does not assert that their effects are compatible or that they cannot form a loop. Existing conflict reservations still govern simultaneous execution.

## Consequences

- Alternative harnesses within one station have deterministic selection even when their declared domains overlap. A lower-priority harness can serve as a fallback without copying exclusions from higher-priority domains.
- A change to a higher-priority domain can redirect work away from lower-priority harnesses. The order and possible shadowing need review when the configuration changes; the record of a selection explains which rule won.
- Adding, removing, or reordering a harness can invalidate the group's complete order and must pass validation before activation.
- This is an **at-most-one** rule, not a promise that a station has a matching harness or that selected work is authorized to pass a process gate. Cross-station contradictions, prerequisites, approvals, and loops require separate decisions.
