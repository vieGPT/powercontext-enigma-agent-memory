---
type: concept
title: Artifact Families, Revisions and Handles
description: Defines the seven built-in Artifact families, immutable Revisions, ArtifactRef and StableHandle identities, Scope-local tags, and the Create, Replace, and candidate write rules that apply to each family.
tags: [artifacts, revisions, artifact-families, stable-handles, concurrency, tags, candidates]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-07T09:04:09.583Z
sources:
  - id: openwiki-source-5d527dc21363023c24fa90ac
    resource: repo://docs/en/docs/workflows/manage-artifact-tags.md
  - id: openwiki-source-3f5bfb980245badd4312b657
    resource: repo://src/powercontext/artifacts/models.py
  - id: openwiki-source-9468400cec6226786b9b0e65
    resource: repo://src/powercontext/artifacts/protocols.py
  - id: openwiki-source-859ac8d3c168625e9f408138
    resource: repo://src/powercontext/builtin/artifacts/atomic_memory/models.py
  - id: openwiki-source-2b14ff31b2960c3f23be261c
    resource: repo://src/powercontext/builtin/artifacts/experience/models.py
  - id: openwiki-source-1389451753138d60ffaa07c4
    resource: repo://src/powercontext/builtin/artifacts/handoff/models.py
  - id: openwiki-source-04abea61d1b61728a3ac17ce
    resource: repo://src/powercontext/builtin/artifacts/profile/management.py
  - id: openwiki-source-ccdbee614ad00083b79944a5
    resource: repo://src/powercontext/builtin/artifacts/profile/models.py
  - id: openwiki-source-1c23fcce70f0e1fd7989f10f
    resource: repo://src/powercontext/builtin/artifacts/prompt/models.py
  - id: openwiki-source-876df217a610abaaf8aa63ba
    resource: repo://src/powercontext/builtin/artifacts/skill/models.py
  - id: openwiki-source-7754257875e5487a8baf1c99
    resource: repo://src/powercontext/builtin/artifacts/topic_memory/admission.py
  - id: openwiki-source-dd6da54641c5fe877e304b33
    resource: repo://src/powercontext/builtin/artifacts/topic_memory/models.py
  - id: openwiki-source-92f205a8f7cf014ee2ba00ca
    resource: repo://src/powercontext/builtin/persistence/artifact_handles.py
  - id: openwiki-source-eaaca96497f97c198e453e99
    resource: repo://src/powercontext/builtin/persistence/artifacts.py
  - id: openwiki-source-5546b1554168a5e96f533f49
    resource: repo://src/powercontext/builtin/persistence/family_management.py
  - id: openwiki-source-ae59440079291b3ce8d6b3b3
    resource: repo://src/powercontext/builtin/persistence/records.py
  - id: openwiki-source-e3af3c2843f364b75038abb6
    resource: repo://src/powercontext/builtin/records.py
  - id: openwiki-source-2f092f75e7ce930d59017ca8
    resource: repo://src/powercontext/builtin/review/service.py
  - id: openwiki-source-d37a6a98f983904386ead9b1
    resource: repo://src/powercontext/builtin/runtime/relational.py
  - id: openwiki-source-da43ec13d0a3646f267bc5bd
    resource: repo://src/powercontext/builtin/tags.py
  - id: openwiki-source-f512aff5e92653f63b08d50b
    resource: repo://src/powercontext/http/_generated/models.py
  - id: openwiki-source-9d25d08497605956bd5fbe48
    resource: repo://src/powercontext/limits.py
  - id: openwiki-source-f25c1b3c7f796f0c6487a9cb
    resource: repo://src/powercontext/server/app.py
generated: { by: "openwiki/0.9.0", at: "2026-11-07T09:04:09.583Z" }
---

# Artifact Families, Revisions and Handles

An **Artifact** is a versioned, Scope-local result. Every Artifact belongs to one **family**, has a stable `artifact_id`, and advances through immutable **Revisions**. Families share the revision storage model, identity types, and ETag discipline, but each family owns its own content schema, validation, and write workflow. Sharing the REST envelope does not make those workflows interchangeable.

