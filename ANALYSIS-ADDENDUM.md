# Addendum to ANALYSIS.md — re-analysis against the full field record

Date: 2026-11-08. Scope: re-analysis only. No patch was written in this pass, and none is
authorized by this document. Owner ruling in force (2026-11-08 19:45 ET): no updates to
upstream tickets until a fix is demonstrated working on the fleet, and whatever works is
then applied to the other fleet machines in use.

Code base read: `repo/` at PR #15420 head `e180b38b`, plus `libdrmtap/` at `5936a7a`.
All line numbers below are from that checkout. Field record: the five observations in the
tasking, plus the journal extract at
`~/workspace/tower-offsite-staging/records/rustdesk-journal-greeter-attempts-2026-11-08.txt`
(sha256 44fabf35016d640e22ac651d2e9e70b5863e7fec8cb8c1dff5e0c7b38289dde2).

## 0. What this addendum corrects

ANALYSIS.md section (c) ranked candidate 1, the session-classification defect (greeter
logind session Type=tty, so the session classifies as "x11"), as the LEADING cause, and
that ranking was reported onward as settled. The full field record falsifies the strong
form of that claim:

- Observation 3: a greeter image appeared at 19:38:37. The greeter session's Type did not
  change between attempts 1-2 and observation 3 (it was still Type=tty when read directly
  at 19:42, observation 2). A pure classification cause predicts no image ever.
- Observation 5: after a cold boot, with the render/input group fix verified still in
  place, the greeter again produced no image. A pure group-access cause predicts an image.

The classification defect is real in code (traced in section 2) and stays on the record as
a defect. It is demoted from "the cause" to one factor in a spawn-context-dependent
system. The new prime suspect is the spawn context itself: everything the greeter
`--server` and the root service latch at spawn time differs between the one success
(a logout-spawn on a just-restarted service, on a warm system) and the failures
(pre-group-change spawns, and a cold-boot spawn roughly 6 seconds after service start).

One further correction to the first analysis's framing: there is no single
"classification" in this code. At least four independent classifiers run in different
processes and can disagree inside one boot (section 2.5). Any single-cause story that
does not say which classifier, in which process, at which moment, is under-specified.

## 1. The five observations, annotated with spawn context

| # | When | Spawn context of the greeter `--server` | Result |
|---|------|--------------------------------------------------|--------|
| 1 | ~18:57 ET | Running system, service already up; greeter spawned on logout (journal: sudo spawn of cosmic-greeter `--server` at 18:57:26, after a prior server exit at 18:57:19). Groups NOT yet fixed. | Authenticated, blank frames. Bus permission-denied lines at start. No capture/display logging anywhere. |
| 2 | ~19:10-19:14 ET | After the 19:12:02 boot (journal: rustdesk.service started 19:12:08, greeter `--server` spawned 19:12:14), greeter visibly awake on the monitor. Groups NOT yet fixed. | Same blank result. The idle/display-asleep confound is eliminated by this attempt. |
| 3 | 19:38:37 ET | Owner's LOGOUT from a running system whose rustdesk service had just been restarted (~19:38). Groups FIXED just before (cosmic-greeter added to render 992 and input 995; video 44 already held). | Greeter image displayed remotely, first time. |
| 4 | 19:38:56 ET | Same configuration as 3. Owner logged in through the remotely viewed greeter. | Greeter server terminated on cue; user server spawned 19:38:58; client did NOT ride through; manual reconnect lands on the desktop and works. Bus-denied lines present in this WORKING configuration. |
| 5 | ~19:42:49 ET | COLD BOOT. Greeter server spawned at boot. Groups verified held after reboot; service active/enabled; package and rendezvous/relay/key config intact. | Greeter did NOT display remotely. |

Deltas between the success (3) and the cold-boot failure (5), same groups, same package,
same config: (i) spawn trigger (logout transition vs boot), (ii) service age and warm
state at spawn (just-restarted service whose DRM producer pre-warm ran while a Wayland
user session was active, vs a service 6 seconds old at boot), (iii) compositor and logind
settling state at spawn, (iv) the root service's harvested environment and caches at
spawn. Section 2 traces why each of these can change the capture outcome in code.

## 2. (a) The spawn-context trace: how capture-path selection depends on when and how the greeter server is spawned

