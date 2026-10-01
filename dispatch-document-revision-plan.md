# Revision plan after reviewing the expanded dispatch document

Reviewed 2026-10-01. This plan addresses the resulting document, rather than repeating the original expansion instructions. It covers every section, all five flow layers, and the comparison tables. The HTML has not been edited during this review.

## Inputs and constraints

- Document to revise: `/home/rbean/code/notes/dispatch-architecture-github-gitlab-kubernetes.html`.
- Reviewed document SHA256: `abe91577259a91b1dc1c50b492d7c93bc51ede3530a3e356f420a3f37ec6b891`.
- Original: `/home/rbean/code/manon.pages.redhat.com/fullsend/docs/dispatch-architecture-github-vs-gitlab.html`, SHA256 `e156da73cc11f2679724884c3799ef6a102657d50a0279465c442b2f1599ec71`.
- Backlog evidence: `/home/rbean/code/notes/dispatch-backlog-snapshot.md`.
- Code evidence: `/home/rbean/code/fullsend-5`, baseline `d462a93207302c4c2385e5f6eaa1bd40f45f6e98`.
- The earlier `dispatch-document-update-plan.md` remains useful for source paths and backlog scope. This revision plan takes precedence for presentation, architecture explanation, and acceptance criteria.

Retain the user's decisions: show pilot and active-active phases; recommend component boundaries without selecting storage/relay technologies; use a Jira example; preserve original text with separate dated corrections; evolve existing Fullsend code. Do not implement services or rewrite the HTML as part of executing this planning task. A subsequent implementation task should edit the HTML and verify the result.

## Findings that determine the revision

The central problem is inconsistent meaning. Some boxes name running components, others describe implementation tasks, limitations, or Jira scope. Several table cells describe an entirely different responsibility from their row. Listing relevant facts is insufficient: the reader needs to follow a transformation and compare equivalent responsibilities.

The most consequential defects are:

1. **Broken comparisons.** In Component Comparison, the Kubernetes cell for “Inference credentials” says “Restricted Job + remote gateway”; “Agent sandbox” describes `repos.Manifest`; “CLI install” describes gateway lifecycle; “Native CI events” contains a GitLab migration note. More rows are affected below.
2. **Flows do not explain interfaces.** “Reload manifest” follows normalization as if it transforms an event. “Adapt authorization” is a task rather than a running component. Connectors such as “candidate ≠ selected run ≠ concurrency identity” are observations, not flows.
3. **The two managed phases are not actually shown.** References to an optional inbox are scattered throughout the document, but the reader cannot see exactly what changes between direct pilot delivery and active-active operation.
4. **Security and credential comparisons mix unrelated boundaries.** An acknowledgement/run ID does not explain dispatcher-to-agent integrity. A private repository mint is not an inference credential path. Refetching an actor does not explain fork protection.
5. **The example is a generic outline.** It does not establish actual matching conditions, show distinct run identities, or walk through redelivery and recovery. It also says the poller “receives hint”, blurring the separate poller and webhook paths.
6. **Historical preservation was not followed.** A normalized visible-text comparison found 40 original blocks among paragraphs, table cells, diagram labels/details, and finding text absent verbatim from the new document. The revision note explicitly says descriptions were revised in place. This needs repair under the user's existing instruction.

Keep the useful material: independent contexts, separation of discovery and selection, source-neutral candidate identity, the failure-case inventory, remote OpenShell execution boundary, and source/backlog references. Give each fact one main explanatory home and cross-reference it elsewhere.

## Rules for the revised explanation

Apply these rules before polishing wording or colors:

- Each layer answers one shared question in all three columns. Each box names a component or an operation with a clear input and output.
- Each table cell completes the sentence implied by its row heading. “Not applicable” or a precise unresolved mechanism is better than unrelated content.
- Separate **the proposed behavior** from **what must change in the code** and **what remains undecided**. Put the latter two in annotations and the evolution section, not in the main flow.
- Distinguish source system, transport, execution platform, tenant/context, and agent. Kubernetes is the proposed execution platform; Jira/GitHub/GitLab remain sources. A role such as Analyst is not automatically an agent name.
- Distinguish candidate reference, normalized evaluation input, selected run, Job, Pod, sandbox, and result. Define them once in a compact glossary near the flow.
- Treat configuration as an input to components. Treat status/results as outputs. Neither belongs on the event-processing arrow without an actual transformation.
- Mark certainty at the right scope. A paragraph about proposed behavior can carry one status label; it does not need a badge on every sentence. An accepted ADR badge applies only to its actual decision.
- Use full sentences where abbreviated text conceals meaning. Replace fragments such as “CAS not atomic”, “Jira does not select pilot”, “no PVC”, and “no namespace per tenant inference”.

