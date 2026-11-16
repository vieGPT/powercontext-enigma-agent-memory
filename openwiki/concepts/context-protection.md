---
type: concept
title: Context and Inference Protection
description: How PowerContext admits final model bytes through check phases before_send and before_release, the allow, block, and withhold verdicts, refusal of protected inference, reuse bindings with their TTLs, and the decision projection and processing-security boundaries that gate what content may enter a model request.
tags: [context-protection, inference-privacy, check-context, reuse-binding, before-send, before-release, decision-projection, processing-security]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-16T11:46:19.799Z
sources:
  - id: openwiki-source-02f24dce454417d7bf9abddd
    resource: repo://src/powercontext/builtin/authz/worker.py
  - id: openwiki-source-0d060442e0f8755edaeee71e
    resource: repo://src/powercontext/builtin/inference/errors.py
  - id: openwiki-source-03618424e07fd3f7427087fe
    resource: repo://src/powercontext/builtin/inference/privacy.py
  - id: openwiki-source-acd133985ece9bdd21beb2f6
    resource: repo://src/powercontext/builtin/inference/protected_transport.py
  - id: openwiki-source-8e71e2a5ae23a278e285e5cd
    resource: repo://src/powercontext/builtin/runtime/application.py
  - id: openwiki-source-dba0875345e17e267953e018
    resource: repo://src/powercontext/builtin/runtime/atomic_memory.py
  - id: openwiki-source-41657f9d8e874e7890576756
    resource: repo://src/powercontext/builtin/runtime/context_bindings.py
  - id: openwiki-source-b578b4c5f465fa93a0adf567
    resource: repo://src/powercontext/builtin/runtime/context_check.py
  - id: openwiki-source-ec5de2c007206fd081f64b3c
    resource: repo://src/powercontext/builtin/runtime/context_reuse.py
  - id: openwiki-source-a2938398ed8944c824737777
    resource: repo://src/powercontext/builtin/runtime/decision_policy.py
  - id: openwiki-source-c18d5d77c87bf380d150b147
    resource: repo://src/powercontext/builtin/runtime/decision_projection.py
  - id: openwiki-source-20a18822be5658d0ccc7f5d9
    resource: repo://src/powercontext/builtin/runtime/topic_memory_processing.py
  - id: openwiki-source-7fcedffdf7ac0f12fb34bac3
    resource: repo://src/powercontext/client/capture.py
  - id: openwiki-source-2d32fe8a08a9420ed136b20a
    resource: repo://src/powercontext/client/session_import.py
  - id: openwiki-source-0e4a909e3f865d9f0f8693be
    resource: repo://src/powercontext/redaction.py
  - id: openwiki-source-f25c1b3c7f796f0c6487a9cb
    resource: repo://src/powercontext/server/app.py
  - id: openwiki-source-2d69328634c5aeb5c5718407
    resource: repo://src/powercontext/server/processing_security.py
  - id: openwiki-source-be2fe6c9bd892492b01e1d49
    resource: repo://tests/test_automatic_prompt_privacy.py
  - id: openwiki-source-9f31641b7b5e454b50b812b3
    resource: repo://tests/test_processing_security.py
generated: { by: "openwiki/0.9.0", at: "2026-11-16T11:46:19.799Z" }
---

# Context and Inference Protection

Context protection is the set of controls that decide whether exact model-bound bytes may leave PowerContext. It has three layers that share one vocabulary of verdicts:

1. **Admission of final bytes** through `check_context`, at two phases: `before_send` (the request body, after the caller's native provider rendering, before anything is written) and `before_release` (the complete raw response, before any content, tool argument, or error is exposed).
2. **Inference protection** at PowerContext's owned provider transports, which runs the same rule engine on every actual HTTP attempt and refuses delivery when enforcement fails.
3. **Reuse and projection bindings** that tie a prepared historical context to its current authorization and source state, and that limit what a decision backend sees.

The protection is opt-in. Its default mode is `disabled`, and the rule engine never calls a model, never redacts, and never rewrites content. The caller still owns model execution and delivery; a check does not authorize spending. For the operator-facing contract, see the [context protection guide](../../docs/en/docs/operate/context-protection.md) and the [inference privacy guide](../../docs/en/docs/operate/inference-protection.md) in the repository docs. This page explains how the implementation fits together.

## Verdicts and phases

The check result carries one of three decisions, `allow`, `block`, or `withhold`, defined by `ContextCheckResult` in [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L123-L136). The rule engine returns the same three values in `PrivacyDecision` ([privacy.py](repo://src/powercontext/builtin/inference/privacy.py#L112-L120)).

| Verdict | Meaning | Action permitted |
| --- | --- | --- |
| `allow` with `coverage.complete=true` | Every supported representation was inspected and no rule fired. | The corresponding action (send or release) may proceed. |
| `block` | A configured forbidden value or format was found. | None. |
| `withhold` | The check could not be completed or the payload uses an unsupported shape, such as an unsupported target, duplicate keys, or an exceeded budget. | None. |
| `allow` with `coverage.complete=false` | Returned only outside enforcing mode. | None: it is not complete coverage. |
| refusal | A protected transport raised `InferencePrivacyError` carrying the phase and a `block` or `withhold` decision. | The call fails before send or before release. |

In enforcing mode, `allow` without complete coverage is never returned to a caller: `check_context` converts it to `withhold` (see the flow below). Refusal is the transport-side form of a non-allow verdict; the exception is deliberately not an unavailable-provider or validation error, so advisory fallbacks and smaller-window retries cannot consume it ([errors.py](repo://src/powercontext/builtin/inference/errors.py#L23-L56)).

The limits that bound a check are module constants and config fields, not entries in [limits.py](repo://src/powercontext/limits.py#L1-L40), which holds identifier and field-length bounds for Scope, Source, Artifact, and skill records. The check and reuse limits are:

| Limit | Default | Defined in |
| --- | --- | --- |
| `max_payload_bytes` (send body) | 1,048,576 bytes; allowed range 256 to 8,388,608 | [privacy.py](repo://src/powercontext/builtin/inference/privacy.py#L87-L96) |
| `max_response_bytes` (release body) | 1,048,576 bytes; allowed range 256 to 8,388,608 | [privacy.py](repo://src/powercontext/builtin/inference/privacy.py#L87-L96) |
| `MAX_CHECK_BODY_BYTES` cap on either phase | 1,048,576 bytes | [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L43-L46) |
| `max_json_depth` | 64; allowed range 1 to 128 | [privacy.py](repo://src/powercontext/builtin/inference/privacy.py#L87-L96) |
| `response_timeout_seconds` | 60 seconds | [privacy.py](repo://src/powercontext/builtin/inference/privacy.py#L87-L96) |
| `max_concurrent_responses` | 16; allowed range 1 to 256 | [privacy.py](repo://src/powercontext/builtin/inference/privacy.py#L87-L96) |
| `MAX_PREPARED_CONTEXTS` | 16 prepared blocks per check | [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L43-L46) |
| `SEND_VALIDITY` | 60 seconds | [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L43-L46) |
| `RESPONSE_CORRELATION` | 300 seconds | [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L43-L46) |
| `REUSE_RETENTION` | 5 minutes | [context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L51-L51) |
| `MAX_REUSE_DEPENDENCIES` | 64, equal to `MAX_ATOMIC_MEMORY_DEPENDENCIES` | [context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L47-L47), [atomic_memory.py](repo://src/powercontext/builtin/runtime/atomic_memory.py#L120-L120) |
| `MAX_REUSE_DEPTH` | 8, equal to `MAX_ATOMIC_MEMORY_DEPENDENCY_DEPTH` | [context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L47-L52) |
| Reuse content in a validated proof | at most 32,768 bytes | [context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L324-L332) |

## Check flow

`check_context` is a pure admission function over the complete carrier. `ScopedContextApplication.check` wraps it, first checks scope access, and supplies a `validate_reuse` callback that reauthorizes retained proofs ([application.py](repo://src/powercontext/builtin/runtime/application.py#L1201-L1229), [application.py](repo://src/powercontext/builtin/runtime/application.py#L1175-L1198)). The Server route parses the carrier, optionally checks the `x-powercontext-resolve-target` header, authorizes `SCOPE_READ` on the request's Scope through the callback, and derives the binding identity from the current subject, actor, groups, and credential ([app.py](repo://src/powercontext/server/app.py#L4156-L4196), [app.py](repo://src/powercontext/server/app.py#L4197-L4211)).

```mermaid
flowchart TD
    A["check_context called with request and config"] --> B{"Body encodes as UTF-8?"}
    B -- no --> E1["Raise InvalidRuntimeRequestError context-payload-encoding"]
    B -- yes --> C{"Body within phase limit and MAX_CHECK_BODY_BYTES?"}
    C -- no --> E2["Raise ContextCheckTooLargeError"]
    C -- yes --> T{"Target profile configured?"}
    T -- no --> W0["Withhold target_unsupported"]
    T -- yes --> D{"Mode is enforcing?"}
    D -- no --> N1["Before_send runs the rule engine for reason codes and before_release is not inspected"]
    D -- yes --> F{"Rule result allow with complete coverage?"}
    F -- "allow but incomplete" --> W2["Withhold with the assessed reason codes"]
    F -- "block or withhold" --> X1["Return the assessed block or withhold"]
    F -- yes --> G{"Phase?"}
    N1 --> G
    G -- before_send --> H["Validate prepared contexts"]
    G -- before_release --> I["Correlate send binding and retained proofs"]
    H -- "locator, marker, or trust error" --> E3["Raise InvalidRuntimeRequestError"]
    H -- "binding or reuse proof error" --> W3["Withhold context-refresh-required"]
    H -- "all locators and proofs verified" --> J["Deadline is earlier of 60 seconds and reuse expiry"]
    I -- "binding or reuse error" --> W3
    I -- "send binding and proofs valid" --> J
    J --> K{"Seal binding?"}
    K -- "token too large" --> W4["Withhold binding-capacity-exceeded"]
    K -- sealed --> A1["Return allow with binding and valid_until"]
```

Caption: the branches of `check_context` in [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L345-L441). Structural errors raise `InvalidRuntimeRequestError` or `ContextCheckTooLargeError` to the caller, while binding and reuse-proof failures are converted into a `withhold` result.

Several behaviors follow from this flow:

- A body that is not valid UTF-8 raises `InvalidRuntimeRequestError` with `context-payload-encoding` before any rule runs ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L357-L361)). A body larger than the smaller of the phase limit and `MAX_CHECK_BODY_BYTES` raises `ContextCheckTooLargeError` ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L362-L364)). The phase limit is `max_payload_bytes` for `before_send` and `max_response_bytes` for `before_release`.
- A `target_profile` missing from the configured targets returns `withhold` with `target_unsupported`, in every mode ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L365-L366)).
- In enforcing mode, a `block` or `withhold` from the rule engine is returned as-is with its reason codes and coverage. An `allow` without complete coverage is returned as `withhold` with the assessed reason codes ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L376-L383)).
- Outside enforcing mode (`disabled`, `shadow`, `advisory`), `check_context` never converts a rule result into `withhold`. `before_send` still runs the rule engine so that reason codes can be reported, while `before_release` is not inspected. The returned decision is `allow` with `coverage.complete` set to false, so a non-enforcing allow is not complete coverage and must not be treated as a permit ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L368-L375), [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L430-L441)). Target, encoding, size, locator, and reuse checks still apply in these modes.
- Structural errors in the carrier, such as an unsupported JSON pointer, a locator that does not cover a whole text block, an overlap, or a non-user message, are not verdicts. They raise `InvalidRuntimeRequestError`. Binding and reuse-proof errors are not raised: they become `withhold` with `context-refresh-required` ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L385-L401)).
- The Server maps these raised errors to HTTP responses: a block to 403, a withhold to 503, an oversized body to 413, and an invalid request to 422 (see the invariants below).

## Admission of prepared context before send

A `before_send` carrier contains the exact JSON text (`powercontext.model-json-payload.v1`) and, for each retained prepared block, a binding plus a text location given as a JSON pointer and half-open UTF-8 byte offsets ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L65-L116)).