### 2.1 The root service loop decides which slot, and COSMIC's greeter never takes the greeter branch

The root `--service` loop (`src/platform/linux.rs`, loop body from ~line 937) calls
`desktop.refresh()` each iteration, then `update_active_user_lookup_cache(&desktop)`,
then picks a server slot:

- If `desktop.username == "root" || desktop.is_login_wayland()`: the login `server` slot
  (the other slot is stopped).
- Else if `desktop.username != ""`: the ordinary `user_server` slot.

`Desktop::is_login_wayland()` (`src/platform/linux.rs:1803-1805`) is
`is_gdm_user(username) && protocol == "wayland"`. `is_gdm_user`
(`libs/hbb_common/src/platform/linux.rs:117-120`) matches only `"gdm"` and `"sddm"`
(lightdm is present but commented out). cosmic-greeter fails the first half by name, and
(Section 2.3) would fail the second half anyway. Consequences specific to COSMIC:

- The drm-feature greeter handling inside the login branch (drop the login server to the
  greeter's uid, 1-second survival grace via `server_survived_grace`,
  `src/platform/linux.rs:742` and ~953-986, root `--server` fallback if the greeter child
  does not stay up) is never reached for cosmic-greeter. The code comment in that block
  concedes the general shape: a greeter `is_gdm_user` does not recognize "never reaches
  this branch — it takes the unprivileged else-branch below already."
- cosmic-greeter's `--server` is therefore spawned and supervised exactly like an
  ordinary user session server, through the `user_server` slot, with none of the
  login-screen-specific handling the PR built for GDM/SDDM.

### 2.2 Session enumeration: first active seat0 session wins, order is loginctl's

`Desktop::refresh` (`src/platform/linux.rs:2062-2136`):

- If the currently tracked session id is still active and on seat0, refresh returns
  early, only harvesting Xwayland environment when Xwayland is running and the session
  is not login-wayland.
- Otherwise it enumerates with `get_values_of_seat0_with_gdm_wayland(&[0, 1, 2])`, the
  NON-skipping variant (`libs/hbb_common/src/platform/linux.rs:204-211`, over
  `_get_values_of_seat0` at 213-261). The underlying iteration takes loginctl's output in
  its own order and returns the FIRST session that is active and on seat0. The skipping
  variant's fallback loop drops Type=tty/unspecified sessions entirely; the variant
  refresh actually uses does not filter by Type at all.
- The stale-list comment at `src/platform/linux.rs:2097-2103` (carried over from
  ANALYSIS.md) sits in this function: the author knew the enumerated view can lag the
  real seat state across transitions.

Timing consequence: at a cold boot, which session is "first active seat0" when the loop
first looks depends on greetd's session registration racing the service start (journal:
service started 19:12:08, greeter spawn 19:12:14). At a logout spawn, the loop has been
tracking sessions continuously and observes the transition from a warm state. The code
contains no settle delay or re-validation keyed to boot.

### 2.3 The protocol classification of the greeter session is "x11" in every spawn — and that is not the whole story

Refresh sets the session protocol from `get_display_server_of_session(sid)`
(`libs/hbb_common/src/platform/linux.rs:167-186`): it reads the session's logind Type;
if the Type is empty, `"tty"`, or `"unspecified"`, it falls back to the CALLER's
`XDG_SESSION_TYPE` environment variable, and if that is unset it defaults to `"x11"`.
The caller here is the root service, whose environment carries no `XDG_SESSION_TYPE`.
Session 25 read Type=tty, Class=greeter (observation 2), so the greeter's protocol is
`"x11"` at every spawn: boot, logout, always. This half of the first analysis is
confirmed by direct read and by code.

What the first analysis under-weighted: this protocol value is only one input, consumed
by `Desktop::refresh` and the service loop. It is not what the per-greeter `--server`
process itself uses to gate its capture routing (Section 2.5), and it did not change in
observation 3, when an image appeared anyway.

### 2.4 The environment the greeter server actually carries is an accident of the spawn path

`try_start_server_` (`src/platform/linux.rs:680-721`) builds the child's explicit
environment from the refreshed `Desktop`: it pushes ONLY non-empty `DISPLAY`,
`XAUTHORITY`, `WAYLAND_DISPLAY`, `HOME`, `DBUS_SESSION_BUS_ADDRESS`, a
forced-display-server override if configured, and `TERM`. It never propagates
`XDG_SESSION_TYPE`, `XDG_SESSION_ID`, or the logind Class.

