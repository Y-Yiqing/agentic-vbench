```yaml
task: voxel/standing-block-census

# 1.
cognitive_level: understanding
# Each answer is a state, not an observation. The count of a block type
# standing at minute twenty depends on every placement and every removal in
# the preceding twenty minutes, so a missed event is wrong there and wrong at
# every later checkpoint. Spotting the events is perception and is not
# sufficient; holding the running state across half an hour is the task.

# 2.
modalities_required:
  video: Both kinds of change are visible only as pixels changing, and the
    answer at any checkpoint is the accumulation of all of them so far.
  audio: not used. The capture is silent.

# 3.
question: >
  At each of thirty listed times, how many camera-placed blocks of each of six
  types are standing in the world?
output_schema: >
  {"census": {"<seconds>": {"birchwood": n, "bookshelf": n, "brick_block": n,
  "gravel": n, "hardened_clay_orange": n, "mossycobble": n}, ...}} for the
  thirty times 60, 120, ... 1800. Counts are integers and score only when
  exactly right.

# 4.
evidence:
  - t=2.7s, video, the first placement; it is still standing at every one of
    the thirty checkpoints and is part of all thirty answers.
  - t=901.4s, video, a removal at the midpoint; every checkpoint from 960
    onwards is wrong by one if it is missed, and every checkpoint before it is
    wrong by one if it is counted early.
  - t=1799.1s, video, the last event before the final checkpoint.

# 5.
ground_truth:
  source: >
    A scripted camera rig inside the game server writes every placement and
    removal it performs, with world coordinates, node name, and its own camera
    pose at that instant. The census is that log replayed forward.
  tier: logged
  verification: >
    Clock: the capture script starts ffmpeg, then drops a GO file that the rig
    polls for every 0.25 s, giving an offset of 4.009 + [0, 0.25] s. Seven
    earlier captures of the identical script measured the offset independently
    from three full-screen sync flashes and got +3.99 to +4.22 s, which
    brackets it. Independently again, placements happen at the crosshair by
    construction, and all 863 of them project to a median 52.7 px from the
    centre of a 1280x720 frame using the logged pose; a clock out by one
    second would move the camera six nodes and scatter them.
    Framing: every event is projected onto its own frame. All 315 removals are
    in frame, at a median apparent size of 34 px and 28 px at the tenth
    percentile. 851 of 863 placements are in frame; the other 12 fall just
    below the bottom edge at the moment they happen and come into view on a
    later pass.

# 6.
scorer:
  metric: >
    The fraction of the 180 (checkpoint, block type) cells whose count is
    exactly right. No gate. An earlier draft gated the primary on the running
    total being within two, which made perfect tracking with the clock five
    seconds out score the same zero as an empty file; the primary separates on
    its own.
  oracle_reward: 1.0
  null_reward: 0.0222

# 7.
difficulty:
  strong_agent_reward: 0.0556
  tool_call_turns: 104
  agent_model: gpt-6-astra, reasoning effort high, via codex exec
  harness: >
    Every one of those 104 actions was proxied into a container built from
    environment/Dockerfile and started with --network none; the recording, the
    agent's scratch files and its answer all stayed inside it, and the host
    directory holding the truth was mode 000 for the whole run. The proxy is
    generator/container_mcp.py and the runner generator/run_arm.sh.
  # It answered all thirty checkpoints and reported tallying the whole
  # recording, so this is a completed attempt, not an abandoned one. Of the
  # ten cells it placed exactly, nine are in the first three checkpoints and
  # one at t=1140; nothing later is right. That is the error propagation the
  # task is built on. The same model given the recording on the host instead
  # scored 0.0389 in 74 turns, and the figure reported here is the higher of
  # the two.
  other_agents: >
    Antigravity (Gemini 3.8 Flash, medium) reached the same container through a
    file bridge, since its own sandbox denies the docker socket, and scored
    0.0056 in 165 actions. Claude Code (Opus 5, high) spent the whole four-hour
    budget building a detection pipeline and never wrote an answer. Both are
    written up in calibration/scores.md.

# 8.
anti_shortcut:
  single_frame: 0.0056    # one frame at t=900 s, 0 tool calls, 30/30 answered
  video_only: n/a, no audio
  audio_only: n/a, no audio
  no_media: 0.0333        # prompt and schema only, 0 tool calls, 30/30 answered
  frame_dump_no_tools: 0.0056  # the whole recording at 1 fps, 1824 frames in 38
                               # contact sheets, 0 tool calls, 30/30 answered
  # Each arm was run twice. Asked plainly, all three declined: "the attached
  # frame cannot establish counts at the 30 requested times", "I can't
  # reliably distinguish and track every placed block across these contact
  # sheets". A refusal scoring zero is not evidence that the degraded input
  # cannot be used, so the arms were re-run with declining ruled out and a
  # best estimate demanded. The numbers above are from those runs. Note that
  # the full frame dump scores no better than a single frame: without tools
  # the extra 1823 frames add nothing.

# 9.
input:
  url: https://huggingface.co/datasets/Maxine668/avb-luanti-standing-census/resolve/main/session.mp4
  sha256: fbf581836598383a4186554a2604e3a3b412954618070b3d681ccc743cf4a327
  length_min: 30
  resolution: 720
```

