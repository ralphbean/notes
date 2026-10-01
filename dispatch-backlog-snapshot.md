# AISDLC-37 direct-child backlog snapshot

Retrieved 2026-10-01. Jira status is not proof of implementation. Query: `parent = AISDLC-37 ORDER BY key ASC`.

## AISDLC-56 — Managed Fullsend central service MVP

Status: Closed. Source: https://redhat.atlassian.net/browse/AISDLC-56

Background
Fullsend can be installed alongside an individual repository, where repository owners manage configuration, event delivery, credentials, and execution infrastructure. Some organizations need those capabilities operated centrally across repositories, forge organizations, and work trackers.
Summary
Build the first managed Fullsend service for a pilot tenant. For now, product changes and service components live in fullsend-ai/fullsend; revisit a separate repository only if the implementation shows a clear need.
Tenant desired state uses the existing repos.yaml format through repos.Manifest, as decided in ADR 0123. The manifest needs to represent repository-less contexts alongside repository-bound contexts. The source of truth and delivery mechanism remain open.
Scope
In scope
Define and implement the initial service architecture in fullsend-ai/fullsend, extending existing Fullsend components where they fit.

Extend repos.Manifest for repository-less and repository-bound execution contexts.

Extend and refactor existing polling and dispatch paths to discover work and launch runs for configured contexts.

Stand up a pilot instance on a cluster we provision.

Demonstrate a pilot team configuring the service through its manifest source and running agents in the cluster.

Collect pilot feedback.

Out of scope
Moving implementation to fullsend-ai/platform before the pilot shows a clear need.

A new tenant REST API or Kubernetes CRD as the configuration model.

Production-scale hardening and operationalization.

Tekton integration; a Kubernetes Job is sufficient for the MVP, while the launcher should allow other implementations.

Operating a central service for upstream or as a public-good service.



## AISDLC-57 — Tenant Configuration with repos.Manifest

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-57

Background
The managed Fullsend service needs one desired-state representation for tenant-level and repository-level execution contexts. The existing repos.yaml format and repos.Manifest describe Fullsend configuration across repositories. ADR 0123 decides to reuse that manifest directly instead of introducing a tenant REST API or Kubernetes resource model.
Some managed work has no repository context. The manifest therefore needs an extension for tenant-level runnable contexts while preserving the existing repository configuration model.
Summary
Extend repos.Manifest and the repos.yaml format for managed tenant configuration. Represent repository-less execution contexts alongside registered repositories, each with its own repository context. Keep the implementation in fullsend-ai/fullsend while we learn whether a separate repository is needed.
The service should load and validate the manifest using the existing Go model. Its source of truth and delivery mechanism remain open.
Scope
In scope
Define and implement the smallest manifest extension needed for repository-less runnable contexts.

Preserve the distinction between tenant-level contexts and independent repository contexts.

Validate the Fullsend configuration for each context and provide actionable errors.

Keep existing per-repository manifests and CLI behavior working.

Provide a filesystem-based way for later components, including pollers and dispatchers, to load the current manifest.

Document the extended manifest with tenant-level and repository-level examples.

Out of scope
Candidate discovery, agent selection, and run dispatch.

A new REST API, Kubernetes CRDs, or API-specific status resources.

Choosing the authoritative manifest source or deployment mechanism.

Site or tenant override layers.

Namespace, identity, quota, or external-credential provisioning; onboarding is tracked separately.

Design considerations
Reuse existing manifest parsing and validation where practical.

Keep repository contexts independent; do not merge their configuration into the tenant context.

Make repository-less contexts explicit without inventing a repository name.

Keep existing per-repository consumers compatible as the schema grows.

Leave future API or CRD evaluation open until the pilot demonstrates a need.

Success criteria
A manifest can describe repository-less and registered repository contexts.

The existing repos.Manifest loader and validator accept the new representation and reject invalid configuration with actionable errors.

Existing per-repository behavior remains compatible.

Later service components can load the manifest from the filesystem without an API server.



## AISDLC-63 — PoC Fullsend Run on OpenShift through remote OpenShell gateway

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-63

Background
The managed Fullsend service needs to launch fullsend run on OpenShift. A future central dispatcher should be able to create a Kubernetes Job for a selected agent run.
OpenShift's Security Context Constraints prevent an ordinary workload from starting OpenShell's sandbox machinery directly. OpenShell supports a Kubernetes-backed gateway which owns sandbox creation in the cluster. This gives us a path where the fullsend run Job remains an ordinary workload and asks the remote gateway to create and operate its sandbox.
Summary
Build a proof of concept which runs fullsend run in a Kubernetes Job on OpenShift and uses a separately installed OpenShell gateway with Kubernetes support. Adapt Fullsend's sandbox-launching path as needed so the Job selects the remote gateway, creates a sandbox through it, transfers its inputs and results, and cleans up the sandbox when the run finishes.
The PoC should demonstrate the execution boundary needed by a later central dispatcher. The dispatcher itself is not part of this Feature; creating the Job resource manually or through a small test harness is sufficient.
Scope
In scope
Install the Agent Sandbox controller and the OpenShell gateway with Kubernetes support on an OpenShift test cluster.

