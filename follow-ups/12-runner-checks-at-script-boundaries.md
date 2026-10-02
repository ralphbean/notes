# ADR 12: Check source assumptions at runner script boundaries

**Status:** Proposed  
**Date:** 2026-10-02

## Context

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) selects a harness from source data and later starts `fullsend run` in an independent Job. The data used to select the run may include a Jira entity, a Git revision, an artifact, and an actor's permissions. Those facts are read through separate services, so they cannot be treated as one atomic snapshot. A fact can change while the run waits for admission or while the agent works.

For example, a review may begin against one PR revision and finish after another commit is pushed. A post script that publishes the review can then report a result for code it did not examine. Refetching before admission reduces the delay between selection and launch, but it does not protect the period between launch and publication. Today, an agent author must put this kind of check in the agent's own post script. That makes the protection depend on each author's implementation.

The runner has a boundary before the pre script starts and another before the post script starts. These are useful places to check whether the source assumptions needed for the run still hold. The source systems continue to govern current entity state and permissions, as described in [ADR 09](09-source-system-authority-for-entity-state.md). A runner check does not make reads across those systems atomic, nor does it prevent a source change immediately after the check.

## Decision

The `fullsend run` contract will support runner-owned validation of the source assumptions relevant to a run before invoking its pre script and again before invoking its post script. The run must carry enough source identity and version evidence for the runner to perform those checks through the appropriate adapters. A check that finds the run's assumptions invalid must prevent the next script from starting and produce a visible outcome; it must not silently continue with the stale input or publish the stale result.

This ADR establishes the runner's responsibility and the two check boundaries. The in-flight steering design will determine the detailed invalidation, cancellation, and restart behavior, including which changes warrant continuing or replacing a run. It will also define the exact evidence and adapter interfaces. This ADR does not set a universal freshness interval or require a cross-provider snapshot.

Until runner support exists, agent authors remain responsible for checking assumptions before consequential actions in their scripts. Even with runner checks, a script that writes to a source may need a source-specific revision precondition or its own final check; the runner cannot close the race between a boundary check and a later external write. The policy for writes when a provider offers no conditional update remains open.

## Consequences

- The same runner behavior can protect local, CI, and Kubernetes executions without giving the runner a dependency on dispatcher or PostgreSQL records.
- A change detected before the pre script avoids starting work against assumptions already known to be stale. A change detected before the post script prevents publication of a result based on invalidated assumptions.
- Source adapters and run inputs will need to expose comparable identity and version evidence for the facts a run relies on. Missing evidence and check failures need visible outcomes; their detailed handling belongs to the steering design.
- Boundary checks reduce stale work but do not provide an atomic snapshot or guarantee that an external write cannot race with a later source change.
