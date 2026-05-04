# SPEC

## §G GOAL
async Python lib + CLI ! control Jandy iAqualink pool systems (iAqua & eXO) over vendor HTTPS APIs.

## §C CONSTRAINTS
- lang: Python ≥ 3.14. modern hints (`Self`, `type[T]`, PEP 604).
- async only. all I/O via `httpx.AsyncClient` (HTTP/2 enabled).
- transport retry: `httpx-retries` ! handles 429 + `Retry-After` + exp backoff.
- license BSD-3-Clause. build = hatchling + hatch-vcs.
- lint = ruff (line=80). format = ruff-format. type-check = mypy on `src/` only AND `ty` (astral) on `src/` only — both must be clean.
- tests = `unittest.IsolatedAsyncioTestCase` + `respx` mocks. pytest filter `error` (DeprecationWarning ignored).
- pre-commit ! pass before merge.
- CLI deps optional (`[cli]` extra: PyYAML, typer).

## §I INTERFACES
external surface. world sees:

**Python API** (`import iaqualink`):
- `AqualinkClient(username, password, *, http_client?, auth_state?)` → async ctx mgr.
  - `await client.login()` / `await client.get_systems()` → `dict[str, AqualinkSystem]`.
  - prop `auth_state: AqualinkAuthState | None` (get/set ! restore session).
- `AqualinkSystem` base. concrete: `IaquaSystem(NAME="iaqua")`, `ExoSystem(NAME="exo")`, `CyclonextSystem(NAME="cyclonext")`, `VrSystem(NAME="vr")`, `VortraxSystem(NAME="vortrax")` (subclass of vr), `CyclobatSystem(NAME="cyclobat")`, `I2dRobotSystem(NAME="i2d_robot")`.
  - `await sys.update()` / `sys.get_devices()` / `sys.online: bool | None`.
  - factory: `AqualinkSystem.from_data(client, data) → subclass per data.device_type`.
- `AqualinkDevice` hierarchy: `AqualinkSensor → AqualinkBinarySensor → AqualinkSwitch → {AqualinkLight, AqualinkThermostat}`.
- exceptions: `AqualinkException` → `{InvalidParameter, ServiceException → {Unauthorized, Throttled}, SystemOffline, SystemUnsupported, OperationNotSupported, DeviceNotSupported}`.

**CLI** (`iaqualink` script, entry = `iaqualink.cli:main`):
- top-level: `list-systems`, `list-devices`, `status`, `turn-on <device>`, `turn-off <device>`, `set-temperature <device> <value>`.
- robot sub-app `iaqualink robot ...` (cyclonext-only unless noted; sub-commands fail-fast on non-cyclonext systems):
  - `start [--cycle floor|floor-wall]` — emits `mode=1`; optional cycle pre-set.
  - `stop` — single canonical exit (cycle / Remote / Lift); emits `mode=0`.
  - `pause` — emits `mode=2`.
  - `status` — refresh + render mode/cycle/remaining/model/total-run.
  - `extend <abs-min>` — set absolute `stepper` (multiple of 15, ≥0).
  - `adjust-time <±delta>` — relative stepper change (string arg, e.g. `+15` / `-15`).
  - `set-cycle <floor|floor-wall>` — write `cycle` field.
  - `remote <forward|backward|left|right|stop>` — emits `mode=2 + direction`.
  - `lift <eject|left|right|stop>` — emits `mode=3 + direction`.
- global flags: `--debug`, `--username`, `--password`, `--config`, `--cookie-jar`, `--system`.

