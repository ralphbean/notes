# ADR 07: Preview routing changes with local configuration lint

**Status:** Proposed  
**Date:** 2026-10-02

## Context

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) evaluates harness rules against a normalized source observation and its applicable context. [ADR 02](02-harness-applicability-from-entity-and-context.md) proposes a harness `domain` for input applicability, while the relationship between `domain` and the existing `trigger` remains open. [ADR 03](03-exclusive-harness-selection-within-a-station.md) makes selection among harnesses in one station deterministic through a deferral order. It also requires expressions and that order to be valid before a configuration revision becomes active.

Those checks cannot tell an author whether a valid edit routes the intended work. If an `urgent` harness outranks `specialist` and its predicate is broadened, it may take observations that `specialist` previously handled. The resulting configuration still has one owner for the station. A narrower predicate may leave observations without an owner. An author editing `repos.yaml` needs to see examples of both outcomes while the change is still local, including observations that did not previously select this station.

The dispatcher already records accepted observations and selected runs, but the proposed observation record has an entity reference and source revision rather than the complete input evaluated by the selector. Refetching that entity later may return different data. A replay of recent observations therefore needs the selection input preserved as it was evaluated.

## Decision

Fullsend will provide a local configuration lint invocation for an edited `repos.yaml`. The command will parse and type-check routing expressions and apply the structural validation required by ADR 03, using the same routing semantics as dispatch. A structural error fails lint. The command name and output format are left to implementation.

When the author has read access to the dispatcher, lint will obtain a bounded set of recent accepted observations for the affected tenant and context. The set will include observations handled by the affected station, observations handled elsewhere, and no-match observations where available. The dispatcher will preserve the normalized selection input and context used for each observation, together with its configuration revision and selection outcome, for a bounded retention period. It will expose these examples through a read-only, access-controlled interface. Credentials must not enter the retained input. Any content needed to replay a predicate must remain faithful to the evaluated input; the author-facing report will minimize or redact sensitive values without changing the predicate result.

Lint will evaluate each preserved input against the active configuration and the proposed local configuration without launching work or changing dispatcher state. For the affected station, its report will show whether each proposed predicate matches and which harness would effectively own the observation after deferral. It will identify examples newly caught, no longer caught, still caught, and still unmatched, including changes of owner between harnesses. The report will show the sample's bounds and when a category has no available example. An author can use the reported cases to inspect the predicate before committing the file.

If the dispatcher or preserved examples are unavailable, lint will report that the observation preview was skipped. Structural validation will still run. The preview is advisory; this ADR does not add a CI or deployment approval gate beyond the configuration validity rules already established by ADR 03.

## Consequences

- Authors can inspect concrete positive and negative examples from recent dispatch activity before committing a routing change. Showing both predicate matches and effective owners makes deferral-induced changes visible.
- The dispatcher must retain enough normalized input to replay selection, not just an entity reference. Bounded retention, read access, and redaction become part of operating the preview.
- The preview describes behavior for its sampled observations under the proposed revision. It cannot prove that every possible input is covered, that stations interact safely, or that a process prerequisite has been met. A no-match example may be intentional.
- An unavailable or sparse sample limits the report. Lint must make that limit visible rather than treating missing examples as evidence that no routing change occurs.
