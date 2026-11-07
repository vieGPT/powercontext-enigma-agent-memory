---
type: Operations runbook
title: Database Migrations, Data Migration and Portable Archives
description: Runbook for PowerContext Server schema revisions (db-migrate), legacy 1.3.0 and 1.4.0 upgrade admission and rejection, maintenance locks and recovery, Processing and Atomic Memory data migrations, the upgrade release gate, and portable .pcb archive export and restore.
tags: [migrations, database, upgrade, archive, operations, sqlite, seekdb, oceanbase]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-07T09:43:24.725Z
sources:
  - id: openwiki-source-a94249c04ae33f0d631c9044
    resource: repo://docs/en/docs/operate/artifact-processing-migration.md
  - id: openwiki-source-ef9b50c9ef8b9abe11967085
    resource: repo://docs/en/docs/operate/atomic-memory-migration.md
  - id: openwiki-source-bbe5d9794e77b49d99514414
    resource: repo://docs/en/docs/operate/database-migrations.md
  - id: openwiki-source-4b90f25d9bacdfbc4b1b79e1
    resource: repo://release/upgrade-support.json
  - id: openwiki-source-418f2306f338afdfcbadc89a
    resource: repo://scripts/check_atomic_memory_migration_shards.py
  - id: openwiki-source-e5db5e5c2b9bc4c5d9f21de1
    resource: repo://scripts/upgrade_release_gate.py
  - id: openwiki-source-7eabfb50d3b9e4892935a078
    resource: repo://src/powercontext/builtin/persistence/atomic_memory_evidence.py
  - id: openwiki-source-490f6c3e6398b0c2adbc5ba6
    resource: repo://src/powercontext/builtin/persistence/atomic_memory_identity.py
  - id: openwiki-source-a6f5ba9958f56301beea0bdd
    resource: repo://src/powercontext/builtin/persistence/migrations/atomic_memory_archive.py
  - id: openwiki-source-c8767a4e013a253fc023649b
    resource: repo://src/powercontext/builtin/persistence/migrations/atomic_memory_references.py
  - id: openwiki-source-2e26c89b8e06915c56c0bb7e
    resource: repo://src/powercontext/builtin/persistence/migrations/atomic_memory_v1.py
  - id: openwiki-source-c499770f18a7e8137434e0db
    resource: repo://src/powercontext/builtin/persistence/migrations/atomic_projection.py
  - id: openwiki-source-0d017396c93839baace10d2b
    resource: repo://src/powercontext/builtin/persistence/migrations/backup.py
  - id: openwiki-source-5219e76482ced55f611abb64
    resource: repo://src/powercontext/builtin/persistence/migrations/connections.py
  - id: openwiki-source-f746b1a78d620cdeccf02874
    resource: repo://src/powercontext/builtin/persistence/migrations/deployment.py
  - id: openwiki-source-9b7384dd37dea09b0b395d65
    resource: repo://src/powercontext/builtin/persistence/migrations/historical.py
  - id: openwiki-source-1ed8342d177e20367e6db6c2
    resource: repo://src/powercontext/builtin/persistence/migrations/locking.py
  - id: openwiki-source-37139bc207aafc946226f16c
    resource: repo://src/powercontext/builtin/persistence/migrations/mysql.py
  - id: openwiki-source-5df7b4318f27c661ca64090c
    resource: repo://src/powercontext/builtin/persistence/migrations/oceanbase.py
  - id: openwiki-source-4ca03022a784c754c3de26c4
    resource: repo://src/powercontext/builtin/persistence/migrations/revision_names.py
  - id: openwiki-source-a6e908fbd7df970384ecdd74
    resource: repo://src/powercontext/builtin/persistence/migrations/server_baselines.py
  - id: openwiki-source-57bb11a86bb181d3aeb1f32f
    resource: repo://src/powercontext/builtin/persistence/migrations/server_readiness.py
  - id: openwiki-source-6aa28a45ee4a6ebef960ed38
    resource: repo://src/powercontext/builtin/persistence/migrations/server_resources/versions/v1_5_0_003_atomic_memory_evidence_projection.py
  - id: openwiki-source-f09a1660968a29408fd9a932
    resource: repo://src/powercontext/builtin/persistence/migrations/sources.py
  - id: openwiki-source-cb31c904a2a4c0bd7dc1036b
    resource: repo://src/powercontext/builtin/persistence/migrations/sqlite.py
  - id: openwiki-source-c925caf620bdcce1682e5a3b
    resource: repo://src/powercontext/builtin/persistence/processing_migration.py
  - id: openwiki-source-0f4a1c14e307b33db7e6138e
    resource: repo://src/powercontext/builtin/portability/bundle.py
  - id: openwiki-source-f83c558a3774fb051e60cbb0
    resource: repo://src/powercontext/builtin/runtime/atomic_memory_rebuild.py
  - id: openwiki-source-7ffc4e7ffdd9e56fa4ca3150
    resource: repo://src/powercontext/cli/archive.py
  - id: openwiki-source-bc58b27142b00aeeb91ebfcf
    resource: repo://src/powercontext/server/cli.py
  - id: openwiki-source-f667698ffdd93bd54ab8c3db
    resource: repo://src/powercontext/server/database_admission.py
  - id: openwiki-source-ca38935e0c998718463fdb4d
    resource: repo://src/powercontext/server/database_migration.py
  - id: openwiki-source-bad9e61f25ada75aaa317ce4
    resource: repo://src/powercontext/server/factory.py
generated: { by: "openwiki/0.9.0", at: "2026-11-07T09:43:24.725Z" }
---

