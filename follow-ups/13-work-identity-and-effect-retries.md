# ADR 13: Defer work identity and effect retries to in-flight steering

**Status:** Deferred  
**Date:** 2026-10-02

## Context

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) gives a source observation, a selected run, and a Kubernetes execution attempt separate identities. Those identities let Fullsend deduplicate delivery of the same provider activity and recover an uncertain Job creation. They do not establish that two different observations request different work. For example, two source activities may both select a review of the same PR revision.

An agent Job can also make an external change, such as posting a review comment, and then fail before Fullsend records successful completion. Retrying that run can repeat the change. A stable Job name or a handled marker for the observation does not establish whether the external change occurred. The design already calls for source-specific reconciliation before an explicit retry, but it does not define a general identity for the intended work or for each external effect.

Whether a later observation should reuse, replace, or start work depends on how Fullsend steers in-flight runs when source input changes. The same steering decision determines when an earlier effect is still relevant and what a retry must inspect. Choosing work keys or an effect journal independently would settle part of that behavior before the steering contract is defined.

## Decision

Defer a separate work-identity and external-effect idempotency contract to the in-flight steering design. This ADR adopts no new obligation key, effect journal, or automatic retry rule. The steering design must address when distinct observations represent the same work, when changed input warrants new work, and how a run with a possible external effect is reconciled before retry.

Until then, the dispatch architecture's existing observation, selected-run, and attempt identities retain their stated scope. An uncertain external effect remains a recovery case; a failed Job must not be taken as proof that no change was made.

## Consequences

- Coalescing or replacing work across distinct observations remains undefined until the steering design is decided.
- Automatic retries cannot claim protection against duplicate external changes on the strength of Job or observation identity alone.
- The steering design will need to specify the evidence required to recognize prior effects and the outcome when that evidence is unavailable.