## Section-by-section revision

### 1. Header, provenance, navigation, and document scope

**Keep:** title, dates, pinned source and backlog links, distinction between Kubernetes and a forge.

**Fix:** the header asserts current source accuracy while the body still contains stale and contradictory statements. Old badge totals take prominent space without explaining the proposal. TOC order differs from the rendered section order: responsibilities/state/example occur before the timeline, but the TOC places them after ship gates.

**Plan:**

- Restore the preservation contract: original GitHub/GitLab text remains explicitly marked as the September 30 snapshot; dated corrections are attached to the affected passages. Recover missing original blocks from the untouched source, not from memory.
- Use an unobtrusive historical-details block for the original title/metadata/badges. Keep corrections visible where they change the reader's interpretation; do not hide all corrections in that block.
- Label the comparison options clearly as GitHub Actions, GitLab CI, and managed Kubernetes execution. Preserve original headings when needed by the text-preservation requirement, adding clarifying captions.
- Retain all existing section anchors. Reorder sections if helpful, then regenerate the TOC in exactly that order.
- Recommended reading order: Summary → Architecture Flows → Worked Example → Component Comparison → CI Responsibilities → State/Recovery → Security → Credentials → Event Coverage/Latency → Component Evolution/Backlog → ADR History → Ship Gates → Open Decisions.
- Do not silently make the preserved original the current recommendation. Use adjacent correction annotations with explicit dates and source links.

**Acceptance:** a reader can distinguish original text, corrected current facts, proposed behavior, and unresolved choices without consulting the plan.

### 2. Summary (`#summary`)

**Problem:** most of the summary recounts GitLab/steering implementation status. The managed proposal gets one sentence. It does not explain the main architectural change, the two phases, or where existing code survives.

**Plan:** preserve historical paragraphs and add a concise leading synthesis of the expanded comparison:

1. The repository paths use CI to deliver events, orchestrate runs, and supply execution context.
2. The managed proposal retains source adapters, normalization, authorization, harness selection and `fullsend run`, while Fullsend takes responsibility for discovery-to-run coordination and a launcher creates Kubernetes Jobs.
3. The pilot uses direct candidate handoff. Active-active adds durable shared coordination around the same selection and launch logic. Webhook ingestion is another producer, not another dispatcher.

Move detailed PR inventories and `ConfigDir`/OWNERS implementation caveats to component evolution; link them from the new summary if necessary. The summary should describe the architecture before its backlog.

### 3. Architecture Flows (`#architecture`)

**Structural repair for all five layers:** add a short question and input/output caption to each layer. Use arrows only for data/control transfer. Put configuration beside the flow with labelled inputs, and implementation gaps beneath the relevant layer. Keep the five-layer overview compact; detailed recovery belongs in its own section.

#### Layer 1 — Entry points

**Mostly sound, but incomplete.** “Scheduled poller” and “Private authenticated webhook” are appropriate entry mechanisms. However, “no required database/broker” and network ownership details dominate their operation, and the flow stops before identifying what is handed off.

- Question: **What causes Fullsend to consider work?**
- Show two producers: scheduled source poller and later webhook receiver.
- Both output a candidate reference through the same handoff contract. The webhook receiver authenticates delivery; the poller uses source API credentials. Neither selects agents.
- Keep the private-cluster constraint as a short receiver annotation; detailed relay alternatives belong in Open Decisions.
- Show manual initiation/replay only as an explicit open entry-path policy, not an implemented capability.
- Correct the contradictory GitHub node title “GitHub App Webhook Events” with an adjacent dated note consistent with the section's Actions-event explanation.

#### Layer 2 — Shim and normalization

**Rebuild the Kubernetes column.** “Resolve source and refresh state” partly fits; “Reload manifest” is not a normalization stage.

