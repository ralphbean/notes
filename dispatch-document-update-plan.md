# Plan: extend the Fullsend dispatch comparison with Kubernetes

Prepared 2026-10-01. This is a document-editing handoff, not a request to implement a Kubernetes dispatcher.

## Files and evidence

- Original, leave unchanged: `/home/rbean/code/manon.pages.redhat.com/fullsend/docs/dispatch-architecture-github-vs-gitlab.html`.
- Working copy, already created byte-for-byte: `/home/rbean/code/notes/dispatch-architecture-github-gitlab-kubernetes.html`.
- Backlog descriptions: `/home/rbean/code/notes/dispatch-backlog-snapshot.md`, fetched using `acli` with `parent = AISDLC-37 ORDER BY key ASC` (12 direct children).
- Fullsend source: `/home/rbean/code/fullsend-5`, updated with a fast-forward pull to `d462a93207302c4c2385e5f6eaa1bd40f45f6e98` on 2026-10-01. The pre-existing untracked `debug.log` was preserved.
- Original HTML SHA256: `e156da73cc11f2679724884c3799ef6a102657d50a0279465c442b2f1599ec71`.

The working HTML remains the original content at this stage. The user's answers are incorporated below; this plan is ready for another instance to execute. Do not publish, commit, modify Fullsend production code, or alter Jira issues as part of this handoff.

## User intent and confirmed preferences

Preserve the existing document's substantive GitHub and GitLab coverage and visual comparison, expanding it with a proposed Kubernetes execution option. Evolve existing Fullsend components rather than starting a separate implementation. Explain which responsibilities CI platforms currently supply and which Fullsend or Kubernetes would own.

The user confirmed all four preferences:

1. Show the single-instance pilot and later active-active service as phases within one Kubernetes column.
2. Recommend component boundaries; leave backend technologies and webhook connectivity alternatives open.
3. Use a Jira candidate as the worked example, with both repository-bound and repository-less context evaluation. This does not choose the pilot provider.
4. Preserve the original text and add dated corrections separately. Do not silently rewrite historical claims. Retain original prose, table-cell content and links, adding clearly labelled “Correction — 2026-10-01” callouts beside affected passages or beneath affected tables. Structural HTML/CSS changes and new headings are allowed, but retain the original title and metadata as historical provenance.

One further behavior choice can remain an explicit open question: when a second legitimate observation arrives during a run, should Fullsend queue it, coalesce it, steer the existing run, or cancel and replace it? Do not silently equate deduplication with any of those policies. Distinguish a conservative proposed serialization policy from accepted decisions.

## Architectural constraints already supported by the backlog

Use these without asking the user to decide them again, unless they explicitly change the scope:

- Reuse `repos.Manifest` and filesystem loading. ADR 0123 identifies mounted ConfigMaps as a typical Kubernetes delivery path; ownership of the authoritative manifests and their deployment remains open.
- Keep implementation in `fullsend-ai/fullsend`. Preserve existing repository installations.
- AISDLC-65: one poller and one dispatcher, one pilot source, direct candidate handoff, no required database or broker. Polling discovers candidates; dispatch fetches current state, evaluates each applicable context independently, and selects every matching agent.
- AISDLC-66: authenticated webhooks feed the same handoff. Polling remains the correctness/recovery path. Public cluster ingress is excluded; this feature owns the external delivery/private-connectivity or relay question.
- AISDLC-147: optional shared inbox and active-active replicas, durable run identity, recovery and visibility. PostgreSQL, Redis Streams/NATS plus a ledger, and source-native coordination are alternatives to evaluate, not settled architecture.
- AISDLC-63: Kubernetes Job runs `fullsend run` as an ordinary restricted OpenShift workload, using a separately installed remote OpenShell gateway. Inputs/results use gateway transfer; the Job does not mount the sandbox's workspace PVC. Static credentials are limited to the PoC.
- AISDLC-64: explore projected workload identity for inference OR repository access through a private mint. Both paths are not required for this PoC, and the public mint must not be modified or assumed usable.
- AISDLC-146: repository-less execution must be explicit. Host, configuration source, task source, target repository, output destination, and credential scope are separate concepts.
- AISDLC-230: private dev/prod ROSA HCP clusters; separate service and agent worker pools and namespaces. Do not infer one namespace per tenant from this requirement.
- No tenant configuration CRD, new tenant REST API, or Tekton dependency for the pilot. Existing OpenShell/Agent Sandbox dependencies are a different matter from inventing a Fullsend tenant CRD.