Configure the OpenShell sandbox service account with the SCC required by the evaluated OpenShell release.

Build or identify an image suitable for running fullsend run in an OpenShift Job.

Define and create a Job resource which executes one representative fullsend run invocation.

Keep the Fullsend Job under an ordinary restricted SCC while the OpenShell gateway creates the sandbox through its Kubernetes driver.

Modify Fullsend where necessary to select and authenticate to the remote gateway without attempting to start local OpenShell infrastructure.

Transfer configuration, repository content, and results between the Job and sandbox through OpenShell's gateway relay and upload/download support.

Use OpenShell's sandbox-owned workspace PVC; do not permit the Fullsend Job to mount that PVC.

Use static GitHub App and inference credentials supplied to the PoC through Kubernetes Secrets (WIF is out of scope).

Demonstrate sandbox and PVC cleanup after success and failure.

Record the OpenShift privileges, gateway authentication shortcut, storage requirements, and remaining production gaps found by the PoC.

Out of scope
A central dispatcher or tenant-driven Job creation.

Tekton integration or general pipeline orchestration.

Mint integration or mint access from the workload.

Workload identity federation for GitHub App access.

Workload identity federation for inference access.

Intra-stage multi-agent orchestration or shared PVC contracts between phases.

Production credential rotation or delivery; static credentials are sufficient for this PoC.

Production hardening, high availability, capacity planning, or general multi-tenant isolation.

Site or tenant configuration overrides.

Success criteria
This Feature is complete when:
an OpenShift Job running under an ordinary restricted SCC successfully executes fullsend run;

Fullsend uses a pre-deployed OpenShell gateway to create its sandbox instead of starting sandbox infrastructure inside the Job;

the agent receives its configuration and repository content, performs representative work, and returns results through OpenShell without a PVC shared with the Job;

static GitHub App and inference credentials from Kubernetes Secrets are sufficient to complete the demonstrated run;

successful, failed, and cancelled runs have demonstrated cleanup behavior for their sandbox resources and workspace PVCs; and

the proof of concept records the changes and unresolved security, authentication, storage, and tenancy work required before a central dispatcher can depend on this execution path.

any changes necessary to fullsend itself and merged and available

References
Managed Fullsend central service MVP

Platform Tenant API

OpenShell OpenShift installation

Tekton Agent Task with OpenShell Sandbox Runtime

Intra-Stage Multi-Agent Orchestration



## AISDLC-64 — PoC Workload Identity Federation from OpenShift

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-64

Background
The OpenShift execution proof of concept in AISDLC-63 uses static credentials so it can focus on running fullsend run through a remote OpenShell gateway. A managed service cannot rely on distributing long-lived GitHub App or inference credentials to every run workload.
OpenShift can project service-account identity into a Pod, but we still need to determine how an external service should trust that identity and turn it into narrowly scoped authority. For repository access, that may require a private mint which accepts claims only from approved clusters, namespaces, and service accounts, then limits tokens to approved repositories.
Summary
Build a proof of concept for Workload Identity Federation from a fullsend run Pod on OpenShift. Demonstrate either inference access from the Pod's projected identity or a repository-scoped token from a private mint. Do not use or modify the public mint.
Keep the proof-of-concept code and resulting ADRs in fullsend-ai/fullsend while we evaluate whether this work needs a separate repository. Document the identity and placement contract that a future dispatcher must follow.
Scope
In scope
Identify the projected service-account token, issuer, audience, and claims available to a fullsend run Pod.

Determine how a relying party validates the cluster issuer and distinguishes authorized workloads.

Evaluate whether authorization should bind to cluster, namespace, service account, tenant, repository, or a combination.

Demonstrate WIF for inference or a private mint for repository access, limiting authority to resources needed by the PoC.

Run fullsend run using federated credentials instead of the corresponding static credential.

Document token audience, lifetime, replay protections, rotation, revocation, and audit behavior.

Record trust and any placement conventions as ADRs in fullsend-ai/fullsend.

Describe the contract a later dispatcher must satisfy when creating a run workload.

Out of scope
Implementing the central dispatcher.

Modifying or depending on the public mint.

Demonstrating both WIF paths; one is sufficient.

Production-ready mint operations, general tenant onboarding, namespace provisioning, or service-account lifecycle automation.

Tekton integration or solving every external provider's federation model.

Design considerations
Treat projected service-account tokens as short-lived workload assertions, not bearer credentials to external services.

Keep workload authentication separate from authorization to repository or inference resources.

Prefer explicit issuer, audience, subject, and claim checks.

Do not assume namespace or service-account conventions until the PoC shows which claims the relying party can validate.

Ensure the design can distinguish tenants sharing a cluster.

Success criteria
A fullsend run Pod on OpenShift uses projected identity to obtain inference access or a repository-scoped token from a private mint.

The path completes representative Fullsend work without the corresponding static credential in the Pod.

