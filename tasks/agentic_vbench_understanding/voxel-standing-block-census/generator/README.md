# How the recording was made

A scripted camera rig inside a Luanti 5.x server running VoxeLibre, captured
off an Xvfb display with software rendering. Nothing is hand-authored: the
world comes from mapgen v7 at `fixed_map_seed = 7`, the rig picks its own
centre and route, and the event log it writes is the ground truth.

| file | what it is |
|---|---|
| `avbcam_init.lua` | the rig mod. Walk, view cone, block palette and its drifting shares, lifetimes, HUD suppression, weather hold, event and camera-pose logging. |
| `h200_run2.sh` | the capture: reset the world, start Xvfb, server and client, start ffmpeg, drop the GO file the rig waits on, stop everything, report. |
| `h200_server.conf`, `h200_client.conf` | the settings that matter, notably `fixed_map_seed = 7`, `avb_centre = 0,0`, `node_highlighting = none` and `selectionbox_width = 0` so the client draws no wireframe around the block under the crosshair. |
| `rig_events.log` | the raw log this recording produced. Columns: time, action, node name, position, extra, camera pose. |
| `gt6_fixed.py` | replays that log forward into the census the verifier grades against. |
| `controls6b.py` | every attack in `calibration/scores.md`. |
| `project.py` | projects each logged event onto its own frame from the recorded camera pose, which is how "was it on screen" was answered. |

## Reproducing

    bash h200_run2.sh 1800            # capture
    python3 gt6_fixed.py rig_events.log 4.13 write
    python3 project.py rig_events.log out.mp4 4.13
    python3 controls6b.py rig_events.log 4.13

The clock offset is `go.epoch - ffmpeg_start.epoch` plus up to one 0.25 s poll
before the rig notices the GO file. `SPEC.md` records the three independent
ways that offset was checked.

## Known imperfections

Twelve of 863 placements fall just below the bottom edge of the frame at the
moment they happen, and come into view on a later pass. The rig's three
full-screen sync flashes, which earlier captures used to align the clock,
rendered only once in this take; the offset comes from the capture script's
own timestamps instead and is corroborated as described in `SPEC.md`.