## Read the source selectively

Read `/home/rbean/code/fullsend-5/AGENTS.md` and the following files. Use commit-pinned GitHub links in the report so evidence remains stable: `https://github.com/fullsend-ai/fullsend/blob/d462a93207302c4c2385e5f6eaa1bd40f45f6e98/<path>`.

| Existing code / document | What it establishes | Managed evolution to explain |
| --- | --- | --- |
| `internal/repos/manifest.go`; `docs/ADRs/0123-tenant-configuration-from-repos-yaml.md` | Existing manifest model and accepted reuse decision | Extend for tenant/repository-less contexts; load latest valid configuration from filesystem |
| `internal/poll/poll.go`, `events.go`, `convert.go`, `dispatch.go`, `state.go` | GitLab discovery, routing, pipeline creation, signed state and CAS persistence remain coupled | Extract discovery/handoff seam; retain repository adapter; replace launch destination for managed mode |
| `internal/jirapoll/poller.go`, `discover.go`, `convert.go`, `lock.go`, `types.go` | Jira discovery, CEL matcher, issue-property coordination and output-file checkpoint boundary | Candidate-only discovery, source-neutral identity, acknowledgement tied to accepted responsibility rather than a local file |
| `internal/cli/poll.go`, `internal/cli/dispatch.go` | Existing CLI wiring and input/output adapters | Extend these entry points; do not invent working flags or a Kubernetes output driver |
| `internal/harnessdispatch/core.go`, `enumerate.go`, `auth.go`, `project.go`, `ref.go`; `internal/normevent/` | Authorization, kill switch, harness enumeration, CEL matching, execution-reference projection | Reuse logic after trusted enrichment; decouple repository assumptions; evaluate independent contexts |
| `internal/cli/run.go`; relevant sandbox/runtime code | Existing execution pipeline and sandbox integration | Add Kubernetes launch adapter and remote-gateway support around the existing runner |
| `.github/workflows/reusable-dispatch.yml` | Actual GitHub routing, matrix jobs, concurrency keys, cancellation | Identify CI-owned orchestration responsibilities being replaced |
| `internal/scaffold/fullsend-repo-gitlab/.gitlab/ci/fullsend-poll.yml`, `fullsend-agent.yml`, `scripts/run-poll-job.sh`, `scripts/run-agent-job.sh`, `scripts/pin-ci-job-identity.sh` | Actual GitLab schedules, resource groups, execution and identity checks | Separate poller concurrency from agent concurrency and from delivery deduplication |
| `internal/mintcore/`; ADRs 0098, 0106, 0113, 0119, 0125 where present | Credential boundaries and planned entity/steering changes | Distinguish accepted direction, implementation and new proposal |

Source findings already verified:

- `harnessdispatch.Dispatch` still requires a non-nil `Event`. `ConfigDir` is documented as a direct child of a repository root for OWNERS authorization. Entity-first managed selection is not already implemented just because an ADR describes it.
- `ExecutionRef` carries agent, role, source repository, event payload, trigger source, status repository and number. It does not yet represent the managed tenant/context/run contract.
- `fullsend dispatch` outputs `gha-matrix` or `json`; those outputs do not schedule a Kubernetes Job.
- GitLab `poll.dispatch` calls `CreatePipeline` directly. `state.go` contains signed watermarks, dispatched keys, failed keys and label snapshots, with CAS persistence. CAS state updates do not by themselves make the pipeline-launch side effect atomic with state persistence.
- Jira's current checkpoint follows a successful dispatch-file write. Its `LockValue.Phase` comment explicitly identifies the pending-to-running handoff as unfinished. Do not claim it is already a durable managed queue or a safe active-active lease implementation.
- Current GitHub reusable dispatch still has `cancel-in-progress: true` groups.
- GitLab poll jobs use a group per poll mode; agent jobs use `fullsend-${STAGE}-${RESOURCE_KEY}`. Their ordering is configured separately.
- The native GitLab MR dispatch template described in the original report has been removed in the updated checkout. `internal/scaffold/fullsend-repo-gitlab/.gitlab-ci.yml` explicitly says native `merge_request_event` dispatch was removed in #7322 and all GitLab events route through the cron poller. Add a dated correction; retain the original coverage text.

