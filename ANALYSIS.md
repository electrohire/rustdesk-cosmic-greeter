# RustDesk unattended-Wayland preview: blank frames at the COSMIC greeter — analysis

Date: 2026-11-08. Host of record: tower-offsite, Pop!_OS 24.04, COSMIC + cosmic-greeter
under greetd, NVIDIA RTX 5060 Ti, proprietary driver 595.84, Wayland.
Package: rustdesk-unattended-wayland 1.5.1 (the `drm`-feature build of upstream PR #15420).
Local tree: `repo/` at PR head `e180b38b58a9e940d4522e460e446e19832e2838`; `libdrmtap/` at
rustdesk-org/libdrmtap main. All line numbers below are from those two trees.

Field facts relied on (verified on the host, fleet record): the RustDesk service is active
and enrolled; two pre-login connections (second on a fresh boot, greeter visibly awake on
the monitor) authenticated, stayed connected, and delivered a blank frame; after a physical
login the same build captured the logged-in COSMIC session normally. Host journal extract:
`~/workspace/tower-offsite-staging/records/rustdesk-journal-greeter-attempts-2026-11-08.txt`
(sha256 44fabf35…).

## (a) Upstream state of PR #15420

- rustdesk/rustdesk PR #15420, "feat(linux): DRM/KMS direct capture for Wayland — no portal
  consent required", author fxd0h, head branch `feature/drm-capture-clean` (fork), base
  master, +6384/−12 across 20 files. **State: MERGED** (last updated 2026-10-07). The
  1.5.1 unattended package is built from this line (`.github/workflows/drm-capture.yml`,
  `build.py --drm` produces the `rustdesk-unattended-wayland` deb).
- Approach: an opt-in `drm` Cargo feature, off by default. The root `--service` (which
  already holds CAP_SYS_ADMIN) dlopens `libdrmtap.so` (built in CI from the RustDesk-controlled
  fork rustdesk-org/libdrmtap, pinned 0.5.2) and reads the KMS scanout in-process; frames
  stream to the unprivileged per-user `--server` over a dedicated authorized `_drm` IPC
  channel; the server encodes and serves the client. Priority DRM → PipeWire → X11, per
  display, with demotion/cooldown on failure. No helper binary, no setcap.
- Tested upstream (PR body): Intel i915 (GNOME), AMD RX560 (GNOME + KDE, and it is their
  GDM-greeter test bed), NVIDIA **Jetson Orin Nano** (nvidia-drm). Desktop NVIDIA proprietary
  + COSMIC/greetd is **not** in the tested matrix.
- Components: `libs/scrap/src/common/drmtap_dl.rs` (dlopen loader, ABI gate),
  `libs/scrap/src/common/drm_reader.rs` (service-side read/validation),
  `libs/scrap/src/common/drm_render.rs` (unprivileged dma-buf→pixels conversion),
  `src/ipc/drm.rs` (producer + `_drm` channel, in the root service),
  `src/server/drm_capturer.rs` (consumer: availability probe, capturer, demotion),
  `src/server/wayland.rs`, `src/server/display_service.rs`, `src/platform/linux.rs`
  (integration + greeter-session spawn), `src/ipc.rs`/`src/ipc/auth.rs` (channel + auth).

## (b) The capture code path, end to end

**Session classification (decides everything).** Capture selection keys off one boolean:
`is_x11()`. In `src/platform/linux.rs:47` it is a lazy static
`hbb_common::platform::linux::is_x11_or_headless()`; scrap re-exports the same
(`libs/scrap/src/common/mod.rs:270`). In hbb_common (`libs/hbb_common/src/platform/linux.rs`):

- `is_x11_or_headless()` (line 128) = `!is_desktop_wayland()`; `is_desktop_wayland()`
  (line 123) = `get_display_server() == "wayland"`.
- `get_display_server()` (line 135) takes the first active seat0 session from `loginctl`
  (`_get_values_of_seat0`, line 213) and asks `get_display_server_of_session(sid)`.
- `get_display_server_of_session()` (line 167) reads the session's logind `Type`. If Type is
  empty, `"tty"`, or `"unspecified"`, it falls back to **the calling process's own**
  `XDG_SESSION_TYPE` env var, and if that is unset it **defaults to `"x11"`**.

The video service branches on this: `src/server/video_service.rs:397`
(`get_capturer_monitor`: `if !is_x11()` → Wayland path, else the X11 `Display::all()`
capturer). The DRM path is only reachable inside the Wayland branch:
`src/server/wayland.rs:176` (`ensure_inited`) and `:398` (`get_capturer_for_display`) gate
on `drm_capturer::is_available()` / `is_available_cached()`; the probe itself
(`src/server/drm_capturer.rs:831`) queries the root service over `_drm`. The PR authors
document the classification trap themselves, twice: `src/server/drm_capturer.rs:965`
("`get_display_server()` answers "x11" whenever loginctl cannot yet name the seat0
session") and `src/server.rs:609` (same observation about `is_x11()`).

**How the per-session server is spawned.** The root service tracks the seat0 session in
`Desktop::refresh()` (`src/platform/linux.rs`, `mod desktop`) and spawns `rustdesk --server`
as the session user via sudo, with an environment harvested from the session
(`try_start_server_`, ~line 685): it pushes only non-empty `DISPLAY`, `XAUTHORITY`,
`WAYLAND_DISPLAY`, `HOME`, `DBUS_SESSION_BUS_ADDRESS`, `TERM` (plus a forced-display-server
override). **It never pushes `XDG_SESSION_TYPE`.** The host journal confirms the spawn
shape: `sudo root -> USER=cosmic-greeter, ENV=XDG_RUNTIME_DIR=/run/user/987,
COMMAND=/usr/share/rustdesk/rustdesk --server`. So a greeter server process has no
`XDG_SESSION_TYPE` of its own, and the classification fallback above reads an empty env.

**Greeter recognition.** There is a designed greeter branch: `Desktop::is_login_wayland()`
(`src/platform/linux.rs:1803`) = `is_gdm_user(username) && protocol == "wayland"`, and when
it fires, `refresh()` deliberately gives the greeter child **no compositor environment at
all** (comment at ~lines 2085–2096: the drm path "talks to the root service over `_drm` and
to a render node, never to the compositor or the portal, which is the entire reason it works
at a login screen"). But `is_gdm_user` (`libs/hbb_common/src/platform/linux.rs:117`) matches
only the usernames `"gdm"` and `"sddm"` (lightdm is present but commented out). The PR's own
comment at `src/platform/linux.rs:2097`–2103 admits the list is stale: "a current GDM runs
its greeter as `gdm-greeter`, which that helper does not match — measured on the test host,
where the greeter server therefore takes the branch below and gets a fully populated
environment." For our host the username is `cosmic-greeter`, equally unmatched, so the
greeter server takes the ordinary-session branch: `get_display_xauth_xwayland()` harvests
`DISPLAY=:0`, `XAUTHORITY`, `WAYLAND_DISPLAY`, and a session-bus address from the greeter
session's Xwayland/portal processes owned by uid 987, and the child is spawned with all of
them set. The same username test also guards the greeter-skip in
`_get_values_of_seat0(ignore_gdm_wayland = true)` (hbb_common line 213 ff.), the variant
`get_display_server()` itself uses.

**The DRM producer (root service).** Started unconditionally with the service when the
feature is on: `src/platform/linux.rs:909` spawns the "drm-producer" thread running
`crate::ipc::start_drm()` (`src/ipc/drm.rs:675`). It is session-agnostic: it enumerates DRM
devices and serves whichever authorized server asks. Authorization is in
`src/ipc/drm.rs:728` (`drm_peer_authorized`: peer uid must be 0 or equal the active session
uid from the service-loop cache) enforced in `handle_drm_conn` (line 737) and re-checked
per frame (line ~889, fail-closed).

**The actual scanout read (libdrmtap).** Pipeline (libdrmtap `src/drm_grab.c:11`–27): find
the primary plane for the target CRTC → refresh `fb_id` via `drmModeGetPlane` (never
cached) → `drmModeGetFB2` (format, handles, strides, modifier — modifier trusted only when
`DRM_MODE_FB_MODIFIERS` is set, `fb2_effective_modifier`) → `drmPrimeHandleToFD` → either
the mapped CPU path or the zero-copy dma-buf export consumed by the unprivileged converter
(EGL detile in `drm_render.rs`). NVIDIA has a dedicated backend (`src/gpu_nvidia.c`):
block-linear modifiers have **no CPU decoder** and return `-ENOTSUP` with an explicit error
(line 84), steering the flow to the EGL path. On the Rust side,
`libs/scrap/src/common/drm_reader.rs` `grab()` (lines 168–285) validates hard: null data or
zero geometry → `WouldBlock`; stride < w*4 or a non-BGRA fourcc → error
"unsupported DRM scanout format". **Nowhere in this stack is a zero-filled frame fabricated
and shipped as valid**; the consumer additionally tracks zero-frame streaks and demotes a
display that delivers them (`src/server/drm_capturer.rs` health/demotion, ~lines 87–331).

**X11 path.** If `is_x11()` is true, the server uses scrap's X11 capturer against the
process's `DISPLAY`. On a Wayland host that display is XWayland. XWayland's root window is
never painted by the compositor (the greeter UI is a native Wayland surface of the greeter's
cosmic-comp instance); an X image grab of the root returns its black background, at the
correct geometry, indefinitely, with no error. The host journal shows the greeter server
attached to X `:0` ("X connection to :0 broken" at its exits) and using a display event
loop throughout both attempts.

**Portal path.** Requires the session bus + xdg-desktop-portal ScreenCast. The journal
shows the greeter server failing to reach any session bus at startup ("Unable to get the
session bus: Could not connect: Permission denied", twice, only for the cosmic-greeter
server; the tpierson servers log no such failure). Portal failure surfaces as an error
(`SCRAP_XDP_PORTAL_UNAVAILABLE` mapping in `src/server/wayland.rs` `map_err_scrap`), not as
frames.

## (c) Root-cause candidates for "blank at the greeter, fine after login", ranked

### 1. LEADING — the greeter session is classified as X11, so the server captures the greeter's XWayland root window (black forever) and never attempts DRM or portal

Chain, every link traced in code above: (i) pre-login, the only seat0 session is the
cosmic-greeter session; (ii) greetd greeter sessions are a known case where
`XDG_SESSION_TYPE` is not set before `pam_open_session` (greetd sets it per *started user
session* from the session file; its own greeter session commonly lands in logind as
Type `tty`/`unspecified` — the greetd project's own history adds an `initial_session`
service knob for exactly this registration problem); (iii) `get_display_server_of_session`
then falls back to the server process's `XDG_SESSION_TYPE`, which is unset by construction
(sudo spawn, `try_start_server_` never propagates it), so classification hard-defaults to
`"x11"`; (iv) `is_x11()` true → X11 capturer on `DISPLAY=:0` → XWayland root → steady
black frames at full geometry, session healthy, no errors — the exact field observation;
(v) independently, `is_gdm_user` not matching `cosmic-greeter` means the designed
no-environment greeter spawn never happens, so the child even receives the harvested
`DISPLAY=:0`/XAUTHORITY that makes the X11 capture succeed silently instead of failing.

For: explains every observation with no extra assumptions (blank but stable connection,
no DRM/Wayland capture ever logged for the greeter server, X attach to `:0` in the journal,
session-bus failure being irrelevant because the portal is never reached, both attempts
identical including on a fresh boot, in-session capture working because the user session's
logind Type is `wayland` — cosmic-greeter passes `XDG_SESSION_TYPE=wayland` in StartSession
for user sessions). The two author comments (drm_capturer.rs:965, server.rs:609) show the
"x11 by default on any loginctl ambiguity" behavior is real and known-fragile.

Against: it depends on one host fact not yet in hand — the greeter session's actual logind
`Type` on this box (see (d)). If Type is `wayland`, this candidate collapses and #2 leads.

### 2. The DRM path IS engaged at the greeter, but the greeter's scanout cannot be converted (NVIDIA tiling/format), yielding demotion churn that presents as blank

For: desktop NVIDIA proprietary is untested upstream (only Jetson); the greeter's
cosmic-comp instance may scan out a different format/modifier than the user session's;
libdrmtap's NVIDIA CPU path openly returns -ENOTSUP for block-linear, so everything rides
on the EGL import in the unprivileged converter succeeding for the greeter's buffers.

Against: the same GPU and compositor family capture fine after login, and the producer is
session-agnostic; more importantly, this stack does not ship silent zeros — reader-side
validation errors, -ENOTSUP, and converter failures all become errors, demotion, and
PipeWire fallback (which here is dead), i.e. log noise and connection errors, not a placid
blank stream. Would also expect "drm:" probe/converter lines in the greeter server's log.

### 3. The `_drm` producer refuses the greeter server (active-uid authorization)

For: authorization is fail-closed and keyed to the service-loop active-uid cache; a stale
or empty cache pre-login would refuse uid 987.

Against: pre-login the active seat0 session *is* the greeter session under the
non-skipping enumeration, so active uid = 987 = peer uid; and a refusal produces probe
failure → DRM unavailable → portal attempt → visible error, not blank frames.

### 4. The portal/PipeWire fallback itself produces the blank

Against, decisively: the journal proves the greeter server has no working session bus, so
the portal cannot even initialize; its failure mode is an error message to the client, not
frames.

### 5. Nothing is being scanned out at the greeter (display asleep / connector disabled)

Eliminated by the field record: attempt 2 ran on a fresh boot with the greeter visibly
awake on the box's own monitor — a live scanout existed on the very CRTC in question.

## (d) Host-side evidence that discriminates

The greeter-user logs are **still needed**; the code plus the service journal make #1 the
clear leader but do not settle it, because candidate #1 turns on the greeter session's
logind `Type`, which is a host fact. Any ONE of these settles it:

1. `loginctl show-session <greeter-sid> -p Type -p Class` while the greeter is up (or the
   greeter server log's own classification lines). Type `tty`/`unspecified`/empty →
   candidate #1 proven. Type `wayland` → #1 dead, go to #2.
2. The greeter server's RustDesk log (`/var/lib/cosmic-greeter/.config/rustdesk/log/`,
   already requested via owner paste). Discriminating lines: X11-capturer creation / absence
   of any `drm:` lines → #1 (the Wayland branch logs its probe as
   `drm: availability probe -> ...` / `drm: consumer cache warmed ...`). Presence of
   `drm: availability probe -> available` followed by grab/convert errors or demotion
   warnings → #2. `_drm peer no longer matches the active session` (ipc/drm.rs:891) → #3.
   `org.freedesktop.portal` / `SCRAP_XDP_PORTAL_UNAVAILABLE` errors → the fallback was
   reached, meaning classification said Wayland and DRM was unavailable (splits #2/#3 from #1).
3. The root service's own log for the attempt windows: which `Desktop` branch ran for the
   greeter (whether `DISPLAY` was set for the child via `set_x11_env`-style logging) —
   corroborates the spawn-side half of #1.

## (e) Smallest upstreamable fix design (for candidate #1)

Two small, independently safe changes; the first is the root fix, the second restores the
PR's own designed greeter behavior. Both are in hbb_common / rustdesk Linux platform code,
no libdrmtap change, no behavior change for sessions logind types correctly.

1. **Read the session's own `XDG_SESSION_TYPE`, not the caller's**
   (`libs/hbb_common/src/platform/linux.rs`, `get_display_server_of_session`, line 167).
   When logind `Type` is empty/`tty`/`unspecified`, the current fallback reads
   `std::env::var("XDG_SESSION_TYPE")` — the env of whatever process happens to ask, which
   for a service-spawned `--server` is empty by construction, guaranteeing the `"x11"`
   default for exactly the unattended case this feature exists for. Instead, resolve the
   session leader's pid (`loginctl show-session -p Leader`) and read `XDG_SESSION_TYPE`
   from `/proc/<leader>/environ` (readable by the root service and by the session user;
   degrade to today's behavior on any failure). This fixes classification for every
   service-spawned server, not just greeters.
2. **Recognize greeters as greeters** (`is_gdm_user`, hbb_common line 117, or a new
   `is_greeter_user` used by `Desktop::is_login_wayland` and the seat0 skip logic). Cover
   the greeter usernames display managers actually use today — `gdm`, `gdm-greeter`,
   `sddm`, `lightdm` (currently commented out), `cosmic-greeter`, `greeter` — or, more
   robustly, key off the logind session `Class=greeter` property, which greetd/GDM/SDDM all
   set and which cannot drift with renames. Effect: greeter servers for all these DMs take
   the designed branch (no compositor env, DRM-only), and `get_display_server()`'s
   greeter-skip works for them as it already does for gdm/sddm.

With (1) alone, a greetd greeter whose compositor exports `XDG_SESSION_TYPE=wayland` in its
session classifies correctly and the DRM path engages. With (2) alone, the spawn env stops
arming the X11 trap but classification can still default wrong — so (1) is the fix to lead
the PR with; (2) is a companion commit.

## (f) Patch decision

**No patch is committed.** The task's gate is a draft patch only if one cause is clearly
established by the code alone. Candidate #1 is established by code + journal + greetd's
documented registration behavior, but its final link — this host's greeter session
`Type` — is a host fact that is one `loginctl` read away and has not been read yet. If that
read returns `wayland`, a classification patch would change nothing on this host and
candidate #2 (libdrmtap/converter, a different repo and a different fix) would lead.
Committing the patch before that read would be building on an unverified premise. The fix
design in (e) is ready to implement the moment the discriminator in (d) lands; the
discriminating log files have already been requested from the host side. No build check was
run, because there is no patch to check (no payload exit code exists to report).
