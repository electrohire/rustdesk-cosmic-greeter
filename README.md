# RustDesk on COSMIC: greeter capture, handover, and CM crash — field analysis

Field investigation by ElectroHire into three RustDesk defects found
operating a small fleet of Pop!_OS 24.04 / COSMIC machines with
RustDesk 1.5.1, including the `rustdesk-unattended-wayland` build (the
DRM capture line from rustdesk/rustdesk
[PR #15420](https://github.com/rustdesk/rustdesk/pull/15420), merged
upstream).

Filed upstream as three issues on rustdesk/rustdesk:

1. [Greeter capture is blank](https://github.com/rustdesk/rustdesk/issues/17084)
2. [Login handover drops the client](https://github.com/rustdesk/rustdesk/issues/17085)
3. [Connection manager crashes per-session on COSMIC](https://github.com/rustdesk/rustdesk/issues/17086)

Related upstream reports, found by a wider duplicate check after
filing:

- [#16891](https://github.com/rustdesk/rustdesk/issues/16891) reported
  the same greeter symptom on COSMIC/greetd (different hardware) and
  was fixed by
  [#16894](https://github.com/rustdesk/rustdesk/pull/16894), merged
  2026-11-08: its cause section names the same two classification
  defects traced here, and the reporter confirmed the fix on their
  machine. Our issue #17084 stands as a second-hardware confirmation
  on the 1.5.1 release build; our prototype patch is superseded by
  #16894's approach, and verifying a build containing #16894 on the
  affected host (cold boot straight to the greeter included) is the
  open step.
- [#15892](https://github.com/rustdesk/rustdesk/issues/15892) covered
  the logout direction of the handover seam in issue #17085 and is
  closed.
- [#16297](https://github.com/rustdesk/rustdesk/issues/16297) reported
  the `--cm` EGL crash family on KDE Plasma (Intel) and is closed;
  issue #17086 is the COSMIC (NVIDIA) environment, with an additional
  crash signature.

This repo is the evidence package behind them: the full code-traced
analysis, the re-analysis against the complete field record, the
first-hand scoring of the greeter server's own logs, and a prototype
patch.

## The three findings

1. **Greeter capture is blank.**
   ([upstream issue #17084](https://github.com/rustdesk/rustdesk/issues/17084)) At the COSMIC greeter (greetd +
   cosmic-greeter), connections authenticate but deliver black frames
   forever. The greeter server's own logs prove every connection was
   captured by the X11 scrap capturer and the DRM/Wayland path was
   never entered: the greeter's logind session is `Type=tty`, session
   classification falls back to the server process's own (unset)
   `XDG_SESSION_TYPE` and defaults to "x11", and `is_gdm_user` does
   not recognize `cosmic-greeter`, so the designed greeter spawn never
   happens either.
2. **Login handover drops the client.**
   ([upstream issue #17085](https://github.com/rustdesk/rustdesk/issues/17085)) When the user logs in, the
   greeter server is replaced by the user-session server; the client
   gets "Reset by the peer" and must reconnect manually, because the
   host-side transition never sends the "restarting" event the
   client's existing auto-reconnect machinery waits for.
3. **The connection manager crashes per-session on COSMIC.**
   ([upstream issue #17086](https://github.com/rustdesk/rustdesk/issues/17086)) On a
   COSMIC Wayland session, the `--cm` Flutter process crashes within
   seconds of every session start (two distinct crash signatures,
   plus GStreamer assertions on the PipeWire path), resetting every
   session on both the standard and the preview 1.5.1 builds.

## Environment of record

- tower-offsite (greeter findings): Pop!_OS 24.04, COSMIC,
  cosmic-greeter under greetd, NVIDIA RTX 5060 Ti (driver 595.84),
  rustdesk-unattended-wayland 1.5.1 at PR #15420 head `e180b38b`
- tower-compute (CM crash): Pop!_OS 24.04, COSMIC Wayland (display
  `:1` via XWayland), NVIDIA RTX 5060 Ti, RustDesk 1.5.1 standard and
  preview builds
- Self-hosted hbbs/hbbr (OSS) rendezvous/relay for all sessions

## Status, honestly

- Finding 1's classification defect is proven in code and in the
  host's own logs. It is not the whole story: one logout-spawn on a
  warm system produced a greeter image once (still via the X11 path),
  and spawn-context state also plays a role. The addendum scores
  every candidate against all five field observations and lists the
  discriminating tests; the producer side has never been observed
  alive on the greeter host, which stays an open question for the
  field test.
- A prototype two-part patch exists in `patch/` (session type from
  the session leader's environment; greeter recognition by logind
  class and the modern greeter account set). `cargo check` passes
  for the `hbb_common` crate one half lives in. The full build was
  blocked in our environment by a missing system gstreamer dev
  library, and the field test has not run yet. It is a prototype,
  not a demonstrated fix.
- Findings 2 and 3 are traced to their mechanisms in the documents
  below; no patch is offered for them yet.

## Documents

- `ANALYSIS.md` — the original code-traced analysis of the greeter
  failure: capture path end to end, ranked root-cause candidates,
  the discriminating evidence, and the fix design.
- `ANALYSIS-ADDENDUM.md` — re-analysis against the full field record.
  It demotes the first analysis's leading cause to a contributing
  factor, traces the spawn-context dependence, maps the
  silent-failure chain, documents the handover seam with its
  smallest fix, and gives the discriminating test plan. Read this
  one for the current state of knowledge.
- `FILE-LOG-SCORING-2026-11-09.md` — first-hand scoring of the
  greeter server's own log files: settles that every connection in
  every spawn used the X11 capturer and no DRM capturer was ever
  created. It also flags that `PATCH-NOTES.md` predates the patch
  implementation pass.
- `PATCH-NOTES.md` — the patch state record as of 2026-11-08
  (superseded in part by the scoring document above; kept for the
  record).
- `patch/rustdesk-platform-linux.diff` — the `src/platform/linux.rs`
  half of the prototype patch, against PR #15420 head `e180b38b`.
- `patch/hbb_common-greeter-session.diff` — the `hbb_common` half:
  two commits (session type from the session leader's environment;
  greeter recognition) against hbb_common `7e1c392`.

## License

MIT (see LICENSE). The analysis documents are shared so the upstream
project and other COSMIC users can use them; attribution appreciated.