**Upstream URLs** (verbatim, do not refactor):
- iAqua: `https://r-api.iaqualink.net/devices.json`, session base `https://p-api.iaqualink.net` (see `IAQUA_SESSION_URL` in `systems/iaqua/system.py`).
- eXO: login=`https://prod.zodiac-io.com/users/v1/login`, refresh=`https://prod.zodiac-io.com/users/v1/refresh`, devices=`https://prod.zodiac-io.com/devices/v1`.
- cyclonext (Zodiac robots, eg Voyager / Alpha series):
  - read: `GET https://prod.zodiac-io.com/devices/v1/{serial}/shadow` (IdToken header). `state.reported.dt == "cyc"`. robot dict @ `equipment.robot[1]` (idx 0 = null).
  - read fields: `mode`, `cycle`, `cycleStartTime`, `direction`, `errors.code`, `vr`, `equipmentId`, `totRunTime`, `durations.*`, `customCyc.{intensity,type}`, `schConf{0..6}*`, `eboxData.*`.
  - write: persistent WebSocket `wss://prod-socket.zodiac-io.com/devices` (Auth header = IdToken). frame:
    ```
    {"version":1,"action":"setCleanerState","namespace":"cyclonext","service":"StateController","target":"<serial>","payload":{"clientToken":"<userId>|<authToken>|<appClientId>","state":{"desired":{"equipment":{"robot.1":{"mode":<n>}}}}}}
    ```
  - mode: 0=stop, 1=start, 2=remote-control-active, 3=lift-system-active. note write key = `"robot.1"` (string, dotted) vs read array idx → adapter ! convert.
  - remote control (mode=2) — `equipment.robot.1.{mode:2,direction:N}`. direction: 0=stop, 1=forward, 2=backward, 3=rotate-right, 4=rotate-left. confirmed live 2026-04-27 RE.
  - lift system (mode=3) — `equipment.robot.1.{mode:3,direction:N}`. direction: 0=idle, 5=eject (climb wall), 6=rotate-left, 7=rotate-right. confirmed live 2026-04-27 RE.
  - exit remote / lift / cycle — single canonical command: send `mode:0`. matches vendor "Stop" button on cleaning cycle screen + screen-dismiss behaviour for Remote / Lift. ∴ CLI `iaqualink robot stop` covers all three.
  - cycle map (partial, observed): 1=floor_only, 3=floor_and_walls. quick/deep/smart values pending RE.
  - features (per OSS refs): START, STOP, FAN_SPEED, STATUS, TIMER, QUICK_CLEAN, PRESET_DEEP_ULTRA, MODE_INFO.
  - WS confirms via `StateReported` event from `StateStreamer`.
  - sources: `tekkamanendless/iaqualink` (Go), `galletn/iaqualink` (HA).
  - field map (vendor app "System Information" panel → API):
    - System Name → `devices.json.name`
    - System Time → `state.reported.aws.timestamp` (epoch ms)
    - First Connected On → `devices.json.created_at`
    - Device Number → `devices.json.serial_number` / shadow `sn` / `deviceId`
    - Firmware Control Box → `state.reported.vr`
    - Firmware Robot → `state.reported.equipment.robot[1].vr`
    - Robot S/N → `state.reported.eboxData.completeCleanerSn`
    - Control Box S/N → `state.reported.eboxData.controlBoxSn`
    - Total Run Time → `state.reported.equipment.robot[1].totRunTime` (UNIT=min, ÷60 → hr)
    - Model Number → ? not in shadow/devices.json. RE candidate endpoints: `r-api.iaqualink.net/devices/{serial}`, `prod.zodiac-io.com/devices/v1/{serial}` (no suffix).
    - App Version → client-side, n/a.
- iAqua API key (hardcoded): `AQUALINK_API_KEY = "EOOEMOW4YR6QNB07"`.

**Files / env**:
- config: `typer.get_app_dir("iaqualink")/config.yaml`. keys: `username`, `password`, `system?`.
- session jar: `typer.get_app_dir("iaqualink")/session.json`. JSON cookie jar. plain-text tokens (documented tradeoff). atomic write via temp file.
- env: `IAQUALINK_USERNAME`, `IAQUALINK_PASSWORD`.

**Constants** (`src/iaqualink/const.py`):
- `KEEPALIVE_EXPIRY=30`, `DEFAULT_REQUEST_TIMEOUT=10.0`.
- `RETRY_MAX_ATTEMPTS=5`, `RETRY_BASE_DELAY=1.0`, `RETRY_MAX_DELAY=30.0`, `RETRY_AFTER_MAX_DELAY=60.0`.

## §V INVARIANTS