An unauthorized trust-boundary claim is rejected, and issued authority is limited to resources needed by the demonstration.

The PoC records the validated claims, token lifetime, and authorization checks.

ADRs are merged in fullsend-ai/fullsend describing the WIF design and any dispatcher placement constraints.

Remaining production mint, federation, and dispatcher work is identified.



## AISDLC-65 — Poll and dispatch managed Fullsend runs

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-65

Background
The managed Fullsend service needs to turn tenant configuration into running agent work. The extended repos.Manifest will describe repository-less and repository-bound contexts, while the OpenShift execution path defines how a selected agent can run through fullsend run in a Kubernetes Job.
Fullsend already has polling, event normalization, routing, and dispatch code for repository installations. Extend and refactor those paths for managed operation while keeping the existing per-repository behavior usable. Pollers discover candidate work without selecting agents; dispatch resolves current manifest configuration, evaluates applicable contexts, and launches selected runs.
Summary
Extend and refactor the existing poller and dispatcher in fullsend-ai/fullsend for managed Fullsend runs. Load tenant and repository contexts from the filesystem through repos.Manifest. The poller discovers candidate work from one source needed by the pilot and hands off candidate references. The dispatcher resolves each candidate against the current manifest, evaluates applicable contexts, and creates a Kubernetes Job for each selected run.
A webhook receiver can later publish to the same candidate handoff; it is not part of this Feature.
Scope
In scope
Extend and refactor existing polling and dispatch paths for managed runs while preserving the repository installation flow.

Load and validate current configuration through repos.Manifest from a filesystem path; leave manifest source and delivery open.

Implement polling for one source needed by the pilot and define a source-neutral candidate handoff.

Keep discovery separate from selection and execution; resolve current entity state before trigger evaluation.

Evaluate candidates against the tenant's repository-less context and each applicable repository context independently.

Select every matching agent while preserving its tenant and execution context.

Create a Kubernetes Job for each selected run using the path proven by AISDLC-63; carry sufficient context to run fullsend run and correlate results.

Demonstrate acknowledgement, retry, duplicate-delivery, poison-candidate, and restart behavior sufficient for the pilot.

Expose signals for polling failures, candidate backlog, dispatch failures, and Job creation.

Demonstrate that manifest updates are observed without restarting or manually reconfiguring components.

Run one poller and one dispatcher for the pilot. Define a candidate handoff that can deliver directly now and use a shared inbox later, while both paths share discovery and dispatch logic.

Out of scope
A tenant REST API or Kubernetes CRDs as the configuration source.

A webhook receiver or source-specific webhook processing.

Every forge or work tracker in the first delivery.

Tekton integration, site or tenant override layers, selection optimizations, and production-scale hardening.

Active-active poller or dispatcher replicas and a shared candidate inbox; tracked in AISDLC-147.

Design considerations
Pollers discover candidates; they do not select agents or launch workloads.

Treat the manifest as desired configuration; do not copy tenant configuration into poller-specific configuration.

Preserve independent tenant and repository contexts.

Make Job creation idempotent enough to control duplicate runs, and keep credentials out of candidate messages.

Preserve existing per-repository dispatch behavior as managed operation is added.

Give candidates stable identities scoped by tenant, registered source, and canonical entity, with provider revision or activity identity where available. Keep credentials and source content out of the handoff.

Advance source progress only after the chosen handoff accepts responsibility for the candidate. In direct mode, a local output file alone is not confirmation that the run was scheduled.

Keep handoff, dispatch selection, and Job launching separate so a shared inbox can later add claims, leases, and retries without changing authorization or CEL behavior. Give selected runs stable identities so redelivery can find an existing Job.

Success criteria
A pilot manifest describes the tenant and repository contexts needed by the pilot.

The poller discovers representative candidate work from the chosen source.

The dispatcher normalizes each candidate and evaluates it against every applicable manifest context.

Every matching agent produces a correlated Kubernetes Job using the fullsend run path from AISDLC-63.

At least one automatically discovered candidate completes a managed run.

Configuration changes are observed without component restarts.

Retry, duplicate delivery, dispatcher restart, and poison-candidate outcomes are demonstrated and documented.

Operators can distinguish polling, candidate-delivery, selection, Job-creation, and workload failures.

The pilot works with one poller and one dispatcher without a required database or broker. The candidate handoff and run identity are documented well enough for AISDLC-147 to add a shared inbox and active-active replicas.



## AISDLC-66 — Receive webhooks for managed Fullsend dispatch

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-66

Background
AISDLC-65 establishes polling and dispatch for the managed Fullsend service. Polling provides the correctness path, but some sources can notify the service when candidate work changes and reduce discovery delay and repeated source queries.
The webhook receiver should live in fullsend-ai/fullsend for now and publish into the candidate handoff used by polling and managed dispatch.
Summary
Add a webhook receiver which authenticates supported source-system deliveries and publishes candidate references through the same handoff used by pollers. The dispatcher should not need a separate selection or execution path for webhook-originated work.
AISDLC-230 keeps the Fullsend clusters private, with no public ingress. This Feature also owns the delivery path from the pilot source to a webhook receiver inside that boundary, including any networking or relay needed to make delivery work.
Shape this Feature in more detail after the candidate contract and managed dispatch path from AISDLC-65 are proven.
Scope
In scope
Implement the webhook receiver in fullsend-ai/fullsend.