- Question: **How does source-specific input become shared evaluation input?**
- Proposed flow: **Source adapter → Fetch and enrich → Normalize for routing**.
- Source adapter accepts a candidate reference and uses trusted registration to choose the source API/conversion code. No per-repository CI shim is required for this managed path.
- Fetch/enrich obtains current entity state and the relevant activity/actor information. Retrieving permissions supplies evidence to authorization; it is not the authorization decision itself.
- Normalization produces the shared entity/event input expected by selection, extending existing conversion code for the proposed entity-first contract. Do not claim today's `harnessdispatch` already accepts a nullable event.
- Output caption: **Normalized entity/activity and actor context, ready for selection**.
- Draw source registration/configuration as an input, not a downstream “reload” box. Pin the configuration used for a selection consistently across layers, without inventing a configuration-delivery technology.
- Distinguish provider observation identity from current entity state: a fresh snapshot does not prove that a claimed historical transition occurred. Link this nuance to state/replay and security.

#### Layer 3 — Routing

**Rebuild the Kubernetes column and correct contradictory introductions.** The current introduction says both existing platforms run two routing paths in parallel, while the GitLab diagram says CEL is not wired to its poller. “Adapt authorization” belongs in implementation notes.

- Question: **Which agents should run in which contexts?**
- Proposed flow: **Resolve applicable contexts → Apply context authorization and kill switches → Evaluate registered harness triggers → Produce selected runs**.
- Evaluate each applicable repository context and the repository-less tenant context independently, using current configuration. “Applicable” must follow configured source/context association, not an accidental repository link or every repository in the tenant.
- For every matching authorized agent, output a selected run with tenant, context, agent, role/authority, target choice and output destination distinguished.
- Show observable outcomes for no match/denied and for evaluation failure. A failed source or configuration read must not be drawn as successful “no work”.
- Move identity distinctions into the glossary/state section. Move event-required and repository-path assumptions into the reuse/gaps table.

#### Layer 4 — Dispatch and credentials

**Make admission and launch intelligible.** “Proposed Job launcher” is relevant, but “Credential PoC boundary” is status information rather than a credential mechanism. The current flow does not say who supplies the trusted Job specification.

- Question: **How does each selected run become an authorized workload?**
- Proposed flow: **Run admission/record → Resolve approved workload specification and credential references → Create or recover Kubernetes Job**.
- Describe run admission as Fullsend logic that applies the chosen concurrency/admission policy and establishes recoverable run identity. The pilot persistence mechanism remains an identified gap.
- Trusted service configuration determines placement, image/configuration references and permitted workload identity. Event payloads cannot supply arbitrary Job definitions or privilege choices.
- The launcher uses its own scoped Kubernetes authority. The run obtains separate source/inference credentials through the selected credential mechanism. Keep these two identities distinct.
- Output: **Correlated Job identity and recoverable admission outcome**, not “agent completed”.
- Static PoC secrets and future workload federation are phase annotations linked to Credentials. Do not promise the federation design is already decided.

#### Layer 5 — Execution and in-run behavior

**Keep the execution boundary, complete the flow, relocate deployment notes.** The remote-gateway box is useful. Separate worker pools are placement context, not an execution step. “Follow-up behavior open” should state where the policy acts.

- Question: **How does the admitted workload perform and finish its work?**
- Proposed flow: **Job Pod invokes `fullsend run` → Remote gateway creates/operates sandbox → Results return and cleanup occurs → Fullsend records/reports outcome**.
- Label the Job Pod and sandbox as separate workloads. The sandbox has its workspace PVC; the run Job does not mount that PVC. Replace the misleading shorthand “no PVC”.
- Include success, failure and cancellation in lifecycle ownership. Job status and application success are related but not interchangeable.
- Move service/agent worker pools to a deployment inset. Keep that inset explicitly tied to the ROSA/OpenShift backlog rather than a universal Kubernetes requirement.
- Show follow-up observations feeding the same entry/selection path. Annotate that their treatment during an active run—queue, coalesce, steer or cancel/replace—is unresolved. Do not draw implemented steering.

#### Add a small two-phase deployment view

The five layers explain behavior. Add a separate view immediately beneath them to explain process boundaries and scaling:

- **Pilot:** one poller → direct handoff → one dispatcher (normalize/select/admit) → Kubernetes launcher/API → run Job → remote sandbox gateway. Filesystem manifest feeds source configuration, selection and launch policy. Direct handoff is a logical contract; do not imply a new public API or mandatory network hop.
- **Active-active:** poller replicas and optional webhook receiver → shared durable inbox → dispatcher replicas → recoverable run records/idempotent launcher → Jobs. Show checkpoint coordination and optional source-query leases beside producers. Indicate which logic remains unchanged.
- Keep inbox, run records and leases as logical responsibilities, not three mandatory products. Storage alternatives remain open.
- Webhook ingestion and active-active scaling are independent additions after the pilot contract, not necessarily a strict sequence.

### 4. Component Comparison (`#components`)

**Reconstruct row meaning before editing prose.** Several cells appear displaced, but a blanket positional shift will not repair all of them. Bind edits to the row label and review every completed row across every column.

Use these Kubernetes cell requirements:

| Row | Required content |
| --- | --- |
| Webhook events | Later authenticated source webhook receiver feeds the shared candidate handoff; source support/path remains to be chosen. Put polling in its own row. |
| Cron schedules | A scheduler invokes the existing polling logic; exact timer/CronJob/service arrangement is proposed/open. |
| Native CI events | No forge CI invocation is required for managed dispatch. Provider events are adapter inputs; remove the GitLab #7322 migration note from this cell. |
| Dispatch trigger | Separate dispatcher invocation from selected-run launch. Original GitHub `workflow_call` and GitLab pipeline API cells currently describe different boundaries; add a clarifying row/note rather than pretending they match. |
| Event normalization | Source adapter and trusted enrichment produce the common evaluation input. Candidate handoff and normalized evaluation input are different objects. |
| Routing, first-party/custom | Same registered-harness selection mechanism; all matching authorized agents in applicable contexts. |
| Deduplication | Observation identity and selected-run identity control different duplicates. Move/correct GitHub concurrency claims; reconcile GitLab's old variable-storage cell with its signed-branch state cell. |
| Concurrency control | Fullsend admission policy controls competing runs for an entity/context/agent; exact granularity and follow-up policy remain open. An inbox alone is not that policy. |
| State storage | Source checkpoints, candidate responsibility, selected-run records and Kubernetes status, with pilot persistence open and later shared durable backend optional. |
| Dispatch integrity | Authorized dispatcher/launcher, trusted run specification, restricted Kubernetes write permissions and run-to-workload binding; no arbitrary workload authority from a candidate. |
| Agent credentials | Source-specific scoped credentials; static secrets for execution PoC, private federation/mint path subject to validation. |
| Inference credentials | Static inference credentials for execution PoC; projected workload identity exchanged for inference access in the relevant federation experiment. |
| Runner / executor | Kubernetes Job Pod running `fullsend run`; gateway-managed sandbox is separate. |
| CLI install | Proposed version-pinned run image containing the CLI, or explicitly open packaging choice. Do not put gateway lifecycle here. |
| Agent sandbox | Remote OpenShell gateway and Kubernetes-backed sandbox; sandbox owns workspace PVC. |
| CI template | Proposed trusted Job specification/launcher adapter replaces forge workflow/pipeline scaffolding for the managed path. |
| Harness evaluation | Input and context semantics; separate current event-based API from proposed entity-first extension. Avoid duplicating the routing row without adding this distinction. |
| Mid-run steering | Follow-up policy remains open; identify the policy owner and link to execution behavior. |

Add missing rows for tenant/configuration source, candidate handoff and run/result history if needed, rather than putting these facts into unrelated existing rows. Preserve original cells and put corrections alongside them or in clearly associated correction rows.

### 5. Security Model (`#security`)

**Rework the Kubernetes column around actual trust boundaries.** For each row, identify the caller, protected resource, proof checked, and authority granted. Run IDs and acknowledgement are recovery mechanisms, not caller authentication.