# Database Migrations, Data Migration and Portable Archives

This runbook separates three mechanisms that operators often confuse:

1. **Schema revisions.** Registered Alembic revisions applied only by an explicit `powercontext server db-migrate apply`. Installing software never migrates a database.
2. **Data migrations.** Offline conversions and repairs of domain data: Processing scheduling state, legacy Memory to Atomic Memory, Atomic search projections, and Topic Memory recovery.
3. **Portable archives.** Logical `.pcb` bundles that move complete scopes with `powercontext archive`. An archive is not a database upgrade and does not replace a deployment-native backup.

## Support status

The installed release gate is in `upgrade-required` mode for target `1.5.0`. The gate's notice tells operators that upgrades from 1.3.0 and 1.4.0 require passing full Server acceptance on SQLite, seekDB and OceanBase, that these paths are not yet verified, and not to upgrade existing databases ([repo://scripts/upgrade_release_gate.py#L63-L66](../../scripts/upgrade_release_gate.py#L63-L66)). The policy file records the same mode ([repo://release/upgrade-support.json#L1-L4](../../release/upgrade-support.json#L1-L4)).

Treat existing databases as not upgradable until the exact target artifact passes the required automatic matrix and the reviewed support declaration changes. The code paths below exist and are tested locally, but local tests do not establish installed-release acceptance.

## Commands at a glance

| Command | Purpose | Writes |
| --- | --- | --- |
| `powercontext server db-migrate status` | Report the schema version without creating a database or control tables. | No |
| `powercontext server db-migrate verify` | Check readiness of the selected bundle. It does not assert cluster readiness. | No |
| `powercontext server db-migrate plan` | Produce a reviewable plan with `plan_id`, backup policy and maintenance scope. | No |
| `powercontext server db-migrate apply` | Back up, migrate, verify. Requires an accepted plan. | Yes |
| `powercontext server processing-migrate` | Processing state. Managed databases delegate to `db-migrate --bundle processing`. | Yes, when applied |
| `powercontext server atomic-memory-migrate` | Legacy Memory to Atomic Memory. Managed databases delegate to `db-migrate --bundle server`. | Yes, when applied |
| `powercontext server atomic-memory-rebuild-projection` | Rebuild current Atomic search rows. Managed databases delegate to `db-migrate --bundle atomic-projection`. | Yes, when applied |
| `powercontext archive export` / `inspect` / `restore` | Portable `.pcb` bundle. `restore --dry-run` validates; `restore --yes` writes. | Restore only |

