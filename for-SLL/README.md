# for-SLL — staging folder

Everything here is intended for the SLL repo. Self-contained: nothing outside
this folder is referenced, and nothing outside it was modified to add it.

Lineage: forked from `bt-webtask-main` as uploaded 2026-08-27. It does **not**
include the recall phase that lives in this repo's `bt-webtask-main_1/two-prong`
— that was a separate line of work and is deliberately not merged in here.

## Contents

| Folder | What it is |
|---|---|
| `two-prong/` | Baseline task. Canonical — changes land here first. |
| `Moving-Linear/` | Moving target, ring slides left-right. **Not finalised.** |
| `Moving-Circular/` | Moving target, ring orbits on the ground plane. **Not finalised.** |
| `analysis/` | Per-participant skill-slope fit and self-rating models. |

Each task folder is independently deployable — Pages serves it from its own
`index.html`.

## Fidelity to the VR task

The coercion engine is a direct port of `NEW_ball_improvement.py`: zone sizes
`[1.2, 0.9, 0.6, 0.3]`, success-ordinal indexing, failure zones `[-5,-4]`/`[4,5]`
shifting 0.5 toward center per failure (max 3.0), failure side chosen by throw
direction. Verified against the Python original on both trial structures.

### Deliberate deviations

1. **Shrinking-ring draw in both conditions.** Success targets come from the same
   formula in improvement and stochastic, so the two differ *only* in the
   temporal order of S/F trials — isolating trial order as the single
   manipulated variable.
2. **Fixed, not sampled targets.** Distance from ring center is a constant per
   trial ordinal rather than a uniform draw:

   | | t1 | t2 | t3 | t4 | t5 | t6 | t7 |
   |---|---|---|---|---|---|---|---|
   | improvement `FFFSSSS` | F 4.0 | F 3.5 | F 3.0 | S 1.05 | S 0.75 | S 0.45 | S 0.15 |
   | stochastic `SSFFSFS` | S 1.05 | S 0.75 | F 4.0 | F 3.5 | S 0.45 | F 3.0 | S 0.15 |

   Success angle is still random; failure side is still set by throw direction.
   Success distances are the midpoints of the annuli previously sampled.

   **This removes all between-participant variance in the coerced outcome.** A
   slope fit to what participants *saw* is therefore a constant per condition.
   See `analysis/` for what to fit instead.

### Bug fixed alongside these

The displayed landing used to be re-derived from the pull and then pushed a
further `ROLL_DIST` past the coerced target, so the ball came to rest ~1.5 units
from center on *every* success regardless of ordinal — the improvement was in the
data but never on screen. The ball now lands `LANDING_OFFSET` (3.0, the VR value)
short of the coerced target and rolls onto it, resting exactly there. Logged
position and displayed position are the same point, as in VR.

## Ghost ball (debug)

Append `?ghost=1` to any experiment URL to draw a translucent ball along the
**uncoerced** trajectory — where the throw would have gone with no manipulation —
alongside the real one, with both distances printed on screen. Off by default; a
participant link will never contain it.

Independently of the flag, the uncoerced landing is logged on every trial for
every participant as `real_x` / `real_y` / `real_dist`. Since the coerced
distances are fixed across participants, this counterfactual is the only
per-participant accuracy signal in the task.

## Moving target

One `ringPosAt(t)` drives both the renderer and the coercion. A coerced landing
is resolved against where the ring **will be** when the ball comes to rest, not
where it is at release. Ring phase and base depth re-roll each trial; the motion
parameters and the ring's position at rest are logged
(`ring_mode`, `ring_period_ms`, `ring_amplitude`, `ring_base_d`, `ring_rest_x`,
`ring_rest_d`).

## Analysis

```bash
pip install pandas numpy statsmodels
python3 analysis/skill_slope.py DATA_DIR -o out/
```

`DATA_DIR` is a directory of the raw jsPsych CSVs DataPipe writes. Produces
`per_participant_slopes.csv` plus regressions of signed self-rating on the fitted
skill slope, with condition as a covariate. The slope is fit to `real_dist` and
negated, so positive means throws getting closer to center.

It also fits the coerced outcome as a design check — within-condition variance
there should be exactly zero. Anything else means some sessions ran a different
build.

## Known issues / not yet decided

- **Moving-target timing is uncalibrated.** At the placeholder `PERIOD_MS: 4200`
  the ring travels more than half a cycle during the ~2.4 s a throw takes, so the
  ball visibly curves to chase it. Either lengthen the period substantially or
  freeze the ring at release — the second is cleaner but changes what the
  manipulation is.
- **No recall phase in these versions.** If the recall phase is meant to ship to
  SLL too, it needs merging in deliberately — and in the moving variants it would
  have to be made ring-relative, since it currently scores clicks against a static
  ring at `RING_D`.
- **U trials still roll forward past the natural landing.** VR instead
  decelerates the ball to a stop 3 m *short* of it. U is excluded from
  participant assignment, so this only matters if you run it.
- **`applyReleaseNoise()` is dead code** — defined, never called. Documented in
  place rather than wired in, since jittering coerced targets would undo
  deviation 2.
- **`fail_side` labels are mirrored** relative to the screen. They follow VR's
  frame (facing -y, +x is the participant's left); the web camera puts +x on
  screen-right. Convention, not a bug — but don't read them as screen sides.
- **DataPipe.** All three versions point at experiment `l4vuPmnrCh5F`. The moving
  variants need their own project before real participants, or their data lands
  in the same OSF project as `two-prong` and balanced assignment drifts.