- Event → dispatcher: authenticate source deliveries; bind them to registered tenant/source; treat content as untrusted even when transport is authenticated; use trusted endpoints and credentials to refetch/enrich.
- Dispatcher → agent: only the authorized launcher may create the approved Job specification; bind run/context to permitted identity and resource scope. Kubernetes API authorization is part of enforcement, not a substitute for application authorization.
- Agent → forge: scoped source credential and authorization limits; static PoC versus intended federation labelled separately.
- Agent → inference: inference-provider authority only. Remove the unrelated repository mint alternative from this row.
- Identity pinning: distinguish dispatcher identity, Job service account and sandbox identity. State which bindings the PoC must validate; do not assume namespace/service-account labels alone establish tenant authority.
- Debug trace guard: generalize the protection to secrets in logs, traces and artifacts across service, Job and sandbox. Do not claim the existing GitHub “no stored PAT” text proves no secret-exposure risk; add a correction.
- Fork protection: source/target provenance, trusted configuration/ref selection, and restrictions on executing untrusted changes with authority. Actor refetch/kill-switch statements do not answer this row.
- Authorization gate: separate initiating actor permission, tenant/context policy, and workload resource scope. Repository-less does not mean authorization-free.
- Add a row or inset for Job ↔ remote gateway/sandbox: gateway authentication, workspace separation and result-transfer boundary. Keep PoC shortcuts visible as limitations.

Preserve the GitLab trigger-token threat model as a platform-specific subsection. Do not generalize it into a Kubernetes trigger-token requirement. Qualify original “unforgeable/trusted/platform-guaranteed” claims: transport provenance does not make issue content or arbitrary caller-controlled inputs trusted.

### 6. Event Coverage and Latency (`#events`)

**Replace the repeated non-answer.** Every Kubernetes event cell currently says “Connector determines coverage; no latency SLA”, including scheduled prioritize, which is a scheduling use case. Repetition fills a column without enabling comparison.

- Separate **event semantics supported by an adapter** from **notification/discovery delay**, **dispatch delay** and **workload start delay**.
- Explain the source-dependent coverage rule once. Each row should state its dependency or unknown: issue creation needs creation/activity discovery; slash commands need comment identity/actor/body; labels need reliable transition/state semantics; PR/MR lifecycle/reviews need the corresponding adapter/API coverage. Mark these proposed requirements, not promised support.
- Add Jira activity explicitly because the worked example depends on it, without making Jira the selected pilot source.
- Scheduled prioritize needs a time-triggered invocation/context policy; it is not just another connector webhook. Mark its managed support as undecided.
- Treat “Stages” as historical built-in routing examples, not a fixed managed agent roster or a guarantee that those are the only matches.
- Preserve the original numerical claims with dated qualifications. Do not carry forward `<1s` as a demonstrated guarantee merely by relabelling it “event trigger”.
- Add a compact latency explanation: source notification or polling wait → candidate backlog/selection → Job scheduling/image/sandbox startup. No invented timings are needed.

### 7. Credential Model (`#credentials`)

**Organize by consumer and resource, not a single primary credential.** Existing cells mix key custody, workload assertions, access tokens, and scope. “Projected inference identity OR private mint repo access” conflates the PoC's success criterion with an inference architecture.

- Keep the existing comparison, adding corrected definitions: the GitHub App key stays at the mint; the workload presents its assertion; the mint obtains/returns a scoped installation token. Do not draw the key as producing the workload's OIDC identity.
- Add a small managed credential-flow table: poller/normalizer → source API; receiver → webhook verification; launcher → Kubernetes API; run/sandbox → source API; run/sandbox → inference; runner → remote gateway; result reporter → explicit output destination.
- For each, show credential holder, validation/issuer, allowed scope and lifecycle/rotation responsibility, with unknowns stated precisely. Do not assume the gateway's credentials are identical to the Job's.
- Execution PoC: identify static source and inference secrets. WIF PoC: demonstrating one of inference federation or private repository mint is sufficient for that experiment. A complete service still needs a valid credential mechanism for every resource it uses.
- “Storage” must answer storage/custody, not only scope. Kubernetes Secrets are sufficient for the stated PoC; later secret delivery is open.
- Retain the project-count row as historical; add that a meaningful managed count depends on consumer/resource/trust boundaries. Avoid suggesting one credential per tenant by default.

### 8. Responsibilities currently supplied by CI (`#responsibilities-ci`)