This page covers the shared model. Family-specific behavior is in the workflow pages linked from [Related workflows](#related-workflows).

## The seven families

The built-in families are `atomic-memory`, `experience`, `skill`, `handoff`, `profile`, `prompt`, and `topic-memory`. Each concrete Artifact type declares its identifier as a class-level `family` value, for example `atomic-memory` in the Atomic Memory models, `topic-memory` in the Topic Memory models, and `profile` as the Profile family constant.[^families]

The legacy `memory` family is not a writable Artifact family. Legacy Memory collections were archived, and their entries are cited as Atomic Memory revisions instead.[^legacy]

| Family | Identity of the current head | Direct Create and Replace | Candidate path |
| --- | --- | --- | --- |
| `atomic-memory` | Server-generated | Routed through the Atomic Memory domain writer, which checks the current revision before preparing the write | Dream proposals for memory are reviewed separately |
| `topic-memory` | Server-generated | Complete `title`, `summary`, and `detail` are validated by Topic Memory admission | Dream proposals are reviewed through the Candidate approval path |
| `experience` | Server-generated | Validated and written through the Experience family writer, which also refreshes the Experience index | Generated proposals are Candidates; approval publishes the revision |
| `skill` | Server-generated | Canonical package content is resolved before the write, then the Skill index is refreshed | Generated proposals are Candidates; approval publishes the revision |
| `handoff` | Fixed Scope singleton, `handoff` by default | Create only when none exists; afterward Replace only | Reviewed Dream handoffs are published through the Candidate path |
| `profile` | Fixed singleton `profile` | Create and Replace coordinate the Profile policy and mark the Profile source window dirty | Generated Profile Candidates are decided through their own review step |
| `prompt` | The registered `prompt_key` | Create requires `prompt_key`; content is checked against the Prompt registry | Reviewed Prompt configuration is published through Artifact compare-and-set |

The Create branch and the Replace branch described below apply to every family. Candidate review is a separate branch that has its own `expected_version` contract.[^candidates]

## Immutable revisions and heads

Each Revision is a stored snapshot. The body row for `(scope_id, family, artifact_id, revision)` is inserted once and never changed. A separate **head** row records the current revision for each logical identity. Reads use the head to find the latest revision, or the exact revision number to load history.[^repo-insert]

`ArtifactRepository.revise` advances the head only if the caller still holds the current revision. The head row is locked and checked, the new body is inserted, and the head is moved with a compare-and-set on the previous revision. If the head moved in between, the write raises `RevisionConflictError`.[^repo-revise]

Lineage is stored per revision. A Revision records its direct source evidence, its upstream Artifact references, and, for copied Artifacts, its publication source and content digest. Lineage is loaded with the Revision and is not rewritten by later revisions.[^lineage]

The read-only `ArtifactCatalog` protocol exposes `get` for an exact revision, `latest` for the current visible revision, and `revisions` in ascending revision order. The `ArtifactStore` protocol commits only new revisions through `add` and `revise`.[^protocols]

## ArtifactRef and ArtifactAddress

`ArtifactRef` is the exact reference to one revision. It has three fields: `family`, `artifact_id`, and `revision`, where `revision` is an integer of at least 1. Validation rejects leading or trailing whitespace, empty values, and over-long values. It also rejects the legacy `memory` family with `InvalidArtifactReferenceError`.[^ref]

`ArtifactAddress` adds a `scope_id` to an `ArtifactRef`. It identifies one exact revision across Scope boundaries and is used by publication lineage.[^ref]

Citations written by clients must keep the family, Artifact ID, and exact revision. The current head can advance while a historical revision stays unchanged.[^docs-artifacts]

## StableHandle: permanent Scope-local names

A `StableHandle` gives one exact Artifact revision a human-copyable name. It has a `family` and a `key`. The key must match the pattern `^[a-z0-9]+(?:-[a-z0-9]+)*$`, meaning lowercase ASCII letters and digits separated by single hyphens, and it may be at most 128 characters.[^handle-model]

Handles are bound with `PUT /v1/scopes/{scope_id}/artifact-handles/{family}/{key}` and resolved with `GET` on the same route. The route path validates `family` and `key` before the handler runs.[^handle-route]

The binding rules are as follows.

- A binding is keyed by `(scope_id, family, handle_key)` and is permanent. The repository has `bind` and `resolve` operations only; no update, release, or delete operation exists.[^handle-repo]
- Binding a handle to the revision it already names returns that same binding. This makes repeated calls idempotent.
- Binding a handle to a different revision raises a conflict. The handler maps it to HTTP `409 Conflict` with the error code `idempotency_conflict`.
- The target revision must already exist in the Scope. Otherwise the binding fails with not found.
- The handle's family must match the target's family, or the binding is rejected as an invalid argument.
- If two binders race on the same key, the losing transaction re-reads the committed binding. It succeeds only when the committed target is the one it requested; otherwise it raises the conflict.

```mermaid
flowchart TD
    A["PUT handle with family and key"] --> B{"Family matches target?"}
    B -- no --> R1["Reject invalid argument"]
    B -- yes --> C{"Key already bound?"}
    C -- yes --> D{"Same revision?"}
    D -- yes --> E["Return existing binding"]
    D -- no --> F["Return 409 conflict"]
    C -- no --> G{"Target revision exists?"}
    G -- no --> H["Return 404 not found"]
    G -- yes --> I["Insert permanent binding"]
```

Figure: the decision path for a handle bind. A conflicting binding is never overwritten.

Verify the family, Artifact ID, revision, and key before binding, because a wrong binding cannot be changed. A mistaken key must be replaced by choosing a new key.[^docs-artifacts]

## Create and Replace

Both writes create an immutable Revision and return the resulting head with an `ETag`. Their behavior is defined by the server, not by the client.[^docs-artifacts]

```mermaid
sequenceDiagram
    participant Client
    participant Route as HTTP route
    participant Records as Records service
    participant Writer as Family writer
    participant Store as Artifact repository
    Client->>Route: POST artifacts with family and content
    Route->>Records: create_artifact
    Records->>Writer: validate create and select artifact_id
    Records->>Store: insert Revision 1 and head
    Store-->>Client: created response with Location and ETag revision 1
    Client->>Route: PUT artifacts family id with If-Match
    Route->>Records: replace_artifact
    Records->>Store: check head against If-Match and insert next Revision
    Store-->>Client: revision response with ETag revision N plus 1
```

Figure: Create issues Revision 1 under a generated or fixed identity. Replace requires the current head as its precondition and then writes the next revision.

**Create** (`POST /v1/scopes/{scope_id}/artifacts`) takes the `family` discriminator and `content`. The request union is discriminated by `family`, so content from one family cannot be submitted to another. The server chooses the Artifact ID for every family except two cases: Prompt uses the required `prompt_key` as its ID, and Handoff uses the Scope's fixed singleton ID.[^create-route][^create-records]

Create of the legacy `family=memory` collection is rejected. The request is reported as `422 legacy_memory_operation_unsupported`, and the response names the Atomic Memory alternatives.[^legacy-error]

**Replace** (`PUT /v1/scopes/{scope_id}/artifacts/{family}/{artifact_id}`) requires an `If-Match` header. A missing header returns `428 precondition_required`. The value must equal the current head's ETag, which has the form `"revision:N"`. A stale value returns `412` with the error code `revision_conflict`. The service checks the head again inside the write transaction, so a late concurrent update also yields `412`.[^replace-route][^etag][^replace-records][^error-map]

Replace of the `memory` collection family is rejected with `BaseOperationNotSupportedError` before any write.[^replace-route]

### Handoff singleton behavior

Handoff is a Scope singleton. Its Create uses the configured Handoff artifact ID, which defaults to `handoff`. Create returns `409 artifact_already_exists` when that Artifact already exists. The error payload sets `use_replace` to true, so clients switch to Replace.[^handoff-create][^handoff-default][^already-exists]

Handoff Replace requires the target to be the Scope's configured singleton. Any other identity is rejected as an invalid request.[^handoff-replace]

## Write versus candidate rules

The family writers decide what a direct write means. The rules differ by family.

- **Direct writers**: Atomic Memory, Topic Memory, Prompt, Profile, Experience, Skill, and Handoff are each registered in a family writer registry. The registry resolves the writer for each family and rejects families without one.[^registry]
- **Candidate-first generation**: Experience and Skill generation produces Candidates. An operator reviews a Candidate and approves it. Approval publishes a revision: without a target it creates a new Artifact under a generated ID, and with a target it revises that Artifact. Approval checks the Candidate's `expected_version` before publishing.[^candidate-approve][^candidate-publish]
- **Review requires its own precondition**: Candidate review uses `expected_version`, not an Artifact `ETag`. Access checks still apply to the selected target in both cases.[^docs-artifacts]

Operators who want to publish reviewed content should use the family's own review path. Direct REST writes are for families whose workflow permits them. Prompt and Profile are examples of direct-write families with extra validation: Prompt checks its content against the registry, and Profile coordinates its policy row when written.[^prompt-writer][^profile-writer]

Topic Memory content is submitted as complete text. Topic Memory admission inspects that content: it rejects recognizable placeholder output and coordination-only text, and it expects supplied content to be supported by the cited evidence or the prior history it revises.[^topic-admission]

## Tags on logical Artifacts

Tags attach to the logical identity `(family, artifact_id)`, not to a revision. Creating or replacing an Artifact does not change its tags, and tags survive Server restarts.[^tags-docs]

Tag normalization accepts 1 to 64 Unicode code points per label, with no outer whitespace and no control, surrogate, or unassigned characters. Matching uses NFC normalization followed by case folding. A target holds at most 32 labels, and a filter accepts 1 to 16 labels. Duplicates after normalization are rejected atomically.[^tag-normalize]

The tag ETag is independent of the revision ETag. A tag replacement sends the tag ETag in `If-Match`, and a missing value returns `428`.[^tags-docs]

Tags are discovery metadata, not permissions or model instructions. They are not added to prompts, Skill packages, or publication payloads.[^tags-docs]

## Invariants and failure semantics

- A revision number is never reused for a given identity, and body rows are not updated after insert.
- Heads advance only through compare-and-set on the previous revision. A race yields `RevisionConflictError`, which the Replace path reports as `412`.
- A handle binding, once committed, is never changed. Re-binding the same target is safe; binding a different target returns `409`.
- Legacy `memory` is rejected at the reference layer, at Create and Replace, and in collection reads.
- Create does not accept a client-chosen Artifact ID. The ID is server-generated, except that Prompt uses its registered `prompt_key` and the Handoff and Profile singletons use their fixed identities.

## Related workflows

- [Atomic Memory](../workflows/atomic-memory.md) covers its create, replace, forget, merge, and restore flows.
- [Experience and Skill lifecycle](../workflows/experience-and-skill-lifecycle.md) covers Candidate generation and approval.
- [Memory and Handoff](../workflows/memory-and-handoff.md) covers the Handoff singleton and its commit flow.
- [Topic Memory](../workflows/topic-memory.md) covers complete-content writes and scoped reads.
- [Sources and processing](sources-and-processing.md) explains the evidence that Artifact lineage records.
- [HTTP and MCP API](../architecture/http-and-mcp-api.md) lists the routes used here.

[^families]: Family identifiers are declared on the concrete Artifact types. repo://src/powercontext/builtin/artifacts/atomic_memory/models.py#L116-L122 ; repo://src/powercontext/builtin/artifacts/topic_memory/models.py#L64-L70 ; repo://src/powercontext/builtin/artifacts/profile/models.py#L31-L32 ; repo://src/powercontext/builtin/artifacts/prompt/models.py#L114-L114 ; repo://src/powercontext/builtin/artifacts/skill/models.py#L121-L127
[^legacy]: The Artifact reference layer rejects the legacy family and the operation mapper names the Atomic alternatives. repo://src/powercontext/artifacts/models.py#L37-L37 ; repo://src/powercontext/artifacts/models.py#L45-L61 ; repo://src/powercontext/server/app.py#L6250-L6300
[^candidates]: Candidate review uses an expected_version contract and Family-owned approval paths. repo://src/powercontext/builtin/review/service.py#L658-L664 ; repo://src/powercontext/builtin/review/service.py#L700-L735
[^repo-insert]: Body rows and the head row are written in separate statements, and the body is keyed by the exact revision. repo://src/powercontext/builtin/persistence/artifacts.py#L800-L845 ; repo://src/powercontext/builtin/persistence/artifacts.py#L190-L245
[^repo-revise]: The revise path locks the head with a compare-and-set update, inserts the next revision, then advances the head. repo://src/powercontext/builtin/persistence/artifacts.py#L340-L368 ; repo://src/powercontext/builtin/persistence/artifacts.py#L370-L460
[^lineage]: Lineage is loaded per revision from source, artifact, and publication relations. repo://src/powercontext/builtin/persistence/artifacts.py#L960-L1000 ; repo://src/powercontext/artifacts/models.py#L119-L132
[^protocols]: The read and write contracts are split into ArtifactCatalog and ArtifactStore. repo://src/powercontext/artifacts/protocols.py#L24-L51
[^ref]: ArtifactRef and ArtifactAddress validation. repo://src/powercontext/artifacts/models.py#L45-L61 ; repo://src/powercontext/artifacts/models.py#L105-L117
[^docs-artifacts]: Public workflow documentation for Create, Replace, handles, and Candidate review. repo://docs/en/docs/workflows/artifacts.md#L1-L79
[^handle-model]: StableHandle key validation. repo://src/powercontext/artifacts/models.py#L64-L93 ; repo://src/powercontext/limits.py#L38-L38
[^handle-route]: Handle bind and resolve handlers. repo://src/powercontext/server/app.py#L2915-L2945 ; repo://src/powercontext/http/_generated/operations.py#L3303-L3340
[^handle-repo]: Handle bind, resolve, and conflict logic. repo://src/powercontext/builtin/persistence/artifact_handles.py#L36-L73 ; repo://src/powercontext/builtin/persistence/artifact_handles.py#L75-L82 ; repo://src/powercontext/builtin/persistence/artifact_handles.py#L107-L117
[^create-route]: Create handler and family discriminator. repo://src/powercontext/server/app.py#L2888-L2912 ; repo://src/powercontext/http/_generated/models.py#L6482-L6510
[^create-records]: Create-side ID selection, lineage source capture, and writer dispatch. repo://src/powercontext/builtin/persistence/records.py#L354-L405 ; repo://src/powercontext/builtin/persistence/records.py#L1061-L1064
[^legacy-error]: Legacy memory collection error mapping. repo://src/powercontext/server/app.py#L6250-L6300
[^replace-route]: Replace handler and If-Match extraction. repo://src/powercontext/server/app.py#L3289-L3318
[^etag]: ETag formatting and If-Match requirement. repo://src/powercontext/server/app.py#L3445-L3452
[^replace-records]: Replace precondition checks inside the write path. repo://src/powercontext/builtin/persistence/records.py#L710-L800
[^error-map]: HTTP mapping for precondition, conflict, and not-found errors. repo://src/powercontext/server/app.py#L6386-L6394 ; repo://src/powercontext/server/app.py#L6412-L6465 ; repo://src/powercontext/builtin/records.py#L249-L256
[^handoff-create]: Handoff Create refuses an existing singleton. repo://src/powercontext/builtin/persistence/family_management.py#L357-L470
[^handoff-default]: The Handoff singleton identity defaults to handoff. repo://src/powercontext/builtin/runtime/relational.py#L516-L516 ; repo://src/powercontext/builtin/runtime/relational.py#L661-L672
[^already-exists]: The 409 payload for singleton Create. repo://src/powercontext/builtin/records.py#L208-L216 ; repo://src/powercontext/server/app.py#L6451-L6461
[^handoff-replace]: Handoff Replace must target the singleton. repo://src/powercontext/builtin/persistence/family_management.py#L429-L452
[^registry]: Family writer registry and its registration list. repo://src/powercontext/builtin/persistence/family_management.py#L93-L115 ; repo://src/powercontext/builtin/runtime/relational.py#L672-L684
[^candidate-approve]: Candidate approval entry point and generic publication. repo://src/powercontext/builtin/review/service.py#L658-L664 ; repo://src/powercontext/builtin/review/service.py#L700-L740
[^candidate-publish]: Generic approval creates a new Artifact when no target exists, or revises the target. repo://src/powercontext/builtin/review/service.py#L721-L735
[^prompt-writer]: Prompt writer validates against the registry and reuses Artifact compare-and-set. repo://src/powercontext/builtin/persistence/family_management.py#L186-L260
[^profile-writer]: Profile writer coordinates its policy row and marks the source window dirty. repo://src/powercontext/builtin/artifacts/profile/management.py#L40-L90
[^topic-admission]: Topic Memory content admission. repo://src/powercontext/builtin/artifacts/topic_memory/admission.py#L175-L230
[^tags-docs]: Tag identity, ETag, and storage rules. repo://docs/en/docs/workflows/manage-artifact-tags.md#L1-L131 ; repo://src/powercontext/builtin/tags.py#L70-L100
[^tag-normalize]: Tag label normalization. repo://src/powercontext/builtin/tags.py#L40-L65