V1: ∀ HTTP request → 10s default timeout (`DEFAULT_REQUEST_TIMEOUT`). ⊥ unbounded.
V2: ∀ 429 response → transport-level retry via `httpx-retries`. ! honor `Retry-After` header, capped @ `RETRY_AFTER_MAX_DELAY=60.0`.
V3: ∀ retry sequence → exp backoff. base=`RETRY_BASE_DELAY`, max=`RETRY_MAX_DELAY`, attempts ≤ `RETRY_MAX_ATTEMPTS=5`.
V4: ∀ `AqualinkSystem` subclass → `NAME: str` ClassVar set. registered in `AqualinkSystem.subclasses` via `__init_subclass__`. `from_data()` ! dispatch on `data["device_type"] == NAME`.
V5: ∀ new system module → import wired in `src/iaqualink/client.py` ∴ `from_data()` discovers @ runtime.
V6: ∀ `system.update()` → check `now - last_refresh ≥ MIN_SECS_TO_REFRESH` (iaqua=5s, exo=50s). skip otherwise.
V7: ∀ `system.update()` exception path → `AqualinkServiceThrottledException` re-raised BEFORE broader `AqualinkServiceException` handler. ⊥ `online = None` on throttle.
V8: ∀ auth-bearing request that raises `AqualinkServiceUnauthorizedException` → retry once via shared `reauth.send_with_reauth_retry()`. ⊥ inline replay of stale request inside `send_request()`.
V9: client systems-discovery & system update paths ! use shared reauth helper. iaqua + exo identical retry semantics.
V10: eXO auth → JWT `IdToken` from `userPoolOAuth` field, sent as `Authorization` header. auto-refresh on 401.
V11: iAqua auth → `session_id` + `authentication_token` in query params.
V12: `AqualinkClient` ! support `async with` (ctx mgr). `__aexit__` closes underlying httpx client.
V13: restored `auth_state` → `__aenter__` skips initial `login()`.
V14: CLI cookie jar restore → only when stored username == requested username. else discard.
V15: CLI systems-discovery → ≤ 1 full login retry on stale jar. on success ! rewrite jar atomically (temp file → rename).
V16: ∀ spec change touching upstream endpoint, field, auth flow → cross-check against `spec/` files (if relevant). divergence ! inline comment with reason.
V17: tests ! use `respx` for HTTP. ⊥ live network.
V18: mypy `src/` clean. `ty check src/` clean. ruff clean. pre-commit green. ! before merge.
V19: Python ≥ 3.14. ⊥ syntax requiring lower minimum.
V20: ∀ new system / device / cli module → mirror existing iaqualink-py idioms (see `systems/exo/system.py`, `systems/iaqua/system.py`):
  - `from __future__ import annotations` first.
  - `LOGGER = logging.getLogger("iaqualink")` (single shared logger).
  - `TYPE_CHECKING`-gated imports for `httpx`, `iaqualink.client.AqualinkClient`, `iaqualink.typing.{Payload, DeviceData}`.
  - `__repr__` over `(name, serial, data)` triple.
  - terse / no docstrings on internal methods. only "why" comments, never "what" (see CLAUDE.md).
  - no `print()`. no f-string log calls (V18). `%`-style logging.
  - exceptions: re-raise `AqualinkServiceThrottledException` BEFORE broader `AqualinkServiceException` (V7).
  - tests: `unittest.IsolatedAsyncioTestCase` + `MagicMock` / `respx` per area; mirror existing `tests/systems/exo/` layout.
  - new public symbols ! exported via package's existing pattern (no `__all__` unless module already declared one).