**Keep and sharpen this section: it directly answers the user's original question.** The current matrix collapses CI, source systems, existing Fullsend code and operators into one “Current CI owner”, then bundles unrelated responsibilities into rows.

- Split current ownership into GitHub path and GitLab path. Identify which part is already Fullsend logic, especially polling checkpoints and actor authorization.
- Use a concise managed split: **Fullsend policy/logic**, **Kubernetes mechanism**, **external service/operator**. Kubernetes can enforce workload deadlines and supply scheduling/status mechanisms; “K8s executes” understates its role. A queue does not supply application fairness automatically.
- Split overloaded rows: delivery authentication vs discovery; scheduling vs recovery/replay; actor authorization vs workload credentials; admission/order vs cancellation/timeout; logs/artifacts vs status reporting/audit.
- Each row answers: who owns it now, who would own it in managed mode, what existing component can be reused, and what behavior remains to be built or chosen.
- Move repeated warnings about concurrency not being deduplication into one definition and link to it. Spend saved space describing actual ownership.

### 9. State, identity, and recovery (`#state-identity-recovery`)

**Strong inventory; missing operational explanation.** Keep the state categories, identities and failure cases. Replace compressed labels with a consistent acknowledgement/recovery model.

- Show two acceptance boundaries: candidate handoff acceptance and selected-run admission. In direct mode, the handoff must not acknowledge responsibility that cannot be recovered. In inbox mode, durable enqueue can accept the candidate before selection; the consumer acknowledgement follows recoverable handling of every selected run or a recorded no-op/rejection.
- For each state category, identify owner, what survives restart, and cleanup/replay implications. Distinguish a logical ledger from a mandatory database.
- Explain no-match, unauthorized, temporarily unreadable source/config, invalid candidate, and failed workload as different outcomes. Recoverable errors cannot be silently acknowledged as ineligible work.
- Expand the failure table with **failure point → durable evidence available → recovering component → next action → acknowledgement condition**. Apply it separately where direct mode and inbox mode differ.
- Address selection across retries: current configuration applies to a fresh selection; once a run set is admitted, recovery must not accidentally reinterpret the same attempt into different agents/targets. Present recording the selected set/configuration revision as a proposed contract, with policy for deliberate reselection/replay left open.
- State that the current GitLab CAS operation protects state writes; the combination of external pipeline creation and state persistence is not one atomic operation. Replace “CAS not atomic”.
- Preserve duplicate Job, duplicate Pod execution, and repeated external effect as separate issues. Explain retention beyond Job deletion without choosing a store.
- Preserve backlog fairness: bounded source queries must allow later candidates to progress, not continuously rediscover the same first batch.

### 10. Worked example (`#worked-example`)

**Replace with an actual example.** Use named but explicitly hypothetical configuration and agents; do not imply a fixed runtime agent inventory.

1. Tenant `example-team` has a registered Jira source, repository-less context `planning`, and repository context `service-a`. Their configured association makes PROJ-42 applicable to those contexts. Another repository context is excluded.
2. A Jira observation with a provider activity identity is discovered. Show a short candidate-reference record with tenant/source/entity/activity identity and no credentials/content. Field names are illustrative.
3. Dispatcher fetches issue/activity and actor permission, loads configuration revision R7, and creates normalized evaluation input. State the illustrative predicates: an authorized request matches a planning-analysis harness; a configured link/condition matches a repository-triage harness. A third harness does not match. Do not select a review agent from a Jira issue without establishing a change proposal/review condition.
4. Show two selected runs with different context/agent identities, explicit target/no-target and output destinations. Roles/permissions are separate fields from agent names.
5. Show two correlated Jobs and their eventual application outcomes.
6. Replay the same observation: it finds the same admitted runs rather than selecting duplicate Jobs.
7. Crash after Job A is created but its response is lost, before Job B exists: recovery finds/validates A and creates B. Show what record makes that possible and label the storage mechanism as open.
8. Add a short active-active variant: the shared handoff accepts once/delivers at least once; another dispatcher recovers using the same identities and admission contract. Do not imply an atomic transaction spanning the inbox and Kubernetes API.

Replace the linear lifecycle table with a branched lifecycle: observed → accepted handoff → selected or no-op/rejected → admitted/Job created → pending/running → succeeded/failed/cancelled → retained result/cleanup. State which statuses describe candidates, runs, or workloads. Avoid a prescribed public API/schema.