Receive webhook deliveries from at least one source required by the pilot and authenticate them using the source's supported mechanism.

Design and provision the webhook delivery path for the pilot source, including any required private connectivity or relay. Define where incoming requests are authenticated, how they reach the private Fullsend receiver, and who operates the delivery infrastructure. Do not expose a public endpoint on the Fullsend cluster.

Identify the tenant or registered source configuration associated with a delivery.

Translate accepted deliveries into the candidate contract established by AISDLC-65.

Handle redelivery and malformed or unauthenticated requests with observable outcomes.

Demonstrate that a webhook candidate reaches the existing dispatcher and launches the same managed run as a polled candidate.

Use the source-neutral candidate reference and handoff contract from AISDLC-65, with stable provider observation identity where available, so a later shared inbox can converge webhook and poll observations.

Out of scope
A separate webhook-specific dispatcher or execution model.

Replacing polling as the correctness and recovery path.

Every forge and work tracker in the first delivery.

General API gateway, ingress, or event-platform work beyond the delivery path needed for the pilot source.

Active-active poller or dispatcher replicas and a shared candidate inbox; tracked in AISDLC-147.

Success criteria
An authenticated pilot-source webhook produces a candidate through the shared handoff; the existing dispatcher processes it; duplicate delivery has a controlled outcome; and polling can rediscover work when webhook delivery is missed.
A webhook from the pilot source reaches the private Fullsend receiver through the provisioned delivery path without public ingress to the cluster. The trust boundary, authentication, forwarding behavior, and operational owner of that path are documented.
The pilot path works without a required database or broker and preserves candidate identity for AISDLC-147 to add a shared inbox later.


## AISDLC-70 — Onboard and authorize managed Fullsend tenants

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-70

Background
The managed Fullsend service needs an explicit process for establishing a tenant and deciding who can change its configuration. With configuration represented by repos.yaml through repos.Manifest (ADR 0123), onboarding no longer provisions tenant API resources. It connects an approved team with the source and authorization mechanism used to deliver its manifest to the service.
Summary
Define and implement the first onboarding process for managed tenants in fullsend-ai/fullsend. An approved team should receive a tenant identity, an initial manifest describing its runnable contexts, and a clear way for its administrators to manage that manifest.
The manifest source of truth and delivery mechanism remain open. The onboarding procedure should use the mechanism selected for the pilot and make approval, initial configuration, authorization, ownership, and handoff explicit and repeatable.
Scope
In scope
Define how a team requests a managed tenant, supplies administrator identities, and receives approval or rejection.

Establish a tenant identity and create initial repos.yaml configuration using the repos.Manifest model from AISDLC-57.

Grant tenant administrators access to the chosen manifest source and document how it controls configuration changes.

Distinguish tenant administrators, service components, and platform operators where needed.

Prevent tenant administrators from reading or changing another tenant's configuration through the chosen source and authorization mechanism.

Provide an auditable, repeatable onboarding procedure and demonstrate handoff to a pilot tenant.

Document how administrator membership is changed or recovered after handoff.

Out of scope
Fully automated signup, billing, entitlements, or external organizational approval.

Repository and external-system credential federation.

Implementing the poller, dispatcher, or run workload.

A tenant REST API, Kubernetes CRDs, or API-specific RoleBindings.

A general-purpose identity-management system or production-scale lifecycle automation.

Design considerations
Keep bootstrap authority held by platform operators separate from ongoing authority delegated to tenant administrators.

Use groups or maintainable access bindings in the chosen manifest source instead of permissions tied permanently to one person.

Make tenant placement and authorization agree so cross-tenant access is difficult to grant accidentally.

Record who approved a tenant and who received initial authority.

Keep the manifest delivery path open until the pilot selects a source of truth.

Success criteria
A pilot team can request onboarding and an authorized platform operator can approve it through a documented process.

The process establishes the tenant identity, initial manifest, and access needed by tenant administrators.

A tenant administrator can update configuration through the chosen source without platform-operator credentials, and cannot access another tenant's configuration.

Service-component and platform-operator access needed by the MVP is represented and tested.

Administrator membership can be updated or recovered through a documented and auditable path.



## AISDLC-143 — Define centralized data source and MCP access models

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-143

Market problem
Teams adopting the centralized AI SDLC deployment have their own context repositories, restricted documents, data sources, and MCP tools. A skill that points to a directly readable repository covers some context, but cannot grant access to protected files or services. Without a defined access model, customers cannot safely bring the context and tools needed for useful workflows, and the platform cannot consistently enforce tenant boundaries.
Proposed capability
Define the customer-facing and platform access models for external data sources and MCP servers in the centralized deployment. Specify how a tenant registers sources and tools, grants access to particular users, agents, or workflows, and supplies service-account credentials to the execution environment without exposing secrets to unauthorized workloads.
The model should cover:
Supported source types and access patterns, including repository-backed context through agent skills, restricted documents and files, and credentialed MCP servers.

