# ADR 05: Own the golden path and tenant customizations

**Status:** Proposed  
**Date:** 2026-10-02

## Context

[ADR 01](01-stations-and-harness-bundles.md) lets an operator recommend a bundle of harnesses as a golden path. Tenants can install harnesses in different contexts, and [ADR 02](02-harness-applicability-from-entity-and-context.md) lets each harness declare the work it accepts. [ADR 03](03-exclusive-harness-selection-within-a-station.md) makes selection among alternatives within a station deterministic. [ADR 04](04-required-station-activity.md) lets platform administrators require a station activity through audit or configuration control.

Those controls do not say who is responsible for the process a tenant ends up running. A tenant could disable the recommended refinement harness, replace it with its own, or change its applicability. The configuration might still be valid for dispatch while some work receives no refinement, or while a later station cannot use the work produced. The platform team cannot reasonably support every tenant-defined combination as if it were the path it published. Tenants also need room to adapt a station without waiting for the platform team to implement each variation.

There is a useful middle ground. The platform team may expose configuration options on its golden harnesses and establish which settings it supports. A tenant using those options should know whether it still has a platform-supported golden path. We need an explicit boundary for that support commitment and for the tenant's responsibility when it changes the path.

## Decision

The platform team owns the good function of the golden path it publishes. That commitment applies when the golden bundle's station harnesses remain enabled and selected as intended, and the tenant uses only configuration options and values that the platform team has declared supported. The platform team defines and maintains that supported configuration surface, including any constraints between options.

A tenant that disables or swaps a golden station harness owns the function of its resulting process. The same applies when the tenant changes the golden harnesses outside their supported configuration surface or adds harnesses that alter their selection or the work products they rely on. The platform team continues to support unchanged golden components, while the tenant owns the integration and behavior of its variation. Adding an independent tenant harness does not make that harness part of the platform-supported golden path.

Fullsend enforces its shared dispatch and security rules for every effective configuration, regardless of support ownership. The platform team owns those rules: registered tenant and source identity, authorization, configuration validation, selection and admission, durable run records, and recovery. Administrator-required registrations from ADR 04 remain effective for tenant variations. Passing Fullsend's validation establishes that the configuration can be dispatched under those rules; it does not certify the tenant's process as a supported golden path.

The effective configuration must make the golden bundle revision, supported tuning, and tenant changes identifiable so the operator and tenant can determine which support commitment applies. This decision does not prescribe the bundle format or configuration delivery mechanism.

## Consequences

- Tenants can adapt or replace station harnesses and take responsibility for the resulting process without asking the platform team to support every variant.
- Tenants can tune golden harnesses within the published supported surface while retaining platform support for the golden path.
- The platform team must document and maintain that surface and investigate failures of the golden path within it. Tenant-specific integrations and changes outside it remain the tenant's responsibility.
- Dispatch validity and process support are separate claims. A configuration can pass Fullsend's checks yet leave work unmatched or produce an unusable work product; selection and run records help identify what happened but do not transfer ownership.