## Edit sequence

1. **Inventory and evidence pass.** Record the eight original section IDs, all original text, comparison rows, and external references. Build a separate correction list before editing. Mark the original as a 2026-09-30 snapshot; verify time-sensitive PR status where adding a new current claim, using `gh pr view`. Do not label all planned work as implemented based on Jira status, or treat AISDLC-56 being Closed as proof that the service exists.

2. **Preserve the visual structure and historical text.** Use “Fullsend Dispatch Architecture: GitHub, GitLab, and Kubernetes” as the new document title/H1; retain the original title and metadata in a historical-provenance block. Explain that Kubernetes is a managed execution/dispatch option which still consumes forge and tracker data, not a third forge. Date the additions and link the pinned source and backlog. Keep the original eight sections and anchors. Retain original badges as explicitly historical; add new labels for the expanded scope rather than recalculating the old counts in place. Place a prominent note above the report stating that the original text is preserved, with dated corrections and a proposed Kubernetes extension.

3. **Add a third architecture column to all five layers.** Use GitHub / GitLab / Kubernetes (proposed). Keep GitLab current/desired distinctions inside its column. Give Kubernetes its own color with readable light/dark variants. Add a distinct “Proposed” legend style; do not reuse “ADR accepted” styling for undecided Kubernetes details. Distinguish proposed pilot from later webhook/active-active additions.

4. **Extend every comparison.** For tables that already split GitLab current/desired, use five total columns: dimension, GitHub, GitLab current, GitLab desired, Kubernetes proposed. For ordinary comparisons, use dimension plus three options. Include Kubernetes in security, credentials, event coverage, execution, scheduling, state, concurrency and risks. Preserve GitLab-specific risks as GitLab-specific.

5. **Add three focused sections.** “Responsibilities currently supplied by CI”, “State, identity, and recovery”, and “Evolution and backlog”. Add TOC links. Use tables and one candidate-to-run flow to make ownership and changes visible.

6. **Extend the ADR timeline and ship gates.** Preserve original entries and add dated status corrections separately. Add ADR 0123's configuration decision. Show the Kubernetes direction as a proposal supported by backlog items, not a fabricated accepted ADR. Separate execution PoC, pilot dispatch, webhook delivery, identity PoC and active-active gates.

7. **Add a worked example and open decisions.** Use the user's selected source. Show one observation selecting more than one agent/context; a duplicate delivery; and a crash after successful Job creation but before acknowledgement. Mark sample field names as illustrative, not an implemented API. End with genuinely unresolved decisions and their relevant backlog links.

8. **Validate content and rendering**, using the checks below. Produce a brief change summary and mention any unresolved decisions left in the report.

## Content for the Kubernetes architecture column

| Layer | Proposed contents |
| --- | --- |
| 1. Entry points | Scheduled polling for pilot source; later authenticated webhook receiver using a private delivery path. Same source-neutral candidate handoff. Direct delivery first; optional durable inbox later. |
| 2. Normalization | Resolve tenant and registered source from trusted configuration; fetch current entity and actor authorization via source APIs; reuse normalization. A webhook is a discovery hint, not authority to choose credentials, namespace or Job specification. |
| 3. Routing | Reload/validate current manifest; evaluate tenant repository-less context and applicable repository contexts independently; authorization and kill switch; reuse CEL selection of every matching agent. Keep candidate, selected-run and concurrency identities distinct. |
| 4. Dispatch and credentials | Proposed launcher creates or finds the Job for a stable selected-run identity; recover ambiguous create outcomes; bind approved placement and credential scope. Static secrets belong to the execution PoC; workload identity/private mint remains a separate validation path. |
| 5. Execution and follow-up | Job invokes existing `fullsend run`; remote OpenShell gateway owns sandbox lifecycle. Track completion/cancellation and result destination. Choose ordering/coalescing/steering explicitly; do not claim generic Job reconciliation supplies these semantics. |

Draw configuration flowing into dispatch, not being copied into each candidate. Show a distinction between “accepted for scheduling”, “Job created”, “Pod running”, and “agent completed”. Acknowledgement of scheduling need not wait for completion, but responsibility must survive the process that acknowledges it.

## Responsibilities that need explicit ownership

Create a matrix with current GitHub/GitLab owner, proposed managed owner, reuse opportunity, and remaining gap. Include:

- Event subscription, inbound endpoint, authentication, delivery identifiers and failed-delivery handling.
- Periodic schedules, missed-work reconciliation, manual initiation and operator replay.
- Dispatch queue, admission/backpressure, tenant fairness and provider API rate limits.
- Concurrency groups, ordering, cancellation, retries and run timeouts.
- Durable run IDs and lifecycle records; correlation from source candidate to selected run to Job/Pod and results.
- Trusted workload identity, scoped credential delivery/refresh, protected configuration/ref selection and actor authorization.
- Runner allocation, isolation, workspace preparation, pinned configuration/harness/image provenance and sandbox cleanup.
- Log collection, artifacts/results, retention, status updates to the source system, auditability and operator visibility.

Assign only the appropriate portions to Kubernetes: it supplies workload resources and lifecycle mechanisms, while Fullsend must define application-level selection, identities, acknowledgements, source integration and concurrency policy. Existing infrastructure can provide logging or storage, but name an integration requirement rather than pretending that the Kubernetes API supplies CI-style history and artifact retention automatically.

## State, deduplication and failure semantics

Distinguish these state categories:

| State | Authority / proposed responsibility |
| --- | --- |
| Source entity content and current permissions | Forge/tracker remains authoritative; refetch during dispatch |
| Desired configuration | Existing manifest model; filesystem delivery; selected configuration revision recorded for an admitted run |
| Discovery checkpoints / provider revisions | Poller state; advance only after handoff acceptance; preserve replay and backlog progress |
| Candidate delivery, claims and retries | Direct pilot contract, then optional shared durable inbox; no credentials or source content in candidates |
| Selected runs and Job association | Dispatcher/run ledger or explicitly documented pilot equivalent; recover partial fan-out and ambiguous creates |
| Entity/agent concurrency and follow-up observations | Explicit Fullsend policy; not implied by inbox deduplication or a poll lease |
| Job/Pod status | Kubernetes; correlate to application outcomes |
| Results, artifacts, steering receipts and historical audit | Explicit retention/ownership; must outlive cleanup as required |

Use separate illustrative keys:

- Candidate: tenant + registered source + canonical entity + stable provider revision/activity where available.
- Selected run: candidate identity + execution context + selected agent, with an explicit policy for configuration revisions and operator replay.
- Concurrency: tenant + context + entity + agent/role, with the granularity shown as a choice to validate.

Explain the following cases in a compact failure table:

- Same provider observation discovered by poll and webhook: converge where provider identity permits; do not assume timestamp equality is sufficient.
- Crash before handoff acceptance: do not advance progress; rediscover.
- Crash after durable handoff but before progress update: redelivery is expected.
- Multiple selected runs, only some created: recover missing runs without duplicating completed admissions.
- Job create succeeds but response/acknowledgement is lost: find and validate the existing Job using stable identity before retrying creation.
- Candidate becomes ineligible before selection: record an observable no-op after fresh state/configuration evaluation.
- Poison candidate: bounded retries, observable quarantine/replay, no silent loss or starvation of later candidates.
- Completed Job is removed: explain retention of run identity/tombstones or a bounded replay horizon. A deterministic Job name alone stops protecting against duplicates after deletion.
- A Job creates a replacement Pod: external effects still need idempotency/reconciliation. One Job does not guarantee one execution.
- Lease expires while a worker is still active: overlapping workers must remain correct; source leases reduce query cost, not replace durable handoff or idempotent launch.

Do not require an inbox database in the pilot diagram. Document the pilot's unresolved persistence and acknowledgement mechanism candidly rather than labeling an in-memory channel durable. Explain how source rediscovery and recoverable Job identity could support the direct path, with exact checkpoint/retention semantics left to implementation validation.

## Backlog mapping

Link every issue directly using `https://redhat.atlassian.net/browse/<key>`.