### 11. ADR Evolution (`#adr-evolution`)

**Retain as history, reduce ambiguity.** The timeline mixes accepted decisions, implementation status, and present-tense PR claims; ADR 0123 appears after 0125 without a clear organizing rule.

- Preserve original timeline text as historical. Attach dated corrections, especially where identity pinning is “in progress” here but implemented elsewhere.
- Separate ADR status from implementation evidence. Accepted entity-first design is not evidence that current dispatch has nullable events.
- Put the added ADR 0123 entry in chronological/numeric order, or explicitly separate “additional relevant decisions” without implying chronology.
- Keep the managed phase roadmap in Evolution/Backlog, linking to it instead of duplicating it at the timeline's end.
- New claims about PR status require verification and a date; the historical snapshot can retain its original date without fresh network research.

### 12. Ship-Gating Risks (`#ship-gates`)

**Turn scope lists into verifiable gates.** The GitLab introduction repeats itself. The managed “Evidence” column frequently names topics rather than evidence: “Evaluate inbox/ledger/takeover/leases/replay/fairness/retention” does not say what constitutes success.

- Preserve original GitLab findings with adjacent dated corrections; distinguish completed foundation work from the remaining webhook integration gates.
- For each managed phase use **capability → demonstration required → unresolved blocker → tracking**.
- Execution PoC: restricted Job completes representative work through the remote gateway, transfers results, and cleans sandbox/PVC resources after success/failure/cancellation. The Job does not mount the sandbox PVC.
- Pilot: demonstrate two-context selection, no-match/error distinction, lost-create-response recovery, partial fan-out, restart and duplicate handling without mandatory broker/database.
- Webhook: authenticate a delivery over the approved private route, converge with polling, and recover missed delivery through polling.
- Identity experiment: prove the chosen one of two credential paths and reject unauthorized claims. Do not present this as proof of every credential flow needed for production.
- Active-active: multiple producers/consumers survive failures around enqueue, claim and create; no lost eligible work, controlled duplicate Jobs, observable backlog/retries, and progress beyond one source batch. Direct mode remains supported.
- Separate experiment completion, pilot readiness, and production operational readiness. Do not invent SLOs or present all production operations as covered by AISDLC-147.

### 13. Evolution and backlog (`#evolution-backlog`)

**Make this the primary explanation of evolution from existing code.** The existing component table is useful but overly abbreviated and too far removed from the proposed flow.

- Add **layer/role → current package responsibility → proposed extraction/adaptation → preserved repository behavior → tracking**.
- Show GitLab discovery/selection/pipeline launch being separated; Jira discovery/CEL/output-file checkpoint becoming candidate handoff; shared normalization/selection retained; new managed launch adapter around `fullsend run`; remote gateway execution adaptation.
- Clarify `harnessdispatch`: `Event` is required; `ConfigDir` uses a repository-root assumption. OWNERS-based role augmentation is conditional, not a universal requirement that an OWNERS file exist.
- Clarify CLI drivers: `gha-event`/`json` are inputs, `gha-matrix`/`json` are outputs, and output projection is not workload launch.
- Add the phase progression here and link to the two-phase architecture view. Execution/configuration/repository-less capability are prerequisites as relevant; webhook and active-active are separable extensions.
- Use full `AISDLC-65` style link labels. Give adjacent features brief scope notes; do not give MCP access or cluster provisioning the same dispatch prominence as selection and handoff.
- Place implementation limitations here once and link from flow annotations. Avoid repeating “not implemented/no K8s driver/public mint out of scope” throughout the main explanation.

### 14. Open decisions (`#open-decisions`)

**Replace vague questions with decisions and their consequences.** Current rows mix fixed backlog constraints with questions and combine unrelated scopes.