`_validate_prepared` enforces these invariants before it allows the send ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L280-L314)):

- A target profile with `require_prepared_contexts` cannot be sent without at least one prepared context. The check raises `context-binding-required` and does not downgrade to a pure check ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L286-L287)).
- Locators must point at a `user` message's text, in the protocol's supported position (`messages` for chat and Anthropic, `input` for Responses). System or developer content cannot be promoted into a prepared block ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L167-L198)).
- Locators must be strictly ordered and non-overlapping, and they must fall on UTF-8 character boundaries ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L201-L212), [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L280-L314)).
- The bytes located must match the verified proof's `content_bytes` and SHA-256 digest, and the proof's trust must be `historical-data` ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L300-L309)).
- Any historical-data marker that appears in the body must be covered by a bound locator. Markers split across native text blocks cannot be covered and are rejected ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L214-L277)).

The send permit's deadline is the earlier of `SEND_VALIDITY` (60 seconds) and the earliest reuse proof expiry ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L43-L46), [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L402-L404)). The send binding is sealed until `correlate_until`, which is `now` plus `RESPONSE_CORRELATION` of 300 seconds, so the response can be correlated for up to five minutes after the send decision ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L405-L405)).

## Correlation before release

A `before_release` carrier contains the complete raw response and the send binding as `request_binding`. `_correlate` opens that binding under the send purpose and requires the same identity, Scope, target profile, policy fingerprint, and phase `before_send` ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L317-L343)). It then reauthorizes every retained prepared binding through `validate_reuse`, so policy or permission changes between send and release stop release. The release permit is sealed until its 60-second deadline; correlation never extends the send deadline ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L422-L429)).