Tenant ownership, identity, authorization scope, and least-privilege access to individual sources, files, tools, and operations.

Service-account credential provisioning, storage, injection, rotation, revocation, and auditability.

How access is propagated to stations and agents through OpenShell providers, including HyperShell constraints.

Onboarding and extension contracts so teams can add context sources and MCP tools through a documented path.

Strategic value
This is a must-have for customer adoption of a centralized deployment: customers need their protected data and approved tools available to agents while retaining control over who can use them. A security-reviewed model gives platform and extension teams a common implementation contract.
Success criteria
Architecture and Security agree on a documented tenant, identity, authorization, and credential lifecycle model for data sources and MCP servers.

The model demonstrates access to a restricted file or document and a credentialed MCP tool using a tenant-managed service account.

Cross-tenant access and access without the required source or tool permission are denied; secret values are not exposed to unauthorized users.

Source and tool access can be granted, audited, rotated, and revoked through documented workflows.

Teams have a documented pattern for repository context via skills and a clear path for sources requiring authentication.

Planned workstreams
Capture customer source and MCP use cases, including file-level and tool-level requirements.

Define tenant boundaries, identities, authorization policy, and enforcement points.

Design credential storage and delivery to OpenShell and HyperShell workloads.

Define registration and configuration interfaces for customer-owned sources, MCP servers, and skills.

Review with Security and validate against representative restricted-data and MCP scenarios.

Open design questions and dependencies
Evaluate HashiCorp Vault and External Secrets Operator, or an equivalent approved solution, for tenant-managed secrets; this is a candidate, not a decision.

Confirm the OpenShell provider contract and HyperShell limitations.

Compare RHAI's MaaS token and third-party-key approach. Coordinate with Eder Ignatowicz, Ralph Bean, and the relevant RHAI and Security contacts, including JJ Aggarwal and Lindani LD Phiri.

Clarify the boundary between skill-based repository context and credentialed data and MCP integrations.

Source
Architecture team discussion supplied by Ella Shulman on centralized data source and MCP access.


## AISDLC-144 — Bring layered configuration into line with ADR 0069

Status: In Progress. Source: https://redhat.atlassian.net/browse/AISDLC-144

Why
ADR 0069 defines the file layers: repo-specific .fullsend/config.yaml overrides the repo-local .fullsend/config.base.yaml, and both fall back to compiled defaults. The Sep 28 configuration meeting made the full source order explicit: CLI flags, local config, base config, environment variables for compatibility, then code defaults.
Today, workflows can pass CLI flags and environment variables that shadow file settings, leaving configuration files ineffective in real runs. Some settings may also lack a config field. We need to trace each setting through the workflows, binary versions, and config layers, then move it safely so centrally managed baselines take effect without breaking repositories that still rely on older workflows or pinned binaries. Central administrators should be able to distribute repo-local base files while each repository keeps its own overlay.
Scope
Inventory each setting across CLI flags, environment variables, .fullsend/config.yaml, .fullsend/config.base.yaml, compiled defaults, shared workflows, and runtime or installer consumers. Identify values that cannot currently be represented in configuration.

Define and document source precedence and per-field merge, fallback, and requiredness rules. Update ADR 0069 or its supporting reference if needed to make the full source order explicit.

Add config fields and accessor behavior for settings that should be centrally managed but have no file representation; align consumers so workflows and environment variables do not unexpectedly shadow file layers.

Plan compatibility changes across workflow and binary version combinations. Keep environment-variable paths where older clients need them, and remove workflow bindings only when the supported upgrade path will not strand existing users.

Verify the fleet update path distributes a repo-local config.base.yaml copy and preserves each repo's .fullsend/config.yaml overlay. Use the existing manifest and repository convergence workflow; runtime commands continue reading the local copy.

Make changes field by field in small pull requests. Use the behavior-test framework for end-to-end precedence checks where it fits, and supplement with branch or vendored-repo checks where needed.

Update config references and relevant workflow, install, and migration documentation as behavior changes.

Acceptance criteria
A reviewed field-level inventory accounts for every supported setting, its available sources, effective precedence, consumers, merge and fallback behavior, and compatibility requirements.

Effective precedence is CLI flags → .fullsend/config.yaml → .fullsend/config.base.yaml → environment variables retained for compatibility → compiled defaults, with field-specific merge and requiredness rules documented and implemented.

Every setting intended for central or repository configuration has a file representation; missing fields are added or their exclusion is explicitly justified.

End-to-end tests verify that workflow-provided values do not silently override higher-priority file settings, and that legacy environment-variable paths continue to work during the supported migration window.

The repository convergence workflow distributes or refreshes the base file locally without overwriting the repo overlay; runtime does not depend on fetching the base file remotely.

Existing installs without a base file and supported older workflow/binary combinations continue to work through the migration. Documentation explains the precedence and upgrade path.

