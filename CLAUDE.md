# SmartSolar MPPT Home Assistant Integration

> **System info**: [../System_info/CLAUDE.md](../System_info/CLAUDE.md) — HA at 192.168.10.15, network, credentials
> **Code search**: `semble search "query" .` — intent-based, ~98% fewer tokens than grep

Home Assistant custom integration for SmartSolar MPPT solar charge controllers. Fetches real-time metrics via HTTP API from `api.smartsolar.io.vn` and MQTT WebSocket Secure from `mqttx.smartsolar.io.vn:8084`. **Current version: v2.0.7**, verified live against HA **2026.10.1** on 2026-10-12 (the test extra pins that same version); every release is deployed to the reference installation and checked with `verify_live.py` right after its tag.

> ⚠️ **v2.0.0 renamed the domain from `smartsolar_mppt` to `smartsolar_ha`.**
> Home Assistant identifies an integration by its folder name, which must equal
> `manifest["domain"]` (`loader.Integration.resolve_from_root()` looks for
> `custom_components/<domain>/manifest.json`), and the domain may only contain
> lowercase letters and underscores — so this is a breaking change: existing
> config entries must be re-added. Entity names were left untouched, so entity
> IDs, dashboards and long-term statistics survive. `smartsolar_mppt` strings
> still appearing in this file are historical notes or entity-name-derived IDs.

## Project Structure

```
custom_components/smartsolar_ha/
├── __init__.py          # Integration entry point, setup/unload, service registration, async_migrate_entry
├── manifest.json        # v2.0.7, domain=smartsolar_ha, config_flow=true
├── const.py             # Constants, SENSOR_TYPES, AGGREGATION, MQTT config, build_device_info helper
├── helpers.py           # Pure helpers: coerce_float, device_logs, as_list, stream_dict, guid_sort_key
├── config_flow.py       # Multi-step config flow: auth → mode → device/project, reauth, reconfigure
├── api.py               # HTTP API client: login, token refresh, real retry/backoff, get_device_status
├── mqtt_client.py       # MQTT client: WSS connect, subscribe, payload parsing, auto-reconnect
├── coordinator.py       # DataUpdateCoordinator: polls API + merges MQTT real-time data + daily stats
├── sensor.py            # Sensor entities (device, project synthesis, project device, daily stats)
├── number.py            # Update-interval entity (RestoreEntity, 1-30s)
├── diagnostics.py       # Config entry diagnostics with secret redaction
├── brand/               # icon.png (256), icon@2x.png + logo.png (512) — HA serves these locally
├── services.yaml        # Service definitions
├── strings.json         # UI strings (English)
└── translations/        # en.json, vi.json
```

> ℹ️ **Brand images:** Home Assistant serves a custom integration's own logo from
> `custom_components/<domain>/brand/` (`loader.Integration.has_branding` checks
> that FOLDER). There is no `brand` key in the manifest schema — HA ignores one,
> so a manifest `brand` block only ever looks like branding that is configured.
> Those three PNGs are the only copies in the repository.
>
> ℹ️ **The integration folder ships only what Home Assistant loads** (plus
> `brand/`). It no longer contains an inner `.gitignore` or an
> `example_configuration.yaml`: `deploy_to_ha.py` uploads every file in that
> folder, so both were being copied to the host for nothing, and the example YAML
> described a `configuration.yaml` platform that does not exist (the integration
> is config-flow only). The unused root `logo.png` was dropped at the same time,
> and `.gitignore` lost the dead rules that had accumulated from the parent
> repository (for files that do not exist here, plus a `logo.png` rule whose
> `!brand/logo.png` exception is now unnecessary).

Root-level files:
```
hacs.json                # HACS metadata (content_in_root=false, min HA 2026.9.0)
pyproject.toml           # Python project config: ruff, mypy, pytest, [test] extra with homeassistant
.pre-commit-config.yaml  # Pre-commit hooks: ruff, yaml/json checks
LICENSE                  # MIT License
upload_to_ha.py          # Minimal uploader: paramiko SSH + base64 + sudo tee
deploy_to_ha.py          # Full deploy gate: lint+types+tests → backup → upload (LF) → sha256 verify → clean restart → wait
verify_live.py           # Asserts the deployed entities are correct on the live instance
tests/
├── conftest.py           # Shared fixtures: mock HA, recorded API responses (deep-copied), coordinator
├── test_api.py           # API client, error hierarchy, retry/backoff, expiry parsing, token invalidation
├── test_sensor.py        # Value extraction, naming, per-sensor aggregation, non-finite rejection
├── test_helpers.py       # coerce_float / as_list / device_logs / stream_dict / guid_sort_key
├── test_mqtt.py          # MQTT payload parsing (both firmware formats), field mapping, reconnect loop
├── test_number.py        # Interval get/set, bounds, RestoreEntity behaviour
├── test_config_flow.py   # Every config-flow step, error mapping, translations
├── test_coordinator.py   # Polling, GUID ordering, daily stats, malformed payloads, MQTT robustness
├── test_diagnostics.py   # Diagnostics content + secret redaction
├── test_const.py         # Sensor metadata invariants
├── test_packaging.py     # manifest.json / hacs.json / brand-asset invariants
├── test_verify_live.py   # verify_live.py payload handling and exit codes
├── test_deploy_script.py # deploy_to_ha.py prune/archive/wait/restart/sha256 + local-gate == CI
└── test_e2e.py           # Real async_setup_entry/unload_entry against a real HA core
.github/workflows/
├── ci.yml                # ruff check + format, mypy (hard gate), pytest with coverage
├── hacs-validation.yml   # HACS validation on push/PR
└── release.yml           # GitHub release on tag push (notes generated by GitHub)
```

