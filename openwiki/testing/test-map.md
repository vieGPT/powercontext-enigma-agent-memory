---
type: testing-guide
title: Test Map and Verification Guide
description: Maps PowerContext's pytest, JavaScript plugin, API contract, integration conformance, hook, CLI, client, and opt-in acceptance suites to the code they protect, with the make and pytest commands to run each one before submitting a change.
tags: [testing, verification, pytest, make, tox, contract-tests, conformance, plugins]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-07T09:04:09.583Z
sources:
  - id: openwiki-source-17f5c5574bdf211c2956cf9a
    resource: repo://integrations/openclaw/plugins/memory-powercontext/src/lifecycle.test.ts
  - id: openwiki-source-6049b64b344a149fb4048602
    resource: repo://integrations/pi/plugins/powercontext/tests/flush.spec.ts
  - id: openwiki-source-012f2c78e3b1446dfc35803f
    resource: repo://Makefile
  - id: openwiki-source-05ccef8d4cf1698187f20464
    resource: repo://pyproject.toml
  - id: openwiki-source-cce5dbe66e8851c74f25605b
    resource: repo://tests/codex_plugin/conftest.py
  - id: openwiki-source-f0a6e7dc03522b2682f88655
    resource: repo://tests/conftest.py
  - id: openwiki-source-da1cc862f5a540f793502703
    resource: repo://tests/e2e/conftest.py
  - id: openwiki-source-3a196bec0b7e7b254c1f9748
    resource: repo://tests/integration_conformance/cases.json
  - id: openwiki-source-30f2f291b783792edac718c2
    resource: repo://tests/integration_conformance/test_cases.py
  - id: openwiki-source-1e54a80b61ff95aaea3dda0c
    resource: repo://tests/integrations/test_python_adapter_compatibility.py
  - id: openwiki-source-e88d71bc921e1d285f34c823
    resource: repo://tests/test_access_control.py
  - id: openwiki-source-c614587b21b95bb1d1e0025a
    resource: repo://tests/test_api_contract.py
  - id: openwiki-source-94a41f4de603e313d1fa261a
    resource: repo://tests/test_handoff_report.py
  - id: openwiki-source-ea57579c85533b3f3493b2f2
    resource: repo://tests/test_hook_characterization.py
  - id: openwiki-source-9d3769fea9a39735ad2f1c0b
    resource: repo://tests/test_integration_manifest.py
  - id: openwiki-source-123aad5092f3e8e77034a45a
    resource: repo://tests/test_topic_memory_server.py
  - id: openwiki-source-0fb6b644bb74f4c54e97b19f
    resource: repo://tests/test_upgrade_release_gate.py
  - id: openwiki-source-ccd2623ea7a88d6abc2d9b5a
    resource: repo://tox.ini
generated: { by: "openwiki/0.9.0", at: "2026-11-07T09:04:09.583Z" }
---

# Test Map and Verification Guide

This page tells you which test suite protects which part of PowerContext and which command runs it. Start from the boundary your change touches, run the narrowest suite that covers it, and widen only when the change crosses components. The named `make` targets in the repository's `Makefile` are the authoritative entry points; `tox` covers the Python version matrix. Do not invent other targets: if a command is not listed in the `Makefile` or `tox.ini`, use `uv run python -m pytest <path>` with a path from the tables below.

