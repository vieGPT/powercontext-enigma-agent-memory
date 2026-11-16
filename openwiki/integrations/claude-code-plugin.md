---
type: integration
title: Claude Code Plugin
description: How the PowerContext Claude Code marketplace plugin wires SessionStart and UserPromptSubmit hooks, the MCP server, the status line, and Scope resolution through POWERCONTEXT_CLAUDE_SCOPE_ID, including the failure outcome when no Scope can be resolved and how it differs from the Codex plugin.
tags: [claude-code, plugin, hooks, mcp, scope, statusline, privacy, bootstrap, integration, marketplace]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-16T11:46:19.799Z
sources:
  - id: openwiki-source-09160a15c85836458c6d2077
    resource: repo://.claude-plugin/marketplace.json
  - id: openwiki-source-d1a75b28fb17d6310d7fe8e6
    resource: repo://integrations/claude-code/plugins/powercontext/.claude-plugin/plugin.json
  - id: openwiki-source-6e802f24c24a8065f42e2fef
    resource: repo://integrations/claude-code/plugins/powercontext/.mcp.json
  - id: openwiki-source-cf9d55b84e1e06fe20d993ff
    resource: repo://integrations/claude-code/plugins/powercontext/bootstrap_state.py
  - id: openwiki-source-777d80715ebde2e97bc8d588
    resource: repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py
  - id: openwiki-source-77eb8104c1b304fa382bbcc4
    resource: repo://integrations/claude-code/plugins/powercontext/hooks/diagnostics.py
  - id: openwiki-source-d15aa9915a0abeaf7f18c1c4
    resource: repo://integrations/claude-code/plugins/powercontext/hooks/hooks.json
  - id: openwiki-source-6739444921b8cfea1da1daa4
    resource: repo://integrations/claude-code/plugins/powercontext/hooks/prepared_context.py
  - id: openwiki-source-756a23005612a8fb0b562ba1
    resource: repo://integrations/claude-code/plugins/powercontext/hooks/prompt_privacy.py
  - id: openwiki-source-6363c9fdf052ecf27f9b1ad9
    resource: repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py
  - id: openwiki-source-b0e7e8f218c8994fc3b4686c
    resource: repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py
  - id: openwiki-source-8393e102ef5f495bc6fd1d7b
    resource: repo://integrations/claude-code/plugins/powercontext/powercontext_client_config.py
  - id: openwiki-source-ba3dcf40d254e56cadb4ce43
    resource: repo://integrations/claude-code/plugins/powercontext/scripts/statusline.py
  - id: openwiki-source-c397489314a42f72a0d60f24
    resource: repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py
  - id: openwiki-source-1b97e4fc746fefe5bcebb4bb
    resource: repo://integrations/claude-code/README.md
  - id: openwiki-source-16419058346606612095788e
    resource: repo://integrations/codex/plugins/powercontext/hooks/hooks.json
  - id: openwiki-source-012f2c78e3b1446dfc35803f
    resource: repo://Makefile
  - id: openwiki-source-f29470b6a16e826f5a684048
    resource: repo://src/powercontext/builtin/scope/application.py
  - id: openwiki-source-e5374e477de7d6683f8772c8
    resource: repo://src/powercontext/cli/system.py
  - id: openwiki-source-f25c1b3c7f796f0c6487a9cb
    resource: repo://src/powercontext/server/app.py
  - id: openwiki-source-eea3f347200d607505a4bc4b
    resource: repo://tests/claude_code_plugin/test_contract.py
  - id: openwiki-source-3bdf3579b68b883e6e60a3fa
    resource: repo://tests/claude_code_plugin/test_hook.py
generated: { by: "openwiki/0.9.0", at: "2026-11-16T11:46:19.799Z" }
---

# Claude Code Plugin

The Claude Code integration is a marketplace plugin that makes Claude Code a client of a running PowerContext Server. It does four things: it recalls one bounded Prepared Context before each user prompt and injects it unchanged, it captures each user prompt as ordinary Content Source evidence, it can inject an opt-in bootstrap package at session start, and it exposes explicit Memory and Handoff operations through an MCP server. A status line can also show the recall token reduction for the current Scope. Every hook path is fail-open, so Server, transport, or contract failures never block normal Claude Code work.

The plugin is a thin client. Durable state (Scopes, Scope bindings, Sources, Memory, Handoffs) lives in the Server. The plugin keeps only a content-free bootstrap receipt on the local machine, plus a best-effort diagnostic throttle file.

## Packaging and entry points