> ℹ️ These workflows run directly now that the project is its own repository.
> Before the split (2026-09-27) the project lived in `smartsolar_mppt/` inside the
> private HA-Config repo, where GitHub ignored them because it only reads
> workflows from the repository root — a root shim re-ran the same gate, and the
> two copies drifted. That shim is gone.

## Architecture & Data Flow

→ **Đã tách:** [docs/architecture.md](docs/architecture.md) — luồng dữ liệu, thành phần.

## Lịch sử phiên bản

→ **Đã tách:** [docs/CHANGELOG.md](docs/CHANGELOG.md) — v1.5.1 → v2.0.7 (audit, rename, deploy, MQTT).

## Development Guidelines

### Running Tests

```bash
py -3.14 -m venv .venv                       # Home Assistant 2026.x needs Python >= 3.14.2
.venv/Scripts/python -m pip install -e ".[test,dev,deploy]"
.venv/Scripts/python -m pytest tests/ --cov=custom_components/
.venv/Scripts/ruff check custom_components/ tests/ upload_to_ha.py deploy_to_ha.py verify_live.py
.venv/Scripts/ruff format --check custom_components/ tests/ upload_to_ha.py deploy_to_ha.py verify_live.py
.venv/Scripts/python -m mypy custom_components/
```

> The `deploy` extra (`paramiko`) is only needed by the SSH scripts. It is a
> separate extra because CI and the test suite never touch it — but a *fresh*
> venv installed with only `.[test,dev]` makes `python deploy_to_ha.py` die with
> `ModuleNotFoundError: No module named 'paramiko'`, which is how that was found.
> `tests/test_packaging.py` asserts the extra is declared.

> The ruff file list is duplicated in `deploy_to_ha.py:SOURCES` and
> `.github/workflows/ci.yml`; keep the three in sync (the local gate used to be
> weaker than CI).

`tests/test_e2e.py` runs the real `async_setup_entry` / `async_unload_entry`
against a real `HomeAssistant` core instance. It deliberately does **not** use
`pytest-homeassistant-custom-component`: that package imports the POSIX-only
`fcntl` module and cannot load on Windows. `tests/conftest.py` does not need the
HA test harness either.

### HA Connection
- URL: `http://192.168.10.15:8123`
- SSH: `vokupt@192.168.10.15` — password via `HA_PASS` env var
- Long-lived token available

### Code Style Targets
- Python **3.14.2+** (required by Home Assistant 2026.x), HA **2026.9+**
- Dependencies: aiohttp >= 3.8.0, aiomqtt >= 2.0
- Use `__slots__` for memory efficiency
- Use `CoordinatorEntity` with `RestoreEntity` for all entities
- `always_update=False` with value-change check
- Shared helpers in `helpers.py` (pure) and `const.py` (metadata)
- Pass `config_entry=` to `DataUpdateCoordinator` — HA 2026 raises a usage
  report when a coordinator relies on the ContextVar

### Key Naming Conventions
- Entity ID: derived by Home Assistant from the device name + entity name (e.g. `sensor.<device_slug>_pv1_pv_voltage`), **not** from the unique ID
- Unique ID: `{entry_id}_{prefix}_{sensor_type}`
- Config keys: `username`, `password`, `mode`, `device_type`, `chipset_ids`, `project_id`, `update_interval`
- PV1/PV2 order = GUIDs sorted **numerically** (`547611` → PV1, `14756976` → PV2)

### Deploying to HA

`deploy_to_ha.py` is the supported path. It refuses to ship unless lint, format,
mypy and the full test suite pass, archives what is currently deployed to
`_local_archive/deployed/`, uploads, clears `__pycache__`, restarts the container,
waits for the REST API to answer **and then for this integration's config entry to
report `loaded`**, so `verify_live.py` can be chained directly to it:

```bash
cd d:/code/smartsolar_ha
.venv/Scripts/python deploy_to_ha.py               # full gated deploy
.venv/Scripts/python deploy_to_ha.py --skip-checks # emergency re-push
.venv/Scripts/python verify_live.py                # assert the live entities are right
```