| Issue | Role in this document |
| --- | --- |
| AISDLC-65 | Core managed polling/selection/Job dispatch and direct-handoff contract |
| AISDLC-66 | Shared webhook ingress path and private-network delivery |
| AISDLC-147 | Optional durable inbox, active-active coordination, recovery and visibility |
| AISDLC-57 / ADR 0123 | Manifest reuse and independent execution contexts |
| AISDLC-63 | Execution adapter and remote OpenShell gateway prerequisite |
| AISDLC-64 | Identity/credential contract and PoC limits |
| AISDLC-146 | Explicit repository-less runner contract |
| AISDLC-70 | Tenant registration and configuration authorization boundary |
| AISDLC-230 | Private connectivity and service/agent placement constraints |
| AISDLC-144 | Configuration precedence/compatibility context; do not invent new site/tenant overlay layers |
| AISDLC-143 | Adjacent data/MCP access work; mention only where needed for credential boundary, not as dispatch implementation |
| AISDLC-56 | Historical umbrella/scope context; Closed is not implementation evidence |

## Corrections and traps to verify

- Preserve original claims as historical text and qualify them in dated corrections. Original sub-second latency claims need qualification: notification latency and queued-run start latency are different; give no invented Kubernetes SLA.
- GitHub Actions native events are not an externally operated GitHub App webhook receiver. Do not describe an unused application webhook secret as a current Fullsend dispatch credential.
- Verify GitLab credential roles against current role-token selection and identity-pinning scripts; the original blanket Maintainer-PAT description may be stale.
- Do not equate GitLab `newest_first` with cancellation or dropping all older jobs. It controls ordering. Do not confuse poll-job resource groups with agent-job resource groups.
- Do not equate GitHub concurrency groups with exactly-once delivery or a lossless application queue.
- Verify the original entity-first, native MR pipeline, sandbox backend and ship-gate claims at the pinned commit. Preserve planned descriptions as planned where code does not support them.
- A repository-local kill switch or OWNERS check cannot silently become the tenant-level authorization policy. Show that adapting these checks is required.
- “Kubernetes native” does not automatically require an operator, CRD, Tekton, a broker, or rewriting the Fullsend execution engine.

## Primary references for platform behavior

- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/): Jobs may start the same program more than once, even with a single completion; TTL cleanup removes Jobs and dependent Pods. Stable launch identity does not imply exactly-once side effects or permanent history.
- [GitHub workflow concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency): use actual Fullsend concurrency configuration when describing present behavior.
- [GitLab resource groups](https://docs.gitlab.com/ci/resource_groups/): serialization and processing order are distinct from application deduplication.

## Acceptance checks

- Original source HTML is unchanged. Original visible text and links remain in the new copy, including the original title/metadata in a provenance block. All eight original topics remain. Corrections are separate, dated and clearly associated with the affected passages; do not rely on readers finding a distant appendix to resolve contradictory claims. Compare extracted original text blocks/cells and links with the final HTML to catch accidental deletions; allow added text and structural changes.
- Five architecture layers each have three clearly labelled options. Tables retain both GitLab states where previously distinguished.
- Display three architecture columns at a comfortable desktop width (for example, widen the page to roughly 1440–1600px); stack them in labelled order on narrow screens. Put wide tables in horizontal-scroll wrappers; do not shrink text to fit five columns into the former 1100px layout.
- Light/dark colors, contrast, print behavior and long code identifiers remain readable. Inspect desktop and mobile rendering with the browser skill if available. Follow that skill's instructions when used.
- No duplicate HTML IDs; TOC fragment links resolve; table cells/header spans align; no broken local resource references. Use existing tooling or a small parser check rather than building a new test framework.
- Every new architecture statement is visibly one of: verified existing code, accepted ADR/backlog requirement, proposed design, or open decision. New current PR-status statements have dated evidence; preserved original statuses are labelled as historical.
- Reuse table names actual existing components; new adapter names are explicitly proposed. No fabricated CLI commands, manifest schema, accepted ADR, performance promise or exact-once guarantee.
- The pilot does not require a database/broker; the active-active path does not imply local files alone are sufficient. Poll/webhook convergence, independent contexts, all matching agents, retries, partial fan-out and post-cleanup replay are explained.
- Private cluster ingress constraint, remote sandbox boundary, static-credential PoC limits and private-mint uncertainty remain visible.

## Suggested instruction to the next Codex instance

Read this plan and `dispatch-backlog-snapshot.md`, then edit only `dispatch-architecture-github-gitlab-kubernetes.html` to carry out the plan. The user's confirmed preferences are recorded above. Use the pinned source paths as evidence, preserve original text and add dated corrections separately, and label unresolved technology and policy choices explicitly. Inspect the rendered HTML and report the resulting file path, meaningful factual corrections, and remaining open decisions. This task is documentation only.
