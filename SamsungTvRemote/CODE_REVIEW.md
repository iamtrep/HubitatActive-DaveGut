# Samsung TV Remote — Code Review & Architecture Audit

Scope: the eight `#include` sources under `src/` (core `davegut.SamsungTVRemote.groovy`
plus the `samsungTvWebsocket`, `samsungTvApps`, `samsungTvPresets`, `SmartThingsInterface`,
`samsungTvST`, `Logging` libraries). `samsungTvTEST` is commented out of the build and is
not audited. Line references are to the `src/` files.

Recommendations are prioritized P0 (correctness / data loss / thread-blocking) through
P3 (cleanliness). Each item names a concrete fix.

## Architecture summary

The modular library split is sound and maps to clear responsibilities: transport
(`samsungTvWebsocket`), app control (`samsungTvApps`), presets (`samsungTvPresets`),
SmartThings transport+parse (`SmartThingsInterface`) vs SmartThings command wrappers
(`samsungTvST`), and logging. The driver is `singleThreaded: true`.

One structural item remains after the current cycle:

1. **App launch uses a deprecated REST path** that may be dead on 2020+ Tizen (P1-4).

(The single-slot send buffer, formerly P0-2, is resolved — see the baseline section.)

## P0 — correctness, data loss

_None open._ The single-slot `state.wsData` buffer (P0-2) has been replaced with an
in-order FIFO queue; see the baseline section.

## P1 — reliability

(P1-1 `ms.channel.unauthorized` and P1-2 art-mode-on-non-Frame are resolved — see the
baseline section. P1-5 was withdrawn: on a single-threaded driver the two poll loops
serialize, and their attribute writes do not overlap in practice, so there is no race.)

### P1-3 SmartThings HTTP calls have no explicit timeout
`SmartThingsInterface.asyncGet` (195–204), `syncGet` (207–235), `syncPost` (237–267) set no
`timeout`. A slow ST endpoint can hang the call (the sync variants block the thread). Fix:
add an explicit `timeout:` to each, consistent with the local REST calls (which use 3–10 s).

### P1-4 App launch uses a deprecated REST path — CLOSED, not viable on this hardware
`samsungTvApps.appOpenByName` (line 32) launches via `POST /api/v2/applications/{id}`, a
legacy endpoint commonly dead on 2020+ Tizen. The proposed fix was to port launch to the
websocket `ed.apps.launch` message.

Probe result (2026-07-01): a `getInstalledApps` diagnostic emitting `ed.installedApp.get`
over the remote channel got **no response** on the 2020 set — the app WS API is not
exposed, so `ed.apps.launch` is not available either. Neither transport can list or launch
apps on this TV. Closed as not-viable; the existing REST-based app features are therefore
likely non-functional on this hardware too (pre-existing, left as-is).

## P2 — minor correctness

### P2-1 Double `exit()` in `appCloseParse`
`samsungTvApps.appCloseParse` lines 79–80 call `exit()` twice on the failure path. Fix:
remove the duplicate unless a repeat is intentional (add a comment if so).

### P2-2 `updateAppName` guard never fails
`samsungTvApps.updateAppName` initializes `appId = " "` (line ~222) then stores app data
under `if (appId != "")` (line 253) — always true, so a failed `httpGet` still writes state.
Fix: test `appId != " "`.

### P2-3 Discarded log/test data in `statusParse`
`SmartThingsInterface.statusParse` (144–188) builds `parseResults` but logs only
`logData` (which never receives it), and line 148 assigns `Map testData = [...]` that is
never used — so the `stTestData` developer dump produces nothing. Fix: log `parseResults`
(or drop it) and either emit or remove the `stTestData` capture.

### P2-4 Implicit-global `respData`
`SmartThingsInterface.deviceSetup` line 59 and `samsungTvST.poll` line 33 assign `respData`
without `def`, creating a script binding rather than a local. Fix: add `def`.

## P3 — cleanliness

- **P3-2** Decide on `samsungTvTEST`: keep as an intentional parked library or drop it and
  its commented `#include`.
- **P3-3** `@CompileStatic` is absent throughout. Converting is high-effort and risky given
  the pervasive dynamic `device`/`state`/`sendEvent` access; recommend only if a specific
  hotspot needs it, not as a blanket change.
- **P3-4** `showMessage()` is a declared-but-unimplemented placeholder; remove the command
  or implement it.
- **P3-5** `pauseExecution` blocking (was P0-1; re-tiered). Sites: `presetExecute` (7 s),
  `presetCreate` (2 s), `fastBack`/`fastForward` (1 s), `setPowerOnMode` ART_MODE (1 s). On
  `singleThreaded: true` these block other commands for the duration, but that block is
  short and gives these sequences atomicity for free. The interaction that made it harmful
  (a blocked flush dropping a buffered send) is gone now that the socket is held open and
  sends go direct. Converting to `runIn` continuations would re-open interleaving that then
  needs hand-rolled guards — net more complex for no real gain. Revisit only the 7 s
  `presetExecute` case, and only if that specific block is observed to bite.

