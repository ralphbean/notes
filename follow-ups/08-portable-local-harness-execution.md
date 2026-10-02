# ADR 08: Keep harness execution portable across local and hosted runs

**Status:** Proposed  
**Date:** 2026-10-02

## Context

The [dispatch architecture](../dispatch-architecture-github-gitlab-kubernetes.html) retains `fullsend run` as the harness runner in repository CI and in the proposed Kubernetes Jobs. Central dispatch adds selection, admission, and durable Job records, but developers also need to run a harness directly in a local checkout. A harness that requires a dispatcher record, Kubernetes Job identity, or a connection to the Fullsend control plane merely to execute would lose that local use.

Hosted execution supplies an entity and its context after source discovery and normalization. A local invocation needs to present the same kind of entity to the harness, including relevant activity when the harness uses it. A separate local input model would make local results less representative of hosted runs. [ADR 02](02-harness-applicability-from-entity-and-context.md) proposes entity and context as dispatch inputs, while [ADR 06](06-harness-domain-and-range-without-universal-artifact-schemas.md) leaves harnesses free of a universal artifact schema. Portability needs to work within those boundaries.

Credentials differ by execution setting. A Kubernetes Job may obtain scoped GitHub App credentials through its workload identity. A person running locally has their own GitHub authentication and may supply Jira credentials. Local execution cannot depend on obtaining a minted App token. The architecture already assigns Job creation, outcome recording, and cleanup to the dispatcher; `fullsend run` executes and exits without writing to the dispatcher database or control plane.

## Decision

`fullsend run <agent-name>` remains directly runnable in a local checkout. The caller supplies the entity, context, and any relevant event in the same representation that a hosted run supplies. Local execution does not introduce a separate entity or harness input schema. The harness uses the same configuration, execution path, output checks, and exit-status meaning in local and hosted settings.

The runner obtains source credentials from its execution setting. A local GitHub run acts with the user's `gh` authentication; local Jira access uses credentials the user provides. It does not request a minted GitHub App token. Hosted runs continue to use the credential path authorized for their environment, including scoped workload credentials for the proposed Kubernetes Jobs. The credential supplied to a sandbox is the credential for that run's setting and scope.

The runner does not need a dispatcher connection, PostgreSQL access, or Kubernetes Job metadata to execute a harness. This applies to local runs and hosted Job payloads. The dispatcher alone creates and manages its selected-run and Job outcome records. A direct local run creates no dispatcher-admitted run record. Its changes to GitHub, Jira, or another source are source facts that later dispatch and gates may evaluate under their normal rules; a successful local exit alone does not establish a shared process decision.

## Consequences

- Developers can run the same harness against a supplied entity before or outside hosted dispatch, using their own source permissions. Local input must remain faithful to the entity representation used by hosted runs.
- Local and hosted runs can exercise the same harness checks, but their source authority may differ. A local success does not prove that the hosted workload identity has the required access, or that a shared gate has accepted the result.
- Harness execution remains independent of the Kubernetes and PostgreSQL coordination records. A replacement coordinator would need to adapt discovery, admission, and outcome accounting while preserving the runner's input and execution contract.
- Local use requires the caller to provide the entity and usable credentials, plus the runtime needed by the harness. Fullsend cannot mint a service identity on the caller's behalf for that invocation.