Claude Code discovers the plugin from a repository-root marketplace that lists one plugin named `powercontext`, sourced from `integrations/claude-code/plugins/powercontext`, at version `0.1.3` [repo://.claude-plugin/marketplace.json#L1-L17]. The plugin manifest declares three user options, `server_url`, `capture_prompts`, and `allow_insecure_http`, and does not declare `hooks` or `mcpServers`; the hook configuration, MCP configuration, and status line script are discovered from their standard locations [repo://integrations/claude-code/plugins/powercontext/.claude-plugin/plugin.json#L1-L38] [repo://tests/claude_code_plugin/test_contract.py#L46-L56].

| Path under the plugin root | Responsibility |
| --- | --- |
| `hooks/hooks.json` | Registers the `SessionStart` and `UserPromptSubmit` hooks |
| `hooks/user_prompt_submit.py` | Per-prompt Scope resolution, recall, prompt capture, and optional flush |
| `hooks/session_start.py` | Opt-in bootstrap context injection and delivery receipts |
| `hooks/prepared_context.py` | Prepare contract: response validation and the bounded recall query (generated copy) |
| `hooks/prompt_privacy.py` | Best-effort credential scrubbing for the recall query (generated copy) |
| `hooks/bootstrap_context.py` | Strict validation of the bootstrap HTTP contract (generated copy) |
| `hooks/diagnostics.py` | Cross-process throttling of failure diagnostics |
| `claude_code_settings.py` | Validated process configuration from environment, plugin options, and stored credentials |
| `powercontext_client_config.py` | Endpoint normalization and saved non-secret client settings (generated copy) |
| `scope_binding_errors.py` | Typed Scope binding failures shared by the entry points |
| `bootstrap_state.py` | Local, content-free bootstrap receipt file keyed by a hashed session ID |
| `scripts/workspace_scope.py` | Scope resolution, workspace binding, bounded HTTP helpers, and the Scope binding CLI |
| `scripts/statusline.py` | Claude Code status line showing estimated token reduction |
| `.mcp.json` | Top-level `powercontext` HTTP MCP server entry |

The files marked as generated carry the header `Generated from src/powercontext_integration_lifecycle/...; do not edit`. The shared helpers are authored once under `src/powercontext_integration_lifecycle/` and copied into the plugin by `make integration-lifecycle`; `make integration-lifecycle-check` fails when a copy drifts from its source [repo://Makefile#L31-L36] [repo://integrations/claude-code/plugins/powercontext/hooks/prepared_context.py#L15-L16] [repo://integrations/claude-code/plugins/powercontext/powercontext_client_config.py#L15-L16]. Change the source module, never the copy.

The hook configuration is deliberately minimal. It has exactly two event keys, `SessionStart` and `UserPromptSubmit`, and each runs `python3` in exec form against a script under `${CLAUDE_PLUGIN_ROOT}/hooks/`. The contract test asserts that no `Stop` hook is registered [repo://integrations/claude-code/plugins/powercontext/hooks/hooks.json#L1-L36] [repo://tests/claude_code_plugin/test_contract.py#L58-L70].

## Hook registration

| Event | Matcher | Script | Timeout | Status message |
| --- | --- | --- | --- | --- |
| `SessionStart` | `startup\|resume\|clear\|compact\|fork` | `hooks/session_start.py` | 10 s | Loading PowerContext context |
| `UserPromptSubmit` | none | `hooks/user_prompt_submit.py` | 10 s | Syncing PowerContext |

`SessionStart` runs only for the five lifecycle sources in its matcher, and the script enforces the same five values again; any other source exits with no output [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L62-L62] [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L98-L105]. The ten-second timeout is the host's limit; the hook's own budget is smaller, as described under [Failure behavior](#failure-behavior).

The MCP server is declared as a top-level `powercontext` entry of type `http`. Its URL is `${user_config.server_url}/mcp`, and its only header is `Authorization` with the value `${POWERCONTEXT_CLAUDE_AUTHORIZATION:-}`, so an absent variable expands to an empty header [repo://integrations/claude-code/plugins/powercontext/.mcp.json#L1-L9] [repo://tests/claude_code_plugin/test_contract.py#L72-L81].

## Installation and configuration flow

`powercontext setup claude-code` is implemented by `ClaudeInstaller`. Its `install` method requires the `claude` CLI on `PATH`, refuses to reuse a marketplace whose source differs from the requested one, adds or updates the `powercontext` marketplace at user scope, installs or updates `powercontext@powercontext` at user scope, and reads back the installed version and cache path [repo://src/powercontext/cli/system.py#L1645-L1670].

`configure` then calls `_claude_configuration_updates`, which merges the non-sensitive options into `pluginConfigs["powercontext@powercontext"].options` in Claude Code's `settings.json`: `server_url` and `allow_insecure_http`, plus `capture_prompts` when the caller supplied it. Unrelated settings are preserved [repo://src/powercontext/cli/system.py#L1264-L1292] [repo://src/powercontext/cli/system.py#L1672-L1682].

The same function sets the status line to a `python3` command that runs the installed `scripts/statusline.py` with `--server-url`, with `refreshInterval` 30. It does so only when no `statusLine` exists or the existing one already points at a PowerContext `statusline.py` (checked by `_is_powercontext_statusline`), so a user's custom status line is left in place [repo://src/powercontext/cli/system.py#L1317-L1326] [repo://src/powercontext/cli/system.py#L1336-L1344].

Authorization is never written into plugin options. It stays in the process environment.

## Scope resolution and POWERCONTEXT_CLAUDE_SCOPE_ID

Every hook and the status line that talk to the Server must resolve a Scope first. `POWERCONTEXT_CLAUDE_SCOPE_ID` is read into `ClaudeCodePluginSettings.scope_id` and treated as absent when blank after trimming [repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py#L144-L144] [repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py#L93-L118].

The plugin does not choose a Scope locally. `resolve_scope_id` sends one `POST /v1/scope-bindings/resolve` request. The body carries `explicit_scope_id`, which is the environment value or `null`, and an ordered `binding_keys` list containing a `session` key when the payload has a `session_id`, followed by a `workspace` key [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L123-L145] [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L191-L197].

The Server applies the order in `resolve_binding`: an explicit Scope ID wins and must exist; otherwise the first binding key with a stored binding wins; otherwise the Server's default Scope is used, and if none exists the call raises `ScopeBindingNotFoundError` [repo://src/powercontext/builtin/scope/application.py#L276-L296]. The HTTP handler `resolve_scope_binding` passes the explicit ID, the keys, and `allow_default` straight through [repo://src/powercontext/server/app.py#L2678-L2688].

```mermaid
flowchart TD
    A["Hook receives cwd and session_id"] --> B{"POWERCONTEXT_CLAUDE_SCOPE_ID set?"}
    B -- yes --> C["Server returns that Scope if it exists"]
    B -- no --> D{"Session binding stored?"}
    D -- yes --> E["Server returns the session-bound Scope"]
    D -- no --> F{"Workspace binding stored?"}
    F -- yes --> G["Server returns the workspace-bound Scope"]
    F -- no --> H{"Server default Scope set?"}
    H -- yes --> I["Server returns the default Scope"]
    H -- no --> J["404 scope_not_found: hook reports a systemMessage diagnostic and sends no recall or capture"]
```

Caption: the Scope resolution order seen by the Hook, with the no-configuration fallback on the right-hand path.

### Fallback when no Scope ID is configured

When `POWERCONTEXT_CLAUDE_SCOPE_ID` is unset, the plugin sends `explicit_scope_id: null`, so resolution depends only on durable Server state:

1. a stored session binding for the Claude Code `session_id`;
2. a stored workspace binding for the checkout;
3. the Server's default Scope.

If no default Scope exists, the Server answers `404` with the error code `scope_not_found`, because `ScopeBindingNotFoundError` maps to that response [repo://src/powercontext/server/app.py#L7107-L7108].

The failure branch depends on the hook. In `UserPromptSubmit`, scope resolution has its own `try` block, separate from recall and capture. A failure is restated by `_scope_resolution_failure` as the error that the hook's own requests would raise for the same response: here an `HTTPError` with status 404, path `/v1/scope-bindings/resolve`, and code `scope_not_found`. `_emit_failure_event` classifies that as `invalid_response` and appends a diagnostic with `http_status` 404 and `error_code` `scope_not_found` to the `systemMessage` output. `main` then returns `0`. No recall, capture, or flush request is sent, and the session's bootstrap receipt is left untouched, because clearing the receipt happens only after resolution succeeds [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L154-L171] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L563-L576]. The result is visible to the user, not silent. A no-configuration install works only when the Server has a binding or a default Scope.

### Workspace and session keys

The workspace key is `{integration: claude-code, kind: workspace, external_id}`, where `external_id` is the SHA-256 of the path that `git rev-parse --show-toplevel` reports. The path is resolved before hashing. When Git is unavailable or fails, the key uses the working directory instead [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L206-L214] [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L375-L390]. The key is never derived from a Git remote or the directory name, so two checkouts with the same name do not share a Scope by accident. The session key carries the trimmed raw `session_id` and is limited to 256 characters [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L199-L204].

Binding is explicit. `workspace_scope.py --bind-scope SCOPE_ID` sends `PUT /v1/scope-bindings` for the workspace key and checks that the Server echoes the same Scope ID; `--clear-scope` posts to `/v1/scope-bindings/clear` [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L147-L168] [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L170-L189] [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L393-L410]. On the Server, `bind` requires the target Scope to exist before it writes [repo://src/powercontext/builtin/scope/application.py#L250-L262].

Setting `POWERCONTEXT_CLAUDE_SCOPE_ID` is an override. It takes precedence over every stored binding and the default Scope, so it should be used only when the work must intentionally go to that Scope.

## Per-prompt recall and capture

The `UserPromptSubmit` hook is the core of the plugin. The sequence below shows one prompt.

```mermaid
sequenceDiagram
    participant CC as Claude Code
    participant H as UserPromptSubmit Hook
    participant S as PowerContext Server
    CC->>H: JSON payload on stdin
    H->>S: POST scope-bindings resolve
    S-->>H: scope_id
    H->>H: Load and clear bootstrap receipt
    H->>H: Scrub secrets and bound recall query
    H->>S: POST context prepare
    S-->>H: prepared-context v1 response
    H->>S: POST sources content when capture is on
    S-->>H: position
    H->>S: POST memory flush only when flush-on-capture is set
    H-->>CC: additionalContext on stdout
```

Caption: one UserPromptSubmit invocation. Scope resolution comes first; recall and capture are then independent requests, and the flush step runs only when explicitly enabled.

The steps, in order, are:

1. **Filter the event.** `main` reads the JSON payload from stdin. Only a `hook_event_name` that normalizes to `userpromptsubmit` is processed; any other event returns `0` with no output [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L131-L142]. The prompt is read from `prompt`, falling back to `user_prompt` [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L230-L236]. A missing or blank prompt, or a non-string `cwd`, records a `skipped` event on stderr and exits with no output [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L147-L150].
2. **Resolve the Scope** as described above. The resolution call and the later recall, capture, and flush requests share one HTTP budget, computed once from `http_budget_seconds` [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L152-L169]. If resolution fails, the hook writes its diagnostic and returns `0` as described above.
3. **Load and clear the bootstrap receipt.** Only after the Scope is resolved, the hook loads the receipt for this session and Scope; a receipt whose stored Scope differs is ignored. It then clears the session's receipt regardless of outcome. The receipt ID is passed to recall as `bootstrap_receipt_id` [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L170-L171] [repo://integrations/claude-code/plugins/powercontext/bootstrap_state.py#L29-L31].
4. **Scrub and bound the query.** `scrub_secrets` replaces recognized credentials with `[REDACTED]` in the query only, and the query is bounded as described in [Recall query bounds](#recall-query-bounds) [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L172-L173].
5. **Recall.** `POST /v1/context/prepare` is sent at most once, with `scope_id`, the query, `max_bytes` set to 8,000, and optional `include_code`, `assembly`, and `bootstrap_receipt_id` fields. Recall failures are reported and recall returns no context; capture still runs [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L174-L185] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L238-L260] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L423-L485].
6. **Capture.** The branch order is fixed. If the scrubber changed the prompt, capture is skipped and a `skipped` event with reason `sensitive_content` is written to stderr. Otherwise, when `capture_prompts` is enabled and the raw prompt is at most 200,000 characters, the original prompt is posted to `POST /v1/sources/content`. A prompt over that length or with capture disabled is not posted and produces no event [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L186-L199] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L262-L295].
7. **Flush, when enabled.** Flush is opt-in and runs only after a successful capture. `_flush_through` calls `POST /v1/memory/flush` until the reported `current_cursor` reaches the captured Source's position, up to `flush_max_calls` times. If the cursor never reaches it, the loop raises and the failure is reported as a `flush_memory` diagnostic [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L208-L223] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L297-L315].
8. **Emit output.** Diagnostics go to `systemMessage`, and recalled context goes to `hookSpecificOutput.additionalContext` with `hookEventName` `UserPromptSubmit`. Nothing is written when both are absent [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L224-L224] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L578-L592].

If anything escapes the steps above, including invalid settings such as a non-numeric timeout or an invalid boolean, the outer `except Exception` in `main` returns `0` with no output [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L131-L135] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L225-L227].

### Recall query bounds

A prompt of at most 8,192 characters and 8,192 UTF-8 bytes is sent after whitespace trimming. A longer prompt keeps its most recent text: the tail is cut to the last 8,192 characters and then to the last 8,192 bytes without splitting a character [repo://integrations/claude-code/plugins/powercontext/hooks/prepared_context.py#L29-L30] [repo://integrations/claude-code/plugins/powercontext/hooks/prepared_context.py#L74-L92]. A long prompt therefore narrows recall instead of failing it. Capture is not bounded by this query limit; it keeps its own 200,000-character ceiling [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L54-L54].

### Prepared Context contract

The Hook accepts only a response with exactly the fields `schema`, `status`, `content`, and `content_bytes`. The schema must be `powercontext.prepared-context.v1`. An `empty` status must have null content and zero bytes. A `ready` status must have non-blank content whose UTF-8 length equals `content_bytes` and does not exceed 8,000 bytes. Any other shape is rejected and nothing is injected [repo://integrations/claude-code/plugins/powercontext/hooks/prepared_context.py#L39-L72].

### Prompt capture identity

Captured prompts become Content Source records with a deterministic source ID: `claude-code-user-prompt:` followed by the SHA-256 of the scope ID, session ID, prompt ID, and prompt text, separated by NUL bytes. Repeating the same prompt in the same session and Scope therefore produces the same Source identity. Metadata records `origin: claude-code`, `event: user_prompt_submit`, the working directory, and, when present, the session and prompt IDs [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L262-L295]. The test suite checks that repeated capture is idempotent and that a captured prompt is not treated as a task outcome [repo://tests/claude_code_plugin/test_hook.py#L883-L933].

Prompt capture does not write Memory. The Server's Source pipeline decides later whether Memory is produced, and the Hook never labels an ordinary prompt as `task-outcome`.

### Secret handling

`scrub_secrets` removes PEM private-key blocks and common credential assignments such as `authorization`, `cookie`, `password`, `token`, and `api_key`, along with bearer tokens and several provider token prefixes. It also removes the active `Authorization` credential from the text. The scrubbed text is used only for the recall query. The capture request is never rewritten, so a recognized secret causes capture to be skipped rather than stored in a modified form [repo://integrations/claude-code/plugins/powercontext/hooks/prompt_privacy.py#L17-L50] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L172-L173] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L186-L188]. This is best-effort recognition, not a guarantee.

## SessionStart bootstrap

`SessionStart` is a separate, opt-in path. The hook reads the event's `session_id`, `cwd`, and `source`, and runs only for `startup`, `resume`, `clear`, `compact`, or `fork`. Any other or missing value exits with no output [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L62-L62] [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L98-L105]. The hook resolves the Scope the same way and posts `POST /v1/context/bootstrap` with the profile `powercontext.scope-bootstrap.v1` [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L176-L221]. The request always carries `enabled` set from `POWERCONTEXT_CLAUDE_BOOTSTRAP_CONTEXT` and `max_bytes` from `POWERCONTEXT_CLAUDE_BOOTSTRAP_MAX_BYTES`; an optional exact Handoff identity comes from `POWERCONTEXT_CLAUDE_BOOTSTRAP_HANDOFF`.

Delivery uses a two-step receipt:

1. The Server returns a ready package with a `receipt_id`.
2. The hook posts `outcome: injected` to `/v1/context/bootstrap/receipts`. Only after the Server confirms the `injected` state does the hook persist the receipt locally and write `additionalContext` [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L121-L160]. If the claim fails, the hook posts `outcome: failed` and emits nothing [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L144-L153].

If the local receipt cannot be saved, the hook does not write `additionalContext`. `save_receipt` raises when no state root is configured, and the error is handled by `_handle_output_failure`, which emits a `delivery_failed` event; in that path the `failed` outcome is not posted, so the Server has already recorded the package as `injected` [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L155-L166] [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L308-L321].

The local receipt is a file under the state root, named by the SHA-256 of the session ID. The root is `POWERCONTEXT_CLAUDE_BOOTSTRAP_STATE_DIR` when set, otherwise `CLAUDE_PLUGIN_DATA`; the root must be an absolute path, and when neither is set no receipt path exists. Under the root the file is `powercontext-bootstrap/<sha256>.json`. It holds only the Scope ID and receipt ID, is written with mode `0600` through an atomic replace, and is ignored when its permissions or shape are wrong [repo://integrations/claude-code/plugins/powercontext/bootstrap_state.py#L163-L174] [repo://integrations/claude-code/plugins/powercontext/bootstrap_state.py#L41-L63] [repo://integrations/claude-code/plugins/powercontext/bootstrap_state.py#L65-L101]. On the next ordinary prompt, that receipt suppresses only the bootstrap revisions that were fully delivered. Artifacts that were truncated and newer revisions remain eligible.

SessionStart's own diagnostics are written to stderr only, not to `systemMessage`. Its HTTP failure mapping differs from recall's: a 403 is reported as `authorization_denied`, and a 404 without an error code as `version_mismatch` [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L343-L360] [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L362-L378]. If the settings cannot be loaded or no package is prepared, the hook clears the receipt it observed and exits with no output [repo://integrations/claude-code/plugins/powercontext/hooks/session_start.py#L104-L116].

Bootstrap is disabled by default and is never selected automatically from Handoffs. Preparation and receipt failures fail open and inject nothing.

## Status line

`scripts/statusline.py` renders one line for Claude Code's status line. It reads the status-line JSON from stdin, takes the working directory from `workspace.current_dir` or `cwd`, and resolves the Scope through the same `resolve_scope_id` call with the session ID [repo://integrations/claude-code/plugins/powercontext/scripts/statusline.py#L267-L288] [repo://integrations/claude-code/plugins/powercontext/scripts/statusline.py#L232-L240].

It then sends two `POST /v1/stats` requests, one for `today` and one for `30d`, each with `selection` `{mode: exact, scope_ids: [scope]}`. A response without validated recall totals is rejected [repo://integrations/claude-code/plugins/powercontext/scripts/statusline.py#L197-L222]. A window with zero comparable preparations is shown as `no data`, not as a measured zero [repo://integrations/claude-code/plugins/powercontext/scripts/statusline.py#L116-L129]. A successful render reads `● PC online · saved <n> today · saved <n> in 30d`, with `cost` in place of `saved` for a negative reduction [repo://integrations/claude-code/plugins/powercontext/scripts/statusline.py#L131-L158].

Failures are classified by `failure_outcome` and shown as one of four fixed lines: `PC auth failed`, `PC version mismatch`, `PC offline · run powercontext doctor`, or `PC invalid response` [repo://integrations/claude-code/plugins/powercontext/scripts/statusline.py#L51-L56] [repo://integrations/claude-code/plugins/powercontext/scripts/statusline.py#L242-L264]. The status line has no `PreToolUse` or `Stop` role and writes no Source or Memory.

## Explicit Memory and Handoff operations

Durable writes do not happen through hooks. The `.mcp.json` entry gives Claude Code direct access to the Server's MCP tools, and a bundled Skill guides Claude to use them when the user explicitly asks to persist Memory or to transfer work. The plugin does not install a `Stop` hook, does not read the transcript, and does not capture Claude's final response. Automatic recall therefore does not depend on Claude choosing to call MCP, and MCP Memory writes do not replace prompt capture.

The MCP header relies only on `POWERCONTEXT_CLAUDE_AUTHORIZATION`. The Hook has a second source, a stored credential file used only when its Server URL matches the effective URL, but the MCP configuration does not read that file [repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py#L139-L143] [repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py#L260-L276] [repo://tests/claude_code_plugin/test_contract.py#L202-L245].

## Server endpoint and transport policy

The effective Server URL is resolved in this order: `POWERCONTEXT_CLAUDE_SERVER_URL`; the explicit `server_url` argument that the status line passes from `--server-url`; `CLAUDE_PLUGIN_OPTION_SERVER_URL`, then `POWERCONTEXT_CLIENT_SERVER_URL`; the saved `server_url` for the `claude-code` host in the client settings file; and finally `http://127.0.0.1:8000`. An unset or blank value falls through to the next source [repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py#L131-L135]. The saved client settings file is `POWERCONTEXT_CLIENT_CONFIG_FILE` when set, otherwise `~/.config/powercontext/clients.json`, and must be version 1 [repo://integrations/claude-code/plugins/powercontext/powercontext_client_config.py#L61-L88].

The URL must be HTTP or HTTPS, must not contain credentials, a query, or a fragment, and may use plaintext HTTP only for loopback hosts unless consent is granted. A trailing `/mcp` is removed [repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py#L278-L305]. Consent resolves in this order: an explicit setting; `POWERCONTEXT_CLAUDE_ALLOW_INSECURE_HTTP`; `POWERCONTEXT_CLIENT_ALLOW_INSECURE_HTTP`; and the saved consent, which applies only when the saved URL equals the effective URL. When the `CLAUDE_PLUGIN_OPTION_ALLOW_INSECURE_HTTP` option is present, it replaces the saved record [repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py#L93-L109] [repo://integrations/claude-code/plugins/powercontext/powercontext_client_config.py#L104-L126]. A non-loopback Server therefore needs HTTPS or explicit consent. Authorization must be a single Bearer credential with no whitespace inside it [repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py#L243-L258].

Requests are JSON over `urllib` with three bounds. Redirects are refused by the custom opener [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L78-L92]. Responses are capped at 1 MiB, enforced in the hook's reader and in the scope client's reader [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L53-L53] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L385-L404] [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L57-L57] [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L356-L368]. Each socket read is clamped to an absolute deadline by a socket wrapper, so a trickling server cannot outlive the budget [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L283-L315] [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L317-L328].

## Failure behavior

Recall and capture are independent. A failed recall does not stop capture, and a failed capture does not remove recalled context. Each recall failure is classified as follows; the Hook injects nothing for a failed recall.

| Condition | Recall outcome | Diagnostic |
| --- | --- | --- |
| HTTP 401 | `authentication_failed` | Emitted |
| HTTP 404 on `/v1/context/prepare` with no structured error code | `version_mismatch` | Emitted |
| HTTP 503, or the Server is unreachable or times out | `server_unavailable` | Emitted, with recovery `powercontext doctor` |
| Any other HTTP status (for example 400, 409, 422, or 404 with an error code), malformed JSON, schema mismatch, or an oversized response | `invalid_response` | Emitted |
| `status: empty` | `empty` | Written to stderr only |

The classification comes from `_http_failure_outcome` and the Prepared Context validator [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L104-L114] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L423-L485].

The same classification applies to the Scope resolution step of `UserPromptSubmit`, because `_scope_resolution_failure` restates the scope client's error in the hook's own vocabulary. A rejected credential becomes `authentication_failed`; a 503, a transport failure, or a timeout becomes `server_unavailable`; a 404 `scope_not_found` or any other status with a code becomes `invalid_response` with its `error_code`; a malformed Scope response also becomes `invalid_response` [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L563-L576] [repo://integrations/claude-code/plugins/powercontext/scripts/workspace_scope.py#L216-L257]. A failed recall, capture, or flush is reported the same way, with the event name in the diagnostic.

Diagnostics contain only the outcome and safe numeric or code fields. They never include the prompt, the Scope ID, the prepared content, the Authorization value, or the response body. Within one hook invocation, each failure outcome is emitted at most once. Across processes, a 60-second cooldown per outcome is kept in a best-effort state file at `POWERCONTEXT_DIAGNOSTIC_STATE_FILE`, or otherwise under `XDG_STATE_HOME` (or `~/.local/state`) in `powercontext/claude-code-diagnostics.json`. Failure outcomes are written to the `systemMessage` output of `UserPromptSubmit`, while informational events such as `empty` and `skipped` go to stderr [repo://integrations/claude-code/plugins/powercontext/hooks/diagnostics.py#L34-L34] [repo://integrations/claude-code/plugins/powercontext/hooks/diagnostics.py#L37-L46] [repo://integrations/claude-code/plugins/powercontext/hooks/diagnostics.py#L79-L107] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L487-L525].

Two timing bounds apply. The Hook's shared HTTP budget defaults to six seconds, and each request is limited to three seconds by default. Every request's timeout is also clamped to the remaining budget, so one slow call cannot extend the others [repo://integrations/claude-code/plugins/powercontext/claude_code_settings.py#L88-L89] [repo://integrations/claude-code/plugins/powercontext/hooks/user_prompt_submit.py#L336-L376].

## Configuration reference

| Variable or key | Default | Effect on the plugin |
| --- | --- | --- |
| `POWERCONTEXT_CLAUDE_SERVER_URL` | `http://127.0.0.1:8000` | Hook Server URL; first in the endpoint order |
| `CLAUDE_PLUGIN_OPTION_SERVER_URL` / `POWERCONTEXT_CLIENT_SERVER_URL` | unset | Plugin-option or client-wide Server URL, used when the environment variable above is unset or blank |
| `POWERCONTEXT_CLAUDE_SCOPE_ID` | unset | Explicit Scope sent as `explicit_scope_id`; overrides bindings and default |
| `POWERCONTEXT_CLAUDE_AUTHORIZATION` | unset | Complete `Bearer <token>` header for the Hook and the MCP entry |
| `POWERCONTEXT_CLAUDE_CAPTURE_PROMPTS` / `CLAUDE_PLUGIN_OPTION_CAPTURE_PROMPTS` | `true` | Enables prompt capture; the `POWERCONTEXT_` name takes precedence |
| `POWERCONTEXT_CLAUDE_FLUSH_ON_CAPTURE` | `false` | Waits for Source processing after capture |
| `POWERCONTEXT_CLAUDE_FLUSH_MAX_CALLS` | `4` | Maximum flush calls per prompt, 1 through 16 |
| `POWERCONTEXT_CLAUDE_BOOTSTRAP_CONTEXT` | `false` | Enables the SessionStart bootstrap package |
| `POWERCONTEXT_CLAUDE_BOOTSTRAP_MAX_BYTES` | `4096` | Bootstrap package byte bound, 512 through 8192 |
| `POWERCONTEXT_CLAUDE_BOOTSTRAP_HANDOFF` | unset | JSON object with the exact `artifact_id` and `revision` (and optional `family`) of a committed Handoff for bootstrap |
| `POWERCONTEXT_CLAUDE_BOOTSTRAP_STATE_DIR` | unset | Absolute root for the local bootstrap receipt; checked before `CLAUDE_PLUGIN_DATA` |
| `CLAUDE_PLUGIN_DATA` | unset | Fallback root for the bootstrap receipt when the state-dir variable is unset |
| `POWERCONTEXT_CLAUDE_CONTEXT_ASSEMBLY` | unset | JSON object sent as `assembly` in the prepare request |
| `POWERCONTEXT_CLAUDE_INCLUDE_CODE` | `false` | Sends `include_code: true` in the prepare request |
| `POWERCONTEXT_CLAUDE_REQUEST_TIMEOUT_SECONDS` | `3` | Per-request timeout; must be positive |
| `POWERCONTEXT_CLAUDE_HTTP_BUDGET_SECONDS` | `6` | Shared Hook HTTP budget; must be positive |
| `POWERCONTEXT_CLAUDE_ALLOW_INSECURE_HTTP` | unset | Consent for plaintext HTTP to a non-loopback Server |
| `POWERCONTEXT_CLIENT_ALLOW_INSECURE_HTTP` | unset | Client-wide consent for plaintext HTTP, checked after the Claude-specific variable |
| `CLAUDE_PLUGIN_OPTION_ALLOW_INSECURE_HTTP` | unset | Plugin option; when present, it replaces the saved consent record |
| `POWERCONTEXT_CLIENT_CONFIG_FILE` | `~/.config/powercontext/clients.json` | Saved non-secret client settings (version 1) |
| `CLAUDE_CONFIG_DIR` | `~/.claude` | Root of the stored credential file `powercontext/credentials.json` |
| `POWERCONTEXT_DIAGNOSTIC_STATE_FILE` | `$XDG_STATE_HOME/powercontext/claude-code-diagnostics.json` (or `~/.local/state/...`) | Location of the cross-process diagnostic cooldown state |

The plugin manifest's `userConfig` keys are `server_url` (default `http://127.0.0.1:8000`), `capture_prompts` (default `true`), and `allow_insecure_http` (no default) [repo://integrations/claude-code/plugins/powercontext/.claude-plugin/plugin.json#L19-L38].

Authorization is environment-only. The Server URL and plugin options must not carry a token. Restart Claude Code after changing its environment, because hooks and the MCP client inherit the environment at launch.

## Differences from the Codex integration

The Codex plugin in `integrations/codex` follows the same Server contract, but its wiring differs in ways that matter when changing either plugin [repo://integrations/codex/plugins/powercontext/hooks/hooks.json#L1-L55].

| Aspect | Claude Code plugin | Codex plugin |
| --- | --- | --- |
| Hook events | `SessionStart`, `UserPromptSubmit` | `SessionStart`, `PreToolUse`, `UserPromptSubmit`, `Stop` |
| Tool binding | None; no `PreToolUse` hook | `PreToolUse` on `mcp__powercontext__.*` |
| Final response | Not captured | `Stop` runs a token-savings hook |
| Hook runtime | `python3` with the standard library only | `uv run --frozen` in the plugin project |
| MCP header | Environment variable expanded in `.mcp.json` | `http_headers_helper` installed by setup |
| Status line | Installed as `statusLine` when none is custom | Not installed by the plugin |

Both plugins use the same Scope resolution endpoint and the same explicit `POWERCONTEXT_*` controls. See the [Codex Plugin](codex-plugin.md) page for its internal details and the [Capability Matrix](capability-matrix.md) for how the two hosts compare.

## Tests that protect this behavior

The focused suite is in `tests/claude_code_plugin`. `test_contract.py` pins the marketplace entry, the two-event hook shape with no `Stop` hook, the MCP header form, the Scope resolution keys, and URL and credential validation [repo://tests/claude_code_plugin/test_contract.py#L31-L129]. `test_hook.py` covers recall injection, capture idempotency, the prompt-length bounds, failure isolation between recall and capture, each Scope resolution failure reaching the host as a single diagnostic with nothing else sent, redirect refusal, and bounded error bodies [repo://tests/claude_code_plugin/test_hook.py#L222-L266] [repo://tests/claude_code_plugin/test_hook.py#L267-L306] [repo://tests/claude_code_plugin/test_hook.py#L410-L484] [repo://tests/claude_code_plugin/test_hook.py#L47-L84] [repo://tests/claude_code_plugin/test_hook.py#L512-L531] [repo://tests/claude_code_plugin/test_hook.py#L883-L933] [repo://tests/claude_code_plugin/test_hook.py#L1107-L1148] [repo://tests/claude_code_plugin/test_hook.py#L1149-L1177]. `test_bootstrap.py` covers the SessionStart receipt sequence and stale-receipt clearing [repo://tests/claude_code_plugin/test_bootstrap.py#L83-L189] [repo://tests/claude_code_plugin/test_bootstrap.py#L274-L300]. `test_statusline.py` and `test_python_floor.py` cover the status line and the Python version floor. The integration README also lists service-chain tests that run against live public and authenticated Servers [repo://integrations/claude-code/README.md#L1-L51].

## Related pages

- [Capability Matrix](capability-matrix.md) compares host coverage across integrations.
- [Codex Plugin](codex-plugin.md) describes the parallel Codex wiring.
- [Prepare and Recall](../workflows/prepare-and-recall.md) explains the Server-side preparation that the recall hook calls.
