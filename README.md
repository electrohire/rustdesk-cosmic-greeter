# RustDesk on COSMIC: greeter capture, handover, and CM crash — field analysis

Field investigation by ElectroHire into three RustDesk defects found
operating a small fleet of Pop!_OS 24.04 / COSMIC machines with
RustDesk 1.5.1, including the `rustdesk-unattended-wayland` build (the
DRM capture line from rustdesk/rustdesk PR #15420, merged upstream).

Filed upstream as three issues on rustdesk/rustdesk (links below).
This repo is the evidence package behind them: the full code-traced
analysis, the re-analysis against the complete field record, the
first-hand scoring of the greeter server's own logs, and a prototype
patch.

## The three findings

1. **Greeter capture is blank.** At the COSMIC greeter (greetd +
   cosmic-greeter), connections authenticate but deliver black frames
   forever. The greeter server's own logs prove every connection was
   captured by the X11 scrap capturer and the DRM/Wayland path was
   never entered: the greeter's logind session is `Type=tty`, session
   classification falls back to the server process's own (unset)
   `XDG_SESSION_TYPE` and defaults to "x11", and `is_gdm_user` does
   not recognize `cosmic-greeter`, so the designed greeter spawn never
   happens either.
2. **Login handover drops the client.** When the user logs in, the
   greeter server is replaced by the user-session server; the client
   gets "Reset by the peer" and must reconnect manually, because the
   host-side transition never sends the "restarting" event the
   client's existing auto-reconnect machinery waits for.
3. **The connection manager crashes per-session on COSMIC.** On a
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