- Use columns: **decision**, **fixed constraints**, **options/tradeoff**, **which behavior/phase it affects**, **evidence needed**, **tracking**.
- Separate: pilot source; direct-handoff durability; run/config revision and replay semantics; same-entity concurrency; manifest authority/delivery; private webhook route; source/inference/gateway credential paths; tenant authorization/placement; shared inbox coordination; retention/operations.
- Explain storage choices only as far as needed to show boundaries: transactional inbox/run records versus broker plus durable run records versus source-native coordination. Do not choose a technology or claim their guarantees are interchangeable.
- Split identity PoC selection from the eventual requirement to provide all needed credentials. “Inference OR mint” is an experiment choice, not a permanent resource-access choice.
- Move fixed constraints out of questions: private ingress, separate service/agent pools, manifest reuse and no mandatory pilot broker are already given.
- Cross-link every consequential open decision from the layer or gate it affects. No orphan questions unrelated to the architecture.

## Execution order for the editor

1. Snapshot the current HTML for comparison. Restore historical text from the original and attach current corrections. Keep the source original untouched.
2. Establish the glossary, five layer questions, ownership model and two phase views. Resolve conceptual inconsistencies before CSS work.
3. Rebuild Component Comparison by row name, then Security and Credentials by boundary/consumer. Do not populate tables from an unlabelled positional list.
4. Rewrite the worked example and state/recovery explanation against the same identities and acknowledgement boundaries. Check that they describe one coherent design.
5. Revise coverage/latency, evolution/backlog, gates and decisions. Move duplicate caveats into their primary sections and leave links.
6. Write the new summary last, so it accurately summarizes the revised design. Align navigation and history/provenance.
7. Perform semantic and preservation checks, then browser/layout checks. Report meaningful changes and unresolved design choices.

## Verification that catches the actual failures

### Semantic checks

- Read each table row aloud as “For [row responsibility], this option uses [cell]”. Reject any cell that answers a different question. Specifically recheck the mismatched rows listed above.
- For every architecture box, identify input, operation/component, output and owner. Move tasks, risks and configuration notes out of the main arrow sequence.
- Trace one candidate through all five layers and the worked example. The names, context selection, selected-run identities, Job relationship and outcome must agree.
- Trace duplicate delivery and lost-create-response recovery in both pilot and active-active. Identify who can safely acknowledge each step and why responsibility survives restart.
- Check every security row for a trust decision and every credential row for a resource-specific credential flow. Run identity alone is not authentication.
- Confirm all declared gaps have a primary explanation and an open decision or gate; all claimed guarantees have a mechanism or are explicitly proposed requirements.
- Cross-check contradictory claims already found: GitLab dual routing vs unwired CEL; variable dedup vs signed-branch state; GitHub event-required code vs nullable-event claim; pinning implemented vs in progress; optional OWNERS vs “requires OWNERS”; no Job PVC mount vs no PVC anywhere.

### Preservation and structure

- Re-run an extracted-text comparison against the untouched original for original paragraphs, cells, node labels/details and findings, and compare original link targets. Require each original passage/link to remain, or document a narrowly structural relocation. Do not treat new prose as preservation of the original.
- Keep corrections dated and locally associated with affected text. Retain anchors and correct the TOC order.
- Check unique IDs, valid fragment links, table cell counts, header scopes and accessible captions. Structural validity alone cannot establish semantic row alignment.
- Replace inherited percentage column widths that already consume 100% before the new Kubernetes column. Use readable widths and horizontal overflow for genuinely wide tables.
- Preserve readable prose rather than minifying new sections into giant lines. This helps later editors maintain row associations and source links.

### Rendering

Use the browser skill when performing the eventual visual inspection. Inspect desktop, narrow/mobile, light/dark and print. This review examined HTML content and structure; it did not perform a rendered visual review.

- Main flows must be understandable without reading tiny implementation caveats.
- Corrections must be distinguishable from proposed behavior and historical text without relying solely on color.
- Side inputs must not look like downstream transformations. Deployment notes must not look like execution steps.
- Avoid forcing every long section/table to stay on one printed page; inspect existing `break-inside: avoid` behavior.

## Ready-to-use handoff

Revise `dispatch-architecture-github-gitlab-kubernetes.html` using this plan. Preserve original text and add dated corrections. Repair semantic comparisons and flows before styling. Show the pilot and active-active topology using the same selection/launch core, and demonstrate the contract with the Jira example and recovery scenarios. Keep technology choices open, tie proposed behavior to existing Fullsend components, and verify table-row meaning as well as HTML structure. Do not implement services or modify Jira/source code. Report the finished file, substantive repairs, remaining decisions and validation performed.