Default runs are hermetic. Suites that need real Codex, model providers, external databases, or installed host applications are opt-in and are skipped unless their flag or environment is set. A passing hermetic run does not prove that a real host or backend behaves the same way; `AGENTS.md` asks contributors to keep actual execution, mocks, emulators, and direct hook invocation distinct when they make external claims ([AGENTS.md](repo://AGENTS.md)).

## Choosing a check

```mermaid
flowchart TD
  Start["Changed code or contract"] --> Contract{"OpenAPI or generated code"}
  Contract -->|yes| ContractCheck["make contract-test"]
  Contract -->|no| Plugin{"Plugin or host hook"}
  Plugin -->|JavaScript plugin| JsCheck["make pi-test or make openclaw-plugin-test or make opencode-test"]
  Plugin -->|Python hook| HookCheck["uv run python -m pytest tests/test_hook_characterization.py"]
  Plugin -->|no| Core{"Server or runtime behavior"}
  Core -->|single boundary| Focused["uv run python -m pytest tests/test_file.py"]
  Core -->|across CLI client and Server| E2E["make e2e-test"]
```

The diagram shows the first decision for a change. Run `make check` before a PR when you touch version references, the integration manifest, workflow actions, or the lock file; run `tox` when the change can affect supported Python versions.

## Make targets

The table lists the test-related targets in the `Makefile`. Commands are shown as the target runs them.

| Target | What it runs | Use when |
| --- | --- | --- |
| `make test` | `pytest -n auto --doctest-modules` over the whole test tree, with the serial Hook-deadline file run afterwards at `-n 0` ([Makefile](repo://Makefile#L44-L55)) | Broad confidence before submitting |
| `make unit-test` | The same pytest run with `--ignore=tests/e2e` ([Makefile](repo://Makefile#L57-L59)) | Fast loop for in-process behavior |
| `make e2e-test` | `tests/e2e` in parallel, then the serial Hook-deadline file ([Makefile](repo://Makefile#L61-L64)) | CLI to client SDK to Server flows |
| `make contract-test` | `api-generate-check`, `js-api-generate-check`, then `tests/test_api_contract.py`, `tests/test_js_operations.py`, `tests/test_desktop_contract.py` ([Makefile](repo://Makefile#L153-L155)) | Any change to the HTTP contract or generated bindings |
| `make api-generate` / `make api-generate-check` | Regenerate or verify Python API models from `openapi/powercontext.yaml` ([Makefile](repo://Makefile#L157-L163)) | After editing the OpenAPI contract |
| `make js-api-generate` / `make js-api-generate-check` | Regenerate or verify JavaScript operation tables ([Makefile](repo://Makefile#L165-L171)) | After editing the contract used by JavaScript integrations |
| `make integration-manifest-check` | Integration capability docs check, then `tests/test_integration_manifest.py` ([Makefile](repo://Makefile#L303-L305)) | Changing integration catalog or capability metadata |
| `make check` | Version, upgrade-policy, workflow-action, integration-manifest, and guidance checks, plus `prek` and `ty` ([Makefile](repo://Makefile#L21-L33)) | Before opening a PR |
| `make js-test` | Installs, tests, builds, and diffs the committed DeepSeek Harness plugin output, then runs its e2e tests ([Makefile](repo://Makefile#L173-L182)) | Changes under `integrations/dsh/plugins/powercontext` |
| `make pi-test` | Installs, tests, and typechecks the Pi package ([Makefile](repo://Makefile#L220-L224)) | Changes under `integrations/pi/plugins/powercontext` |
| `make openclaw-plugin-test` | Installs, tests, typechecks, and builds the OpenClaw memory plugin ([Makefile](repo://Makefile#L201-L206)) | Changes under `integrations/openclaw/plugins/memory-powercontext` |
| `make opencode-test` | Installs, tests, typechecks, builds, and checks the committed OpenCode bundle ([Makefile](repo://Makefile#L212-L218)) | Changes under `integrations/opencode/plugins/powercontext` |
| `make cindy-test` | Builds the Cindy hook adapter, checks its worker bundle, runs `node --test`, and packages it ([Makefile](repo://Makefile#L189-L195)) | Changes under `integrations/cindy` |
| `make dify-test` | Checks the isolated Dify SDK plugin and runs `tests/e2e/test_dify_tools_http.py` ([Makefile](repo://Makefile#L226-L235)) | Changes under `integrations/dify` |
| `make harness-check` | Lints, typechecks, and tests the Bub replay harness under `e2e/bub` ([Makefile](repo://Makefile#L116-L122)) | Changes to the replay harness |
| `make opendal-test` | Lints, typechecks, and tests the OpenDAL connector ([Makefile](repo://Makefile#L107-L114)) | Changes under `integrations/opendal` |
| `make evaluation-unit-test` | Runs the evaluation unit suites with `-m "not live"` and the agent tool recovery suite ([Makefile](repo://Makefile#L66-L76)) | Changes under `evaluation/` only |
| `make code-seekdb-test` | Runs native code indexing against an embedded seekdb instance ([Makefile](repo://Makefile#L78-L81)) | Native code indexing with seekdb |
| `make real-e2e-test` | Runs the opt-in real Codex Experience and Skill tests with `--run-real-e2e` ([Makefile](repo://Makefile#L83-L88)) | Real host and model acceptance, with credentials configured |
| `make topic-memory-r8-acceptance` | Runs the bounded topic-memory product-chain harness, by default layers `e0` through `e4` ([Makefile](repo://Makefile#L90-L96)) | Topic-memory product acceptance |
| `make docs-test` | Lints, tests, and builds the website ([Makefile](repo://Makefile#L289-L293)) | Documentation site changes |

Packaging and release gates such as `make version-check`, `make upgrade-policy-check`, `make release-check`, and `make upgrade-release-test` are described in the `Makefile` and are separate from the behavior suites on this page.

## Python matrix with tox

`tox.ini` defines the `py311`, `py312`, `py313`, and `py314` environments. Each one runs `uv sync --locked`, then `pytest --doctest-modules tests` with coverage, then `ty check` ([tox.ini](repo://tox.ini#L1-L21)). Use `tox` for changes that affect supported Python versions. Extra arguments are passed through as pytest positional arguments.

## pytest configuration and collection controls

`pyproject.toml` sets the default collection roots to `tests` and `evaluation/memory/atomic_extraction/tests`, and registers the opt-in markers `native_service`, `real_e2e`, `zcode_host_acceptance`, `zcode_live_model`, and `cross_host_acceptance` ([pyproject.toml](repo://pyproject.toml#L270-L278)).

The root [`tests/conftest.py`](repo://tests/conftest.py#L47-L77) controls behavior that every test inherits:

- An autouse fixture restores the shared artifact-family profile registry after each test, so a plugin registration in one test cannot change another Server's readiness.
- Unless `--run-real-e2e` is set, developer `POWERCONTEXT_SERVER_`, `POWERCONTEXT_CLIENT_`, and `POWERCONTEXT_CODEX_` variables are removed, client settings and host homes (`HERMES_HOME`, `DSH_HOME`) point at per-test temporary paths, and `localhost` bypasses proxies.
- [`pytest_collection_modifyitems`](repo://tests/conftest.py#L154-L188) deselects ZCode host acceptance and cross-host tests unless their marker or file is selected explicitly. Selecting them without prerequisites fails rather than silently passing.
- [`pytest_ignore_collect`](repo://tests/conftest.py#L190-L207) excludes `tests/release_upgrades`, the Alembic migration `env.py`, and `tests/e2e/real_experience_skill` unless `--run-real-e2e` is passed.

The options registered in [`pytest_addoption`](repo://tests/conftest.py#L80-L152) include `--run-real-e2e`, `--real-e2e-mode` (`baseline`, `configured`, or `all`), `--real-codex-timeout`, `--real-e2e-env-file`, `--run-zcode-acceptance`, `--zcode-model-config`, `--zcode-generation-env`, `--require-dsh-runtime`, `--require-dsh-config-runtime`, `--require-claude-code-python-floor`, and `--cross-host-acceptance-output`. Use `make real-e2e-test` for the real Codex path rather than assembling these options by hand.

## Suite map by protected code

### Server, access control, and HTTP contract

| Suite | Protects | Representative tests |
| --- | --- | --- |
| [`tests/test_access_control.py`](repo://tests/test_access_control.py#L62-L118) | The authorization service: receipt commit proof separate from identity reservation, logical artifact share as read-only across versions, memory share targeting one logical entry, and administration roles that do not become content writers | `test_receipt_commit_proof_is_separate_from_identity_reservation`, `test_logical_artifact_share_is_read_only_across_all_versions` |
| `tests/test_access_http.py`, `tests/test_access_mcp.py`, `tests/test_access_adapters.py`, `tests/test_access_snapshot_boundary.py` | Access enforcement at the HTTP, MCP, and adapter boundaries and at snapshot reads | Per-boundary access cases |
| `tests/e2e/test_access_control_http.py`, `tests/e2e/test_access_control_regressions.py` | End-to-end access control through the running Server, including regressions | Regression cases named for the defect |
| `tests/test_server.py`, `tests/test_server_http_limits.py`, `tests/test_server_uds.py`, `tests/test_mcp.py` | Server application wiring, HTTP request limits, Unix domain socket transport, and MCP surface | Server factory and transport cases |
| [`tests/test_api_contract.py`](repo://tests/test_api_contract.py#L188-L193) | The checked-in OpenAPI contract at `openapi/powercontext.yaml`: the namespaced request-id header, typed public shapes, and declared responses | `test_contract_uses_the_namespaced_request_id_header`, `test_every_access_protected_operation_declares_the_unavailable_response` |
| [`tests/test_js_operations.py`](repo://tests/test_js_operations.py#L36-L60) | The JavaScript operation generator covers every OpenAPI operation with method, path, location, and scope | `test_js_operations_cover_every_openapi_operation` |

The contract is the source of truth. After editing it, run `make api-generate` and `make contract-test`, as `AGENTS.md` requires ([AGENTS.md](repo://AGENTS.md)). The generated `src/powercontext/http/_generated/` code and the JavaScript operation tables are verified for freshness by `make api-generate-check` and `make js-api-generate-check`; the contract tests then check properties of the contract itself.

### Topic memory

[`tests/test_topic_memory_server.py`](repo://tests/test_topic_memory_server.py#L143-L214) covers the public topic-memory HTTP shapes, mapping of missing or cross-scope values to not-found, a 503 response when flush is requested without a processing capability, and redacted fallback failure output. Its log and span tests check that the failure log and span agree without exposing private error text ([test_topic_memory_server.py](repo://tests/test_topic_memory_server.py#L294-L300)). Product-chain and privacy behavior is covered by `tests/e2e/test_topic_memory_*.py`, and the layered acceptance run is `make topic-memory-r8-acceptance`.

### Handoff report

[`tests/test_handoff_report.py`](repo://tests/test_handoff_report.py#L27-L60) checks that the handoff report uses the same scope selection as the rest of the Server (`subtree`, `exact`, and `all` modes). A second case checks that Markdown output keeps scope organization and exact addresses ([test_handoff_report.py](repo://tests/test_handoff_report.py#L90-L100)), and a third checks that the statement count does not grow with the number of scopes ([test_handoff_report.py](repo://tests/test_handoff_report.py#L157-L170)). Runtime handoff behavior is covered by `tests/e2e/test_handoff_runtime.py`.

### Client SDK and receivers

`tests/test_client.py` and `tests/test_client_transport_policy.py` cover the Python client SDK and its transport policy. `tests/client/test_skill_receiver.py` and `tests/client/test_receiver_service.py` cover the remote skill receiver, which applies Skill packages to a target environment and reports results.

### CLI, setup, and installation

The CLI suites are mostly named for their command group: `tests/test_cli.py`, `tests/test_cli_scope.py`, `tests/test_config_cli.py`, `tests/test_config_wizard*.py`, `tests/test_installation*.py`, `tests/test_system_cli.py`, `tests/test_setup_transport.py`, and the per-host CLI files such as `tests/test_cli_zcode.py`, `tests/test_opencode_cli.py`, and `tests/test_openclaw_cli.py`. Installation tests use the `installed_agent_checks`, `server_checks`, and `hermes_activation` fixtures in `tests/conftest.py` to isolate native installation behavior from separately exercised runtime probes ([conftest.py](repo://tests/conftest.py#L208-L247)).

### Host hooks and plugins

- **Hook characterization.** [`tests/test_hook_characterization.py`](repo://tests/test_hook_characterization.py#L14-L30) runs installed copies of the Claude Code and Codex plugin hooks with the host's own invocation style and a scripted Server. It records the requests the Server receives, the hook's stdout, stderr, and exit status, and files left for later hooks, then compares them with the recorded expectations. The parametrized case runs per host and case ([test_hook_characterization.py](repo://tests/test_hook_characterization.py#L81-L91)). The fixtures live under `tests/fixtures`; this page does not document their contents. The module is skipped on Windows because the recorded state includes POSIX file modes. After a reviewed behavior change, the docstring says to regenerate expectations with `uv run python tests/test_hook_characterization.py` and review the diff.
- **Codex plugin.** The `tests/codex_plugin` suite loads plugin scripts by file path from `integrations/codex/plugins/powercontext` and isolates `CODEX_HOME`, diagnostic state, and bootstrap state per test ([conftest.py](repo://tests/codex_plugin/conftest.py#L22-L62)). `test_recall.py` checks that recall emits bounded, untrusted context ([test_recall.py](repo://tests/codex_plugin/test_recall.py#L93-L100)).
- **JavaScript plugins.** `make js-test`, `make pi-test`, `make openclaw-plugin-test`, and `make opencode-test` run each package's own test runner and typecheck. Their behavior is described by the plugin pages, not repeated here.

### Integration conformance

The integration conformance suite is a shared behavior contract for pending-Source delivery, consumed by both Python and JavaScript:

- [`tests/integration_conformance/cases.json`](repo://tests/integration_conformance/cases.json#L1-L60) defines schema version 1 cases, each with `modes`, `input`, and `expected`. The cases cover cursor catch-up, malformed acknowledgements, flush failure, duplicate delivery, scope isolation, closing and cancellation boundaries, best-effort fail-open, required-mode fail-closed, and extraction unavailable.
- [`tests/integration_conformance/test_cases.py`](repo://tests/integration_conformance/test_cases.py#L20-L48) checks that the fixture has exactly the ten required case IDs, that each case has non-empty input and expected blocks, and that `required_mode_fail_closed` runs only in `required_capable` mode with `host_continues` set to `false`.
- The Pi flush spec ([flush.spec.ts](repo://integrations/pi/plugins/powercontext/tests/flush.spec.ts#L27-L37)) and the OpenClaw lifecycle test ([lifecycle.test.ts](repo://integrations/openclaw/plugins/memory-powercontext/src/lifecycle.test.ts#L35-L40)) load the same JSON file. The Pi spec looks up a case by ID in `best_effort` mode and fails if it is missing.

Changing a case's expectations therefore affects the Python fixture test and the JavaScript plugin tests together. Run `uv run python -m pytest tests/integration_conformance` and the relevant plugin target.

### Adapters and optional APIs

[`tests/integrations/test_python_adapter_compatibility.py`](repo://tests/integrations/test_python_adapter_compatibility.py#L17-L73) runs the LangChain, LangGraph, and Pydantic AI adapter settings in a separate isolated interpreter, both with and without the core text-assembly API. It checks that `context_assembly` works when the API is present and fails with a clear validation message when it is not. `tests/integrations/test_hermes_provider.py` covers the Hermes provider.

### Builtin runtime and native code

`tests/builtin/` holds unit tests for the builtin runtime: native code indexing (`test_native_code*.py`), publication, scope application, tags, processing diagnostics, and work receipt models, with subdirectories for artifacts, dream, inference, persistence, portability, review, and runtime. The native command and service paths are also exercised by root-level suites such as `tests/test_native_command.py` and `tests/test_native_service_diagnostics.py`.

### Integration manifest

[`tests/test_integration_manifest.py`](repo://tests/test_integration_manifest.py#L28-L45) checks that the agent-host integrations in the manifest match the setup installers, that integration directories and evidence paths resolve, and that the tool surfaces match. `make integration-manifest-check` runs it after the generated capability docs check.

### End-to-end acceptance

`tests/e2e/` holds cross-component scenarios that run from the CLI or client SDK through a running Server. Examples include `tests/e2e/test_runtime_server.py`, `tests/e2e/test_mcp_transport.py`, `tests/e2e/test_server_cli.py`, the host service chains such as `tests/e2e/test_codex_service_chain.py` and `tests/e2e/test_zcode_service_chain.py`, and `tests/e2e/test_dify_tools_http.py`. The `Makefile` runs the Hook-deadline file `tests/e2e/test_zcode_service_chain.py` serially so unrelated parallel work cannot starve its deadlines ([Makefile](repo://Makefile#L46-L49)).

Database-backed migration scenarios use a `migration_backend` fixture that always runs with SQLite, runs with seekdb only when `POWERCONTEXT_TEST_MIGRATION_SEEKDB=1`, and runs with OceanBase only when `POWERCONTEXT_TEST_MIGRATION_OCEANBASE_URL` is set. The OceanBase path refuses a URL without an explicit management database and verifies a real OceanBase version before creating its per-test database ([e2e conftest.py](repo://tests/e2e/conftest.py#L30-L108)).

### CI and release tooling tests

Some root tests check repository tooling rather than product behavior. `tests/test_ci_pipeline.py` and `tests/test_ci_changes.py` exercise CI selection and retry logic, `tests/test_upgrade_release_gate.py` checks the release gate, and `tests/test_release_version.py` checks version synchronization. The release gate's required mode, for example, executes its cases but cannot publish as supported ([test_upgrade_release_gate.py](repo://tests/test_upgrade_release_gate.py#L94-L104)). Run these when you change the corresponding scripts, not as evidence about Server behavior.

## Opt-in and host-dependent suites

| Suite | Opt-in control | Notes |
| --- | --- | --- |
| Real Codex Experience and Skill tests (`tests/e2e/real_experience_skill`) | `make real-e2e-test`, which passes `--run-real-e2e`, `--real-e2e-mode`, `--real-codex-timeout`, and `--real-e2e-env-file` | Uses real Codex and configured services; skipped at collection otherwise |
| ZCode host acceptance | `--run-zcode-acceptance`, with `--zcode-model-config` or `--zcode-generation-env` for live mode | Requires an explicitly configured ZCode CLI and isolated Server; marked `zcode_host_acceptance` and `zcode_live_model` |
| Cross-host work continuity | Marker `cross_host_acceptance` or the `test_cross_host_work_continuity.py` file | Actual Pi RPC to ZCode CLI with isolated SQLite and controlled inference |
| Native service tests | Marker `native_service` | Mutates a disposable current-user service manager on a matching platform |
| Code indexing with seekdb | `make code-seekdb-test` | Runs against an embedded seekdb instance |
| Bub replay harness | `make harness-check` for hermetic checks; `make harness-acceptance` and `make harness-paired` against an existing Server | The compose targets (`make harness-compose-check`, `make harness-compose-acceptance`) build isolated environments |

Do not treat a skipped opt-in suite as passing. When a change claims real-host or real-backend behavior, run the matching target with its prerequisites and report what actually executed.

## Practical guidance

- Start with the focused file for the module you changed, for example `uv run python -m pytest tests/test_handoff_report.py -q`. Add `make contract-test` for any `openapi/` edit and `make e2e-test` when a CLI, client, and Server path changes together.
- Changes to hook scripts should run the hook characterization suite, not only the plugin's own unit tests, because that suite exercises the installed copies with each host's invocation style.
- Changes to the conformance fixture should run the Python fixture test and every JavaScript plugin that loads it.
- Changes to the OpenAPI contract require regeneration before tests: `make api-generate`, `make js-api-generate`, then `make contract-test`.
- Keep the design guidance in `AGENTS.md` in mind when adding tests: assert through public interfaces, add regression tests for defects likely to recur, and avoid tests that only freeze implementation details ([AGENTS.md](repo://AGENTS.md)).

## Related pages

- [HTTP and MCP API](../architecture/http-and-mcp-api.md) describes the contract surfaces that the contract and transport suites protect.
- [Source map](../architecture/source-map.md) locates the source directories that these suites import.
- [Quickstart](../quickstart.md) covers installing the development environment that the commands above assume.