Where those values come from in refresh: since `is_login_wayland()` is never true for
cosmic-greeter, refresh does not take the bare-environment greeter branch. It takes
`get_display_xauth_xwayland` (`src/platform/linux.rs:1843-1872`) or
`get_display_xauth_wayland` (1812-1841): harvesters that scan for portal / Xwayland /
ibus processes owned by the session uid and copy `DISPLAY` / `XAUTHORITY` /
`WAYLAND_DISPLAY` / `DBUS_SESSION_BUS_ADDRESS` out of those processes' environments,
retrying up to 10 times at 300 ms. What they find depends on which of the greeter's
helper processes exist at that moment, which is a spawn-timing fact, not a config fact.

`run_as_user` (`src/platform/linux.rs:1231-1305`) then spawns through sudo. On this
system `SUDO_E_PRESERVES_ENV` holds, so the spawn is
`sudo -E XDG_RUNTIME_DIR=/run/user/<uid> -u <user> <exe> --server`, with the explicit
environment set on the sudo process. Under `sudo -E` the child ALSO inherits the root
service's own environment, subject to sudoers policy. The journal confirms this spawn
form (every greeter spawn is the same sudo line). The root service's own environment is
not static: `set_x11_env` (`src/platform/linux.rs:781-789`) writes `DISPLAY` and
`XAUTHORITY` into the SERVICE's environment when the display changes. So a greeter
server spawned after a user session (observation 3's logout spawn, minutes after the
service had been tracking tpierson's session and its displays) can inherit materially
different `DISPLAY`/`XAUTHORITY` values than one spawned 6 seconds after boot
(observation 5), with identical code, identical config, and identical session Type.

The fallback spawn form (`sudo -u <user> -- env XDG_RUNTIME_DIR=... KEY=VALUE...`) passes
only the explicit set; which form a host uses is itself an environment fact
(`SUDO_E_PRESERVES_ENV` is probed), recorded here because it changes the analysis on
other fleet machines.

### 2.5 Inside the child: one memoized latch, and three classifiers that can disagree

The per-greeter `--server` process:

- `IS_X11` (`src/platform/linux.rs:47`) is a lazy static evaluated ONCE per process:
  `hbb_common::platform::linux::is_x11_or_headless()`. `is_x11()` at
  `src/platform/linux.rs:2361-2362` returns the memoized value for the process's whole
  life. A server spawned before its session state settles keeps whatever verdict it was
  born with.
- `get_display_server` (`libs/hbb_common/src/platform/linux.rs:135-165`): forced-env
  override first; then seat0 enumeration; if that yields nothing, the process's own
  `XDG_SESSION_ID`, then `/proc/self/sessionid`, then its own `XDG_SESSION_TYPE`,
  default `"x11"`. The `--server` runs OUTSIDE the greeter's logind session (spawned by
  the root service via sudo), so its self-referential fallbacks describe the service's
  context, not the greeter session it serves.
- The DRM producer in the ROOT service classifies independently and unmemoized:
  `drm_prewarm` (`src/ipc/drm.rs:642-672`) polls `scrap::is_x11()`
  (`libs/scrap/src/common/mod.rs`, ~line 270, the unmemoized form) every 2 s for a 30 s
  budget. At a cold boot whose only session is the Type=tty greeter, that read stays
  "x11" for the whole budget and the pre-warm is skipped (the `_drm` listener still
  runs; consumer handshakes still enumerate fresh). At the 19:38 restart, the service
  restarted while tpierson's Wayland session was active: `is_x11()` was false at once,
  and the pre-warm primed the reader (`DrmReader::open(None, 0)` + `grab_desc()`, as
  root). Producer warm state at the moment the greeter server later spawns is therefore
  completely different between observations 3 and 5.
- The consumer side classifies a third time: `warm_availability`
  (`src/server/drm_capturer.rs:964-985`) retries while `scrap::is_x11()` (10 x 300 ms),
  and `is_available` (831-890) probes the producer; the routing gates proper use
  `is_available_cached()` (826), cache only, never probing.