`db-migrate` accepts `--bundle server|artifacts|topic-memory-recovery|processing|atomic-projection`. The default is `server`, and only the `server` bundle can report full Server readiness ([repo://src/powercontext/server/database_migration.py#L44-L48](../../src/powercontext/server/database_migration.py#L44-L48), [repo://src/powercontext/server/database_migration.py#L280-L294](../../src/powercontext/server/database_migration.py#L280-L294)). The archive group has no `validate` subcommand; dry-run is the `--dry-run` flag on `restore` ([repo://src/powercontext/cli/archive.py#L69-L122](../../src/powercontext/cli/archive.py#L69-L122)).

## Revision chain

Migration names use `v<major>_<minor>_<patch>_<sequence>`. Stored IDs for older revisions are opaque aliases. The release-based names are presentation metadata only ([repo://src/powercontext/builtin/persistence/migrations/revision_names.py#L19-L55](../../src/powercontext/builtin/persistence/migrations/revision_names.py#L19-L55)). Frozen resources, plan IDs and recovery evidence keep their original IDs (`s1300`, `r1400`, `s1400` to `s1402`).

Two registered source histories reach the same head `v1_5_0_003`. The default source is the 1.3.0 baseline. The 1.4.0 source runs through a separate frozen bridge ([repo://src/powercontext/builtin/persistence/migrations/deployment.py#L46-L57](../../src/powercontext/builtin/persistence/migrations/deployment.py#L46-L57)).

```text
1.3.0: v1_3_0_000 -> v1_4_1_001 -> v1_4_1_002 -> v1_4_1_003 -> v1_4_1_004 -> v1_4_1_005 -> v1_4_1_006 -> v1_5_0_001 -> v1_5_0_002 -> v1_5_0_003
1.4.0: v1_4_0_000 -> v1_4_1_001 -> v1_4_1_002 -> v1_4_1_003 -> v1_4_1_004 -> v1_4_1_005 -> v1_4_1_006 -> v1_5_0_001 -> v1_5_0_002 -> v1_5_0_003
```

Each step has a file in the bundled `server_resources/versions/` directory. For example, `v1_5_0_003` registers the disposable evidence projection ([repo://src/powercontext/builtin/persistence/migrations/server_resources/versions/v1_5_0_003_atomic_memory_evidence_projection.py](../../src/powercontext/builtin/persistence/migrations/server_resources/versions/v1_5_0_003_atomic_memory_evidence_projection.py)).

## Admission and legacy rejection

Admission classifies the database before any write. The selected source determines which steps may run.

```mermaid
flowchart TD
    A["Plan or startup check"] --> B{"Any schema objects present?"}
    B -- "No" --> C["uninitialized: apply initializes"]
    B -- "Yes" --> D{"pc_schema_revision present?"}
    D -- "Yes" --> E{"Single revision at head?"}
    E -- "Yes" --> F["ready after checks"]
    E -- "No" --> G["migration_required"]
    D -- "No" --> H{"Layout matches a registered 1.3.0 or 1.4.0 source?"}
    H -- "Yes" --> I["Plan historical steps and checkpoints"]
    H -- "No" --> J{"Complete released Server schema recognized?"}
    J -- "Yes" --> K["server_migration_unavailable"]
    J -- "No" --> L["unknown_baseline: no changes made"]
```

Caption: admission of an unmanaged or managed database, as implemented in the SQLite and MySQL-family runners.

Branches, in the order the code applies them:

- **Managed storage with a revision.** The control table must have the registered single-column shape. Multiple revision rows are rejected as `incompatible_schema`. An unknown control-table shape is `recovery_required` ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L384-L400](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L384-L400)). Pending revisions make the plan `migration_required`.
- **Unmanaged storage with a recognized layout.** `select_source` compares the layout against the default and bridge source bundles. A unique match selects that source and its historical steps. More than one matching contract is `incompatible_schema` ([repo://src/powercontext/builtin/persistence/migrations/historical.py#L75-L95](../../src/powercontext/builtin/persistence/migrations/historical.py#L75-L95)).
- **Historical steps are state-based.** Admission records only the steps the data requires: `historical_structure`, `historical_scopes`, `historical_processing` and `historical_atomic_memory` ([repo://src/powercontext/builtin/persistence/migrations/historical.py#L114-L137](../../src/powercontext/builtin/persistence/migrations/historical.py#L114-L137)). Software version strings never authorize adoption.
- **Unrecognized schema.** For SQLite, a non-empty schema with no selected source and no baseline match raises `unknown_baseline` and makes no changes. A schema that matches a complete released Server baseline, but for which no path is selected, raises `server_migration_unavailable` ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L384-L443](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L384-L443), [repo://src/powercontext/builtin/persistence/migrations/server_baselines.py#L40-L75](../../src/powercontext/builtin/persistence/migrations/server_baselines.py#L40-L75)).
- **Baseline matching.** A complete schema fingerprint must match a frozen baseline exactly. An inconsistent frozen baseline resource, or a fingerprint claimed by several releases, is `invalid_bundle` ([repo://src/powercontext/builtin/persistence/migrations/server_baselines.py#L60-L75](../../src/powercontext/builtin/persistence/migrations/server_baselines.py#L60-L75)).
- **Recorded evidence.** If a maintenance journal exists, its recorded release checksums must still match. A changed checksum is `checksum_conflict` ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L95-L120](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L95-L120)).
- **Source selection by journal.** A journal selects its recorded source chain. Two source chains that both claim the same journal are `recovery_required` ([repo://src/powercontext/builtin/persistence/migrations/sources.py#L36-L53](../../src/powercontext/builtin/persistence/migrations/sources.py#L36-L53)).

Startup uses the same classification but never migrates managed storage. See [Startup and readiness](#startup-and-readiness) below.

## Applying a schema migration

Run the sequence below for every database. Stop all API hosts, Workers, SDK clients and automatic restarts before `apply`. The command cannot discover external writers.

```bash
powercontext server db-migrate plan --env-file .env --backup auto
powercontext server db-migrate apply --env-file .env --plan-id PLAN_ID --backup auto --maintenance-confirmed --yes
powercontext server db-migrate verify --env-file .env
```

`apply` plans first. A `ready` result returns the verify result with no changes. Otherwise it checks declarations and backup policy, then runs the locked sequence:

```mermaid
flowchart TD
    P["Accepted plan with plan_id"] --> Q{"Declarations valid?"}
    Q -- "No" --> R["Blocked: confirmation or backup error"]
    Q -- "Yes" --> S["Acquire target lock"]
    S -- "Held by another process" --> T["migration_locked"]
    S -- "Acquired" --> U["Re-plan and compare with accepted plan"]
    U -- "Different" --> V["plan_changed"]
    U -- "Same" --> W["Create recovery point and journal"]
    W --> X["Historical conversion steps if required"]
    X --> Y["Apply pending revisions one at a time"]
    Y --> Z["Verify schema, data and domain"]
    Z --> AA["Journal marked complete"]
```

Caption: the locked apply order shared by the SQLite and MySQL-family runners. The recovery point is taken before any structural change.

Key rules:

- **Consent.** `apply` requires an explicit `--plan-id` matching the reviewed plan, `--maintenance-confirmed`, and a backup declaration. `--backup manual` requires `--backup-confirmed`. `--backup skip` requires `--accept-no-backup`. Automation requires both `--backup` and `--plan-id` ([repo://src/powercontext/server/database_migration.py#L147-L242](../../src/powercontext/server/database_migration.py#L147-L242), [repo://src/powercontext/server/database_migration.py#L396-L411](../../src/powercontext/server/database_migration.py#L396-L411)). Automatic backup that is unavailable blocks the command with `backup_unsupported` unless the operator chooses another policy. The SQLite and MySQL-family runners apply the same checks ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L318-L344](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L318-L344), [repo://src/powercontext/builtin/persistence/migrations/mysql.py#L402-L427](../../src/powercontext/builtin/persistence/migrations/mysql.py#L402-L427)).
- **Plan binding.** The plan is recomputed after the lock is acquired and again before the write transaction. Any difference raises `plan_changed` ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L241-L275](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L241-L275), [repo://src/powercontext/builtin/persistence/migrations/mysql.py#L430-L444](../../src/powercontext/builtin/persistence/migrations/mysql.py#L430-L444)).
- **Recovery point.** SQLite uses the Online Backup API, verifies integrity and creates the manifest exclusively. A partial file is kept as evidence and never reported as complete. `create_backup` refuses unless writers are declared stopped ([repo://src/powercontext/builtin/persistence/migrations/backup.py#L131-L200](../../src/powercontext/builtin/persistence/migrations/backup.py#L131-L200)).
- **Completion.** The journal is marked `complete` only after schema, data and domain validation succeed ([repo://src/powercontext/builtin/persistence/migrations/mysql.py#L470-L526](../../src/powercontext/builtin/persistence/migrations/mysql.py#L470-L526), [repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L296-L316](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L296-L316)).

### Journal and interrupted runs

The maintenance journal is `maintenance.json` in `<database>.pc-migration-state`. It records the state (`preparing`, `active` or `complete`), the database identity, the source revision, the bundle and configuration digests, the backup policy and reference, and the release checksums ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L34-L108](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L34-L108)).

Retry rules:

- Repeat the same bundle and configuration. A changed bundle, configuration digest or historical conversion definition raises `recovery_required` or `plan_changed` instead of resuming ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L444-L490](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L444-L490)).
- A partially migrated database does not get a new automatic backup. Switching to `auto` after partial progress raises `recovery_required` ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L612-L620](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L612-L620)).
- A missing journal for a database with pending revisions is `recovery_required`. Do not back up partial state in that case ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L384-L443](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L384-L443)).
- Keep the original backup. A backup-required error means the original recovery point is unavailable and must not be replaced after schema changes.

The 1.3.0 upgrade has a precondition the code cannot verify. Stop all writers and drain queued or running 1.3.0 Dream tasks before upgrading, because the target cannot execute the old prompt contract ([repo://docs/en/docs/operate/database-migrations.md#L73-L73](../../docs/en/docs/operate/database-migrations.md#L73-L73)).

## Locks and admission failures

Each backend uses a different lock. None of these is a cross-host lock for the whole database.

| Backend | Lock | Failure codes and behavior |
| --- | --- | --- |
| SQLite | Non-blocking OS file lock on `<database>.pc-migration.lock`. The lock file is kept after release so waiters and newcomers share one inode. | Contention raises `migration_locked`. A platform without an OS-backed lock raises `migration_lock_unsupported` ([repo://src/powercontext/builtin/persistence/migrations/locking.py#L31-L62](../../src/powercontext/builtin/persistence/migrations/locking.py#L31-L62)). |
| SQLite startup | Read-only probe of an existing lock file with a shared non-blocking lock. The probe never creates the lock file. | An active maintenance lock refuses startup as `migration_locked`. Unsupported inspection raises `migration_lock_unsupported` ([repo://src/powercontext/builtin/persistence/migrations/locking.py#L64-L96](../../src/powercontext/builtin/persistence/migrations/locking.py#L64-L96), [repo://src/powercontext/builtin/persistence/migrations/server_readiness.py#L123-L134](../../src/powercontext/builtin/persistence/migrations/server_readiness.py#L123-L134)). |
| Embedded seekDB | Local lock on the engine directory. A nonempty directory must contain recognized native storage. A missing directory is not created on a read-only path. | Unrecognized storage raises `unsupported_target` ([repo://src/powercontext/builtin/persistence/migrations/connections.py#L103-L145](../../src/powercontext/builtin/persistence/migrations/connections.py#L103-L145)). |
| OceanBase | Coordinator OS lock on one fixed host, plus a durable run receipt. | Requires `--lock-coordination single-host` and `--evidence-dir` ([repo://src/powercontext/builtin/persistence/migrations/deployment.py#L117-L232](../../src/powercontext/builtin/persistence/migrations/deployment.py#L117-L232)). |

OceanBase details:

- The coordinator lock is an OS lock, not a database-wide or cross-host lock. All migrators for the deployment must use the same host and evidence directory ([repo://src/powercontext/builtin/persistence/migrations/oceanbase.py#L1-L21](../../src/powercontext/builtin/persistence/migrations/oceanbase.py#L1-L21)).
- A lock loss or a changed lock file or directory raises `migration_lock_lost`. A symbolic link, a non-regular file, or a platform without an OS lock raises `migration_lock_unsupported` ([repo://src/powercontext/builtin/persistence/migrations/oceanbase.py#L49-L120](../../src/powercontext/builtin/persistence/migrations/oceanbase.py#L49-L120)).
- Every started run writes a receipt. A clean close clears it. An unclean exit keeps it. The next attempt raises `recovery_required` until the operator passes the exact token with `--acknowledge-previous-run`. An acknowledgment with no pending run raises `stale_ack` ([repo://src/powercontext/builtin/persistence/migrations/oceanbase.py#L316-L336](../../src/powercontext/builtin/persistence/migrations/oceanbase.py#L316-L336), [repo://src/powercontext/builtin/persistence/migrations/mysql.py#L354-L385](../../src/powercontext/builtin/persistence/migrations/mysql.py#L354-L385)).
- `--acknowledge-previous-run` is accepted only for OceanBase; other backends raise `unsupported_option` ([repo://src/powercontext/builtin/persistence/migrations/deployment.py#L117-L232](../../src/powercontext/builtin/persistence/migrations/deployment.py#L117-L232)). The acknowledgment asserts that the previous remote operation and background DDL have ended. The coordinator cannot verify that.

Other frequent blocking codes:

| Code | Meaning and action |
| --- | --- |
| `confirmation_required` | Plan ID or backup declaration missing. Review a plan and pass the explicit flags. |
| `maintenance_required` | `--maintenance-confirmed` missing. Stop all writers first. |
| `plan_changed` | Database, bundle, configuration or options changed after review. Create a new plan. |
| `backup_unsupported` | Automatic backup unavailable. Choose `manual` or `skip` and review a new plan. |
| `migration_required` | Startup found pending revisions. Run `db-migrate apply`. |
| `embedding_profile_mismatch` | Configured embedding profile differs from stored managed storage, or the profile is incomplete. |
| `unsupported_upgrade_configuration` | A partial bundle was asked to handle a vector configuration, or a managed database has no registered transition. |
| `unknown_baseline`, `server_migration_unavailable`, `incompatible_schema` | Schema not registered for upgrade. No changes were made. |

Embedding profiles are admitted strictly. A configured model requires a profile ID and a dimension ([repo://src/powercontext/builtin/persistence/migrations/deployment.py#L87-L101](../../src/powercontext/builtin/persistence/migrations/deployment.py#L87-L101)). The snapshot admission path compares the stored profile with the configured one ([repo://src/powercontext/builtin/persistence/migrations/sqlite.py#L209-L239](../../src/powercontext/builtin/persistence/migrations/sqlite.py#L209-L239)). Partial bundles reject vector configuration before a plan or backup is created ([repo://src/powercontext/builtin/persistence/migrations/deployment.py#L117-L140](../../src/powercontext/builtin/persistence/migrations/deployment.py#L117-L140)).

## Startup and readiness

The Server calls `prepare_server_storage` before serving ([repo://src/powercontext/server/factory.py#L221-L230](../../src/powercontext/server/factory.py#L221-L230)). That function never migrates managed storage:

- Empty local persistent SQLite or seekDB storage is initialized through the registered chain. A non-ready plan raises an error that tells the operator to stop writers and run `powercontext server db-migrate apply` ([repo://src/powercontext/builtin/persistence/migrations/server_readiness.py#L123-L163](../../src/powercontext/builtin/persistence/migrations/server_readiness.py#L123-L163)).
- OceanBase requires an explicit `db-migrate` job for an uninitialized remote database. Startup then validates the exact layout, the stored profile and the domain state ([repo://src/powercontext/builtin/persistence/migrations/server_readiness.py#L139-L152](../../src/powercontext/builtin/persistence/migrations/server_readiness.py#L139-L152)).
- A full-Server check requires exactly one revision at the bundle head. It verifies retained objects, runs a live foreign-key check, and validates the domain ([repo://src/powercontext/builtin/persistence/migrations/server_readiness.py#L26-L48](../../src/powercontext/builtin/persistence/migrations/server_readiness.py#L26-L48), [repo://src/powercontext/builtin/persistence/migrations/server_readiness.py#L50-L91](../../src/powercontext/builtin/persistence/migrations/server_readiness.py#L50-L91)).
- An unfinished portable restore receipt blocks normal startup and workers. Only the explicit restore path may open it ([repo://src/powercontext/builtin/persistence/migrations/server_readiness.py#L35-L47](../../src/powercontext/builtin/persistence/migrations/server_readiness.py#L35-L47)).

The managed Ops snapshot admission (`powercontext`'s private admission entry) validates a private copy of the main file and WAL. It is read-only. Its errors return only safe reason codes and never database contents or configuration ([repo://src/powercontext/server/database_admission.py#L79-L143](../../src/powercontext/server/database_admission.py#L79-L143)).

## Processing migration

`processing-migrate` upgrades existing Topic Memory processing state so that the Scope-based Artifact Processing Supervisor can start. The command has two paths:

- **Managed database.** The command delegates to `db-migrate --bundle processing`. An identity marker without a revision is blocked until the full Server upgrade finishes ([repo://src/powercontext/server/cli.py#L132-L190](../../src/powercontext/server/cli.py#L132-L190), [repo://src/powercontext/server/cli.py#L233-L262](../../src/powercontext/server/cli.py#L233-L262)).
- **Unmanaged SDK storage.** The domain-only procedure runs. Its plan, apply and verify steps are described in [the Processing migration docs](../../docs/en/docs/operate/artifact-processing-migration.md).

Startup bootstrap is conservative. A fresh database with no legacy data receives a `complete` marker. A database with the legacy processing schema, or existing processing data, raises `ProcessingSchemaNotReadyError` and must be migrated offline ([repo://src/powercontext/builtin/persistence/processing_migration.py#L154-L189](../../src/powercontext/builtin/persistence/processing_migration.py#L154-L189), [repo://src/powercontext/builtin/persistence/processing_migration.py#L213-L236](../../src/powercontext/builtin/persistence/processing_migration.py#L213-L236)).

Mode switches between global and dedicated processing need a new `--migration-id` each time. Keep the same ID only to resume the same switch ([repo://src/powercontext/server/database_migration.py#L67-L70](../../src/powercontext/server/database_migration.py#L67-L70)). The migration preserves Topic Pending values, Source Cursors, CAS generations, and request counters. Uncovered Sources become ordinary dirty state, and a Scope with unresolved legacy Pending is retained as an accepted invocation ([repo://docs/en/docs/operate/artifact-processing-migration.md](../../docs/en/docs/operate/artifact-processing-migration.md)).

## Atomic Memory migration

The conversion task `powercontext.memory.v1-to-atomic-memory.v1` converts legacy Memory collections into independent `atomic-memory` artifacts ([repo://src/powercontext/builtin/persistence/migrations/atomic_memory_v1.py#L100](../../src/powercontext/builtin/persistence/migrations/atomic_memory_v1.py#L100)). Its key rules:

- **Identity.** A new identity is a UUIDv5 of the canonical JSON `(scope_id, memory_artifact_id, entry_id)`, prefixed `mem-`. Entry IDs shared by different containers stay distinct ([repo://src/powercontext/builtin/persistence/atomic_memory_identity.py#L1-L30](../../src/powercontext/builtin/persistence/atomic_memory_identity.py#L1-L30)).
- **Blocking custom prompts.** A custom `memory.extract` Prompt blocks the task until it is explicitly converted to the Atomic contract ([repo://src/powercontext/builtin/persistence/migrations/atomic_memory_v1.py#L700-L722](../../src/powercontext/builtin/persistence/migrations/atomic_memory_v1.py#L700-L722)).
- **Archive, not delete.** Legacy collections are copied to `pc_memory_artifact_archive` ([repo://src/powercontext/builtin/persistence/migrations/atomic_memory_archive.py#L30-L40](../../src/powercontext/builtin/persistence/migrations/atomic_memory_archive.py#L30-L40)). They leave public tables, search and online reads. The archive has no online read surface, and the task does not downgrade the database.
- **Reference decisions.** Candidates that cite only whole collections need a decision file. Only `replace` is accepted, and its replacement evidence must be existing exact Atomic revisions, not a legacy collection ([repo://src/powercontext/builtin/persistence/migrations/atomic_memory_references.py#L174-L190](../../src/powercontext/builtin/persistence/migrations/atomic_memory_references.py#L174-L190)).
- **Owner decisions.** Ownerless legacy entries under enforced access need an explicit operator attestation through `--owner-decisions`. The CLI parses the decision file before any database work ([repo://src/powercontext/server/cli.py#L369-L449](../../src/powercontext/server/cli.py#L369-L449)). The decision format is validated in the conversion module ([repo://src/powercontext/builtin/persistence/migrations/atomic_memory_v1.py#L133-L165](../../src/powercontext/builtin/persistence/migrations/atomic_memory_v1.py#L133-L165)).

The CLI routes by storage ownership. On a managed database, it delegates to `db-migrate --bundle server` and rejects `--decisions` and `--owner-decisions`. On an unmanaged legacy database, it runs the domain-only conversion with those files ([repo://src/powercontext/server/cli.py#L369-L449](../../src/powercontext/server/cli.py#L369-L449)).

Atomic evidence for a revision is stored in a disposable projection, `pc_atomic_memory_evidence`. Invalid or missing declarations are kept as neutral evidence. Trust is never backfilled from the current registry ([repo://src/powercontext/builtin/persistence/atomic_memory_evidence.py#L58-L109](../../src/powercontext/builtin/persistence/atomic_memory_evidence.py#L58-L109), [repo://src/powercontext/builtin/persistence/atomic_memory_evidence.py#L118-L130](../../src/powercontext/builtin/persistence/atomic_memory_evidence.py#L118-L130)).

## Atomic projection and Topic recovery bundles

`--bundle atomic-projection` repairs derived current Atomic rows and the SQLite FTS helper on a current managed database. It preserves immutable history, lifecycle, tags, permissions and Source or Processing state. It runs only in the registered no-vector configuration and cannot register vector-profile changes ([repo://src/powercontext/builtin/persistence/migrations/deployment.py#L117-L140](../../src/powercontext/builtin/persistence/migrations/deployment.py#L117-L140)). Invalid authority stops the repair with `invalid_authority`. A failed post-repair check stops it with `projection_verification_failed` ([repo://src/powercontext/builtin/persistence/migrations/atomic_projection.py#L100-L118](../../src/powercontext/builtin/persistence/migrations/atomic_projection.py#L100-L118), [repo://src/powercontext/builtin/persistence/migrations/atomic_projection.py#L175-L185](../../src/powercontext/builtin/persistence/migrations/atomic_projection.py#L175-L185)).

`server atomic-memory-rebuild-projection` has two branches, selected by storage ownership:

- **Managed databases** delegate `plan`, `verify` and `apply` to `db-migrate --bundle atomic-projection`, and the managed bundle rules above apply.
- **Unmanaged databases** accept only `apply`. `plan` and `verify` fail with `plan and verify require managed Server storage`. `apply` also requires `--maintenance-confirmed` after writers are stopped; without it the command stops before any database work. It then runs the standalone `rebuild_atomic_memory_projection` through `_atomic_memory_maintenance`, passing the configured embedding model when one is present ([repo://src/powercontext/server/cli.py#L495-L578](../../src/powercontext/server/cli.py#L495-L578)).

The standalone rebuild is not the no-vector bundle. It accepts a configured embedding profile only when the supplied model's profile matches it, and it raises `maintenance-required` or `embedding-profile` before any work otherwise ([repo://src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L474-L484](../../src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L474-L484)).

Current-row selection covers every Atomic Memory head, not only active ones. The rebuild reads head identities in `(scope_id, artifact_id)` keyset batches ([repo://src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L118-L126](../../src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L118-L126)). For each identity it loads the state and the latest revision. An `active` state publishes a freshly prepared current row, and any other state removes that identity's current row. Each identity commits in its own transaction after the prepared content is rechecked against the authority, so an interrupted run can leave partial derived repair and should be repeated while maintenance remains in force ([repo://src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L458-L531](../../src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L458-L531)).

After the loop, one transaction deletes obsolete current rows. A current row is obsolete when no Atomic Memory head for it is `active` and unmerged, which also covers orphan rows with no head ([repo://src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L252-L263](../../src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L252-L263), [repo://src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L532-L553](../../src/powercontext/builtin/runtime/atomic_memory_rebuild.py#L532-L553)). The same transaction rebuilds the evidence projection and runs a verification that requires both evidence and evidence projection. The report's `ready` value sets the exit code. Its counts include `rebuilt_rows`, `removed_rows` and `embedding_calls`.

`--bundle topic-memory-recovery` covers only the registered Topic recovery layouts for unmanaged storage. Its output reports `server_ready=false`. Any database containing `pc_schema_revision` is rejected before backup or writes ([repo://src/powercontext/server/database_migration.py#L44-L48](../../src/powercontext/server/database_migration.py#L44-L48), [repo://src/powercontext/builtin/persistence/migrations/deployment.py#L117-L140](../../src/powercontext/builtin/persistence/migrations/deployment.py#L117-L140)).

## Portable archives

A `.pcb` bundle is a logical ZIP that carries complete scopes in portable form. It holds a `manifest.json` and a `records.ndjson` stream. Format version `1` is the only version writers produce ([repo://src/powercontext/builtin/portability/bundle.py#L127-L140](../../src/powercontext/builtin/portability/bundle.py#L127-L140), [repo://src/powercontext/builtin/portability/bundle.py#L1181-L1240](../../src/powercontext/builtin/portability/bundle.py#L1181-L1240)).

```mermaid
sequenceDiagram
    participant Op as Operator
    participant CLI as archive CLI
    participant Svc as PortableBundleService
    participant DB as Target database
    Op->>CLI: archive restore bundle dry-run
    CLI->>Svc: validate bundle
    Svc->>DB: read target identities only
    Svc-->>CLI: already present, conflicts, compatibility
    Op->>CLI: archive restore bundle yes
    CLI->>Svc: restore bundle
    Svc->>DB: stage verified records in temporary index
    Svc->>DB: one transaction writes records and receipt
    Svc->>DB: rebuild projections then mark receipt ready
```

Caption: restore validates first, stages records outside the target transaction, then writes authoritative rows and marks projections ready.

### Export

- **Authorization first.** The selected scope IDs are validated and authorized before any database query. For this offline command, the authority is the OS and database permission to read the configured deployment. It never uses Server bearer tokens ([repo://src/powercontext/builtin/portability/bundle.py#L526-L560](../../src/powercontext/builtin/portability/bundle.py#L526-L560), [repo://src/powercontext/cli/archive.py#L150-L152](../../src/powercontext/cli/archive.py#L150-L152)).
- **Consistent snapshot.** One consistent-snapshot transaction streams rows to a temporary NDJSON file. The archive is published with a replace only after its structure and dependencies pass. A failed export keeps any previous file at the destination ([repo://src/powercontext/builtin/portability/bundle.py#L560-L590](../../src/powercontext/builtin/portability/bundle.py#L560-L590)).
- **Destination guard.** On SQLite, export refuses a destination that is the source database or a file with its `-wal`, `-shm` or `-journal` names, case-insensitively ([repo://src/powercontext/builtin/portability/bundle.py#L1132-L1149](../../src/powercontext/builtin/portability/bundle.py#L1132-L1149)).

### Inspect and validate

`inspect` verifies the ZIP structure, the manifest, per-record digests, the record count, record classes and the scope list. It never opens the target database ([repo://src/powercontext/builtin/portability/bundle.py#L592-L595](../../src/powercontext/builtin/portability/bundle.py#L592-L595), [repo://src/powercontext/builtin/portability/bundle.py#L1181-L1240](../../src/powercontext/builtin/portability/bundle.py#L1181-L1240)).

`restore --dry-run` validates against the target without domain writes. It reports `already_present`, `conflicts`, required and unsupported Source types and Artifact families, and whether a projection rebuilder exists ([repo://src/powercontext/builtin/portability/bundle.py#L597-L646](../../src/powercontext/builtin/portability/bundle.py#L597-L646)). A missing SQLite target is validated without creating its file or parent directory ([repo://src/powercontext/cli/archive.py#L154-L178](../../src/powercontext/cli/archive.py#L154-L178)).

### Restore

- **Confirmation.** `restore` without `--dry-run` requires `--yes` ([repo://src/powercontext/cli/archive.py#L104-L122](../../src/powercontext/cli/archive.py#L104-L122)).
- **Pre-write conflicts.** Restore revalidates and aborts before any write when validation finds conflicting target records. It also aborts when the target is incompatible with the bundle's source types or artifact families ([repo://src/powercontext/builtin/portability/bundle.py#L681-L700](../../src/powercontext/builtin/portability/bundle.py#L681-L700)).
- **Staging and transaction.** Verified records are staged in a temporary disk index before the write transaction opens. Records replay in dependency order inside one transaction. An integrity error at write time becomes a conflict and rolls back ([repo://src/powercontext/builtin/portability/bundle.py#L1241-L1257](../../src/powercontext/builtin/portability/bundle.py#L1241-L1257), [repo://src/powercontext/builtin/portability/bundle.py#L895-L925](../../src/powercontext/builtin/portability/bundle.py#L895-L925)).
- **Idempotency.** Identical records count as `already_present`. A different payload at the same identity never overwrites the target ([repo://src/powercontext/builtin/portability/bundle.py#L895-L925](../../src/powercontext/builtin/portability/bundle.py#L895-L925)).
- **Receipt and projections.** A durable receipt keyed by `bundle_id` moves from `authoritative_restored` to `ready` only after projection rebuild completes. If rebuild fails, rerun the same restore. `projections_ready=false` keeps the target out of normal serving until recovery ([repo://src/powercontext/builtin/portability/bundle.py#L728-L775](../../src/powercontext/builtin/portability/bundle.py#L728-L775)).

### What is and is not portable

The bundle carries Scope identity and hierarchy, Source journal heads and Source records, referenced Source definition manifests, accepted evidence bindings, Artifact revisions with their lineage and heads, Atomic Memory merge relationships, Candidate versions and decision heads, managed Skill packages and manifests, Profile policies and Prompt revisions, and Work and Handoff records stored as Sources ([repo://src/powercontext/builtin/portability/bundle.py#L154-L170](../../src/powercontext/builtin/portability/bundle.py#L154-L170)).

The following classes are excluded: access bindings, audit records, credentials and provider secrets, evaluation receipts, external Skill registrations, host-local installation state, search projections and indexes, Source cursors and scheduler state. Archives are not encrypted. Checksums detect corruption, not authenticity ([repo://docs/en/docs/operate/portable-archive.md](../../docs/en/docs/operate/portable-archive.md)).

### Limits and exit codes

The archive file is limited to 512 MiB. The manifest is limited to 256 KiB. Records are limited to 2 GiB uncompressed, 2,000,000 records, and 8 MiB per NDJSON line ([repo://src/powercontext/builtin/portability/bundle.py#L127-L140](../../src/powercontext/builtin/portability/bundle.py#L127-L140)). Bundle IDs must be canonical lowercase UUIDs ([repo://docs/en/docs/operate/portable-archive.md](../../docs/en/docs/operate/portable-archive.md)). A manifest with an unknown `format_version` is rejected before any target write ([repo://src/powercontext/builtin/portability/bundle.py#L1341-L1360](../../src/powercontext/builtin/portability/bundle.py#L1341-L1360)).

Exit codes:

- `2`: invalid or unreadable bundle, or an uninspectable SQLite target filename.
- `3`: incompatible target, conflict, or pending recovery.

These are raised by the archive CLI's error-to-exit mapping ([repo://src/powercontext/cli/archive.py#L230-L268](../../src/powercontext/cli/archive.py#L230-L268)).

## Upgrade release gate

`scripts/upgrade_release_gate.py` validates the policy, runs every required backend case against the target wheel, and writes a `result.json` into a fresh output directory. Each case failure is recorded, and the run fails if any required backend fails ([repo://scripts/upgrade_release_gate.py#L521-L558](../../scripts/upgrade_release_gate.py#L521-L558)). Local regression tests do not replace this run.

Policy validation:

- The policy `target_version` must match the OpenAPI release ([repo://scripts/upgrade_release_gate.py#L100-L125](../../scripts/upgrade_release_gate.py#L100-L125)).
- The 1.3.0 baseline must cover SQLite, seekDB and OceanBase. Every acceptance suite must live under `tests/release_upgrades` ([repo://scripts/upgrade_release_gate.py#L100-L170](../../scripts/upgrade_release_gate.py#L100-L170)).
- The automatic 1.3.0 matrix lists twelve configurations: no-vector, Code, vector and vector-plus-Code on each backend ([repo://release/upgrade-support.json#L10-L76](../../release/upgrade-support.json#L10-L76)).
- A persistence-input change requires a refreshed compatibility review. The digest covers every managed persistence file, with CRLF normalized to LF ([repo://scripts/upgrade_release_gate.py#L82-L98](../../scripts/upgrade_release_gate.py#L82-L98)).

Acceptance evidence:

- The JUnit report must contain every named full-Server phase, including `test_legacy_startup_is_rejected` and `test_fresh_upgrade_equivalence`. Any failure or skipped test rejects acceptance ([repo://scripts/upgrade_release_gate.py#L49-L58](../../scripts/upgrade_release_gate.py#L49-L58), [repo://scripts/upgrade_release_gate.py#L207-L214](../../scripts/upgrade_release_gate.py#L207-L214)).
- OceanBase shard reports must have matching collections and disjoint, exactly-once coverage. A missing or duplicate node fails verification ([repo://scripts/check_atomic_memory_migration_shards.py#L28-L76](../../scripts/check_atomic_memory_migration_shards.py#L28-L76), [repo://scripts/check_atomic_memory_migration_shards.py#L78-L86](../../scripts/check_atomic_memory_migration_shards.py#L78-L86)).
- Publication requires `mode` to be `supported-upgrades`. In `upgrade-required` mode, a release run fails even when every case passes ([repo://scripts/upgrade_release_gate.py#L521-L558](../../scripts/upgrade_release_gate.py#L521-L558)).

## Operator checklist

1. Confirm the target and source. Read the `source_migration` and `target_migration` fields in the plan ([repo://src/powercontext/server/database_migration.py#L280-L294](../../src/powercontext/server/database_migration.py#L280-L294)).
2. Stop every writer, Worker, SDK client and automatic restart. Pause input and manual writes.
3. Take an independent backup, or accept the automatic recovery point. Do not replace an original backup after schema changes.
4. Run `plan`, review the output, then `apply` with the `--plan-id` and an explicit backup policy.
5. Run `verify` and confirm `ready` with `server_ready=true` before restarting writers.
6. For OceanBase, use one fixed maintenance host and one evidence directory for every migrator. Acknowledge any pending run only after confirming the previous remote operation ended.
7. For archives, run `restore --dry-run` first, then `restore --yes`. Confirm `projections_ready` is `true` before serving.
8. Keep the previous binary until a restore drill succeeds across a major release.

## Related pages

- [Persistence and storage](../architecture/persistence-and-storage.md) describes the storage backends and profiles this runbook operates on.
- [Troubleshooting](troubleshooting.md) covers common startup and maintenance failures.
- The Atomic Memory model that the conversion targets is described in the source (`src/powercontext/builtin/persistence/migrations/atomic_memory_v1.py`); no dedicated wiki page exists yet.
