# ADR 01: Stations and harness bundles

**Status:** Proposed  
**Date:** 2026-10-02

## Context

Fullsend runs harnesses. A harness configures one agent's execution, and its trigger can make it eligible for dispatch. We sometimes call harnesses “stages,” while the [historical architecture discussion](https://docs.google.com/document/d/1D6ED3wARvKtaQ71lvTmefoQIIU8e3iBnExch1yLayjs/edit) calls steps in an end-to-end lifecycle “stations.” Those words currently suggest more shared process structure than Fullsend defines.

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) explains how Fullsend selects and runs harnesses. It does not define an end-to-end process. A downstream operator may nevertheless want to assemble, share, and install a set of harnesses, then recommend that set to their users. We need names for the role a harness can fill and for the set an operator distributes before deciding whether any process behavior belongs in Fullsend.

## Decision

A **station** is a named role. A harness may declare that it fulfills a station; a harness does not need a station claim to run. The claim describes the harness's intended role. It does not grant authority or prove that the harness fulfills it. Station is distinct from the existing harness `role` field, which Fullsend uses for credentials.

Fullsend will support a **bundle**: a shareable, installable declaration of a set of harnesses. A bundle has no inherent order. Its harnesses keep their own configuration and dispatch triggers.

A downstream operator may designate a bundle as **golden** for the people they support. That designation expresses the operator's recommendation; Fullsend does not prescribe a universal golden bundle.

## Consequences

- Operators can distribute a coherent set of harnesses without declaring a sequence of work.
- Station claims and bundle membership add descriptive structure. They do not introduce gates, transitions, prerequisite checks, or other process rules. Installing a bundle does not change how its harnesses are selected for dispatch.
- A future decision can define stronger process behavior if it is needed. This ADR leaves the declaration format, installation mechanism, and station naming scope to that work.
