# ADR 04: Govern required station activity without harness dependencies

**Status:** Proposed  
**Date:** 2026-10-02

## Context

[ADR 01](01-stations-and-harness-bundles.md) gives harnesses station roles. [ADR 02](02-harness-applicability-from-entity-and-context.md) resolves their applicability from an entity and context, and [ADR 03](03-exclusive-harness-selection-within-a-station.md) selects at most one owner within each station. An organization may also expect a particular activity at a station. For example, it may expect threat modelling during feature refinement.

The next station needs a refined feature in Jira, a published pull request, or another work product in a system of record. It should not need to know which harness produced that work product. A dependency on a particular prior harness would couple otherwise independent harnesses and could stop useful downstream work when the required activity was performed by a person, another tool, or a replacement harness.

Gate decisions concern work products. A refinement gate may judge whether the feature contains an adequate threat model, but the fact that a named threat-modelling harness ran does not by itself approve the feature. Conversely, an organization may want to check that its tenants actually use a required harness even when that harness's execution is not a condition of a downstream gate. We need a way to govern that expectation without turning harness execution history into a phase dependency.

## Decision

Harness definitions will not require another harness to have executed. Downstream stations may depend on work products and gate decisions recorded in systems of record, but they will not halt or defer solely because a required activity or harness run is absent from an earlier station. Gates decide on the work product, not on the activity that produced it. A gate may require content or a linked artifact as part of the work product; it does not treat a harness run as proof of acceptance.

Fullsend will support two ways to govern required station activity:

1. **Audit.** Tenants are responsible for configuring and operating the harnesses expected to fulfill their stations. Fullsend records and exports the relevant configuration revisions, selection decisions, no-match outcomes, and run outcomes. An organization compares those records with the applicable work products in systems of record to check whether tenants followed its requirements. A compliance finding does not change downstream dispatch eligibility.
2. **Insertion.** Platform administrators control the effective dispatcher configuration so they can insert a required harness into a tenant's configuration or prevent its removal. Tenant changes that would remove or suppress the required registration are rejected before activation. The existing [`repos.yaml` tenant manifest](https://github.com/fullsend-ai/fullsend/blob/main/docs/ADRs/0123-tenant-configuration-from-repos-yaml.md) remains the configuration model; the source and delivery of administrator-controlled configuration are left open by that decision.

These mechanisms can be used together. Insertion controls what is installed and eligible for selection; selection and run records still show whether the activity actually ran or failed. Under ADR 03's at-most-one rule, an administrator-required harness must be the selected owner of its station for applicable work. An additional required harness that should run alongside that owner needs a distinct station role, or its activity must be included in the owner's work.

If ADR 03 is abandoned or replaced with selection that permits multiple harnesses in one station, administrators could instead inject an additional required harness into that station alongside tenant harnesses. That would require its own rules for compatible effects and selection; this ADR does not choose them. Either way, downstream harnesses acquire no dependency on which earlier harnesses executed.

## Consequences

- Tenants can change how they produce a work product without changing downstream harness definitions. Gate decisions remain about the product presented for acceptance.
- Audit mode provides evidence for compliance review but does not prevent a tenant from omitting required activity. Run records alone cannot reveal work that was never discovered. Insertion mode prevents removal through controlled configuration but cannot by itself prove that a run completed or that its output was adequate.
- Platform administrators need authority over the effective configuration and a way to reject tenant changes that defeat mandatory registrations. The exact GitOps and configuration-delivery mechanics remain to be designed.
- Requirements about execution controls, such as logging and sandbox policy, are outside this decision.