> ⚠️ **Why the extra wait:** Home Assistant answers on `/api/config` as soon as
> its HTTP component is up, *before* the integration platforms are added. The
> deploy used to return at that point, so a verification run chained immediately
> after it found **zero** entities and reported every check as missing — observed
> live on 2026-09-28 (0 entities at once, 38 five seconds later). `restart_ha()`
> now also polls `/api/config/config_entries/entry` until the folder-named domain
> reports `loaded`, and refuses the deploy when the entry stays in
> `setup_retry`/`setup_error`. The outcomes are kept apart: a listing that was
> *read* but does not mention the domain is retried up to
> `HA_MISSING_ENTRY_ATTEMPTS` times, then reported and the deploy continues (a
> fresh install, or a domain rename, has nothing to verify); a listing that could
> **not** be read (HTTP error, timeout, non-list body) is neither absence nor
> failure, so it keeps waiting and then fails the deploy — treating it as absence
> would report success without ever having seen `loaded`.
>
> ℹ️ **A `loaded` entry is still not the last word.** Home Assistant writes entity
> states asynchronously *after* the entry loads, so a verifier chained right after
> the deploy read a half-filled state machine: 12 of the 38 entities at the moment
> `loaded` was printed, the rest appearing within ~10-20 s (measured 2026-09-28).
> `verify_live.py` therefore waits — bounded to `SETTLE_ATTEMPTS` ×
> `SETTLE_INTERVAL` (12 × 5 s) — until every entity it asserts on exists, then
> reports the ones that never appear. The chain is verified live: the deploy
> printed `smartsolar_ha is loaded` and the immediately chained verification
> exited 0.
>
> ⚠️ **The host's `find` is BusyBox's.** `prune_remote()` used to ask for
> `find … -printf '%P\n'`, which BusyBox rejects; the error was swallowed by
> `|| true`, so the empty listing was read as "the host matches the repository"
> and **pruning never removed anything on a real host** — stale files (an inner
> `.gitignore`, `example_configuration.yaml`, modules deleted long ago)
> accumulated silently while the deploy printed `0 stale file(s) removed`. It now
> runs plain `find <dir> -type f`, trims the absolute prefix locally, and treats
> an empty listing as an error instead of a clean result.

> ⚠️ **Changing the domain breaks the deployed config entry.** Home Assistant
> resolves an entry by its `domain`, so an entry stored for `smartsolar_mppt`
> cannot load once the integration is `smartsolar_ha` — it logs
> `Integration 'smartsolar_mppt' not found` and every entity goes away.
> `deploy_to_ha.py` detects a leftover `smartsolar_mppt/` folder and warns about
> it, but it does **not** migrate the entry.

> ✅ **Migrate the entry instead of re-adding it** (validated live on 2026-09-28
> for `smartsolar_mppt` → `smartsolar_ha`, 37 entities, no entity_id or
> statistics loss):
>
> 1. With HA **running**, back up `.storage` on the host:
>    `sudo cp /homeassistant/.storage/core.{config_entries,entity_registry,device_registry} /somewhere/`
>    and tar the currently deployed integration folder.
> 2. `sudo docker stop homeassistant` — HA rewrites `.storage` on shutdown, so
>    editing while it runs is silently discarded.
> 3. In `.storage`, rewrite the domain in three places, then write the files back
>    as **UTF-8 without BOM**:
>    * `core.config_entries` → the entry's `"domain"`
>    * `core.entity_registry` → every entity's `"platform"` (one per sensor)
>    * `core.device_registry` → the device's `"identifiers"` pair
> 4. Upload the new tree, delete the old `custom_components/<old_domain>/`, then
>    `sudo docker start homeassistant`.
>
> Because the **entry_id is unchanged**, every entity's unique_id is unchanged
> too, so HA re-binds the same registry rows and the same entity_ids — dashboards
> and long-term statistics keep working and no credential is re-entered. Deleting
> the entry and re-adding it instead produces a new entry_id, hence new
> unique_ids, hence `_2`-suffixed entity_ids and orphaned statistics.
>
> Notes from doing it for real: HAOS's supervisor can restart the container on
> its own after a `docker stop`, so confirm the folder/`.storage` state and do a
> final clean restart before trusting the result; and `configuration.yaml`'s
> `logger.logs` still named the old domain, which silently hid the new
> integration's INFO lines until it was updated (this can be applied without a
> restart via the `logger.set_level` service).

> `clear_entities.ps1` is the opposite tool (it *deletes* both domains' entities
> and devices from the registries, for a clean re-add); run it only with HA
> **stopped**, since it refuses to run while HA answers on its API.

> `verify_live.py` needs `SMARTSOLAR_USER` / `SMARTSOLAR_PASS` in `.env` for the
> per-device WiFi check: it asks the cloud which devices report a `signalQuality`,
> because 'unknown' is only correct for a device whose payload omits it. Without
> those two keys that check reports `SKIP` instead of guessing.

`upload_to_ha.py` remains as a dependency-light uploader (no test gate).

HA runs the config-entry `data` from disk, so changing `update_interval` in the
number entity persists through `config_entries.async_update_entry`; the value is
re-read on the next reload.