Boundaries
This work follows ADR 0069's per-repository installation model and does not restore deprecated per-organization configuration. It keeps the base file in each target repository; central distribution updates those copies. It does not define a remote configuration service or reopen security and inference authorization decisions that ADR 0069 explicitly defers.
References
ADR 0069 — Ready-made configuration presets for simplified installation

Sep 28, 2026 meeting notes and transcript

Layered configuration slides



## AISDLC-146 — Run Fullsend agents without a target repository

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-146

Background
A managed Fullsend service needs to run some agents whose work has no target code repository. A Jira-only task or a tenant-level analysis can still have a configuration source, an execution host, an input, and an output. Today fullsend run requires --target-repo, and several runtime paths assume that directory is a repository.
The immediate reason to separate these concepts is to make those agents runnable in OpenShift Jobs once the OpenShift execution path is ready. We can establish the runtime contract first. GitHub Actions and GitLab CI are useful proofs: each requires a repository or project to host a workflow, but that host should not become the agent's target by accident.
Summary
Enable fullsend run to execute a harness explicitly without a target repository. Give the agent a scratch working directory and the configured harness content, with no implicit target checkout, repository metadata, or repository-scoped authority. Keep repository-targeted runs working as they do today.
Implement this in fullsend-ai/fullsend. A direct local invocation and representative GitHub Actions and GitLab CI workflows should show that the run's host is independent of its target context. This provides a runtime capability for later managed polling, dispatch, and OpenShift execution.
Scope
In scope
Add an explicit repository-less invocation mode to fullsend run, mutually exclusive with --target-repo. An omitted or mistyped repository path must not silently select this mode.

Load the harness, agents, skills, and configuration from --fullsend-dir while keeping that directory distinct from a target repository.

Provide a scratch workspace for runtimes that need a working directory. Do not copy the workflow host repository into the sandbox as target code or inject its repository instructions as target context.

Identify and handle runner assumptions about Git checkout, repository paths, environment variables, pre/post scripts, validation, artifacts, and cleanup. Repository-dependent harness behavior should fail clearly rather than use a fabricated repository.

Keep workflow host metadata, task or event source, target context, status/output destination, and credential scope distinct. Rename variables and types when their current names blur those roles. Add small insulating interfaces where they help separate infrastructure-specific behavior from target-repository behavior.

Ensure GitHub Actions or GitLab CI hosting does not implicitly select the workflow repository as the target, status destination, or GitHub App token scope. Repository access, if an agent needs it as an external resource, must be requested explicitly.

Show a representative repository-less harness completing a direct local run and producing an inspectable result.

Show the same runtime mode invoked directly from a GitHub Actions workflow and a GitLab CI pipeline. In each case, the repository/project may host the workflow or configuration, while the agent run has no target repository.

Preserve the behavior of existing repository-targeted fullsend run callers.

Out of scope
Extending repos.Manifest or choosing a managed tenant configuration schema; that can follow once the runtime behavior is clear.

Polling, automatic agent selection, or the managed dispatcher in AISDLC-65.

Creating OpenShift Jobs or integrating a remote OpenShell gateway; AISDLC-63 covers the OpenShift execution proof of concept.

Reworking the existing repository-centered reusable dispatch workflows to route repository-less events.

Making every built-in agent work without a repository, or redesigning all forge abstractions at once.

Design considerations
Treat the workflow repository/project as an execution host and possible configuration source. It is a target repository only when the caller explicitly chooses it.

A runtime may still need a working directory when there is no repository. Give it a scratch directory without assigning Git semantics to it.

Current GitHub/GitLab environment detection is useful for CI integration but should not determine whether the agent has a target repository. In particular, review forge, REPO_FULL_NAME, target-directory, token-mint, and status-routing paths for conflated meanings.

Keep the no-target choice visible in the CLI and any action or pipeline adapter. Avoid a sentinel repository name or empty checkout that merely hides a repository assumption.

Preserve normal harness post-processing and security checks where they apply. Make repository-specific requirements explicit instead of bypassing them wholesale.

Success criteria
A custom agent runs through fullsend run with an explicit no-target mode, without a target checkout or Git repository, and produces an inspectable result.

The same mode works in a manually invoked GitHub Actions workflow and a GitLab CI pipeline. Their host repository/project is not copied into the sandbox or exposed as the target repository.

No repository-scoped token, status repository, or repository identity is inferred solely from the CI host. Any resource or output destination used by the agent is explicit.

A harness that requires a target repository fails early with a useful error.

Existing repository-targeted runs still work with their current CLI and workflow contracts.

The resulting runner interface is usable by a later OpenShift Job without requiring a repository checkout.

References
Managed Fullsend central service MVP (AISDLC-56)

Poll and dispatch managed Fullsend runs (AISDLC-65)

OpenShift run proof of concept (AISDLC-63)

Tenant configuration decision (ADR 0123)



## AISDLC-147 — Deploy managed Fullsend polling and dispatch active-active

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-147