Answer to (a), in one paragraph: the greeter server's capture path is selected from
state latched at spawn (the memoized `IS_X11`, the harvested display/xauth environment,
the service's mutated own environment inherited through `sudo -E`), from caches whose
contents depend on service history (producer pre-warm state, the active-uid cache the
service loop maintains, the consumer availability cache with its 30 s negative TTL), and
from a session enumeration that takes the first active seat0 session in loginctl order
with no boot-settle handling. The logout spawn of observation 3 and the cold-boot spawn
of observation 5 differ on every one of those axes while sharing session Type, groups,
package, and config. That is the spawn-context dependence the field record shows.

## 3. Candidates re-scored against all five observations

Legend: **E** explains the observation, **C** consistent with it, **X** contradicted by
it, **—** not engaged by it.

| Candidate | 1 (blank, pre-groups, logout spawn) | 2 (blank, pre-groups, boot spawn, display awake) | 3 (IMAGE, groups fixed, logout spawn, restarted service) | 4 (handover drop; bus lines in working config) | 5 (blank, groups held, cold boot) |
|---|---|---|---|---|---|
| C1. Type=tty → "x11" classification routes the greeter server off DRM/portal onto X11 capture of the greeter's XWayland | E | E | **X** as sole cause (Type unchanged; image appeared) | — | C (only in combination with C2) |
| C2. Spawn-context latching: memoized `IS_X11`, harvested/inherited environment, producer pre-warm state, service caches all fixed at spawn and different at boot vs logout | C | C | E | — | E |
| C3. `_drm` peer authorization against a stale or empty active-uid cache (fail-closed, cache-only re-auth) | C | C | C | — | C (strongest at cold boot, where the cache is youngest) |
| C4. Device-group access (render/input) for the unprivileged convert | E (as a gate on the GPU-convert path) | E (same) | E (first image coincides with the fix) | — | **X** as sufficient cause (groups held, no image) |
| C5. Session-bus death kills the PipeWire/portal fallback | E (as the reason the fallback yields nothing) | E (same) | C | E for exoneration: bus-denied lines present while WORKING, so the bus is not the capture blocker | E (as fallback-death, not as primary) |
| C6. cosmic-comp DRM-master contention / compositor not settled at spawn | C | C | C | — | C (boot journal of the 19:12 boot carries "Unable to become drm master, assuming unprivileged mode" and "Failed to initialize drm lease global for: card0") |
| C7. Handover seam (separate defect, Section 5) | — | — | — | E | — |

Notes on individual candidates:

**C1 — demoted, not discarded.** The code defect is exactly as ANALYSIS.md traced it,
including the contributing defect it admitted: greeter recognition matches only
gdm/sddm, and `get_display_server_of_session` defaults a tty session to "x11". What the
field record kills is the inference "therefore the greeter server captures XWayland and
never attempts DRM, in every spawn." In observation 3 something produced real frames
with the classification unchanged. Either the successful frames did not travel the path
C1 describes (producer/consumer reads are unmemoized and can diverge from the child's
latch), or the X11 path itself, fed a different harvested environment in that spawn,
captured a live XWayland that was actually presenting the greeter. This analysis does
not claim to know which; the test plan (Section 6) is built to find out. Any future fix
that only repairs the classification, without explaining observation 3, should be
treated as unproven.

**C2 — prime suspect.** It is the only candidate that engages the actual delta between
observations 3 and 5. Its sub-mechanisms are independently visible in code
(Sections 2.4-2.5): the child's one-shot `IS_X11` latch; the harvest-based environment;
the service's self-mutated environment inherited under `sudo -E`; the producer pre-warm
that primes at a warm restart and skips at a cold boot; the 30 s negative availability
TTL. C2 also retro-explains observations 1-2 without needing groups to be the whole
story (both were spawns whose latches and caches were born in unfavorable states), while
leaving room for C4 to have been a genuine additional gate in 1-2.

**C3 — live, unproven.** `drm_peer_authorized` (`src/ipc/drm.rs:728-734`): root always;
otherwise the peer uid must equal the active uid. The per-frame re-check
(`src/ipc/drm.rs`, ~886-895) uses `active_uid_cached()` (`src/ipc/auth.rs:218`), which
reads ONLY the service-loop cache and fails closed on a miss or mismatch, closing the
stream with a warn. The cache is maintained by the service loop from its refreshed
`Desktop` (`update_active_user_lookup_cache`, loop at ~938). At a cold boot the cache's
early contents, and whether it holds 987 by the time a consumer connects, are timing
facts nobody has read yet. C3 cannot explain observations 1-2 by itself (the greeter was
the active session for long stretches there), but it is a plausible cold-boot-specific
contributor and it is cheap to discriminate (T3/T6).

**C4 — necessary, proven insufficient.** The unprivileged convert context
(`RenderConverter::open_render`, `libs/scrap/src/common/drm_render.rs:25-47`) runs in
the greeter `--server` and needs a usable render node: the render group, or the logind
ACL that follows the ACTIVE session. Before the fix, cosmic-greeter had neither render
group nor (outside its own session's ACL windows) guaranteed access; the tower-offsite
diagnostic found the render group had no members at all. After the fix, observation 3
produced the first image and observation 5 none. So group access gates something real
(the GPU-convert path; note the code's fallback on convert-open failure is to REQUEST
the producer's CPU-converted path, `src/server/drm_capturer.rs:418-445`, so even this
gate is not mechanically a black-screen guarantee), and it is not the whole story.
Whatever fix is finally demonstrated must keep the group fix in place; it must not be
credited as the fix.

**C5 — exonerated as blocker, confirmed as fallback-death.** Observation 4 settles it:
the session-bus permission-denied lines appear in the working configuration too. The
bus still matters structurally: when the DRM front path is unavailable or demoted, the
fallback is PipeWire/portal (`src/server/wayland.rs:398-445`), which needs the session
bus the greeter server provably cannot reach (its `DBUS_SESSION_BUS_ADDRESS` is
inherited through sudo and does not point at the greeter session's bus). At the greeter,
every demotion lands on a dead end. That is why failures at this seat present as a
placid authenticated black session rather than an error.

**C6 — plausible contributor, field question.** The 19:12 boot's journal shows
cosmic-comp failing to become DRM master and failing to initialize the DRM lease
global. Whether libdrmtap's open needs master, and how master transitions interact with
a producer that starts at service start, was not fully traced in the libdrmtap tree in
this pass; C6 is scored on the timing correlation only and is targeted by T5.

**The successful frame source in observation 3 is unidentified.** This addendum
deliberately does not name it. Naming it without evidence is how the first analysis
went wrong.

## 4. (b) The silent-failure chain: where a device-access or open failure is swallowed

Overarching field fact: on this build, the greeter user has NO session logs at all
(host-verified: `/var/lib/cosmic-greeter/.config/rustdesk/` holds only the two config
files; no log directory exists for the greeter user or under root). Every log line
quoted below from the consumer (`--server`) side is therefore invisible at the greeter
by construction. The root service's lines are the only possible witnesses, and the
journal extract contains NONE of the DRM lines below for any attempt — itself a datum
(it suggests the producer/consumer machinery never got far enough to log, or that the
service's output is not in the extract's filter; T3 resolves which).

The chain, in the order a frame would travel:

1. **Producer device open.** `DrmReader::open` (`libs/scrap/src/common/drm_reader.rs:
   107-140`) returns `None` on `drmtap_open` failure with a single info line,
   "drmtap_open failed; DRM capture unavailable" (line 133). No device path, no errno.
2. **Enumeration skips silently.** `drm_enumerate_all_displays` (`src/ipc/drm.rs:
   176-262`): in the multi-device path a device whose open returns `None` is skipped;
   the only trace is a debug line when that device also reports zero displays. The
   `/dev/dri` path is the same shape, with a summary info line counting only what
   opened. A card that will not open can vanish from the advertised display list with
   no error anywhere.
3. **Pre-warm aborts quietly.** `drm_prewarm` (`src/ipc/drm.rs:642-672`): if the
   session still reads X11 after the 30 s budget, one info line ("session still reads
   as X11 after 30s; skipping the pre-warm (the _drm listener still runs)") and
   return; if the priming open returns `None`, one info line ("pre-warm skipped (no
   reader; cache refresh requested)"). Root service only.
4. **Consumer availability verdict is cached and quiet.** `is_available`
   (`src/server/drm_capturer.rs:831-890`): an Ok-but-empty display list logs at info,
   "drm: availability probe -> no displays", and publishes Unavailable for the 30 s
   `NEGATIVE_TTL` (line 746). Probe errors increment toward `DRM_PROBE_MAX_FAILURES`
   (line 778, value 5) with info lines before Unavailable is published.
   `warm_availability` (964-985) retries 10 x 300 ms while `scrap::is_x11()`, then logs
   at info, "drm: consumer cache warm found no producer at startup (will probe
   lazily)". Routing gates use `is_available_cached()` (826) and never probe, so a
   stale negative verdict steers silently until something re-probes.
5. **Per-display build falls through to a dead fallback.** `wayland::
   get_capturer_for_display` (`src/server/wayland.rs:398-445`): if `is_available()`
   builds the DRM capturer and that build errors per-display, it logs warn,
   "drm capturer for display {} unavailable (...); falling back to PipeWire", and calls
   `ensure_pipewire_inited()`. At the greeter the PipeWire/portal path cannot produce
   frames (C5). The client-visible result is an authenticated session that waits for an
   image indefinitely. Note also the gate at the top of the same function (402-404):
   if the child's latched `is_x11()` is true, this function is never called at all and
   the X11 capturer serves the session instead — the C1 route, equally silent.
6. **Convert failure degrades in info lines.** The consumer picks the display's render
   node and calls `RenderConverter::open_render` in the unprivileged server
   (`src/server/drm_capturer.rs:418-445`). If it returns `None` — the exact signature
   of a render-node access failure pre-group-fix — the code logs at info, "drm:
   requesting the CPU-converted frame path for display {} (... drmtap_open_render
   found no usable /dev/dri/renderD*)", and sets need_cpu. `open_render` itself logs
   the failure at info (`libs/scrap/src/common/drm_render.rs`, ~41-44). During
   streaming, a WouldBlock from convert is ignored silently (`drm_capturer.rs`,
   ~line 516); other convert errors end the stream with an info line, "drm capture
   stream ended: {reason}", and set prefer-cpu.
7. **Producer worker closes with warns nobody sees.** `drm_capture_worker`
   (`src/ipc/drm.rs:970-1155`), opening the reader AS ROOT for the chosen
   (device, crtc): a failed open logs warn, "drm: failed to open crtc {}; closing
   _drm connection", and closes. `grab_desc` WouldBlock errors increment a stall
   counter silently; only after 150 stalls (~5 s at 33 ms) does it log at info, "drm:
   capture stalled (no frame); closing _drm connection". Unsupported switches to the
   CPU-mapped path with a warn; other errors log warn, "drm: capture error", and
   close.
8. **Authorization closes fail-closed with one warn.** Per-frame re-auth against the
   cache-only active uid (Section 3, C3): on mismatch or unknown, warn, "drm: _drm
   peer no longer matches the active session (or it is unknown); closing". Root
   service only.
9. **Demotion ends at the same dead end.** `DisplayHealth` (`drm_capturer.rs`): four
   zero-frame sessions demote a display to PipeWire for a 30 s+ doubling cooldown —
   at the greeter, demotion is indistinguishable from failure (C5).

**Lines that WOULD discriminate, if they were visible:** the pre-warm outcome line
(primes vs skips vs no-reader separates C2's producer half from everything else);
"availability probe -> no displays" vs a probe-error streak (enumeration/open failure
vs handshake failure); "drm: _drm peer no longer matches..." (C3, decisive);
"requesting the CPU-converted frame path" (C4's convert signature, decisive);
"capture stalled (no frame)" (a live but starved producer, pointing at C6/compositor);
"failed to open crtc" (producer-side open failure). None appear in the journal extract
for attempts 1, 2, 3, or 5. Establishing where the root service's output actually goes
on this package is a precondition for using any of them (T3).

## 5. (c) The handover seam

What happens at login (observation 4): the greeter `--server` is a per-session child of
the root service. At login the service stops it ("X connection to :0 broken" at
19:38:56) and spawns the user-side `--server` about 2 s later (19:38:58). The client's
media connection belongs to the old process and ends with it.

Client side: nothing carries that connection across, and the code shows exactly why.
In `src/client/io_loop.rs:261-270`, when the peer stream closes cleanly, the client
checks `is_restarting_remote_device()`. If set, it emits the "restarting" msgbox, and
the Flutter layer treats that as a reconnect control event (`flutter/lib/models/
model.dart:935-957`): an immediate `sessionReconnect` plus a second attempt after a
5 s silent window (`_restartReconnectSilentDelaySecs`, model.dart:59), inside a 5-minute
grace (`RESTART_REMOTE_DEVICE_GRACE`, `src/client.rs:99`; flag logic at 2789-2812). But
the flag is set in exactly one place: when the CLIENT itself requested the restart
(`src/ui_session_interface.rs:567` marks it before the client sends
`restart_remote_device`, `src/client.rs:2783`). A host-side session transition sets
nothing, on either side. The close therefore falls through to msgbox "error" /
"Connection Error" / "Reset by the peer" (`io_loop.rs:266-268`), which does not match
the one generic auto-retry the client has (`shouldAutoRetryOnOffline`, model.dart:
990-1020, requires the error text to be exactly "Remote desktop is offline"). Result:
error dialog, manual reconnect — precisely observation 4.

Server side: the root service performs the stop/start and knows a transition is
happening, but it owns no client connections and has no channel to ask the dying
`--server` to notify them. The dying `--server` sends nothing on exit.

**Smallest change that closes the seam, and its side: server side.** When the Linux
`--server`'s served session is ending (its session/desktop watcher observes the seat0
session change that will terminate it), it should send its connected sessions the
EXISTING "restarting" msgbox control event before exiting. The client already
implements the entire recovery for that event — immediate reconnect, a spaced second
attempt, a 5-minute grace window — and the successor `--server` registers the same
RustDesk ID with the rendezvous server, so the reconnect lands on the user session.
The observed 2 s spawn gap sits well inside the built-in cadence. No protocol change;
no client change required in principle. Scoping notes: fire only on an observed
session-transition termination, not on arbitrary process death, unless masking crashes
behind a reconnect window is an accepted trade; and the whole item is subject to the
owner's ruling — it is demonstrated on the fleet before any upstream contact, like
everything else here. The client-side alternative (auto-retry on any "Reset by the
peer") is broader, masks genuine failures, and duplicates machinery the protocol
already has; it is not the recommended side.

## 6. (d) Discriminating test plan

All tests are reads, or a service restart, on the fleet's own box, executed by the
owning chat / owner (this analysis contacts no host). Predictions are stated in
advance; each outcome names what it proves or kills.

**T1 — Cold-boot greeter server identity (QUEUED as an owner paste).** At a cold-boot
greeter, before any login: find the greeter `--server` pid; read
`/proc/<pid>/cmdline`, `/proc/<pid>/environ` (NUL-split), and its session id
(`ps -o sid= -p <pid>`); plus `loginctl` and
`loginctl show-session <greeter-id> -p Type -p Class -p Active -p State`.
- Environ carries `DISPLAY=:0` (+ XAUTHORITY into Xwayland) and NO `WAYLAND_DISPLAY`:
  the harvest took the Xwayland branch at boot; C1/C2's front-path story for cold boot
  is alive, and the producer/auth theories must explain why DRM never engaged behind
  it.
- Environ carries `WAYLAND_DISPLAY`: refresh classified the spawn as Wayland at boot;
  C1's front-path story dies for cold boot, and weight shifts to C3/C6 and the
  producer.
- The pid's own session id is NOT the greeter's logind session (expected, per the sudo
  spawn): confirms the child's self-classification fallbacks describe the service's
  context, not the greeter's (Section 2.5) — a code-reading check made visible.

**T2 — Same reads at a logout-spawned greeter.** Repeat T1 immediately after a greeter
server spawns on logout (the observation-3 configuration), before connecting.
- Any environ delta vs T1 is a spawn-context fact with config held constant: it
  directly evidences C2's environment half and names the differing variable.
- Identical environs: C2's environment half dies; weight moves to producer warm state
  (C2's other half), C3, and C6.

**T3 — Unfiltered root-service output across one cold-boot attempt.** First establish
where the service's stdout goes (`systemctl cat rustdesk.service`, StandardOutput),
then capture the service's complete output from boot through one failed connection
attempt — unfiltered, not greps around the greeter pid.
- "session still reads as X11 after 30s; skipping the pre-warm": confirms the
  producer never primed at cold boot (C2 producer half, first half).
- "drm: _drm peer no longer matches the active session (or it is unknown)": C3
  confirmed for that attempt, decisively.
- "drm: capture stalled (no frame)": producer alive but starved; points at
  C6/compositor scanout.
- "availability probe -> no displays": enumeration/open failure on the producer side
  (step 1-2 of the silent chain).
- "requesting the CPU-converted frame path": C4's convert signature — would mean the
  front path DID engage and the render node failed even with groups held (revives C4
  in ACL form; pair with T4).
- Total absence of every DRM line: the consumer never asked the producer — routing in
  the child (C1/C2 latch) is where the attempt died.

**T4 — Device access as the greeter user, at cold boot.**
`sudo -u cosmic-greeter test -r /dev/dri/renderD128`, same for the card node,
`ls -l /dev/dri`, `getfacl` on the render node, and the greeter session's Active state.
- Render node unreadable despite group membership (ACL not applied, or a by-path node
  differing): C4 revives in ACL form for cold boot.
- Readable: C4 is closed for cold boot; observation 5's cause sits upstream of the
  convert (routing, producer, or auth).

**T5 — Compositor timing at cold boot.** From the boot journal: timestamps for
cosmic-comp start, its DRM-master lines, greetd session registration, rustdesk.service
start, and the greeter `--server` spawn.
- cosmic-comp becomes master (or logs its master failure) AFTER the greeter server /
  producer start: C6's timing form is live.
- Master settled seconds before the spawn: C6 weakens sharply.

**T6 — Service-restart-only replication (cheapest decisive test).** At a cold-boot
greeter, with no reboot and no login: restart ONLY rustdesk.service, then connect.
- Image appears: spawn context governs (C2 in some form) — this replicates
  observation 3's restart+respawn conditions minus the logout. Pure static
  classification (C1 alone) dies, since Type, groups, and config are untouched.
- Still blank: C2's service-history half weakens; the difference between 3 and 5 then
  sits in the logout transition itself (session history, ACL state) or in C3/C6;
  T3's lines during T6 discriminate further.

**T7 — Wait-and-retry at cold boot.** Connect (fails), wait >60 s untouched (past the
30 s negative TTL and the pre-warm budget), reconnect.
- Second attempt works: a TTL/cache state was the gate (C2 cache half / step 4 of the
  silent chain).
- Identical failure: cache theories weaken; the block is persistent state
  (classification latch, auth, or compositor), not a stale verdict.

**T8 — Handover confirmation (after any capture fix).** Connect to the greeter, log in
remotely, record the client's exact behavior: "Reset by the peer" error dialog vs any
auto-reconnect. Confirms Section 5's reading on the live build and scopes the
handover fix's field test.

Sequencing: T1+T3+T4+T5 are one bundle at one cold boot (T1 is the already-queued
owner paste; T3-T5 are reads the owning chat can take over SSH while the box sits at
the login screen). T6 and T7 follow in the same sitting if T1-T5 leave C2 and C3 both
alive. No test requires code changes.

## 7. Open unknowns this analysis does not close

- The successful frame source in observation 3 is unidentified (Section 3). Every
  candidate above is scored for consistency with the record, not for proven mechanism.
- Where the root service's log output actually lands on this package (T3's first
  step). If it is a file rather than the journal, several "absent lines" readings in
  the existing record need re-examination.
- libdrmtap's master requirements for `drmtap_open` were not fully traced in this
  pass; C6 remains a field question by design.
- Whether `SUDO_E_PRESERVES_ENV` holds on the other fleet machines (it changes
  Section 2.4's inheritance analysis per machine, and the owner's ruling requires
  whatever works to be applied fleet-wide, so this must be re-derived per machine,
  not assumed).

## 8. Bottom line

1. The classification defect is real code and stays on the record; the claim that it
   is the settled sole cause is withdrawn. It cannot explain observation 3.
2. The group fix is necessary and proven insufficient. It cannot explain
   observation 5.
3. The prime suspect is spawn-context state — child latches, harvested and inherited
   environment, producer warm state, and service caches — because it is the only
   candidate that engages the actual delta between the one success and the cold-boot
   failure. T6 is the cheapest decisive test; T1 (already queued) plus T3-T5 form
   the evidence bundle.
4. The handover seam is a separate, confirmed defect with a traced cause and a
   smallest fix on the server side, reusing the client's existing "restarting"
   reconnect machinery.
5. Per the owner's ruling: nothing here goes upstream, and no patch is designed
   beyond the handover note in Section 5, until a fix is demonstrated working on
   the fleet — cold boot included — and whatever works is then applied to the other
   machines in use.
