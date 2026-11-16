# Greeter server file logs — first-hand scoring (2026-11-09, Desktop Access chat)

Evidence set: `~/workspace/tower-offsite-staging/records/greeter-rustdesk-logs/`
(server/ + check-hwcodec-config/, 14 files, pulled from tower-offsite
2026-11-09 ~02:45 ET; local hashes computed here). Key file:
server/rustdesk_rCURRENT.log, 55,200 B, sha256
c1396a6ddb674981708f5da4fc8f48779fc29d3162cd38ed069d9048b7242490 — the
19:49:21 cold-boot (boot ebc27d7a) greeter server's own log, including the
owner's authenticated black attempt. Scored against T1/T3 and the
ANALYSIS-ADDENDUM.md section 4(b) discriminating lines. All greps run
first-hand on the delivered files.

## Findings

1. **Every connection, in every spawn, captured via X11 scrap. No DRM
   capturer was ever created.** All 7 server logs repeat the same sequence
   per connection: "#displays=1, current=0, width=1920, height=1080,
   name:HDMI-A-1" (video_service.rs:487), "Try create capturer with
   capturer privacy mode id 0" (:516), "Create capturer from scrap" (:365),
   and "X11 pointer has 10 buttons" (enigo xdo). Scrap creation counts per
   log: 18-57-26: 2, 19-10-02: 4, 19-12-14: 1, 19-38-37: 4, 19-41-57: 2,
   19-42-49: 1, rCURRENT: 2. A grep for any DRM capturer creation across the
   set returns nothing. This is the addendum's C1 route observed directly:
   the child's latched is_x11() meant wayland::get_capturer_for_display was
   never called, so no per-display DRM build, no PipeWire fallback warn, no
   convert lines — the X11 capturer served every session, silently.

2. **The consumer cache-warm line is the ONLY drm line in the entire set,
   and it appears in all 7 spawns.** "drm: consumer cache warm found no
   producer at startup (will probe lazily)" (drm_capturer.rs:2041):
   18:57:29, 19:10:06, 19:12:18, 19:38:41, 19:42:01, 19:42:53, 19:49:24.
   No log ever records a producer being found, and none of the addendum's
   other discriminating lines appear anywhere: no "availability probe ->
   no displays", no "_drm peer no longer matches", no "requesting the
   CPU-converted frame path", no "capture stalled (no frame)", no "failed
   to open crtc". Consequence: "no producer at startup" is a host-wide
   constant, NOT a cold-boot variable — the C2 producer half cannot be what
   distinguishes the 19:38 sitting from the cold boots. Combined with T3
   (the root --service emitted zero DRM/pre-warm lines across the cold
   boot), the producer side has never once been observed alive on this
   host. That question stays open for the field test (see below); it is not
   a build-scope item.

3. **19:38 vs the cold-boot pair — the tower report's distinguishing marks
   hold against that pair, with two refinements.** In 19-38-37: SWITCH
   error (service.rs:308) at 19:38:38.497, 0.080 s after the first capturer
   creation (19:38:38.417), an immediate second display/capturer cycle, and
   3 display inner-loop entries inside ~19 s before the owner's physical
   login ended the greeter server at 19:38:56. Neither cold-boot log has a
   SWITCH. Refinements: (a) SWITCH is not unique to 19:38 in the full set —
   18-57-26 (x2) and 19-10-02 (x2) also record it, in the same position
   (~0.08–0.1 s after a capturer creation, followed by re-creation); it
   marks a display switch/re-creation event, not the owner's presence as
   such. (b) rCURRENT had TWO connection cycles, not one: #1266 entered the
   display loop 19:49:41.70 and exited 19:49:45.67 (3.96 s), #1267 exited
   19:50:05.65 with "Connection closed: Peer close". 19-42-49 had one.
   Shape conclusion unchanged: every cycle is scrap, every display loop is
   short, and the server's last line (19:54:21) is the X11 connection dying
   when the owner logged in physically.

4. **Rendezvous color, corrected.** The tower summary called the startup
   network noise "DNS failures". First-hand it is mixed: the 19-12-14 spawn
   shows genuine transient name-resolution failures ("Temporary failure in
   name resolution", including the rendezvous mediator); the 18-57-26 spawn
   and rCURRENT (43 mentions of :21114) show the name resolving fine (nat
   response from mini-server.tailbfa48d.ts.net:21116, 3.1 ms latency) with
   ConnectionRefused on the hbbs HTTP API port 21114 and TCP-proxy fallback
   via 21116. Either way: color, not cause — sessions were established
   through the relay path in the same logs.

5. **Addendum premise corrected by this delivery.** Section 4(b) opens with
   "the greeter user has NO session logs at all (host-verified:
   /var/lib/cosmic-greeter/.config/rustdesk/ holds only the two config
   files...)". That is falsified: the logs live under
   .local/share/logs/RustDesk, and this delivery is exactly the consumer-side
   witness the addendum scored as invisible by construction. (Noted
   2026-11-08 ~20:41 ET; the addendum text still carries the old claim.
   PATCH-NOTES.md is likewise stale: it records the patch as halted and
   unimplemented, but a later implementation pass produced the patch tree
   the 2026-11-08 build rerun verified at step 0. Both files want an
   amendment pass; not done here.)

## What this settles and what it does not

- SETTLED: the consumer-side unknowns are closed. The failure shape scored
  from T3 ("consumer never asked the producer") is confirmed and sharpened:
  the consumer asked once at startup in every spawn, found no producer, and
  no connection ever reached the DRM path at all — routing (the C1 latch)
  sits in front of everything. The patch's routing scope is final for the
  BUILD; the file logs add no new code obligations to it.
- NOT SETTLED: whether a producer exists once routing lets the consumer
  probe for real. First field-test watch items, in order: (i) on the
  patched build's first greeter connection, which discriminating line
  appears — a producer interaction, "availability probe -> no displays",
  "capture stalled", or "failed to open crtc" (the last two are root-side,
  so the sitting must also capture the root service's output, file logs
  included if the package writes them); (ii) T5, the greeter compositor's
  DRM-master state; (iii) the open environ question, whether the greeter
  session leader's environ carries XDG_SESSION_TYPE (patch part 1).
- The field test remains gated on: the build verdict (libwebm fetch
  authorization, owner, pending), and the owner's cold-boot hold (no
  tower-offsite reboot while the Spec 049 campaign runs).
