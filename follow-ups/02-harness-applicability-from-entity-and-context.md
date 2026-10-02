# ADR 02: Resolve harness applicability from entity and context

**Status:** Proposed  
**Date:** 2026-10-02

## Context

[ADR 01](01-stations-and-harness-bundles.md) lets an operator install a bundle of harnesses. A bundle gives those harnesses no order and does not select runs. The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) evaluates harnesses in repository and tenant contexts, but it does not yet explain how a source entity reaches those contexts or how variants for different kinds of work apply.

The planned normalized entity model gives dispatch a stable entity identity and kind with an optional event. The proposed harness `domain` describes accepted inputs; `range` describes intended outputs. The relationship between `domain` and the current `trigger` remains to be settled during that work.

## Decision

Tenant configuration identifies the tenant’s registered source scopes and the repository and tenant contexts available for dispatch. A Jira project can be one such source scope; it does not define the tenant.

Dispatch resolves the contexts to evaluate from verified source identity where that identity is sufficient. When a source entity does not identify a repository context, tenant configuration must explicitly link its source scope to the applicable context. Dispatch evaluates each applicable context independently.

Within a context, each installed harness declares its own input applicability through its `domain`. Dispatch evaluates that domain against the normalized entity, including an optional event when relevant. A `range` describes the harness’s intended output; it does not select input work. We introduce no separate work-profile object or bundle-level routing rule.

## Consequences

- One tenant can use different harness variants for different entities and contexts without assigning a workflow to each Jira project.
- Cross-system work, such as a Jira issue intended for a repository, needs an explicit context link when the entity itself cannot establish one. Missing context is an observable configuration outcome.
- Bundles remain installable collections. Their station claims remain descriptive, and harness declarations remain responsible for applicability.
- Existing `trigger` declarations need a compatible migration path to entity-oriented `domain` evaluation. This ADR does not decide whether `trigger` is retained or deprecated.