Background
AISDLC-65 establishes a managed polling and dispatch path for a pilot with one poller and one dispatcher. That path should still work without a database or broker, including existing repository installations. A managed service will eventually need multiple replicas so a restart or failed pod does not stop discovery or dispatch. If two pollers observe the same work, we need to preserve work and avoid launching duplicate agent Jobs.
Summary
Explore and implement an active-active deployment of managed fullsend poll and fullsend dispatch, building on the candidate handoff and run identity from AISDLC-65. Keep the shared inbox optional so the single-instance and repository workflows can run without added infrastructure. Choose the inbox technology after testing the coordination and operational trade-offs.
Scope
In scope
Run multiple poller and dispatcher replicas against the same tenant configuration and candidate sources, with failure recovery and controlled duplicate delivery.

Define a source-neutral candidate reference with tenant, registered source, canonical entity identity, and a stable provider revision or activity identity where available. Keep credentials and source content out of the inbox; dispatch resolves current entity state and current manifest configuration.

Define the acknowledgement boundary: poll progress advances only after the handoff accepts responsibility for a candidate; dispatch acknowledges only after the selected run is durably recorded and Job creation has a recoverable outcome.

Coordinate retries, backoff, poison candidates, operator replay, retention, and backlog visibility. Preserve at-least-once recovery while making Job creation idempotent under candidate redelivery and worker crashes.

Evaluate source-subscription or shard leases to reduce duplicate provider queries. Define lease scope, renewal, expiry, takeover, and any fencing needed for checkpoint writes. A lost lease must not make duplicate execution or skipped work possible.

Show how webhook deliveries from AISDLC-66 can enter the same candidate handoff and converge with polling for the same provider observation.

Keep the shared backend optional. Verify that one poller and dispatcher can still run through the direct handoff without a database or broker.

Exercise concurrent discovery, replica shutdown and takeover, retry after enqueue and Job-creation failures, and a backlog larger than one poll batch. Demonstrate that later candidates are not starved behind repeatedly rediscovered earlier ones.

Out of scope
Choosing a database, broker, or lease mechanism before the implementation evidence is available.

Exactly-once execution of external side effects; retries need stable identities and idempotent effects.

Production-scale capacity planning or support for every forge and work tracker.

Design options to evaluate
PostgreSQL inbox and run ledger. A candidate table could use a unique observation key for deduplication, INSERT ... ON CONFLICT for concurrent writers, and row claims with FOR UPDATE SKIP LOCKED or expiring claims for dispatch replicas. Poll leases and checkpoints could live in the same store. This offers one transactional place for candidate and run state, at the cost of operating a database and managing queue-table growth.

Redis Streams or NATS JetStream plus a run ledger. Consumer acknowledgement and redelivery are built in, but stable deduplication, poll checkpoints, and idempotent Job creation still need durable state. Evaluate the extra components and failure boundaries against the database option.

Source-native coordination and repeated discovery. Jira issue properties or forge state can limit duplicate work without a shared service, as in existing repository paths. Evaluate whether their lock and checkpoint semantics can actually support multiple managed replicas, and whether a bounded poll that discards overflow can guarantee progress through a large backlog. This may remain the appropriate infrastructure-free mode even if managed active-active uses an inbox.

Poll leases and the inbox solve different problems: leases can reduce repeated source queries, while deduplication and a durable handoff protect correctness when leases overlap or expire. The storage and lease design should be selected together with the provider cursor and replay behavior.
Success criteria
Two or more poller replicas and two or more dispatcher replicas can process a pilot source while preserving every eligible candidate and avoiding duplicate Jobs for the same selected run.

A replica can fail before or after enqueue, claim, and Job creation; another replica recovers the work without losing it or launching an uncontrolled duplicate.

Poll and webhook observations of the same provider revision converge on one candidate identity where the source supplies enough identity to do so.

Operators can see source lag, inbox backlog, retries, poison candidates, lease ownership or contention, and the candidate-to-run-to-Job chain.

The no-infrastructure single-instance mode from AISDLC-65 remains usable and shares the same discovery, authorization, and selection rules.



## AISDLC-230 — Provision dev and prod ROSA HCP clusters for Fullsend operations

Status: New. Source: https://redhat.atlassian.net/browse/AISDLC-230

Fullsend needs separate development and production clusters for service operations and agent execution. Dev should support PoCs and iterative development. Prod should run real workflows: the poller and dispatcher service workloads, plus the pod data plane for agent workloads.
Use ROSA with hosted control planes (HCP) as the default. HCP keeps control-plane components in Red Hat's AWS account, avoids dedicated infrastructure nodes, and has a smaller minimum EC2 footprint than ROSA classic. HCP also adds a per-cluster service fee, so validate the full cost for both environments using the proposed worker, availability-zone, storage, and networking choices. See the current ROSA pricing and architecture comparison. Prefer HCP unless the actual cost estimate or workload requirements point to a better option.
Use the Konflux infra project as a reference for infrastructure-as-code structure, provisioning flow, and cluster lifecycle management. Its repository includes Terraform, environment configuration, templates, and CI. We can adapt the patterns that fit Fullsend.
For both clusters, private means that the Kubernetes API, console, ingress routes, and Fullsend service and agent workload endpoints do not accept public internet ingress. Operators reach administrative endpoints through the Red Hat VPN, and end users reach Fullsend-facing endpoints through the Red Hat VPN. Automation and other approved service callers use explicitly defined private network paths. Public subnets or NAT may support necessary outbound traffic, but must not make cluster or workload endpoints publicly reachable.
Definition of done
Separate dev and prod ROSA HCP clusters are provisioned in the agreed accounts and regions, with availability-zone topology appropriate to each environment.