## §T TASKS
id|status|task|cites
T1|x|`AqualinkClient` core + httpx HTTP/2 transport|V1,V2,V3
T2|x|`AqualinkSystem` registry + `from_data` factory|V4,V5
T3|x|iAqua system + devices|V6,V7,V11
T4|x|eXO system + devices|V6,V7,V10
T5|x|shared reauth helper (`reauth.py`)|V8,V9
T6|x|exception hierarchy|V7
T7|x|CLI (Typer) discovery + control commands|I.cli
T8|x|session jar persistence (atomic, JSON)|V13,V14,V15
T9|x|httpx-retries integration (V2/V3)|V2,V3
T10|x|default request timeout 10s|V1
T11|.|? eXO temp unit configurable from panel (TODO @ `systems/exo/system.py:35`)|-
T12|x|f-string logging migration (G004 ignore removed)|-
T13|x|removed SLF001 ignore (rule not in active ruff selection ∴ no-op cleanup)|-
T14|.|? consolidate `iaqualink_session.json` plain-text token tradeoff → consider OS keyring|-
T15|x|add `cyclonext` system subclass `systems/cyclonext/{system,device}.py` (NAME="cyclonext", MIN_SECS_TO_REFRESH=50, eXO-style JWT auth, shadow URL = `prod.zodiac-io.com/devices/v1/{serial}/shadow`). parse `state.reported.equipment.robot[1]` → sensors {control_box_fw=`vr`, robot_fw=`robot[1].vr`, robot_sn=`eboxData.completeCleanerSn`, control_box_sn=`eboxData.controlBoxSn`, total_run_time_min=`robot[1].totRunTime`, mode, errors_code, equipment_id, cycle, cycle_start_time}; binary sensor {running = `mode != 0`}; switch {cycle start/stop}. expose system-level {name, system_time=`aws.timestamp`, first_connected=`devices.json.created_at`, device_number=`sn`}. register import in `client.py`. tests `tests/systems/cyclonext/` w/ respx fixture (read-path).|V4,V5,V6,V7,V10,B1
T16|x|cyclonext write path via persistent WebSocket `wss://prod-socket.zodiac-io.com/devices` (action=`setCleanerState`, namespace=`cyclonext`, mode 0/1/2). add `httpx-ws` or `websockets` dep behind `[robot]` extra. impl `start()`, `stop()`, `pause()`. clientToken = `{userId}|{authToken}|{appClientId}`.|V10,T15
T17|x|reverse-engineer cyclonext cycle/mode value table (quick, deep, smart, custom) using mobile-app traffic capture. doc mode→cycle in `systems/cyclonext/const.py`.|T15,T16
T18|x|add CLI surface for cyclonext: `iaqualink robot start <device>`, `robot stop <device>`, `robot status <device>`. expose cycle selection once T17 done.|I.cli,T15,T16
T19|x|model number = `devices.json[*].id` (e.g. 649888). already fetched at discovery; surfaced as cyclonext `model_number` device + CLI `robot status` line.|T15
T20|x|cyclonext runtime extension (±15 min). RE confirmed `stepper` field += `stepperAdjTime` per app +/- press. write path = `setCleanerState` with `equipment.robot.1.stepper=<abs-min>`. impl `set_runtime_extension(min)` + `adjust_runtime(delta)` + CLI `robot extend <abs>` & `robot adjust-time <±delta>`.|T16,T17,T18
T21|x|generalize WS frame builder; add `vr` system family (Polaris VRX iQ+ etc.). namespace=vr, shadow `equipment.robot.{state,canister,errorState,totalHours,prCyc,durations,cycleStartTime,stepper,sensors.sns_1}`. cycles 0=wall_only, 1=floor_only, 2=smart, 3=floor_and_walls. start frame `equipment.robot.state=1`. add return-to-base + 4-way remote control.|V4,V5
T22|x|add `vortrax` family (VortraX TRX 8500 iQ). same shadow shape as vr; product number from `eboxData.completeCleanerPn`. share parser with vr.|V4,V5,T21
T23|x|add `cyclobat` family (battery cyclonext). shadow `equipment.robot.{main,battery,stats,lastCycle,cycles}`. action=`setCleaningMode`, payload `main.ctrl=1`. battery sensors + cycle types 0..3.|V4,V5
T24|x|add `i2d_robot` family (older Polaris IQ). REST protocol via `r-api.iaqualink.net/v2/devices/{serial}/control.json`. hex-encoded request/response. params: `request=OA11` (status), `0A1240` (start), `0A1210` (stop). state/error/mode maps.|V4,V5
T25|x|cyclonext Remote + Lift System impl. helpers `remote_{forward,backward,rotate_left,rotate_right,stop}` (mode=2 + direction 1/2/4/3/0) and `lift_{eject,rotate_left,rotate_right,stop}` (mode=3 + direction 5/6/7/0). exit via mode=0. CLI `iaqualink robot remote <fwd\|back\|left\|right\|stop>` and `robot lift <eject\|left\|right\|stop>`.|V4,V5,T15
T26|x|unify cyclonext stop. drop CLI `robot exit-remote` (redundant: `mode=0` frame already sent by `robot stop`). update help text + docs to note `stop` exits any active mode (cycle / Remote / Lift).|T15,T25
T27|x|style audit. compare cyclonext / vr / vortrax / cyclobat / i2d_robot modules against `systems/exo/` + `systems/iaqua/` style. align idioms per V20: terse docstrings, comment style, import grouping, `__repr__`, no extraneous `__all__`, mirror exo's exception ordering. update tests where they drift from existing test_system.py patterns.|V20
T28|x|add `ty` (astral) static type-checker alongside mypy. dev dep wired, pre-commit local hook runs `ty check src/`, exits 0; `[tool.ty.rules]` warns (instead of errors) on three pre-existing legacy patterns; tracked for cleanup @ T29. suggested_commands memory updated.|V18
T29|x|tighten ty rules (`invalid-method-override`, `unresolved-attribute`, `invalid-assignment`) back to `error` once base-class param renames + cast()-style narrowing land. fix sites: `device.py` placeholder `_` params for `set_temperature`/`set_brightness`/`set_effect_*`; iaqua/system.py `target_temperature` access; system.py subclass registry typing; CLI robot system narrowing; `_robot_ws.py` ws_connect fallback typing.|V18,T28

## §B BUGS
id|date|cause|fix
B1|2026-04-27|`device_type=="cyclonext"` ∉ `AqualinkSystem.subclasses` ∴ robot-only account → empty CLI + lone stderr warning. fix tracked @ T15.|-