## Reference cross-check

Four independent Samsung-TV implementations were audited at source level and used to
validate protocol choices and surface gaps in this driver:

| Implementation | Language | Role |
|---|---|---|
| samsungtvws (`xchwarze/samsung-tv-ws-api`) | Python | de-facto WS protocol reference (token auth, `ms.remote.control`, `ed.apps.launch`, art-app channel) |
| Toxblh `samsung-tv-control` | TypeScript/Node | WS control library commonly used in Node-RED flows |
| openHAB Samsung TV binding | Java | long-lived-connection binding |
| Home Assistant `samsungtv` | Python | async integration built on samsungtvws |

**Convergent recommendations adopted this cycle** (reference support in parentheses):
- Power-on by Wake-on-LAN only; do not send `KEY_POWER` when the set is reachable, since
  it toggles (HA wakes via WoL only; samsungtvws separates power from the key channel).
- `Option:"false"` in the `ms.remote.control` payload (present in all four).
- Treat a websocket failure as a power-off signal rather than trusting REST `PowerState`
  alone (HA `is_alive`; samsungtvws).
- A settle/cooldown window around power-off (HA power-off handling).
- Wake-on-LAN to more than one port / repeated packet (common WoL practice in HA).

**Reference-supported items still open** (mapped to the findings above):
- App launch via the WS `ed.apps.launch` emit instead of REST `/applications/{id}`; all
  three of samsungtvws, Toxblh, and HA use the WS path, and the REST path is legacy →
  **P1-4**.
- Handle `ms.channel.unauthorized` (token rejection is surfaced by samsungtvws) → **P1-1**.
- Guard the Frame art power-hold `Release` on a still-open socket (defensive; matches the
  per-send liveness checks in HA) → minor, Frame-only.
- Art-mode request/response correlation by id (samsungtvws correlates art request ids) →
  cosmetic, Frame-only, low value here.

**Considered and not adopted:**
- openHAB holds the connection open with an application-level keepalive. This driver now
  keeps the socket open (idle-close default `never`) and the Hubitat platform pings it
  every 30 s, so an app-level keepalive would be redundant.

## Future direction — SmartThings OAuth integration (parent app + child drivers)

`tvChannel`/volume/input/modes are SmartThings-cloud-only; the local Samsung API does not
expose them. The PAT path is effectively dead for new setups — tokens issued after
30 Dec 2024 expire in 24 h (legacy pre-2024 tokens still work up to 50 y). The durable
alternative is OAuth, which on Hubitat is an app-level capability. This would be a new
integration, not a patch to this driver.

Shape:
- **Parent app** — hosts the SmartThings OAuth flow (self-enabling OAuth), stores and
  refreshes tokens, enumerates devices via `GET /devices` (filter to Samsung TV
  capability/type), and creates one child per TV. Matches the `integrations/<Name>/`
  convention (vendor manager + child drivers).
- **Child driver (per TV)** — design decision: **hybrid vs cloud-only.** Recommended
  hybrid: keep the existing *local* websocket control (power/keys/WoL) and use the parent's
  cloud poll only for the ST-only attributes. Cloud-only would discard the local control
  that is this driver's main value.

Token model: one account-level authorization covers all TVs on the account; each TV is
addressed by `deviceId`. Access tokens are short-lived; refresh tokens keep it alive (with
rotation/expiry to handle — confirm exact TTLs at build time). One-time user setup:
register a SmartApp/OAuth client in the SmartThings Developer Workspace (client id/secret,
redirect_uri → the parent's OAuth endpoint).

Open scope questions: hybrid vs cloud-only child; and whether this lives in this fork or as
a separate integration in the user's own namespace. Effort is substantial — treat as a
deliberate feature, not part of the review cleanup.

## Baseline — resolved in the current fork cycle

For reference, so these are not re-flagged (all built from `src/`, deployed as 2.3.9i):

- `appData` `==`→`=` NPE; app-launch `httpPost` body; four preset defects (null guard,
  `!connectST`, casing, log field); `state.wsData` init in `updated()`.
- Reliable power-off: non-Frame single key; Frame connection-gated hold.
- Wake-on-LAN-only power-on (removed the `KEY_POWER` toggle); dual-port WoL (9 and 7).
- `Option:"false"` on key payloads.
- Websocket-failure power cross-check + post-off cooldown + optional idle-close preference.
- Version-change self-reinitialization on the first poll after a code push.
- Single-slot `state.wsData` send buffer replaced with an in-order FIFO `state.wsQueue`
  (drained on open, cleared on failure, bounded); dead `xxxsendMessage` removed. Fixes
  dropped keys in cold-socket bursts (e.g. multi-digit `channelSet`).
- `configure()` no longer opens the art-mode websocket on non-Frame sets (gated on
  `frameTv`). (P1-2)
- `parse()` handles `ms.channel.unauthorized`: resets the token so the next connect
  re-pairs with the on-screen prompt. (P1-1)
