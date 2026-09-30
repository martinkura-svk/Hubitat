# Moes ZigBee Wall Switch 1/2/3-Gang — Driver Guide for AI Agents

This repo contains a **Hubitat Elevation (HE) Zigbee driver** (Groovy) for **Tuya TS0601 (cluster
0xEF00) multi-gang wall switches**: Moes ZTS-EU 1–4 gang, Zemismart 4/6-gang, OZ Smart 1–4 gang,
Mercator Ikuü 5/6-gang, NOVADIGITAL, Eyzee, Trato/Zemismart keypads, Zemismart 4-gang screen
switch, Moes Star Feather SFL02-Z
(1–4 gang), Nova Digital Topazio, Quero automatização (1/2/3/6-gang), and assorted Tuya relay boards.
Original author: Martin Kura (martinkura-svk); maintained since 1.0.1 mostly by **kkossev**, with
community contributions (hhorigian, Nathanael-Shermett).

Community thread: https://community.hubitat.com/t/re-release-moes-zigbee-wall-switch-1-2-3-gang-tuya-ts0601-switches/138120
Current version: **1.5.3 (2026-09-30)** — see `version()` / `timeStamp()` in the driver.
Zemismart screen-switch master/relay control, configuration, and labels are implemented for testing; target-device operation
has not yet been confirmed. This guide was updated on 2026-09-30.

**Bug work:** a reviewed list of known bugs with exact locations, fixes, and verification steps is in
[BUGS.md](BUGS.md) in this folder (written 2026-07-11). If you were asked to fix bugs in this driver,
work from that list — do not re-derive the findings. Mark items `[x]` there only after the user
confirms a fix works on the hub. Items marked ASK USER / VERIFY ON DEVICE must not be changed
without confirmation.

> **ARCHITECTURE WARNING:** this is the **legacy monolithic architecture** — one self-contained
> driver file. There are **no** `#include kkossev.*` libraries, **no** `deviceProfilesV3` map,
> and **no** amalgamated/"lib included" build to keep in sync. It is also **not** in the kkossev
> repo style: no `logDebug/logInfo` helpers (logging is inline `if (debugLogging) { log.debug ... }`),
> no healthCheck, no `checkDriverVersion()` — settings are NOT auto-migrated on upgrade.
> Do not try to apply V3 conventions here.

> **FILE NAME WARNING:** the driver file is named **`Moes ZigBee Wall Switch`** — it has **no
> `.groovy` extension**. Do not rename it: HPM (`packageManifest.json`) and `importUrl` point to
> `https://raw.githubusercontent.com/martinkura-svk/Hubitat/main/Moes%20ZigBee%20Wall%20Switch`.

---

## 1. Files in this folder