## Measured attacks

Nulls, using only what the prompt states or what can be estimated off the
video without tracking anything.

| arm | reward |
|---|---|
| all zero | 0.0000 |
| every cell = 45 / 65 / 85 | 0.0000 / 0.0056 / 0.0222 |
| total linear in time to the true final, split evenly | 0.0222 |

Diagnostics, handed pieces of the answer key. These bound the corresponding
real attack rather than baselining it.

| arm | reward |
|---|---|
| exact per-checkpoint total, split evenly | 0.0500 |
| exact per-checkpoint total, split by the whole-video type mix | 0.0500 |
| honest tracking that misses 2% of events | 0.1722 |
| honest tracking that misses 5% | 0.1333 |
| honest tracking that misses 10% | 0.0222 |
| honest tracking that misses 20% | 0.0000 |
| perfect tracking with the clock 10 s out | 0.4167 |
| perfect tracking that ignores removals entirely | 0.1556 |

## How this task was arrived at

Four earlier versions of the same footage asked for a ledger of events rather
than a state, and none of them could be brought under the gate.

| version | question | length | strong agent |
|---|---|---|---|
| v1 | which block was dug or placed, and when | 180 min | 0.3162 |
| v2 | the same, with a texture-measured confusable palette | 15 min | 0.7341 |
| v3 | for each removal, when that block had been placed | 15 min | 0.3613 |
| v4 | this census, six types | 15 min | 0.1667 |
| v6 | this census, thirty minutes, six types | 30 min | 0.0556 |

The measured reasons, in the order they were established.

A voxel game's class identity is always calibratable. The textures are the
game's own fixed PNGs, so one look settles what cobblestone is forever.
Selecting the palette by measured texture distance pulled the mean pairwise
distance from 3.707 down to 1.842 and did not move the identification rate at
all, which is why v2 was the easiest version of the five.

Independent events cannot be made hard enough. A ledger of 139 separate
removals is 139 separate problems, each scored on its own, and a solver that
gets 46 per cent of them scores 0.46. Nothing short of making the footage
unreadable brings that under 0.10, and unreadable is the wrong kind of hard.

Fewer block types make the task harder, not easier, which is the opposite of
the intuition. The metric is an exact count, so what matters is the size of
the number in each cell. Relabelling one fixed event stream into 20, 10, 6 and
4 types and replaying honest tracking that misses a fifth of the events gives
0.424, 0.298, 0.151 and 0.128 as the count per cell rises from 6.7 to 33.7.

The placement mix has to drift. With a constant uniform palette the count per
type is just the running total divided by the number of types, and
extrapolating the total linearly scores 0.125. Redrawing a skewed share for
each type every two minutes drops that to 0.03.

The gate belongs in the primary metric, not bolted on top of it. The first
draft required the running total to be within two before any cell counted,
which scored perfect tracking with a five second clock error the same as an
empty submission. The hardest task already in this family reaches zero because
its own primary metric reaches 0.026, not because a gate trips.