Two points matter for implementers:

- The send permit records a payload digest of the approved headers and body, but the release correlation path does not compare the release payload with that digest. The release check verifies the response against current policy and rule engine, and it confirms that the send binding is valid. Matching the released response to the exact approved request is the executor's job: it must retain each call's `request_binding` locally, since a stateless Core cannot prove which approved request an external executor actually sent ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L146-L152), [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L406-L421)).
- A policy change alters the policy fingerprint, so a binding sealed under the old policy fails correlation. The fingerprint hashes the whole configuration, including the forbidden values, without storing those values in metadata ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L139-L144)).

## Inference protection at owned transports

Built-in OpenAI and Anthropic clients, configured embedding and rerank paths, and the MiniMax and JEV paths install `ProtectedAsyncTransport` at the final HTTP layer, after native SDK rendering ([composition.py](repo://src/powercontext/builtin/runtime/composition.py#L2184-L2186), [minimax.py](repo://src/powercontext/builtin/inference/minimax.py#L98-L108), [jev_reranking.py](repo://src/powercontext/builtin/artifacts/memory/jev_reranking.py#L203-L203)).

```mermaid
stateDiagram-v2
    [*] --> Disabled
    Disabled --> PassThrough: mode disabled
    [*] --> Observed: shadow or advisory
    Observed --> SendRequest: decision recorded only
    [*] --> Enforced: enforcing
    Enforced --> Withheld: request not inspectable or target unsupported
    Enforced --> Checked: inspect_json before_send
    Checked --> Refused: block or withhold
    Checked --> Sent: allow
    Sent --> Buffered: complete raw response within timeout and byte limit
    Sent --> Uncertain: transport error or timeout
    Buffered --> ReleaseChecked: inspect_json before_release
    ReleaseChecked --> Refused: block or withhold
    ReleaseChecked --> Released: allow with exact approved bytes
    Uncertain --> Refused: withhold send_or_response_uncertain
```

Caption: the protected transport's send and release states, from [protected_transport.py](repo://src/powercontext/builtin/inference/protected_transport.py#L65-L190).

Details that follow from the transport code:

- In `shadow` and `advisory`, the transport records the decision but never refuses delivery, and it does not inspect or buffer responses. Only `enforcing` buffers the response and refuses ([protected_transport.py](repo://src/powercontext/builtin/inference/protected_transport.py#L65-L71), [protected_transport.py](repo://src/powercontext/builtin/inference/protected_transport.py#L121-L124)).
- In enforcing mode the release decision is enforced before the SDK receives a response. After the complete raw response is buffered, `_assess` raises `InferencePrivacyError` for any non-allow verdict; only an allowed verdict reaches the code that builds the returned `httpx.Response` from the exact approved bytes ([protected_transport.py](repo://src/powercontext/builtin/inference/protected_transport.py#L177-L190)).
- In enforcing mode, the transport reads a streaming request body into memory before inspection. A stream it cannot read, a body above `max_payload_bytes`, a target that does not match a trusted profile, or a method other than POST is withheld before send ([protected_transport.py](repo://src/powercontext/builtin/inference/protected_transport.py#L73-L96), [protected_transport.py](repo://src/powercontext/builtin/inference/protected_transport.py#L105-L111)).
- A response with a non-identity `content-encoding`, a body above `max_response_bytes`, a timeout, or a transport error is withheld at release. Timeouts and transport errors carry the reason `send_or_response_uncertain`, and their usage is recorded as incomplete ([protected_transport.py](repo://src/powercontext/builtin/inference/protected_transport.py#L130-L161)).
- The SDK receives only the exact approved decoded bytes; compression and length headers are removed. Streaming responses (`text/event-stream`) use the SSE inspector rather than the JSON inspector ([protected_transport.py](repo://src/powercontext/builtin/inference/protected_transport.py#L162-L172), [protected_transport.py](repo://src/powercontext/builtin/inference/protected_transport.py#L180-L190)).
- Refusals are raised as `InferencePrivacyError` and are not absorbed by advisory decision fallbacks. Usage known at the failure remains attached and is reported.

The Memory write gate and Topic Memory workers inherit this behavior: a refusal leaves the Source window pending rather than publishing or acknowledging it, as described in the [inference privacy guide](../../docs/en/docs/operate/inference-protection.md).

## Rule engine

`PrivacyPolicy.inspect_json` implements the admission rules shared by the transport and `check_context` ([privacy.py](repo://src/powercontext/builtin/inference/privacy.py#L334-L421)). Its order determines which reason code a caller sees:

1. `disabled` allows, and an unknown target profile withholds with `target_unsupported`.
2. Endpoint mismatch, payload too large, duplicate or unsupported headers, and a non-JSON content type withhold.
3. Non-UTF-8 bodies withhold with `encoding_unsupported`.
4. A forbidden value or pattern in headers or text blocks with `block` and `forbidden_content`, which is a complete-coverage verdict.
5. Invalid or too-deep JSON, or duplicate keys, withhold. Opaque image, audio, and file carriers and incomplete tool arguments withhold.
6. For `before_send`: the model must match the profile, unknown fields withhold, streaming requires `allow_streaming` on OpenAI Chat, provider state (`store: true`, `extra_body`, or `truncation: auto`) withholds, `n` must be 1, and cache settings must be approved. Budget checks run last.
7. For `before_release`: a response model must match the profile or its listed `response_models`, and the response must be complete.

Budget checks (`_budget`, [privacy.py](repo://src/powercontext/builtin/inference/privacy.py#L424-L455)) apply to generating protocols. An enforcing generating target without a declared context window and output limit withholds with `budget_unconfigured`. A request whose output limit is missing, zero, or above the configured maximum withholds. The window check requires `validated_utf8_token_bound`, and it withholds when the payload plus reserve plus output limit exceeds the declared window. The rule engine does not treat a character estimate as a hard bound.

Rule matching covers configured exact secret values and the bounded formats `email`, `private_key`, and `api_token`. It is not a general PII classifier or semantic DLP system.

## Reuse bindings and their lifetimes

`prepare` with `include_reuse` returns a `PreparedContextWithReuse` whose `reuse` binding is produced by `prepare_reuse` ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L261-L301)). Before sealing, `prepare_reuse` runs these branches:

1. No Atomic Memory application configured: raises `context-reuse-unsupported`.
2. Roots from another Scope or a family other than `atomic-memory` raise `context-reuse-cross-scope-unsupported` or `context-reuse-family-unsupported` while the roots are selected ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L123-L135)).
3. Roots plus code dependencies above `MAX_REUSE_DEPENDENCIES` raise `context-reuse-dependency-limit`. The dependency snapshot, which includes ancestors and Sources, is checked against the same bound.
4. A cited code file whose digest changed, or whose Scope cannot be read, raises `context-reuse-source-changed` or `context-reuse-code-unavailable`.
5. Otherwise the binding captures the content's digest and byte count, the ordered Atomic Memory roots, a complete dependency snapshot, one dependency per cited code file (its path and SHA-256 digest), and the expiry ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L288-L300)). Reuse is therefore supported only for Atomic Memory roots within a single Scope.

`validate_reuse` reopens the binding and compares the current state with the recorded one ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L304-L357)):

- A binding whose Scope or identity differs is rejected with `context-binding-identity-mismatch` ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L317-L319)). `check_context` reports that rejection as `context-refresh-required`.
- Malformed signed payloads, a root or code dependency that does not validate, a content digest that is not a SHA-256 hex value, or content above 32,768 bytes are rejected as `context-binding-invalid` ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L324-L334)).
- The Atomic Memory dependency snapshot must equal the recorded one, otherwise the proof fails with `context-reuse-source-changed` ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L346-L351)).
- Each cited code file must still have its recorded digest in the Scope's worktree. A file the repository would no longer index counts as removed. Edits to uncited files or an index rebuild do not invalidate the proof ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L138-L152), [context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L239-L258)).
- The combined dependency count is capped at `MAX_REUSE_DEPENDENCIES` ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L352-L353)).

A rejected `context-reuse-source-changed` may carry a content-free `ContextReuseInvalidation` for in-process callers. It lists at most 16 changes and 4,096 bytes of canonical JSON, and it never changes the rejection itself ([context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L47-L52)).

Lifetimes and binding properties:

| Artifact | Lifetime | Source |
| --- | --- | --- |
| Reuse binding | 5 minutes (`REUSE_RETENTION`) | [context_reuse.py](repo://src/powercontext/builtin/runtime/context_reuse.py#L51-L51) |
| Send permit deadline | 60 seconds (`SEND_VALIDITY`), capped by the earliest reuse expiry | [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L43-L46), [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L402-L404) |
| Send permit correlation | 5 minutes (`RESPONSE_CORRELATION`) | [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L43-L46), [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L405-L405) |
| Release permit | 60 seconds, not extended by correlation | [context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L402-L404) |

`BindingCodec` seals each token as authenticated, encrypted AES-GCM with an audience that includes the key ID, deployment ID, generation, and purpose, so a token cannot be replayed for a different purpose ([context_bindings.py](repo://src/powercontext/builtin/runtime/context_bindings.py#L33-L108)). A caller-supplied key must be exactly 32 bytes. Without one, the key is ephemeral and bindings do not survive restart. A sealed token larger than `max_token_bytes`, whose default is 65,536 bytes, raises `context-binding-too-large` inside the codec; `check_context` reports that case as `binding-capacity-exceeded` ([context_check.py](repo://src/powercontext/builtin/runtime/context_check.py#L422-L429)). A binding never grants access: every use rechecks current authorization.

Reuse is not a cache. There is no stored prompt or response cache, and reuse does not create a provider KV cache.

## Redaction helpers

Redaction is separate from admission. [redaction.py](repo://src/powercontext/redaction.py#L31-L55) collects known secret values: environment values for variables whose names contain a sensitive part (`api_key`, `authorization`, `cookie`, `password`, `secret`, `token`) and whose values have at least 8 characters, plus secrets read from the Codex `auth.json`. `redact_known_secrets` replaces those values with `[REDACTED]`, longest first. Decision projection, Client capture, and session import call this helper ([decision_projection.py](repo://src/powercontext/builtin/runtime/decision_projection.py#L206-L206), [capture.py](repo://src/powercontext/client/capture.py#L48-L48), [session_import.py](repo://src/powercontext/client/session_import.py#L250-L250)). The check and rule engine never use it: admission never redacts or restores values.

## Decision projection and processing security

The same verdict discipline extends to decision backends and to background processing, though with separate boundaries.

**Decision projection.** A decision backend receives only a bounded projected request, not the authoritative object. `KnownSecretDecisionRequestSanitizer` replaces known secret values and provider-key prefixes, recognizes JSON evidence values, and returns either a permitted projection with content-free redaction facts (`subject_redacted`, `evidence_redacted`) or a withheld projection with one of the reason codes `projection_material_removed`, `projection_sanitizer_failed`, or `projection_sanitizer_invalid` ([decision_projection.py](repo://src/powercontext/builtin/runtime/decision_projection.py#L51-L60), [decision_projection.py](repo://src/powercontext/builtin/runtime/decision_projection.py#L164-L227)). `apply_decision_request_sanitizer` rejects any result that changes the decision kind or question, or that exceeds the subject and evidence bounds, so a sanitizer can only replace the subject and evidence ([decision_projection.py](repo://src/powercontext/builtin/runtime/decision_projection.py#L287-L325)).

Decision policy maps the backend's outcome to its own verdict vocabulary, `ALLOW`, `DENY`, or `UNKNOWN`, separate from the transport verdicts. A fallback or abstain becomes `UNKNOWN` with coverage `UNADJUDICATED`, and a deterministic local rule can record an adjudicated verdict with no model usage ([decision_policy.py](repo://src/powercontext/builtin/runtime/decision_policy.py#L49-L100), [decision_policy.py](repo://src/powercontext/builtin/runtime/decision_policy.py#L101-L143)). Observation logs record only bounded fields, not a content ledger ([decision_policy.py](repo://src/powercontext/builtin/runtime/decision_policy.py#L144-L171)).

**Processing security.** Scheduled artifact processing in a child process reconstructs Server authorization from trusted settings rather than from model output ([processing_security.py](repo://src/powercontext/server/processing_security.py#L1-L19)). `build_worker_security` returns `None` when it is disabled or the access mode is `disabled`, and otherwise validates the provider before any principal is chosen ([processing_security.py](repo://src/powercontext/server/processing_security.py#L38-L41)). The background principal comes from `background_principal_id`, falling back to the legacy static principal; with neither set in enforced mode it raises `ValueError` ([processing_security.py](repo://src/powercontext/server/processing_security.py#L42-L51)). The serialized spec carries the principal, deployment identity, and static-preset flag, and omits credentials ([processing_security.py](repo://src/powercontext/server/processing_security.py#L52-L56)). Validation raises `BuiltinConfigurationError` before dispatch if the authorization provider or relational repository was injected or is not the standard builtin type ([processing_security.py](repo://src/powercontext/server/processing_security.py#L77-L86)). Topic Memory publications are authorized by Scope authority alone, with no per-topic owner, through the worker's `topic_commit` operation ([worker.py](repo://src/powercontext/builtin/authz/worker.py#L151-L158)), which the Topic Memory processing path invokes ([topic_memory_processing.py](repo://src/powercontext/builtin/runtime/topic_memory_processing.py#L2069-L2069)).

## Invariants and failure semantics

- Only `allow` with complete coverage permits an action. Every other path, including a refused reuse proof, a non-enforcing allow, or an incomplete response, yields no send or release.
- Enforcing mode disables SDK transport retries, while the output-schema retries that remain are bounded and every actual attempt is checked.
- Usage that a protected response consumed is retained on refusal, and cache-read input is not double counted.
- Admission never redacts or restores values. A changed payload, including a reversible placeholder substitution, has different bytes and therefore needs its own admission. Literal placeholder text in an approved body stays ordinary content.
- A correlated release proves only that current policy and the retained proofs still hold; it does not prove execution exactly once or that the model understood its evidence.
- The Server maps errors from the check path to HTTP responses. An `InferencePrivacyError` becomes 403 with code `inference_privacy_block` for a block and 503 with code `inference_privacy_withhold` for a withhold, both with the phase and reason codes. `ContextCheckTooLargeError` becomes 413 `context_payload_too_large`, and `InvalidRuntimeRequestError` (including invalid locators and encodings) becomes 422 `invalid_request` ([app.py](repo://src/powercontext/server/app.py#L7232-L7246), [app.py](repo://src/powercontext/server/app.py#L7252-L7267)).

## Tests that anchor this behavior

- [test_automatic_prompt_privacy.py](repo://tests/test_automatic_prompt_privacy.py#L1-L147) runs each automatic prompt hook against a controlled loopback server. For credential-bearing prompts, the hook skips source capture, keeps preparation running, does not forward the secret in any request, and logs only a `sensitive_content` reason.
- [test_processing_security.py](repo://tests/test_processing_security.py#L43-L65) checks that worker identity prefers the explicit background principal and excludes credentials. [test_processing_security.py](repo://tests/test_processing_security.py#L67-L87) checks that injected or subclassed authorization providers fail before background dispatch, and [test_processing_security.py](repo://tests/test_processing_security.py#L119-L157) checks that the background CLI preserves the deployment identity.

The page's source is limited to the runtime, inference, and server modules cited above. Repository docs under `docs/en/docs/operate/` describe the operator contract and were used for the public status codes and the bounded streaming profile. Behavior beyond the owned transports, such as provider-side retention or routing, is outside this protection's guarantees.

## Related pages

- [Sources and processing](sources-and-processing.md) describes the Source and processing lifecycle that feeds Memory writes gated by these checks.
- [Gateway architecture](../architecture/gateway.md) describes the Server layer that exposes the check and prepare routes.
- [Prepare and recall workflow](../workflows/prepare-and-recall.md) describes the prepare path that produces prepared context and reuse bindings.