| File | Role |
|---|---|
| **`Moes ZigBee Wall Switch`** | **The production driver** (v1.5.3, no extension). Keep implementation changes in this file. |
| `AGENTS.md` | This agent guide, formerly `CLAUDE.md`. Keep it consistent with approved driver changes. |
| `packageManifest.json` | HPM manifest. **Lags the driver: 1.3.5 vs 1.5.3** — update `version`, `releaseNotes`, `dateReleased` when releasing. |
| `README.md` | Generic repo boilerplate (license note only) — no driver documentation. The real history is the header comment block of the driver. |
| `.hubitat/metadata.json` | Hubitat VS Code extension config — DEV hub driver code id **1218**. Contains both an old-folder entry and the current-folder entry. Read the file for current editor revision numbers; these are not driver release versions. |
| `.git/` | **This folder IS a git clone** of `martinkura-svk/Hubitat`. Check current branch and worktree status before editing; do not assume a clean checkout. Releases land on `main` (kkossev contributes via commits/PRs to Martin Kura's repo). |

### Editing workflow
- Bump `static String version()` and `static String timeStamp()` **only when the user
  says so** — never after individual bug fixes (see memory `hubitat-bugfix-workflow`).
- Add a `*  revision x.y.z - YYYY-MM-DD - <author> - description` line to the header history block
  again only when the user asks for a version bump.
- There is **no `checkDriverVersion()`**: new preferences do NOT get defaulted on upgrade. Existing
  installs keep old settings; new settings are `null` until the user saves preferences. Write code
  accordingly (null-guard every new `settings?.x` use).
- On release: update `packageManifest.json`, commit and push to `main` of `martinkura-svk/Hubitat`.
- `definition(name: 'Moes ZigBee Wall Switch 1/2/3-Gang', namespace: 'Moes 1.31', ...)` — the
  misleading "1/2/3-Gang" name and the odd `'Moes 1.31'` namespace are **frozen**: changing either
  orphans every existing install. Do not "fix" them.

---

## 2. Supported device families and model detection

Family detection uses `device.getDataValue('manufacturer')` via helper predicates. The screen
switch helper additionally requires model `TS0601`.
**Always use the helpers, never compare manufacturer strings inline.** A device is exactly one of:

| Helper | Manufacturers | Family | Parse path |
|---|---|---|---|
| `isTratoZemismartKeypad()` | `_TZE204_rzdkn5rx`, `_TZE204_zpvusbtv`, `_TZE204_rkbxtclc` | Trato/Zemismart keypads 1–3 gang | main `parse()` for switch DPs, `parseTratoZemismartKeypad()` → `processTuyaDP()` for the rest |
| `isMoesStarFeatherSwitch()` | 12 mfrs: `_TZE200_ydkqbmpt/z3u99qxt/stvgmdjz` (1G), `uenof8jd/tzyy0rtq/hktk6hze` (2G), `wv9ukqca/zo0cfekv/rd8cdssd` (3G), `dq8bu0pt/9dhenr94/hmabvy81` (4G) | Moes Star Feather SFL02-Z + Nova Digital Topazio TPZ (scene-less) | `parseMoesStarFeatherSwitch()` → `processStarFeatherDP()` (main parse skipped entirely) |
| `getStarFeatherGangCount()` | (same list) | returns 1–4 | gates preferences and on()/off() loops |
| `isQueroSwitch()` | `_TZE284_ms97nkyy` (6G), `_TZE284_nzns7udm` (1G), `_TZE284_1aqlsquf` (2G), `_TZE284_pgxndxp4` (3G) | Quero automatização | `parse12GangSwitch()` → `process12GangDP()` (main parse skipped) |
| `isQuero12GangSwitch()` | `_TZE284_ms97nkyy` only | the 6-physical + 6-virtual model | different DP mapping inside `process12GangDP()` |
| `getQueroPhysicalCount()` / `getQueroGangCount()` | (Quero list) | physical relays / total children. **`getQueroGangCount()` is currently dead code** — `setupChildDevices()` hardcodes the counts. |
| `is8GangNvxorhcj()` | `_TZE204_nvxorhcj` | Nova Digital Topazio 8-gang | traditional path with special DP 0x65/0x66 = gangs 7/8 mapping |
| `isZemismartScreenSwitch()` | `_TZE284_xibaabmu`, model `TS0601` | Zemismart 4-gang screen switch | `parseZemismartScreenSwitch()` → `processZemismartScreenDP()` (main parse skipped) |
| *(none — default)* | everything else | "traditional" Moes/Zemismart/OZ/Tuya switches | inline logic in `parse()` + `processTuyaDP()` only if Trato |

Gang/child count is a separate hardcoded manufacturer switch in `setupChildDevices()` —
**when adding a fingerprint you must also add the manufacturer there**, or it falls to
the `default` = 4 buttons. Note it uses `device.data.manufacturer` (equivalent to `getDataValue`).

Special cases to know about:
- `_TZE204_dqolcpcp` (QA 12-way relay): 12 children are created but **only relays 1–6 work**
  (traditional parse handles DPs 01–06 only; `componentOn/Off` for children 07–12 sends bogus DP
  hex). Known limitation since 1.0.8 — documented in the header ("only the first 6 relays").
- `_TZE204_dvosyycn` (ZXYH 8IN/8OUT board): deliberately limited to 6 relays (header, 1.1.7).
- `_TYZB01_ltundz9m`, `_TZ3000_jyupj3fw` (TS0006, changelog 1.1.5): present in `setupChildDevices()`
  but have **no fingerprints** and the driver has no ZCL 0x0006 parsing — support is questionable
  (BUGS.md C-item, ASK USER).
- `setupChildDevices()` also lists manufacturers with no fingerprint anywhere
  (`_TZE200_wfxuhoea`, `_TZE200_1ozguk6x`, `_TZE200_mua6ucdj`, `_TZE200_vhy3iakz`, `_TZ3000_uim07oem`)
  — leftovers for manually-assigned devices; harmless, leave them.
- The `_TZE284_*` Quero fingerprints include cluster `ED00` and `controllerType: "ZGB"` — newer
  Tuya stacks; keep that form for future `_TZE284_` devices.

---

## 3. Message flow (parse pipeline)

`parse(String description)` handles **only** `catchall:` / `read attr -` descriptions on
the Tuya cluster **0xEF00**; every other cluster is silently ignored (there is no ZCL on/off,
battery, or Basic-cluster handling in this driver).

Routing order inside `parse()`:
1. `isMoesStarFeatherSwitch()` → `parseMoesStarFeatherSwitch(descMap)`, return.
2. `isQueroSwitch()` → `parse12GangSwitch(descMap)`, return.
3. `isZemismartScreenSwitch()` → `parseZemismartScreenSwitch(descMap)`, return.
4. Command `00`/`01`/`02` with `data.size() >= 7` → **inline traditional switch logic**:
   - `switchFunc = data[2]` (the Tuya DP), `switchAttr = data[3]` (the DP *type* byte),
     `switchState = data[6] == '01'`.
   - DP 01–06 with type `01` (bool) → route to child `${device.id}-<DP>`; parent `switch` becomes
     `on` if any child on, `off` when the last one turns off. For `is8GangNvxorhcj()`, DP `65`/`66`
     map to children 07/08.
   - Anything else: Trato keypads → `parseTratoZemismartKeypad()`; others → debug log only
     (**config readbacks are NOT evented for plain traditional devices** — BUGS.md B-item).
   - **Only the first DP of a multi-DP frame is processed on this path** (no loop).
5. Command `24` → `syncTuyaDateTime()` (Tuya time sync request; replies with UTC + local epoch).

The three legacy family parsers (`parseTratoZemismartKeypad`, `parseMoesStarFeatherSwitch`,
`parse12GangSwitch`) all share the same **multi-DP loop** pattern:
`dp = data[2+i]`, `dp_id = data[3+i]` (actually the DP *type* byte — misnamed), `fncmd_len = data[5+i]`,
`fncmd = getTuyaAttributeValue(data, i)` (big-endian), `i += fncmd_len + 4`.
Short frames (`dataLen <= 5`): command `0B` = ACK (ignored), `24` = time sync, `11` = ignored (Quero).
`parse12GangSwitch` additionally guards `descMap.data == null` ("check-in") — the other two do not
(Star Feather one is BUGS.md A1).

The screen-switch parser handles DP commands `00`, `01`, `02`, `05`, and `06`, retaining byte
payloads and reading both length bytes. It validates the complete frame before applying records,
rejects truncated frames, validates known DP types, and skips unknown DPs. Command `24` uses
the common time-sync helper; `25` replies with gateway-connected payload `010001` on endpoint 01.
Global ZCL responses and command `0B` are ignored. Reported endpoints other than 01 are rejected.
Configuration and label events are collected until all records have been processed; duplicate DPs
publish the last valid value. Master reports never fabricate individual relay states.

`getTuyaAttributeValue()` decodes the value **big-endian** (Tuya DP order) into a
signed 32-bit int.

## 4. Tuya DP maps per family

### Traditional devices (inline in `parse()` + `processTuyaDP()`, reachable for Trato only)

| DP | Meaning | Event |
|---|---|---|
| 0x01–0x06 | gang 1–6 state (bool) | child `switch` + parent aggregate (in `parse()`) |
| 0x07–0x09 | auto/countdown timer gang 1–3 | log only |
| 0x0D | master switch | log only (event deliberately commented out) |
| 0x0E | power-on behavior (0 off / 1 on / 2 last state) | `powerOnBehavior` |
| 0x0F | indicator mode (0 none / 1 lit when on / 2 lit when off) | `lightIndicatorMode` |
| 0x10 | backlight switch (bool) | `backlightMode` |
| 0x13 | delay-off schedule | log only |
| 0x1D–0x1F | restart status gang 1–3 | log only |
| 0x20 | unknown | event `unknown0x20` (undeclared attribute) |
| 0x65 | child lock (Trato) | `childLock` |
| 0x66 | backlight level 0–100 (Trato) | `backlightLevel` |
| 0x67 / 0x68 | LED on-color / off-color (Trato, 9-color enum) | `onColor` / `offColor` |
| 0xD1 / 0xD2 | cycle / random schedule | log only |

### Moes Star Feather (`processStarFeatherDP()`)

| DP | Meaning | Event |
|---|---|---|
| 0x01–0x04 | scene button 1–4 pressed | `pushed` 1–4 (**no PushableButton capability declared** — BUGS.md B-item) |
| 0x12–0x15 | mode_l1–l4: switch vs scene | `switchMode_l1..l4` |
| 0x18–0x1B | state_l1–l4 | child `switch` (gang 1 falls back to parent event if no child) + parent aggregate |
| 0x1E–0x21 | countdown timer gang 1–4 (s) | log only |
| 0x24 | backlight mode (bool) | `backlightMode` |
| 0x25 | indicator mode | `lightIndicatorMode` |
| 0x26 | power-on behavior | `powerOnBehavior` |
| 0x67 | induction mode | `inductionMode` |
| 0x68 | vibration mode (Gear 0–3) | `vibrationMode` |
| 0x69–0x6C | momentary timer gang 1–4 (s) | log only |

### Quero automatização (`process12GangDP()`)

| DP | 6-gang `ms97nkyy` (12 children) | 1/2/3-gang |
|---|---|---|
| 0x05, 0x06 | unknown (logged) | virtual scene 1/2 → child `physical+1` / `physical+2`; fncmd 0=toggle, 3=on, 2=off |
| 0x07–0x0C | virtual scenes → children 07–12 (same 0/2/3 protocol) | 0x07 = virtual scene 3 → child `physical+3` (0x08–0x0C would mis-map — BUGS.md C-item) |
| 0x18–0x1D | physical relays 1–6 → children 01–06 | physical relays (over `getQueroPhysicalCount()` → logged unknown) |
| 0x24–0x26 | unknown (logged) | alternate state reports for relays 1–3 |
| 0x65 | backlight brightness 0–99% | same → `backlightLevel` |

### Zemismart screen switch (`processZemismartScreenDP()`)

| DP | Meaning | Encoding / event |
|---|---|---|
| 0x01–0x04 | physical relays 1–4 | BOOL `01`, one byte 00/01; child `switch` + parent aggregate |
| 0x0D | native master switch | BOOL; `screenMasterSwitch` (separate from parent aggregate) |
| 0x0F | indicator mode | ENUM; `screenIndicatorMode`: off / on_off_status / switch_position |
| 0x10 | backlight switch | BOOL; `backlightMode`: off / on |
| 0x13 | color while delayed | ENUM; `delayColor`, existing `ColorOptions` |
| 0x1D–0x20 | power-on behavior per relay | ENUM; `powerOnBehavior_l1`–`powerOnBehavior_l4`: off / on / last state |
| 0x65 | child lock | BOOL; `childLock`: off / on (preference retains spelling `childLlock`) |
| 0x66 | backlight brightness | VALUE, four bytes, 0–100; `backlightLevel` |
| 0x67 / 0x68 | colors when on / off | ENUM; `onColor` / `offColor`, existing `ColorOptions` |
| 0x69–0x6C (105–108) | physical display labels 1–4 | RAW `00`, UTF-8; `screenLabel1`–`screenLabel4` string attributes |
| 0x6F | radar configuration | ENUM; `radarConfig`: none / 10s / 20s / 30s / 45s / 60s |

DPs 0x07–0x0A (countdowns), 0x0E (global relay_status), 0x18 (test_bit), and 0xC9
(cycle_schedule) are diagnostics only. Countdown values are decoded as unsigned four-byte
seconds and debug-logged; the others retain DP/type/raw bytes in debug logs. No countdown
preferences, attributes, commands, or writes are implemented for this family. Do not use DP 0xD1
from another model's cycle-schedule mapping. DP 0x13 is a color setting, not a schedule editor.

Identity evidence: [marcusvrsilva's post 56](https://community.hubitat.com/t/138120/56) and the
pairing screenshot supplied by the user in this implementation conversation. The screenshot
confirms model `TS0601`, manufacturer `_TZE284_xibaabmu`, endpoint `01`, profile `0104`,
application `4E`, input clusters `0004,0005,EF00,0000,ED00`, and output clusters `0019,000A`.
The fingerprint uses those cluster lists and `controllerType:'ZGB'`; it does not restrict application version.
Do not transfer identity details from other forum users or similar products.

Protocol reference: Zigbee2MQTT's
[ZMS-206US-4 converter](https://github.com/Koenkk/zigbee-herdsman-converters/blob/8c6213a285159cb8ebb63436db2aef7c39565772/src/devices/zemismart.ts#L863),
used with Home Assistant. This is not a ZHA implementation or proof of successful Hubitat hardware testing.
Do not add proprietary `E000/D0/D2` handling from another manufacturer's captures without target logs.

### Writes (commands to device)

All writes go through `getTuyaCommand(dp, dpType, hexValue)` / `sendTuyaCommand()` → Tuya cluster
0xEF00 command 0x00, payload = random `PACKET_ID` + dp + type + 2-byte length + value — **except**
the traditional-device paths which use hand-rolled raw frames
(`he cmd ... 0xEF00 0x00 {0001<dp><type><len><value>...}`) with hardcoded transid `01`.
DP type constants: RAW='00' (used for screen labels), BOOL='01', VALUE='02' (4 bytes),
STRING='03', ENUM='04', BITMAP='05'. `SETTIME` is unused.

- Traditional `on()`/`off()`: one raw frame toggling **DPs 01–03 only** (BUGS.md B-item for
  4–6-gang devices). 8-gang nvxorhcj: 8 single-DP commands (01–06,65,66). Star Feather / Quero:
  loop over gang count, DP 0x18+n. Quero 12-gang: one raw 6-DP frame (0x18–0x1D).
- `componentOn/Off(cd)`: childId → DP per family (SF: 0x18+n BOOL; nvxorhcj: 01–06/65/66;
  Quero: physical 0x18+n BOOL, virtual 0x05+n ENUM with 03=on/02=off; traditional: childId used
  directly as DP hex in a raw frame).
- Screen switch: parent `on()`/`off()` send one BOOL write to DP 0D, followed by an additional
  one-second delay and a DP query. No automatic four-relay fallback. `componentOn/Off()` still
  use DPs 01–04. All use the existing command builder and existing child IDs on endpoint 01.
- `refresh()` → `queryAllTuyaDP()` = cluster 0xEF00 command **0x03** (query all DPs). For the
  screen switch, this is preceded by Basic-cluster reads of attributes 0004, 0000, 0001, 0005,
  0007, FFFE. Common installation/initialization/refresh paths therefore use the same sequence.
- `syncTuyaDateTime()`: 0xEF00 cmd 0x24, length prefix `0800` for the screen switch (upstream
  little-endian UINT16), existing `0008` for other families, then UTC epoch + local epoch
  (each uint32, 1970-based); replies to the device's `24` request.

---

## 5. Child devices & the parent `switch` attribute

- Children are Hubitat built-in **`Generic Component Switch`**, DNI `${device.id}-NN`
  (`%02d`, 1-based), `isComponent: true`, created by `setupChildDevices()` →
  `createChildDevices(buttons)` on `installed()` and `initialize()`. Single-gang devices get
  **no** children — events go to the parent directly.
- Quero devices create `N physical + N virtual` children (virtual = scene buttons exposed as
  toggleable switches, states tracked driver-side since the device only reports presses).
- `deleteObsoleteChildren()` removes children whose DNI doesn't start with `device.id` or equals
  `${device.id}-00`.
- Parent `switch` = OR of all children: traditional path computes it inline in `parse()`;
  SF/Quero use `updateParentSwitchState(childId, newState)` which counts children
  as on, using `newState` for the child being updated (children's `currentValue` lags within the
  same event cycle — that's why the override parameter exists; preserve it).
- Screen-switch reports retain `state.screenRelayStates`, apply the full batch, then call the
  same aggregate helper with its optional third `reportedStates` map. Unknown relay states must
  not produce a false all-off report. Four children are explicitly selected in `setupChildDevices()`.
- DP 0D publishes `screenMasterSwitch` independently. After a batch, a changed master report
  with no valid relay updates triggers one DP query. `state.screenMasterState` is stored before
  that query to prevent identical replies causing a loop. Parent `switch` remains the OR of
  reported child states; never infer four child values from one master report.
- `getChildId(cd)` = last 2 chars of the DNI.

## 6. Commands & preferences

**Commands:** `on`, `off` (all gangs), `refresh`, `initialize` (capability — **also runs at every
hub reboot**, see BUGS.md B1), and `setPreference(name, value)` (added 1.5.2 by
Nathanael-Shermett for runtime control, e.g. changing `lightIndicatorMode` from Rule Machine; it
infers the type, calls `device.updateSetting`, patches `settings[key]` in-memory, then calls
`updated()` — no validation, see BUGS.md).
For the screen switch, `setPreference()` validates recognized names, types and values before
updating settings. Unsupported names (including countdowns and global powerOnBehavior) are
rejected without mutation or sending. Labels preserve text/spaces and use type `text`; logging
preferences use bool and brightness uses number. Writes still occur only through `updated()`.
Other families retain their existing behavior. There is no `setScreenLabel` command.
With `_DEBUG = true` (default false): `testParse`, `queryAllTuyaDP`, `test` (broken —
calls a nonexistent method, BUGS.md C1), `tuyaTest`.

**Preferences are conditional on the family**:
- All: `infoLogging`, `debugLogging` (both default true; debug auto-off after 24 h via
  `runIn(86400, logsOff)`). `powerOnBehavior` is shown for all except the screen switch.
- Screen switch: four text preferences `screenLabel1`–`screenLabel4`, always visible for this
  exact device; no enable toggle. Optional configuration preferences: `screenIndicatorMode`,
  `backlightMode`, `backlightLevel`, `childLlock`, `onColor`, `offColor`, `delayColor`, `radarConfig`,
  and `powerOnBehavior_l1`–`powerOnBehavior_l4`. All new configuration starts unset, without
  selected defaults; null/empty means do not configure. Keep global powerOnBehavior hidden.
- Trato keypad: `backlightLevel` (0–100, DP 0x66), `backlightMode` (0x10), `lightIndicatorMode`
  (0x0F), `onColor`/`offColor` (0x67/0x68), `childLlock` (sic — **the typo'd name is frozen**,
  renaming orphans saved settings; DP 0x65).
- Star Feather: `backlightMode` (0x24), `lightIndicatorMode` (0x25), `vibrationMode` (0x68),
  `inductionMode` (0x67), per-gang `switchMode_lN` (0x12+), `countdown_lN` (0x1E+),
  `momentary_lN` (0x69+), gang count gated by `getStarFeatherGangCount()`.
- Quero: `backlightLevel` 0–99 (DP 0x65).
- Traditional (else): `switchLightMode` off/on/position (DP 0x0F).

`updated()` pushes preferences: always `switchLightModeConfig() + powerOnBehaviorConfig()`,
plus the per-family blocks. Note the Trato block **re-writes** DPs 0x10 and 0x0E that the two
generic config functions already wrote (BUGS.md B-item), and the generic functions branch
internally on the same family helpers — read all of `switchLightModeConfig()` /
`powerOnBehaviorConfig()` before touching `updated()`.

Screen settings follow the existing configuration structure: `switchLightModeConfig()` handles
indicator/backlight, `powerOnBehaviorConfig()` handles per-relay startup modes, and `updated()`
adds `zemismartScreenConfig()` (brightness/lock/colors/radar) plus `screenLabelConfig()`.
All helpers return command lists for the common send at the end of `updated()`. Each helper
owns distinct DPs. Existing maps are reused; only ScreenIndicatorOptions and RadarConfigOptions
are new. `screenPreferenceOptions()` / `validScreenPreference()` share screen-only validation.
Invalid settings warn and are skipped individually; other valid settings are still sent.
Persistent configured controls reapply on each explicit save. Initialization, refresh, and reports
do not send configuration. Do not call `countdownConfig()` for this family.
Do not add separate lifecycle methods, label commands, or child-creation/reset calls to `updated()`.

- Labels change only the physical display; never automatically rename Hubitat children.
- Encode as UTF-8 RAW, with the payload length calculated from bytes by the common sender.
- Reject more than 12 UTF-16 code units or malformed surrogates; do not truncate. Preserve accents
  and spaces. Invalid labels produce no write for that label.
- Save Preferences sends changed or unconfirmed labels. Suppress only when both the last
  requested text and the report-backed attribute match. This permits retrying missed writes.
  Unconfigured blank/whitespace-only defaults do not overwrite factory text. Emptying a
  previously requested label sends an explicit clear. No automatic/background label retries.
- `state.screenLabelRequests` tracks requested values, separately from the report-backed
  `screenLabel1`–`screenLabel4` attributes. Sending a command does not confirm display changes.
- `initializeVars()` initializes only missing label settings; optional configuration stays unset.
  Retain saved configuration, text, and label state
  across initialization/restart. Refresh does not write labels. Logging defaults remain in the
  shared initialization path.

## 7. Known quirks / intentionally odd code (verify before "fixing")

1. `initialize()` calls `initializeVars()` which **unconditionally resets legacy preferences to
   defaults** — and capability `Initialize` means this runs at every hub reboot (BUGS.md B1,
   ASK USER — behavior inherited from the original 2021 driver). New screen-label settings are
   null-guarded; this exception does not fix or authorize changing the legacy reset behavior.
2. `refresh()` contains a commented-out family gate and a commented-out `lastCheckin()` path —
   since ~1.4.x it always sends `queryAllTuyaDP()` for every family. Deliberate simplification;
   don't resurrect the comments without asking.
3. `lastCheckin()` and the `lastCheckin` attribute are dead (only referenced from the commented
   block in `refresh()`).
4. `dp_id` in all three DP loops is actually the DP **type** byte — misnamed but harmless; it is
   passed around and never used for logic.
5. The scene behavior of Trato keypads (gangs in "scene mode" sending enum DPs) is **not** exposed
   as button events — presses land in `processTuyaDP()` cases 0x01–0x06 as debug logs. Config-only
   support was the 1.3.0 scope.
6. `sendTuyaCommand()` contains the classic `delay = 201` undeclared-binding-variable idiom inside
   the `zigbee.command()` call — standard Tuya-driver pattern, leave it.
7. Header changelog 1.1.6 says `_TZE204_rzdkn5rxc` but the fingerprint (correctly, 8-char suffix)
   says `_TZE204_rzdkn5rx` — the changelog has the typo, not the code.
8. `packageManifest.json` lags the driver by design between releases; sync it only on release.

## 8. Conventions

- Logging: inline `if (debugLogging) { log.debug ... }` / `if (infoLogging) { log.info ... }` —
  there are **no** logDebug/logInfo helper functions; match the inline style.
- No `mfgCode` anywhere — everything is Tuya cluster 0xEF00.
- Multi-byte Tuya DP values are **big-endian** (`getTuyaAttributeValue`); lengths are 2 bytes.
- Events do not carry `type: physical/digital` in this driver — don't add it piecemeal.
- No committed automated test suite. Hardware verification is on-hub: paste into HE "Drivers code" (code id 1218 via the
  VS Code extension), pair or hit Save Preferences, watch Live Logs with debug logging on.
- The file carries a `groovylint-disable` header (line 1); keep new code lint-clean but do not
  mass-reformat.
- A local Groovy 4 CLI is available on this machine (`groovy -e ...`) for verifying language
  semantics — HE's sandbox differs in places (see the Libraries BUGS.md cross-findings), so use it
  for Java/Groovy semantics only, not HE API behavior.

## 9. Current verification and deployment status (2026-09-30)

- The expanded screen-switch implementation passed 32 focused local packet-replay groups using mocked Hubitat
  APIs, including representative traditional, Trato, Star Feather, and Quero regressions. Coverage
  includes all enum values, brightness boundaries, invalid preferences/reports, master-only query
  loop prevention, aggregate state, report-only diagnostic DPs, label retry, and time prefixes. The temporary
  harness is outside the repository; there is no committed automated test suite.
- The expanded implementation compiled successfully on the DEV hub as temporary driver 5246
  (unique name, fingerprints removed, no assigned devices). The temporary driver was deleted
  and its absence verified. Installed driver 1218 remained at editor revision 164 with two
  assigned devices; it was not updated during this implementation.
- DEV hub: `http://192.168.0.151`, production driver code ID **1218**. The user reported a manual
  upload before the final removal of the label toggle. Do not assume subsequent local edits are
  deployed, and do not automatically upload or release when editing documentation.
- Relay/master control, physical-press reports, aggregate state, configuration controls,
  restart/refresh operation, and display labels still require confirmation on the target device.
  Test all four positions, accented text, the length limit, retrying unconfirmed labels, and explicit
  clearing. Test lock/unlock and per-relay startup behavior under device-owner control. Keep
  BUGS.md completion gates intact.
- Writable countdowns/cycle schedules, MCU firmware discovery, child naming, other manufacturer
  IDs, proprietary E000/D0/D2 handling, and unrelated legacy fixes remain out of scope. A local
  version bump is not an HPM release or authorization to publish.