Before provisioning, a network plan records each environment’s VPC and subnet CIDRs, availability-zone placement, expected initial and maximum worker footprint, and spare IP capacity for growth. Validate the proposed ranges against Red Hat VPN address space and networks hosting Fullsend dependencies, including any relevant cluster pod and service ranges. Record the chosen topology and the capacity assumptions behind it.

No cluster API, console, ingress route, or Fullsend workload endpoint is publicly reachable. No internet-facing load balancer, route, or public IP exposes one. Operator access to administrative endpoints and end-user access to Fullsend-facing endpoints work through the Red Hat VPN; non-human callers have approved private paths. Required private DNS resolution and routing work from each authorized access location.

Outbound traffic from both cluster and workload networks starts from a deny-by-default baseline. Allow only the destinations and protocols needed for ROSA operation, GitOps, image pulls, secrets, observability, and approved Fullsend integrations. Record the purpose and owner of each allowance, and review the list when dependencies change. An unrestricted internet route or NAT gateway alone does not satisfy this requirement.

Verify the access model in dev and prod: operator and end-user access through the Red Hat VPN succeeds; approved non-human callers can use their private paths; public attempts to reach cluster and workload endpoints fail; required outbound dependencies work; and an unapproved outbound destination is blocked. Record the tested paths and results.

Provisioning and ongoing cluster configuration are represented in versioned infrastructure-as-code and GitOps-managed configuration, with dev and prod differences explicit and reviewable.

Document the source of truth for each layer, including AWS and ROSA infrastructure, GitOps bootstrap, ongoing cluster configuration, and secrets. Name the owning repository and controller for each, keep Terraform and GitOps ownership from overlapping, and use protected, separate remote Terraform state for dev and prod with locking.

Document and use a change process for both environments: review the proposed infrastructure plan and GitOps changes before applying them, restrict production apply rights to authorized operators or automation, and detect and reconcile configuration drift or emergency changes made outside the source repositories.

Both clusters can be bootstrapped into GitOps management, and supported configuration changes can be reconciled from source control.

Bootstrap each cluster repeatably from a newly provisioned state through installation of the GitOps controller and its initial configuration. Document the order of operations, who or what runs each step, how the bootstrap runner reaches the private cluster API, which identities and credentials it needs and where those credentials are stored, and the source repository and path it reconciles. Demonstrate a configuration change reconciling from source control after bootstrap in both dev and prod.

Dev supports PoCs and Fullsend development. Prod can host the Fullsend poller and dispatcher service workloads and the agent-workload pod data plane for real workflows.

Run the poller and dispatcher on a dedicated service worker pool, and agent workload pods on a separate agent worker pool. Place the service workloads and agent pods in separate namespaces. Use scheduling and resource controls so agent pods cannot consume service pool capacity.

Plan for agent load to grow without assuming a known concurrency target today. Document initial per-agent resource assumptions and independently adjustable worker-pool scaling limits, quotas, and capacity monitoring. Demonstrate that the agent pool can scale within its configured range and that agent load does not starve the poller or dispatcher; revisit limits as real usage becomes available.

A current total-cost estimate for both clusters covers ROSA service fees and underlying AWS infrastructure, and confirms the selected HCP topology fits the availability and workload needs.

Networking, identity, secrets, workload isolation, and scaling expectations are agreed for the service and agent workload planes.

Operators have documented ownership and instructions for access and routine configuration changes.

This cluster foundation supports related Fullsend service and agent work, including AISDLC-65, AISDLC-66, AISDLC-146, and AISDLC-147.
Out of scope
Provisioning application support services such as a shared database or broker is outside this cluster feature. AISDLC-147 or later work will choose and deliver any shared inbox infrastructure needed for active-active polling and dispatch.

Building an external webhook delivery path or relay is outside this cluster feature. AISDLC-66 or later work will address delivery to a private Fullsend endpoint; this feature does not add public ingress for webhooks.

Defining and verifying ongoing operational readiness, including alert ownership, backup and restore, upgrades, recovery objectives, and recovery exercises, is outside this cluster feature. AISDLC-231 owns that follow-up work.

Notes
Use the Konflux infra team's provisioning and management patterns as input where they fit.

Consult #forum-konflux-infrastructure for advice on HCP provisioning, GitOps bootstrap, dev/prod separation, and day-two cluster operations. Capture applicable guidance and remaining choices in the implementation notes.
