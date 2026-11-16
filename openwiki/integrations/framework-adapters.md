---
type: "Reference"
title: "Framework Adapters"
openwiki_generated: true
verified:
  - by: openwiki/0.9.0
    at: 2026-11-16T11:46:19.799Z
sources:
  - id: openwiki-source-2105e0d1930dd7a02f527f79
    resource: repo://docs/en/docs/integrations/minimax.md
  - id: openwiki-source-766c32a768024e49f7d3ba31
    resource: repo://docs/en/docs/workflows/ingest-text-files-with-opendal.md
  - id: openwiki-source-a4a505efa079a203d126f14d
    resource: repo://integrations/bub/pyproject.toml
  - id: openwiki-source-89d1efeed816f2f8d4298f55
    resource: repo://integrations/bub/src/powercontext_bub/plugin.py
  - id: openwiki-source-22ec35f456bd812a94878d36
    resource: repo://integrations/cindy/plugins/powercontext-hooks/main.js
  - id: openwiki-source-00277400acd241d0ad9c598a
    resource: repo://integrations/cindy/src/worker.ts
  - id: openwiki-source-c6668ad613deb24a82e99c8c
    resource: repo://integrations/dify/plugin/powercontext_dify/client.py
  - id: openwiki-source-593dcf4f69241ff207d6629a
    resource: repo://integrations/dify/plugin/powercontext_dify/policy.py
  - id: openwiki-source-366e06f8bd89a485c9509720
    resource: repo://integrations/dify/plugin/powercontext_dify/tool.py
  - id: openwiki-source-fbbd8efab6d42ecd091b17d5
    resource: repo://integrations/langchain/src/powercontext_langchain/client.py
  - id: openwiki-source-75123cb5b1709f5f9798b13c
    resource: repo://integrations/langchain/src/powercontext_langchain/middleware.py
  - id: openwiki-source-ce9514a455fe7e8f9f307c81
    resource: repo://integrations/langchain/src/powercontext_langchain/settings.py
  - id: openwiki-source-ba19b14fddc5547a0c9e4f00
    resource: repo://integrations/langchain/src/powercontext_langchain/tracing.py
  - id: openwiki-source-fa64b15cfaec81d7b3c1ae32
    resource: repo://integrations/langgraph/src/powercontext_langgraph/client.py
  - id: openwiki-source-cd0c2e99a8958bf1f317463f
    resource: repo://integrations/langgraph/src/powercontext_langgraph/recall.py
  - id: openwiki-source-d1cf48189788f8ea8f7a4a4c
    resource: repo://integrations/langgraph/src/powercontext_langgraph/runtime.py
  - id: openwiki-source-4d6d919ce92f2e4801697dc2
    resource: repo://integrations/langgraph/src/powercontext_langgraph/settings.py
  - id: openwiki-source-481f5c65d41411557cf22350
    resource: repo://integrations/langgraph/src/powercontext_langgraph/tools.py
  - id: openwiki-source-66357d7997e94e0e435bc607
    resource: repo://integrations/minimax/plugins/powercontext/powercontext.mcp.json
  - id: openwiki-source-865dea515613260801694436
    resource: repo://integrations/opendal/src/powercontext_connector_opendal/cli.py
  - id: openwiki-source-d985f64ecbba20ebee7c7849
    resource: repo://integrations/opendal/src/powercontext_connector_opendal/connector.py
  - id: openwiki-source-86df5137e831c75793fbbf45
    resource: repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py
  - id: openwiki-source-8659542dca41e806728ab588
    resource: repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/scope.py
  - id: openwiki-source-11c3a58389866c506eca50f4
    resource: repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/settings.py
  - id: openwiki-source-b5738f1cd7b2265a7c1a6d95
    resource: repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/toolset.py
  - id: openwiki-source-25381ecb444a61faf5a361a6
    resource: repo://src/powercontext/cli/minimax.py
  - id: openwiki-source-ada48487ce77f173d033229e
    resource: repo://src/powercontext/client/client.py
  - id: openwiki-source-b624210e9826c2c6dfc85219
    resource: repo://src/powercontext/client/transport_policy.py
  - id: openwiki-source-6e423d23f5768d9310faf5f4
    resource: repo://src/powercontext/http/_generated/operations.py
  - id: openwiki-source-c679f009aeb3c68ed320bfab
    resource: repo://src/powercontext/sources/connectors.py
generated: { by: "openwiki/0.9.0", at: "2026-11-16T11:46:19.799Z" }
---


# Framework Adapters

PowerContext ships four Python framework adapters: LangChain middleware, LangGraph recall, a Pydantic AI capability, and
a Bub plugin. Each is a thin layer over the public `powercontext.client.PowerContextClient` facade. The same page also
records the platform integrations (Dify, Cindy, MiniMax) and the OpenDAL Connector, which either call the HTTP contract
from their own runtime, install a native plugin, or run as an independent worker.

This page describes what each entry point does on the request path, where it hooks into its host, what it does when
recall returns nothing or fails, and which invariants hold across them. For the transport layer that all Python adapters
share, see [Client, CLI, and transports](../architecture/client-cli-and-transports.md). For the HTTP and MCP surface, see
[HTTP and MCP API](../architecture/http-and-mcp-api.md). For the recall flow itself, see
[Prepare and recall](../workflows/prepare-and-recall.md). For the broader matrix of host capabilities, see
[Capability matrix](capability-matrix.md).

## Adapter map

