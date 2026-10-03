# Lightning Lateral Tuning Log

Truck: 2025 F-150 Lightning, comma four, BluePilot bp-7.0 + chestnut (Lebowski big model, v0.11.2 port).
Test road: same narrow road each drive (lane width p50 ~2.5 m). One change per drive.
Run after each drive: `"/data/Lightning Fork/lat-tuning/lat_test.sh"` (newest route on the comma).

Metrics (engaged, above 25 mph, 0.4-1.5 Hz band = the ping-pong):
- **truck actual wobble** (m/s^2) - lower is better
- **truck / requested** steady curvature - 1.0 = truck does what openpilot asks
- **BP sent / requested**, **truck / sent** - where the shortfall happens
- **lag** - truck response delay in the wobble band

| # | Date | Route | Model | Change | Truck wobble | Truck/requested | BP sent/req | Truck/sent | Feel |
|---|---|---|---|---|---|---|---|---|---|
| 0 | 10-03 | 0000004c | bp-7.0 old big | baseline (blend 0.4) | 0.026 | 0.54 | 0.76 | 0.76 | ping-pong |
| 1 | 10-03 | 0000004e | Lebowski | model swap only (blend 0.4) | 0.028 | 0.69 | 0.83 | 0.93 | "half as bad", over-corrects |
| 2 | 10-03 | 0000004e seg 20-34 | Lebowski | curvature blend 0.4 -> 0 (custom_profile_curv=1) | (not pulled) | | | | WORSE: corrects twice as fast, right-left-right-left; can't see the road is straight for the next half mile and go slow and easy |
| 3 | 10-03 | 0000004e (later segs) | Lebowski | curvature blend 0.7 | (not pulled) | | | | WORSE |
| 4 | 10-03 | 0000004f (qlog) | Lebowski | blend 0.4 + speed-scaled look-ahead (20 ft @10 mph .. 50 ft @40 .. 80 ft @70) in BP lateral_curv_ext | 0.033 | **0.97** | 0.96 | 1.01 | **MUCH BETTER - "almost Tesla"** (new baseline) |

Undo test 2: BluePilot settings > custom tuning profile OFF (or `custom_profile_curv=0`).

Notes:
- Test 2 lesson: the blend was doing real smoothing. Without it the raw delay-compensated action reaches the truck and it chases every small correction. Bracket the other way next (0.7).
- Live streaming on the comma while driving caused "selfdrive lagging" - never again; drive, stop, then pull.
- Test 4 numbers come from qlogs (10 Hz) - wobble not exactly comparable to rlog (100 Hz) rows; tracking ratios are.
- Over the hotspot, pull qlogs (8 MB/18 min) not rlogs (200 MB) and say so up front.

## Longitudinal (openpilot long + Experimental, e2e speed from Lebowski)

| # | Date | Route | Change | Speed surge no lead (mph rms) | Follow gap swing (s rms) | Truck accel lag / gain | Feel |
|---|---|---|---|---|---|---|---|
| L0 | 10-03 | 00000050 (qlog, 6 min) | baseline: op long + Experimental, personality Aggressive, longitudinalActuatorDelay 0.15 (default) | 0.46 | n/a (29 s following) | 0.8 s / 0.99 | ping-pong follow; surges at 50 (too fast/too slow) |
| L1 | 10-03 | 00000051 (qlog) | longitudinalActuatorDelay 0.15 -> 0.5 (opendbc car/ford/interface.py) | 3.05 (polluted by 55->30 limit drop) | n/a (no lead) | 0.8 s / 0.62 | pulsing less but still there; decel exaggerated (EV regen); 55->30 limit = near slam (op REQUESTED -2.43 m/s^2) |
| L2 | 10-03 | 00000051 seg 6-12 (qlog) | SpeedLimitMode 3 -> 2 (assist -> warning) to stop limit-drop braking; keep L1 | (route merged with L1) | following 1.3 min: surge 0.11 mph, gap 1.59 s +-0.01 | 0.8 s / 0.63 | following much better, still not ACC-smooth |
| L3 | 10-03 | 00000052 (qlog, 8.3 min, 51 % following) | EV regen handoff: brakes engage at -0.5 (was -0.14), release -0.4, precharge -0.45/-0.35; gentler slowing stays on motor regen (FORD_EV_CARS only, longitudinal_ext.py) | 0.63 | following surge 0.98 mph, gap 1.78 s +-0.05 | **0.2 s / 0.95** (accel 0.89, gentle slow 1.11, firm 0.95) | **MUCH SMOOTHER** |
| L4 | 10-03 | 00000053 + 00000054 (qlog) | longitudinalActuatorDelay 0.5 -> 0.25 (match the post-L3 truck) | n/a (followed ~98 %) | following 4.8 min: surge 1.52 mph, gap 2.57 s +-0.13; 2.4 min: surge 0.42, gap 3.24 s +-0.09 | 0.4 s / 0.89; 0.2 s / 0.83 | **"two good drives"** - settled setup |

Undo L1: `/data/bp_long_backup/revert.sh` + reboot.

EV finding (L0): slow-down requests get 1.28x (-0.26 asked -> -0.38), speed-up 0.46-0.67x. Ford carcontroller sets gas=INACTIVE (-5) when op_gas < MIN_GAS (-0.5); on the Lightning 'gas inactive' = motor coast-regen on top of the brake request. Panda ford.h only allows gas >= -0.5 or inactive.
Undo L3: `/data/bp_long_backup/revert_L3.sh` + reboot.

Settled setup 10-03 (two good drives): steering test #4 (blend 0.4 + speed-scaled look-ahead) - steering on those drives: truck/requested 0.92 / 0.82, BP sent/req 0.96 / 0.90, wobble 0.036 / 0.040 (qlog). Speed: op long + Experimental (e2e), L3 EV regen handoff, L4 delay 0.25, SpeedLimitMode 2, personality Aggressive.