| Adapter | Entry module | Host surface | Server operations used | Automatic behavior |
| --- | --- | --- | --- | --- |
| LangChain | `integrations/langchain/src/powercontext_langchain/middleware.py` | `AgentMiddleware` for `create_agent` | `resolve_scope_binding`, `prepare_context`, `capture_content_source` | Recall before each model call. Capture after a completed turn only when `auto_capture=True`. |
| LangGraph | `integrations/langgraph/src/powercontext_langgraph/recall.py`, `tools.py` | `pre_model_hook` node and `BaseTool` list | `resolve_scope_binding`, `prepare_context`, `search_memory`, `remember_memory` | Recall through `llm_input_messages`. Memory tools run only when the model calls them. |
| Pydantic AI | `integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py`, `toolset.py` | `AbstractCapability` and `FunctionToolset` | `resolve_scope_binding`, `prepare_context`, `capture_content_source`, `flush_memory`, `search_memory`, `remember_memory` | Recall once per run. Event capture only when `capture_events=True`. |
| Bub | `integrations/bub/src/powercontext_bub/plugin.py` | `PowerContextPlugin` registered under the `bub` entry-point group | `prepare_context`, `capture_content_source`, `flush_memory`, and scope resolution through `resolve_scope_id` | Recall before each LLM call. Event capture only when `capture_events` is true. |
| OpenDAL | `integrations/opendal/src/powercontext_connector_opendal/cli.py`, `connector.py` | Independent process or embedded `RemoteConnectorWorker` | Source Definition registration, checkpoint load and compare-and-swap save, and Source submission through `RemoteConnectorWorker` | One full scan per run. Creates Sources, not Memory. |
| Dify | `integrations/dify/plugin/main.py`, `powercontext_dify/tool.py`, `client.py` | Dify plugin provider and `pc_*` tools | `resolve_scope_binding`, `get_scope`, and the operations named by each tool | No automatic recall or capture. The model calls tools explicitly. |
| Cindy | `integrations/cindy/plugins/powercontext-hooks/main.js` and `integrations/cindy/src/worker.ts` | Ghost hook for `will-user-message` plus read tools | `resolve_scope_binding`, `prepare_context`, `search_memory`, `capture_content_source` | Recall rewrites the user message. The original prompt is captured as a Source. |
| MiniMax | `src/powercontext/cli/minimax.py` (`MiniMaxInstaller`) and `integrations/minimax/plugins/powercontext/` | Native MiniMax plugin with a streamable HTTP MCP entry | None directly. The host opens the MCP session to `/mcp`. | None. The plugin does not capture conversations. |

## Hook points and recall branches

Every framework adapter reads the latest user text, asks the Server for bounded context, and returns a model request
that may carry that context. The table lists where each adapter hooks in and what happens when recall returns nothing
or raises.

| Adapter | Hook point | When recall returns nothing | When recall fails |
| --- | --- | --- | --- |
| LangChain | `wrap_model_call` and `awrap_model_call`, once per model call | No human text: the handler runs unchanged. Empty envelope or unexpected envelope type: `_prepare` returns `None` and the handler runs unchanged. | Validation error, `ClientError`, or other exception: `_prepare` returns `None`. A first HTTP 401 or 403 is logged at error level. |
| LangGraph | `PowerContextRecall.__call__` as the `pre_model_hook`, once per model step | No human text or no content: `llm_input_messages` is the run's messages unchanged. | Client or unexpected error: the hook returns the run's messages unchanged and the failure is not cached, so the next model step retries. Settings error: `_prepare_once` returns `None` and the error is logged once. |
| Pydantic AI | `before_model_request`, once per model request | No query or no content: earlier context blocks are stripped and the request is sent without a new block. Injection state is not set, so a later model request in the same run queries again. | `ClientError` in `_prepare_context` returns `None` and is reported to the auth reporter. Other exceptions from the request build or `prepare_context` propagate. |
| Bub | `before_llm_call`, once per LLM call | No prompt, no scope, a request that already carries the marker, or no content: `None` is returned and the call proceeds unchanged. | `CLIENT_ERRORS` in scope resolution or `_prepare_context` are recorded and `None` is returned. Scope is not cached on failure, so the next call retries. |

```mermaid
flowchart TD
  A["before_llm_call receives request"] --> B{"Latest user text empty?"}
  B -- yes --> Z1["Return None"]
  B -- no --> C{"Scope resolved?"}
  C -- no --> Z2["Return None and record scope failure"]
  C -- yes --> D{"A message already has the marker?"}
  D -- yes --> Z1
  D -- no --> E["Query is recall text or prompt"]
  E --> F{"prepare_context call succeeds?"}
  F -- no --> Z3["Return None and record prepare failure"]
  F -- yes --> G{"Content empty?"}
  G -- yes --> Z1
  G -- no --> H["Return request with marker system message first"]
```

Caption: the Bub recall branch in `before_llm_call`. Every branch that does not inject context returns `None`, so the
model request proceeds unchanged.

## Shared request contract

Every adapter reaches the Server through `PowerContextClient` or, in Dify and Cindy, through an equivalent HTTP
call. The Python facade maps each method to a generated operation. For the operations used here:

- `resolve_scope_binding` posts to `/v1/scope-bindings/resolve` ([client.py](repo://src/powercontext/client/client.py#L510-L512), [operations.py](repo://src/powercontext/http/_generated/operations.py#L1071-L1079)).
- `prepare_context` posts to `/v1/context/prepare` ([client.py](repo://src/powercontext/client/client.py#L1077-L1087), [operations.py](repo://src/powercontext/http/_generated/operations.py#L1326-L1334)).
- `capture_content_source` posts to `/v1/sources/content` and expects `202` ([client.py](repo://src/powercontext/client/client.py#L590-L593), [operations.py](repo://src/powercontext/http/_generated/operations.py#L1140-L1148)).
- `flush_memory` posts to `/v1/memory/flush` ([client.py](repo://src/powercontext/client/client.py#L973-L975), [operations.py](repo://src/powercontext/http/_generated/operations.py#L1903-L1911)).

Two rules apply to every Python adapter:

1. **Scope is resolved by the Server, not derived locally.** Each operation first calls `resolve_scope_binding`. The
   explicit ID is the invocation or settings value, or `None` for the Server default. The adapters never infer a Scope
   from Git, the path, the agent, or the prompt. The Pydantic AI and Cindy adapters resolve once per run or per
   invocation and reuse or re-resolve as described below. Bub resolves once per turn state and caches the result.
2. **Recalled content is untrusted evidence.** Each adapter prefixes the prepared content with the marker
   `PowerContext host-supplied context`, followed by the instruction that the block is untrusted historical evidence.
   LangChain uses this prefix in its appended system block
   ([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L39-L39)). LangGraph, Pydantic AI,
   and Bub use the same marker ([recall.py](repo://integrations/langgraph/src/powercontext_langgraph/recall.py#L35-L35),
   [capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L59-L59),
   [plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L55-L55)).

The Python transport policy is also shared. `PowerContextClient` refuses to send plaintext HTTP to a non-loopback
host unless the caller sets `allow_insecure_http` or supplies a caller-owned `http_client` with
`trust_transport_security=True`. Loopback clients bypass proxy discovery
([client.py](repo://src/powercontext/client/client.py#L360-L420)). The settings base class resolves the
endpoint and consent for every adapter ([transport_policy.py](repo://src/powercontext/client/transport_policy.py#L150-L237)).
Saved consent does not transfer to a different endpoint.

## LangChain middleware

`PowerContextMiddleware` subclasses LangChain's `AgentMiddleware`. It works with `create_agent` and uses only the
public middleware hooks. Its constructor takes `auto_capture: bool = False` and `trace_content: bool = False`
([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L49-L63)).

Recall runs on every model step. `wrap_model_call` reads the latest human message. If there is none, it calls the
handler unchanged. Otherwise it calls `_prepare` and, if content is returned, overrides the request's system message
with a copy that appends a text block. The override is local to that `ModelRequest`, so the recalled block never enters
agent state or a checkpointer ([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L65-L82),
[middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L268-L273)). The async variant
`awrap_model_call` follows the same logic with `await`
([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L83-L97)).

`_prepare` resolves the configuration, opens one client, resolves the Scope, then calls `prepare_context` with the query
clamped to 8192 characters and `max_bytes` from settings. If settings include `context_assembly`, it is forwarded as
`assembly` ([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L131-L164)). A
response that is not a `PreparedContext` and a response with status `EMPTY` both return no content.

Capture is opt-in. `auto_capture` defaults to `False`, and `after_agent` and `aafter_agent` return immediately unless
it is true. When it is enabled, `_completed_turn` takes the latest human text and, as the assistant text, either the
serialized `structured_response` or the final AI message. The final AI message is empty when it still carries tool
calls, so an incomplete turn is skipped ([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L98-L129),
[middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L239-L266)). The source body uses
`User:` and `Assistant:` headers, is bounded to 200,000 characters, and carries `integration=langchain` and
`kind=agent-turn` metadata. Its source ID is `langchain-agent-turn-` followed by the SHA-256 of scope, user text, and
assistant text ([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L166-L190),
[middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L275-L301)).

The synchronous hooks use `asyncio.run` only when no event loop is running. Inside a running loop they log one warning and
return without work, telling the caller to use `ainvoke` or `astream`
([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L192-L205)). The same applies to
synchronous recall and to synchronous capture.

```mermaid
sequenceDiagram
  participant Agent as LangChain agent
  participant MW as PowerContextMiddleware
  participant Client as PowerContextClient
  participant Server as PowerContext Server
  Agent->>MW: wrap_model_call with latest human text
  MW->>Client: resolve_scope_binding
  Client->>Server: POST /v1/scope-bindings/resolve
  MW->>Client: prepare_context with bounded query
  Client->>Server: POST /v1/context/prepare
  Server-->>MW: PreparedContext
  MW-->>Agent: handler with appended system block
  Agent->>MW: after_agent with completed turn
  MW->>Client: capture_content_source when auto_capture is on
  Client->>Server: POST /v1/sources/content
```

Caption: one model step with recall, then capture at the end of a completed turn. Capture only happens when
`auto_capture=True`.

Failures are fail-open. Validation errors, `ClientError`, and unexpected exceptions return `None` for recall, and
capture logs and returns without raising. Authentication failures are logged once per middleware instance, not once per
operation. `__init__` initializes a single `_configuration_error_logged` flag
([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L58-L63)). Recall
([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L131-L164)) and capture
([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L166-L190)) both report client
failures through `_log_client_failure`, which shares that flag
([middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L207-L222)). The first HTTP 401 or
403 that either operation receives is logged at error level, naming `POWERCONTEXT_LANGCHAIN_TOKEN` or
`PowerContextScope(token=...)`. Later 401 or 403 responses from either operation on the same instance, and all other
client failures, are logged at debug level.

Tracing is split into two adapter-owned helpers in `tracing.py`. `genai_client_context` attaches a local OpenTelemetry
context value, `powercontext.genai_client`, around adapter SDK calls. It is never written to wire baggage, so cores that
understand the marker can enrich their own dependency spans, and older cores ignore it
([tracing.py](repo://integrations/langchain/src/powercontext_langchain/tracing.py#L29-L41)). `recall_content_observation`
records recall content only when `trace_content=True`. When enabled, it starts a span named `PowerContextMiddleware recall`
on the `powercontext.langchain` tracer, with `gen_ai.provider.name=powercontext` and `gen_ai.operation.name=retrieval`,
and records the query and the prepared content, each bounded to 16,384 bytes with a truncation flag. When disabled it
creates no span. An exception inside the recall block marks the span as an error and re-raises
([tracing.py](repo://integrations/langchain/src/powercontext_langchain/tracing.py#L43-L89),
[middleware.py](repo://integrations/langchain/src/powercontext_langchain/middleware.py#L131-L164)).

Configuration uses `PowerContextLangChainSettings` with prefix `POWERCONTEXT_LANGCHAIN_`. `max_bytes` must be 512 through
32768 and defaults to 8000. `timeout` defaults to 10 seconds. Invocation-level `PowerContextScope` values override the
environment field by field. `ResolvedConfig` keeps the token out of the representation
([settings.py](repo://integrations/langchain/src/powercontext_langchain/settings.py#L24-L61),
[client.py](repo://integrations/langchain/src/powercontext_langchain/client.py#L27-L81)).

## LangGraph adapter

The LangGraph package has three public parts: `PowerContextRecall`, the three Memory tools, and `PowerContextScope`
read from the graph runtime. Its docs describe the `create_react_agent` wiring
([__init__.py](repo://integrations/langgraph/src/powercontext_langgraph/__init__.py#L14-L31)).

`PowerContextRecall` is a `pre_model_hook`. Its `__call__` takes the run's messages and the latest human text. If no
human text exists, it returns the messages unchanged. If preparation yields no content, it also returns the messages
unchanged. Otherwise it builds `llm_input_messages` as one leading `SystemMessage` followed by the unchanged messages.
The hook never writes recalled content to `messages`, so the persisted history does not accumulate context from earlier
turns ([recall.py](repo://integrations/langgraph/src/powercontext_langgraph/recall.py#L77-L88),
[recall.py](repo://integrations/langgraph/src/powercontext_langgraph/recall.py#L43-L69)).

`_prepare_once` resolves configuration from the active scope. If that fails, `_log_config_failure` logs once per instance
and recall is skipped ([recall.py](repo://integrations/langgraph/src/powercontext_langgraph/recall.py#L90-L100),
[recall.py](repo://integrations/langgraph/src/powercontext_langgraph/recall.py#L163-L171)). `_prepare` then opens a client
and calls `resolve_server_scope`, which posts a scope-binding resolution on every model step. Only after that does it
build a cache key from the scope ID, the human message identifier or text, and a digest of `max_bytes` and `assembly`.
A cache hit returns the stored content without a `prepare_context` call. The cache is a bounded dictionary of 64 entries
that drops the oldest key when full. A `ValidationError`, `ClientError`, or other exception returns `None` before the
result is cached, so the next model step retries. The scope component keeps one shared hook from serving one tenant's
content to another run ([recall.py](repo://integrations/langgraph/src/powercontext_langgraph/recall.py#L102-L147),
[recall.py](repo://integrations/langgraph/src/powercontext_langgraph/recall.py#L149-L161),
[recall.py](repo://integrations/langgraph/src/powercontext_langgraph/recall.py#L186-L199),
[client.py](repo://integrations/langgraph/src/powercontext_langgraph/client.py#L95-L103)).

The Memory tools are `powercontext_search`, `powercontext_remember`, and `powercontext_context`, returned by
`powercontext_tools()` ([tools.py](repo://integrations/langgraph/src/powercontext_langgraph/tools.py#L129-L131)). Each
resolves its config from the runtime scope, opens a client, resolves the Scope, and makes one request:
`search_memory` for search, `remember_memory` for writes, and `prepare_context` for bounded context. Failures do not
raise. `ValidationError` returns an invalid-argument message, `ClientError` returns an unavailable message with the
exception type, and other exceptions return a generic unavailable message. The stated reason is that an empty search
result would otherwise look like "no memory exists" to the model
([tools.py](repo://integrations/langgraph/src/powercontext_langgraph/tools.py#L52-L127),
[tools.py](repo://integrations/langgraph/src/powercontext_langgraph/tools.py#L132-L148)).

`current_scope()` reads `PowerContextScope` from `langgraph.runtime.get_runtime`. It returns `None` when called outside
a runnable context, when no graph runtime is attached, or when the runtime context has the wrong type. In each of those
cases the hooks and tools use environment settings
([runtime.py](repo://integrations/langgraph/src/powercontext_langgraph/runtime.py#L24-L39)).

Connection behavior matches LangChain. `open_client` borrows a context-variable shared `httpx.AsyncClient` when
`shared_http_client(...)` is active and otherwise creates its own client with `config.timeout`. The shared client's
`trust_transport_security` flag defaults to false
([client.py](repo://integrations/langgraph/src/powercontext_langgraph/client.py#L104-L124),
[client.py](repo://integrations/langgraph/src/powercontext_langgraph/client.py#L126-L147)). The settings class uses
`POWERCONTEXT_LANGGRAPH_` and the same field defaults, and it carries a bare `token` rather than a complete
`Authorization` header ([settings.py](repo://integrations/langgraph/src/powercontext_langgraph/settings.py#L24-L61)).

## Pydantic AI capability and toolset

The Pydantic AI package offers two entry points. `PowerContext` is the `AbstractCapability` that adds automatic recall
and optional capture. `PowerContextToolset` is a `FunctionToolset` of the same three Memory tools for applications
that do not want automatic work ([__init__.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/__init__.py#L14-L20)).

Each run gets its own state. `PowerContext.for_run` and `PowerContextToolset.for_run` create a `_RunState` holding the run
ID, conversation ID, a lock, the resolved Scope, and capture counters. `PowerContextToolset.__aenter__` creates one
`PowerContextClient` per run, enters it, and resolves the Scope through `resolve_scope_binding` once. If resolution
fails, the client is closed before the exception propagates. `__aexit__` closes the client
([toolset.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/toolset.py#L81-L144),
[toolset.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/toolset.py#L146-L171),
[scope.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/scope.py#L24-L62)).

Recall runs in `before_model_request`, which the capability reaches through `PowerContext` at
[capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L66-L93). When capture is
enabled, the hook first captures the prompt once per run. It then branches in this order:

1. If run state already records injection (`context_injected`), or the request already carries a block for this run
   (`_has_current_run_context`), it sets `context_injected` and returns `_leading_context`. That moves the run's context
   block to the front of the model request without a Server call.
2. Otherwise it strips earlier PowerContext context blocks from the request and from the persisted history.
3. If the latest user text is empty, it restores the stripped history and sends the request without a new block.
4. It calls `_prepare_context`. If that returns no content, it restores the history and does not set `context_injected`,
   so a later model request in the same run queries again.
5. If content returns, it sets `context_injected`, builds a run-scoped `SystemPromptPart` with the marker, inserts it
   at the current-run boundary in persisted history with `_insert_context`, and prepends it to the model request
   ([capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L117-L158),
   [capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L226-L244),
   [capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L378-L394),
   [capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L396-L413),
   [capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L435-L451)).
   A new run prepares again even when it continues an earlier message history, because state is per run.

Capture is opt-in through `capture_events`, which defaults to false. When it is on, `after_model_request` records visible
text and tool-call arguments, `after_tool_execute` records tool arguments and results, and `after_run` flushes the rest.
`_capture_event` increments a run-local sequence under the run lock, renders the event with `render_capture_event`, and
posts a Source. Its metadata includes schema `powercontext.pydantic-ai-capture-event/v1`, origin `pydantic-ai`, and the
run ID. Every `capture_checkpoint_every` successful captures trigger `_flush_locked`. Flush repeats `flush_memory` until
the cursor reaches the captured position or stops making progress
([capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L160-L187),
[capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L189-L211),
[capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L213-L224),
[capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L246-L300),
[capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L302-L346)).

Failure behavior differs by path. `_prepare_context` catches only `ClientError`; any other exception from building the
request or calling `prepare_context` propagates to the model call. Capture and flush catch `ClientError` and, for
capture, other exceptions as well, and log at debug level without raising. A `ClientError` from the Memory tools is
re-raised as `ModelRetry`, so the model can see that the call failed and retry. A 401 or 403 is logged once as a
configuration warning, and only the first such failure is reported
([capability.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/capability.py#L226-L244),
[toolset.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/toolset.py#L55-L77),
[toolset.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/toolset.py#L239-L252)).

Settings validate at construction. The base URL must be HTTP or HTTPS, must not carry credentials, query, or fragment,
and is normalized without a trailing slash. `token` must be a bare printable token with no `Bearer` scheme and no
whitespace. `scope_id` is trimmed and bounded. `capture_checkpoint_every` is 1 through 100, and `capture_max_bytes` is
512 through 32768 ([settings.py](repo://integrations/pydantic-ai/src/powercontext_pydantic_ai/settings.py#L28-L108)).

```mermaid
stateDiagram-v2
  [*] --> RunBound: for_run creates run state
  RunBound --> ScopeResolved: client enter resolves scope
  ScopeResolved --> Recalled: first model request prepares context
  Recalled --> Recalled: later model requests reuse injection
  Recalled --> Flushed: after_run flush final
  Flushed --> [*]
```

Caption: the per-run lifecycle of the Pydantic AI capability. Recall is injected once per run, and final flush runs at
run end when capture is enabled. If preparation returns no content, the run stays in the pre-injection state and the
next model request prepares again.

## Bub plugin

The Bub package `powercontext-bub` registers `PowerContextPlugin` under the `bub` entry-point group, with the name
`powercontext` ([pyproject.toml](repo://integrations/bub/pyproject.toml#L28-L29)). Its settings class,
`PowerContextSettings`, is a Bub `@config(name="powercontext")` that reads the `POWERCONTEXT_BUB_` prefix and is frozen
with extra fields ignored ([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L70-L93)).

Defaults and bounds are: `base_url` `http://127.0.0.1:8000`, `api_token` unset, `scope_id` unset with a minimum length of
1, `timeout` 10 seconds, `max_bytes` 8000 bounded to 512 through 32768, `capture_events` false,
`capture_checkpoint_every` 5 bounded to 1 through 100, `capture_max_bytes` 8192 bounded to 512 through 32768,
`capture_log` unset, and `trust_transport_security` false. `context_assembly` is validated against the installed core,
and a core without text-assembly support rejects it ([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L95-L104)).

The plugin's `__init__` loads the settings and resolves the transport, then creates separate locks for scope resolution
and capture ([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L141-L150)). `load_state` seeds one
`_powercontext` entry in the turn state. It carries the explicit `scope_id`, the workspace binding key when Bub exposes a
workspace, a cached `scope_id` set to `None`, the transport settings, the counters, `prompt_captured` set to `False`, and
`recall_texts` ([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L151-L176)).

`build_prompt` is registered `tryfirst`. It asks the other `build_prompt` implementations for the prompt and then records
the sender's text, not Bub's framed content, as the recall text for that prompt. A re-entrant call returns `None` so the
delegation does not loop ([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L177-L216)). Recall
therefore ignores channel metadata and the time stamp that Bub adds around the message. `system_prompt` returns the fixed
`GUIDANCE` text that tells the model about the injected context and the `powercontext.search` and `powercontext.remember`
tools ([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L217-L221)).

`before_llm_call` is the recall hook. It runs these branches in order, and each non-injecting branch returns `None`:

1. If the request has no latest user text, return.
2. Resolve the Scope through `_scope_id`. On failure, return.
3. If `capture_events` is on and this turn has not captured its prompt, capture a `user_prompt` event.
4. If any request message already contains the marker, return without recall.
5. Use the recorded sender text for this prompt, or the prompt itself, as the query. If the query is empty, return.
6. Call `_prepare_context`. If it returns no content, return.
7. Otherwise return the request with a `republic.system` message carrying the marker and the prepared content in front
   of the existing messages. Only the request changes; the turn state keeps no copy of the recalled content
   ([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L222-L255)).

`_scope_id` caches a successful resolution in the turn state and reuses it. When resolution raises one of
`CLIENT_ERRORS`, it stores the error name, writes a `scope` capture record with status `failed`, returns `None`, and
leaves the cache empty, so the next hook retries ([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L307-L337)).
`_prepare_context` sends the query through `_bounded_recall_query`, which keeps the most recent text within 8192
characters and 8192 bytes. A `CLIENT_ERRORS` failure records the error and returns `None`. A success returns the prepared
content and records its status and size
([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L339-L365),
[plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L483-L493)).

`open_client` uses a plain `PowerContextClient` unless `trust_transport_security` is true. In that case it supplies a
caller-owned `httpx.AsyncClient` and marks the transport as trusted, so the client accepts plaintext HTTP to a
non-loopback host only when the operator opted in ([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L106-L139)).

When `capture_events` is on, `after_llm_call` captures `llm_result` events, `after_tool_call` captures `tool_result`
events, and `save_state` runs a final flush under the capture lock
([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L257-L296)). `_capture_event` needs a resolved scope,
increments a sequence under the capture lock, and builds a source ID with the `bub-event:` prefix from the scope, session,
sequence, event, and run ID. It posts the rendered event with origin `bub` and kind `agent-trajectory`. On a
`CLIENT_ERRORS` failure it records the failure and returns without advancing `captured_events` or `captured_position`.
On success it advances both, and every `capture_checkpoint_every` captures it calls `_flush_captured_sources`
([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L367-L422)). `_flush_captured_sources` skips when no
new position was captured, makes one `flush_memory` call, and stores `current_cursor` as the flushed position. Unlike the
Pydantic AI capability, it does not loop until the cursor reaches the target
([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L424-L459)). Every capture, failure, and checkpoint
record is appended as JSON lines to `capture_log` when that setting is set, and nothing is written otherwise
([plugin.py](repo://integrations/bub/src/powercontext_bub/plugin.py#L461-L473)).

Bub's tests live in the end-to-end harness, not in the integration package: `e2e/bub/tests/test_bub_recall_query.py`
covers the sender-text query rule, the empty-sender skip, subagent prompts, and the marker message format;
`test_bub_scope_binding.py` checks that recall and capture use only the Server-resolved Scope;
`test_bub_capture.py` checks credential redaction before the client boundary; `test_bub_transport_trust.py` checks the
plaintext refusal and the transport vouch; and `test_bub_core_compatibility.py` checks the entry point and the optional
assembly API.

## Dify plugin

The Dify package is a Python plugin that Dify loads through `main.py`, which starts `DifyPluginEnv` and `Plugin`
([main.py](repo://integrations/dify/plugin/main.py#L15-L20)). It exposes one provider and 20 `pc_*` tools. Each tool YAML
names a Python class with an `operation` attribute, for example `pc_search` maps to `search_memory`
([pc_search.py](repo://integrations/dify/plugin/tools/pc_search.py#L17-L21)).

Connection settings are validated by `Connection` in `client.py`. Exactly one of `scope_id` or
`binding_external_id` must be set. The token must be non-empty, the URL must be HTTP or HTTPS with no credentials, and
plaintext HTTP is accepted only for loopback or with `allow_insecure_http`. `max_bytes` is 512 through 32768. The
default request timeout is 10 seconds and the generation timeout is 120 seconds. `context_assembly` is validated against
the contract schema ([client.py](repo://integrations/dify/plugin/powercontext_dify/client.py#L96-L157)).

The provider validates credentials by resolving the Scope and reading `get_scope`
([powercontext.py](repo://integrations/dify/plugin/provider/powercontext.py#L16-L24)). Scope resolution is
`resolve_scope`. With a configured ID it sends `explicit_scope_id`. With a binding key it sends an
`integration: dify`, `kind: configured-scope` binding. It rejects a response whose Scope differs from the configured ID
([client.py](repo://integrations/dify/plugin/powercontext_dify/client.py#L276-L291)).

`PowerContextTool._invoke` runs the sequence: build `Connection`, resolve the Scope, validate and build the request,
dispatch through `call_memory`, and emit the result
([tool.py](repo://integrations/dify/plugin/powercontext_dify/tool.py#L139-L190)). `request_for` rejects any parameter not
in the schema, decodes JSON-text parameters, applies defaults, and injects the Scope and `max_bytes`. Hidden parameters
such as `scope_id`, `max_bytes`, and `assembly` are never exposed to the model
([tool.py](repo://integrations/dify/plugin/powercontext_dify/tool.py#L59-L137),
[policy.py](repo://integrations/dify/plugin/powercontext_dify/policy.py#L1-L30)). For `capture_content_source`, a metadata
`origin` of `dify` is set and secret-looking content is rejected before any write.

`Client.call` sends the request through `httpx.Client` with `trust_env=False` and `follow_redirects=False`, and reads the
body in streamed chunks. Responses over 4 MiB are rejected. A JSON response must validate against the contract for its
status. Write operations that fail at transport or with a 5xx response are marked unconfirmed, so the outcome is
reported as `unknown` rather than `error`
([client.py](repo://integrations/dify/plugin/powercontext_dify/client.py#L159-L275)).

Memory tools have translation boundaries in `call_memory`. `list_memory_entries` maps to `list_atomic_memories` and
defaults `states` to active entries unless inactive entries are requested. `revise_memory_entry` sends `If-Match` as a
revision ETag. `retire_memory_entry` changes lifecycle state to `forgotten` using the state version
([client.py](repo://integrations/dify/plugin/powercontext_dify/client.py#L293-L330)).

The tool returns a text envelope with `ok`, `operation`, `status`, `data`, and `error`, then JSON and variable messages.
A successful empty result becomes status `empty`, not an error
([tool.py](repo://integrations/dify/plugin/powercontext_dify/tool.py#L139-L190)).

## Cindy hooks

The Cindy integration has two parts. The Ghost script `main.js` runs in the browser sandbox. It answers host messages and
delegates Server work to a Node worker started by Cindy's broker ([main.js](repo://integrations/cindy/plugins/powercontext-hooks/main.js#L21-L38)).
The worker `worker.ts` exposes two entry points: `runHook` for the `will-user-message` event and `runReadTool` for read
tools ([worker.ts](repo://integrations/cindy/src/worker.ts#L105-L149),
[worker.ts](repo://integrations/cindy/src/worker.ts#L151-L252)).

The Ghost resolves a bearer-token credential only inside the worker. It forwards the hook with `deadline_at` and a 2-second
timeout. The worker's `HOOK_BUDGET_MS` is 1800 milliseconds. The worker caps the prepare call at 800 milliseconds so that
capture has time to run ([worker.ts](repo://integrations/cindy/src/worker.ts#L14-L16),
[worker.ts](repo://integrations/cindy/src/worker.ts#L151-L185)). The Ghost always sends an `allow` verdict after its local
timeout.

`runHook` runs these branches in order. If the hook is disabled, it returns skipped. If the deadline has passed, it returns
`hook_deadline` with capture `not_sent`. If the text already contains the Cindy context markers, it returns
`prepared_context_present` and does not recapture. If the text contains a secret or the credential, it returns
`sensitive_content`. Otherwise it resolves the Scope through a `cindy`/`session` binding and calls `prepare_context`
with `max_bytes`. Only a `ready` status with content produces a rewrite. Capture runs independently and is skipped when
`capture_prompts` is false or the text exceeds `MAX_SOURCE_LENGTH`. The source ID is a SHA-256 over scope, session,
hook, and text, and metadata includes `event: user_prompt_submit` and `capture_stage: will-user-message-input`
([worker.ts](repo://integrations/cindy/src/worker.ts#L151-L252)).

The Ghost appends the prepared content to the user message only when the result is ready, the message has no markers,
and the total length is within 16,000 characters. The appended block states that the Scope was resolved for the message
and that the historical evidence is untrusted
([main.js](repo://integrations/cindy/plugins/powercontext-hooks/main.js#L70-L104)).

Read tools are `resolve_scope`, `prepare_context`, and `search_memory`. A `status` tool answers from the last observed
hook outcome without any Server call ([main.js](repo://integrations/cindy/plugins/powercontext-hooks/main.js#L24-L38)).
The Ghost removes model-supplied `session_context` and substitutes the trusted session ID. `runReadTool` resolves the
binding on every call and never captures. It validates arguments, rejects `query` values containing secrets, and bounds
`prepare_context` `max_bytes` to 512 through 8192
([worker.ts](repo://integrations/cindy/src/worker.ts#L105-L149)).

## MiniMax native plugin

MiniMax is not a Python framework adapter. `powercontext setup minimax` installs a native plugin that contains the
`powercontext-project-context` Skill and a streamable HTTP MCP connection. The MiniMax host owns the MCP session, and the
plugin itself does not capture conversations ([minimax.md](repo://docs/en/docs/integrations/minimax.md#L1-L18)).

The packaged MCP declaration points to `http://127.0.0.1:8000/mcp`
([powercontext.mcp.json](repo://integrations/minimax/plugins/powercontext/powercontext.mcp.json#L1-L9)). `_mcp_entry`
accepts only `type: streamable-http`, requires an enabled flag that is a boolean, requires a literal URL that ends in `/mcp`,
and rejects `${...}` placeholders in headers. It accepts at most one `Authorization` header
([minimax.py](repo://src/powercontext/cli/minimax.py#L169-L194),
[minimax.py](repo://src/powercontext/cli/minimax.py#L196-L203)).

`MiniMaxInstaller.install` validates the package manifest and its resources before writing. It checks the file inventory
hashes recorded under the PowerContext data directory, refuses conflicts with user-owned or unmanaged files, stages
the new plugin tree, and publishes it with the installation record
([minimax.py](repo://src/powercontext/cli/minimax.py#L274-L345)). The installer's `configure` path updates the private
`mcp.json` entry and the installed bundle together, and it preserves an existing disabled private entry
([minimax.py](repo://src/powercontext/cli/minimax.py#L347-L420)).

Qualification is narrower than the code path. The MiniMax docs state that plugin-only MCP tool injection was not reliably
qualified for CLI 0.6.9 or 0.7.2, and that setup, doctor, and upgrades do not change that result
([minimax.md](repo://docs/en/docs/integrations/minimax.md#L9-L13)).

## OpenDAL Connector

The OpenDAL package is an independently scheduled worker. It reads files through an OpenDAL-backed fsspec filesystem and
submits UTF-8 snapshots through the shared Connector lifecycle. It does not run inside the Server and does not create
Memory directly ([connector.py](repo://integrations/opendal/src/powercontext_connector_opendal/connector.py#L84-L119)).

The `powercontext-connector-opendal` command requires `--base-url`, `--scope-id`, `--binding-id`, `--service`, and
`--source-namespace`. It accepts `--root`, repeated `--storage-option KEY=VALUE`, repeated `--pattern`, `--max-files`, and
`--max-file-size`. It builds the connector, a `ConnectorBinding`, and a `PowerContextClient` that reads
`POWERCONTEXT_TOKEN` from the environment. It then calls one `RemoteConnectorWorker.run` and exits 0 only when the status
is `COMPLETE` ([cli.py](repo://integrations/opendal/src/powercontext_connector_opendal/cli.py#L31-L75),
[cli.py](repo://integrations/opendal/src/powercontext_connector_opendal/cli.py#L88-L101)).

`from_service` imports `opendalfs` and raises `ImportError` when the package is unavailable, reporting the Python 3.12
requirement. It forces `use_listings_cache=False` so that each run sees files created outside the worker
([connector.py](repo://integrations/opendal/src/powercontext_connector_opendal/connector.py#L120-L150)).

`run` executes the following steps in order, and each step has branches that matter for the checkpoint:

```mermaid
flowchart TD
  A["Load checkpoint and list files under root"] --> B{"More than max files?"}
  B -- yes --> X["Raise file-limit error"]
  B -- no --> C{"Size over max file size?"}
  C -- yes --> R["Reject item"]
  C -- no --> D["Read bytes and compute sha256 digest"]
  D --> E{"Read failed?"}
  E -- yes --> F["Fail item"]
  E -- no --> G{"Digest equals checkpoint?"}
  G -- yes --> S["Skip unchanged file"]
  G -- no --> H{"Valid UTF-8?"}
  H -- no --> R
  H -- yes --> I["Submit text-file snapshot Source"]
```

Caption: the per-file decision path of the OpenDAL Connector. Unchanged files are skipped, and reject or fail outcomes
prevent the checkpoint from advancing.

The checkpoint is a schema-versioned map from relative path to `sha256:<hex>` digest. Its validator rejects absolute
paths, parent escapes, and uppercase hex
([connector.py](repo://integrations/opendal/src/powercontext_connector_opendal/connector.py#L60-L82)). The default
patterns select Markdown, text, reStructuredText, and AsciiDoc files. Relative paths are sorted before processing
([connector.py](repo://integrations/opendal/src/powercontext_connector_opendal/connector.py#L51-L52),
[connector.py](repo://integrations/opendal/src/powercontext_connector_opendal/connector.py#L229-L245)).

`ConnectorLifecycle` owns the commit rule, not the connector. It saves the proposed checkpoint only when the completion
status is `COMPLETE`, the checkpoint differs from the previous one, and no item was rejected or failed. It saves with
`expected=previous`, which is a compare-and-swap. If any item was rejected or failed, the result is reported as
`INCOMPLETE` and the previous checkpoint remains committed
([connectors.py](repo://src/powercontext/sources/connectors.py#L256-L300)).

The connector declares the `text-file-snapshot` Source definition. Its Source ID is derived from namespace, path, and
content digest, and a projection exposes the text as text evidence with provenance metadata
([source.py](repo://integrations/opendal/src/powercontext_connector_opendal/source.py#L79-L180)).

The OpenDAL docs say that the default per-file limit is 2 MiB. The code default is `DEFAULT_MAX_FILE_SIZE = 256 * 1024`,
which is 256 KiB ([connector.py](repo://integrations/opendal/src/powercontext_connector_opendal/connector.py#L51)). Treat the
code value as authoritative until the docs are corrected. The docs also mention a `POWERCONTEXT_SCOPE_ID` variable, but
the CLI requires `--scope-id` as an argument ([cli.py](repo://integrations/opendal/src/powercontext_connector_opendal/cli.py#L88-L101)).

## Cross-cutting invariants

- **Fail-open for host work.** Recall, capture, and flush do not raise into the agent or the graph in the LangChain,
  LangGraph, and Bub adapters. In the Pydantic AI capability, `ClientError` from recall, capture, and flush is logged and
  swallowed, but a non-`ClientError` exception from recall propagates. The Pydantic AI Memory tools raise `ModelRetry`
  for the model to handle, and the Dify and LangGraph tools return structured failure text.
- **Explicit or default Scope only.** No adapter derives a Scope from Git, paths, agents, or prompts. Cindy's plugin read
  tools and Dify's configured scope both require an explicit binding or an explicit ID.
- **Capture is always a Source.** Every capture path creates Source evidence through `capture_content_source` or the
  connector lifecycle. Extraction into Memory happens later through the normal Server pipeline.
- **Recall never persists.** LangChain and LangGraph keep recalled content out of `messages` and the checkpointer.
  Pydantic AI keeps it in the run-scoped request and the current-run boundary of its history. Bub returns a changed
  request and keeps only the sender's recall text in turn state.
- **Secrets are rejected before write.** Dify and Cindy scan capture content and some read queries for credential-like
  text. The Python middlewares rely on the caller's transcript policy, as the LangChain docs state.

## Configuration summary

| Adapter | Environment or input | Key defaults |
| --- | --- | --- |
| LangChain | `POWERCONTEXT_LANGCHAIN_BASE_URL`, `_TOKEN`, `_SCOPE_ID`, `_TIMEOUT`, `_MAX_BYTES`, `_ALLOW_INSECURE_HTTP` | Base URL `http://127.0.0.1:8000`, timeout 10 s, max bytes 8000, capture off |
| LangGraph | `POWERCONTEXT_LANGGRAPH_*` with the same fields | Same defaults, token as a bare value |
| Pydantic AI | `POWERCONTEXT_PYDANTIC_AI_*`, including `_CAPTURE_EVENTS`, `_CAPTURE_CHECKPOINT_EVERY`, `_CAPTURE_MAX_BYTES` | Capture off, checkpoint every 5 captures, capture max 8192 bytes |
| Bub | `POWERCONTEXT_BUB_*`, including `_BASE_URL`, `_API_TOKEN`, `_SCOPE_ID`, `_CAPTURE_EVENTS`, `_CAPTURE_LOG`, `_TRUST_TRANSPORT_SECURITY` | Base URL `http://127.0.0.1:8000`, timeout 10 s, max bytes 8000, capture off, checkpoint every 5, capture max 8192 bytes |
| Dify | Plugin credentials: `server_url`, `api_token`, exactly one of `scope_id` or `binding_external_id` | `max_bytes` 8000, timeout 10 s, generation timeout 120 s |
| Cindy | Plugin settings: `enabled`, `server_url`, `auth_mode`, `scope_id`, `capture_prompts`, `max_bytes` | Max bytes 6000 for hooks, hook budget 1800 ms |
| MiniMax | `powercontext setup minimax`, `POWERCONTEXT_MINIMAX_AUTHORIZATION` | Endpoint `http://127.0.0.1:8000/mcp` |
| OpenDAL | CLI flags, `POWERCONTEXT_TOKEN` | Patterns for Markdown and text, max files 10000, max file size 256 KiB |

## Focused tests

- LangChain middleware: `tests/langchain_middleware/test_middleware.py` covers recall without persistence
  (`test_middleware_injects_recall_without_persisting_it`), capture off by default
  (`test_middleware_does_not_capture_completed_turn_by_default`), capture on completion, structured response capture
  (`test_middleware_captures_tool_strategy_structured_response`), and reaching the end of an agent run when the server is
  unreachable, both async and sync (`test_async_agent_reaches_end_when_server_unreachable`,
  `test_sync_agent_reaches_end_when_server_unreachable`). `tests/langchain_middleware/test_tracing.py` covers bounded,
  task-local recall content, and `test_default_langfuse_tracing.py` covers the default trace tree.
- LangGraph adapter: `tests/langgraph_adapter/test_recall.py` covers the empty-context case
  (`test_recall_supplies_no_prefix_when_context_is_empty`), the untrusted label
  (`test_recall_injects_context_labelled_untrusted`), no accumulation across turns, graph completion for an over-limit
  prompt, scope-isolated caching (`test_recall_isolates_cached_context_across_scopes`), and cache refresh when assembly or
  the byte budget changes. `test_tools.py` and `test_scope.py` cover the Memory tools and runtime scope resolution.
- Pydantic AI adapter: `tests/pydantic_ai_adapter/test_capability.py` covers one injection per run
  (`test_context_is_replaced_once_per_run_when_reusing_old_history`), the new-messages boundary across turns, empty context
  not being injected, unreachable-server fail-open for recall (`test_unreachable_server_fails_open_for_recall`), and a
  single credential-free configuration warning. `test_capture.py`, `test_toolset.py`, and `test_settings_scope.py` cover
  capture and flush, the toolset, and settings validation.
- Bub: the end-to-end suites in `e2e/bub/tests/`, listed under [Bub plugin](#bub-plugin) above.
- OpenDAL: `integrations/opendal/tests/test_opendal_connector.py` covers checkpoint persistence across a Server restart,
  retention of the previous checkpoint before a rejected item, oversized-file rejection without aborting, and the
  source-to-Memory loop.
- Dify: `integrations/dify/tests/test_plugin_contract.py` covers exact memory revision handling, secret rejection before
  write, and the rule that a missing binding never falls back to a default Scope or calls Memory.
- Cindy: `integrations/cindy/tests/hooks.test.cjs` and `settings.test.cjs`.
- MiniMax: `tests/test_cli_minimax.py` exercises the MiniMax CLI installer.

## Related pages

- [Capability matrix](capability-matrix.md) compares host capabilities across integrations.
- [Client, CLI, and transports](../architecture/client-cli-and-transports.md) describes the shared Python Client that the
  framework adapters call.
- [HTTP and MCP API](../architecture/http-and-mcp-api.md) lists the operations and the MCP projection behind these calls.
- [Prepare and recall](../workflows/prepare-and-recall.md) describes the Server-side preparation that each adapter calls.
