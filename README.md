# Lab 9 — Gyroscope parameters: robot program + analysis

| File | What it is |
|---|---|
| `lab9_gyro.qrs` | **TRIK Studio program for the real robot** (visual blocks). Runs the whole experiment and logs to `lab9_gyro_78.csv`. |
| `lab9_gyro_quicktest.qrs` | The same program with short waits (3 s warm-up, 10 s static) to check that the robot turns. Logs to `lab9_quicktest.csv`. Not for the report. |
| `Lab9_Gyroscope_parameters.ipynb` | Colab notebook: offset b0 + own calibrate_gyro(), drift d and heading integrals, measurement limits ±ω_max, noise σ, results table |

---

## 1. Experiment (Option A, open-loop in-place rotation)

| `phase` | Name | What the robot does | Duration | Variable (robot, ms) |
|---|---|---|---|---|
| — | warm-up | still, motors off, **not logged** | 2 min | Timer `120000` (`.js`: `T_warm`) |
| `0` | static | still, `u = 0`, logged | 3 min | `T_static = 180000` |
| `1` | sin | `u = sat±100(A_sin · sin(2π·tau/T))` | 36 s | `T_sin = 36000` |
| `2` | rect | `u = sat±100(A_rect · sgn(sin(2π·(tau − T_sin)/T)))` | 36 s | `T_rect = 36000` |
| `3` | rest | still, `u = 0`, logged | 3 s | `T_rest = 3000` |

* Period `T = 4000` ms. The amplitude steps `A = 30 → 60 → 150 %` every `3 * T` = 12 s, so each amplitude gets 3 periods. 150 % is clipped to 100 %.
* Right wheel **M3 / E3** gets `+u`, left wheel **M4 / E4** gets `−u`, so the robot turns in place. `u > 0` → CCW (left) turn.
* Total logged time `T_total = 255000` ms (4 min 15 s). With the warm-up, a run takes about 6 min 15 s.
* One log line every ~10 ms. Lines are collected in `buf` and written in blocks of 50.

## 2. Log file `lab9_gyro_78.csv`

| Column | Robot variable | Source | Unit |
|---|---|---|---|
| `t_us` | `t_us` | `gyroscope[3] − t_us0`: gyro timestamp `read()[3]` from the start of logging | µs |
| `t_ms` | `t` | `time() − t0`: script clock from the start of logging (backup clock) | ms |
| `gz_cal` | `gz_cal` | `gyroscope[2]` = `read()[2]`, z rate. TRIK `calibrate()` is never called, so no bias is removed | mdeg/s |
| `gz_raw` | `gz_raw` | `readRawData()[2]`, **`.js` log only** | raw units (LSB) |
| `encL` | `encL` | `encoder4`, left wheel | encoder units (degrees in TRIK Studio) |
| `encR` | `encR` | `encoder3`, right wheel | encoder units |
| `u` | `u` | power sent to M3 (M4 gets `−u`) | % |
| `phase` | `phase` | 0 static, 1 sin, 2 rect, 3 rest | — |

```
t_us,t_ms,gz_cal,encL,encR,u,phase            (.qrs)
```


## 3. Robot program `lab9_gyro.qrs`

### Blocks, in execution order

| # | Block | Setting | Why it is needed |
|---|---|---|---|
| 1 | Initial Node | — | program start |
| 2 | Remove File | `lab9_gyro.csv` | `Write To File` appends; without this a new run is added below the old one |
| 3 | Function **Init** | see variables below | phase lengths, period, empty buffer with the CSV header |
| 4 | Print Text | `WARM-UP 2 min, do not touch` | tells people not to move the robot |
| 5 | Timer | `120000` ms | 2-min warm-up, not logged (a number, not a variable: Timer blocks need a literal) |
| 6 | Print Text | `LOGGING: static 3 min, then rotation` | logging has started |
| 7 | Function **Start** | `t0`, `t_us0`, `t = 0` | zero point of both clocks |
| 8 | Pre-conditional Loop | `t < T_total` | repeats blocks 9–16 until all phases are done |
| 9 | Function **Sample** | see variables below | reads the sensors, computes `u` and `phase`, adds one CSV line to `buf` |
| 10 | Motors Forward | `M3`, power `u` | right wheel |
| 11 | Motors Forward | `M4`, power `0 - u` | left wheel, opposite direction → turn in place |
| 12 | Condition | `n >= 50` | write to the file only every 50 lines (writing every sample slows the loop) |
| 13 | Write To File | `lab9_gyro.csv`, text `buf` | *(true branch)* saves 50 lines |
| 14 | Function **Flush** | `buf = ""; n = 0` | *(true branch)* empties the buffer |
| 15 | End If | — | joins the true / false branches (needed for code generation) |
| 16 | Timer | `10` ms | sample period ≈ 10 ms; back to the loop |
| 17 | Write To File | `lab9_gyro.csv`, text `buf` | *(after the loop)* saves the last < 50 lines, which include the rest phase |
| 18 | Print Text | `DONE - log saved` | time to read the tape mark and download the log |
| 19 | Final Node | — | program end |
| — | Comment | experiment description | for people only, not executed |

### Variables

**Function Init** (set once)

| Variable | Value | Meaning | Unit |
|---|---|---|---|
| `T_static` | `180000` | static phase length | ms |
| `T_sin` | `36000` | sin phase length | ms |
| `T_rect` | `36000` | rect phase length | ms |
| `T_rest` | `3000` | rest phase length | ms |
| `T_total` | `T_static + T_sin + T_rect + T_rest` = 255000 | total logged time, loop condition | ms |
| `T` | `4000.0` | period of the sin / rect signal | ms |
| `buf` | `"t_us,t_ms,gz_cal,encL,encR,u,phase\n"` | text buffer; starts with the CSV header | — |
| `n` | `0` | number of lines in `buf` | — |

**Function Start** (once, right after the warm-up)

| Variable | Value | Meaning | Unit |
|---|---|---|---|
| `t0` | `time()` | script clock at the start of logging | ms |
| `t_us0` | `gyroscope[3]` | gyro timestamp at the start of logging | µs |
| `t` | `0` | elapsed time; must have a value before the first loop check | ms |

**Function Sample** (every loop pass, ~10 ms)

| Variable | Formula | Meaning | Unit |
|---|---|---|---|
| `t_us` | `gyroscope[3] - t_us0` | gyro timestamp since the start | µs |
| `gz_cal` | `gyroscope[2]` | z angular rate | mdeg/s |
| `encL` | `encoder4` | left encoder | deg |
| `encR` | `encoder3` | right encoder | deg |
| `t` | `time() - t0` | elapsed time since the start of logging | ms |
| `tau` | `t - T_static` | time since the rotation run started (negative during static) | ms |
| `s1` | `(sgn(tau) + 1) / 2.0` | 0 before the sin phase, 1 from it on | — |
| `s2` | `(sgn(tau - T_sin) + 1) / 2.0` | 1 from the rect phase on | — |
| `s3` | `(sgn(tau - T_sin - T_rect) + 1) / 2.0` | 1 from the rest phase on | — |
| `m_sin` | `s1 - s2` | 1 only inside the sin phase | — |
| `m_rect` | `s2 - s3` | 1 only inside the rect phase | — |
| `k_sin` | `floor(tau / (3 * T))` | amplitude step in the sin phase: 0, 1, 2 | — |
| `A_sin` | `30 + 30·k_sin + 30·k_sin·(k_sin − 1)` | amplitude 30 / 60 / 150 | % |
| `k_rect` | `floor((tau - T_sin) / (3 * T))` | amplitude step in the rect phase | — |
| `A_rect` | same formula with `k_rect` | amplitude 30 / 60 / 150 | % |
| `u_sin` | `A_sin * sin(2*pi*tau/T)`, then clipped | sine command | % |
| `u_rect` | `A_rect * sgn(sin(2*pi*(tau - T_sin)/T))`, then clipped | square-wave command | % |
| *(clip)* | `x = (abs(x + 100) - abs(x - 100)) / 2.0` | `sat±100(x)`: limits to ±100 % | % |
| `u` | `floor(m_sin*u_sin + m_rect*u_rect + 0.5)` | motor power, 0 in static and rest | % |
| `phase` | `m_sin + 2*m_rect + 3*s3` | 0 static, 1 sin, 2 rect, 3 rest | — |
| `buf` | `buf .. t_us .. "," .. t .. "," .. gz_cal .. "," .. encL .. "," .. encR .. "," .. u .. "," .. phase .. "\n"` | adds one CSV line | — |
| `n` | `n + 1` | line counter | — |

At an exact phase boundary (`tau` exactly 0, `T_sin` or `T_sin + T_rect`) `sgn` returns 0, so for that one sample `phase` is x.5. The notebook rounds it down to the earlier phase.

## 4. How to run on the robot

1. Charge the battery. Put the robot on a hard floor with room to spin.
2. Run `lab9_gyro_quicktest.qrs` first. The robot should start turning about 13 s after start.
3. Open `lab9_gyro.qrs` in TRIK Studio, choose the real TRIK robot, then upload and run it.
4. **Do not touch the robot or the table** during the warm-up, the static phase and the rotation. The robot starts turning only after **5 minutes**.
5. In the first sine half-period (`u > 0`) the robot must turn **left (CCW)**. If it turns right, write that down.
6. Download `lab9_gyro_78.csv` from the robot.

---

## 5. Notebook `Lab9_Gyroscope_parameters.ipynb`

Open it in Colab and upload `lab9_gyro_78.csv` (the first cell asks for it). Set the configuration, then *Runtime → Run all*.

It does four things: **offset**, **drift + heading integrals**, **measurement limits**, **noise**.

> **Units:** on the robot `t` and the `T_*` lengths are in **ms**. In the notebook `t` is in **s** and rates are in **deg/s** (`omega_z = gz_cal / 1000`).

### Settings (first cell)

| Variable | Default | Meaning |
|---|---|---|
| `LOG_PATH` | `"lab9_gyro.csv"` | robot log |
| `T_CAL_BASELINE` | `30.0` | baseline window for `b0`, s |
| `HEADING_TOL_DEG` | `1.0` | allowed heading error from a wrong `b0` after 3 min, deg (sets `T_cal_rec`) |


### Data

| Variable | Meaning | Unit |
|---|---|---|
| `log` | the CSV, repeated gyro samples removed, `phase` rounded down, plus `t` and `omega_z` | — |
| `log.t` | time from `t_us` (from `t_ms` if the gyro timestamp is not increasing) | s |
| `log.omega_z` | `gz_cal / 1000`, measured rate ω_z | deg/s |
| `static` | rows with `phase == 0` | — |
| `t_st`, `omega_st` | time from the start of the static phase, and its rate | s, deg/s |
| `fs` | gyro sample rate | Hz |
| `rot` | rows with `phase >= 1` (rotation run) | — |

### Results

| Section | Variable | Meaning | Unit |
|---|---|---|---|
| 1 Offset | `b0`, `b0_se` | mean of the first `T_CAL_BASELINE` s, standard error | deg/s |
| 1 Offset | `b0_T`, `se_T` | b0(T) and its standard error for `T_grid` = 0.5 … 60 s | deg/s |
| 1 Offset | `T_cal_rec` | recommended calibration time: from here b0(T) stays within ±`HEADING_TOL_DEG`/180 deg/s of b0(60 s) and 2·SE is below that | s |
| 1 Offset | `b0_cal` | result of `calibrate_gyro()` | deg/s |
| 2 Drift | `t_win`, `b_win` | centres and means of the 30-s windows | s, deg/s |
| 2 Drift | `d` | bias drift, slope of the window means | deg/s per min |
| 2 Integrals | `theta_raw` | integral of the raw rate (static log) | deg |
| 2 Integrals | `theta_cal` | integral of `omega_st − b0_cal` | deg |
| 2 Integrals | `slope_raw` | uncorrected heading drift (compare with `b0·60`) | deg/min |
| 3 Limits | `omega_max_pos`, `omega_max_neg` | largest / smallest rate in the rotation run | deg/s |
| 3 Limits | `n_pos`, `n_neg` | samples at exactly that value; ≥ 5 = flat top = saturated | — |
| 4 Noise | `noise` | static rate with the line `b0 + d·t` removed | deg/s |
| 4 Noise | `sigma` | noise standard deviation σ (R = σ² for Lab 12) | deg/s |

### Functions

| Function | Does |
|---|---|
| `b0_window(T)` | mean and standard error of the first `T` s of the static log |
| `calibrate_gyro(log, T_cal=None, k_mad=4.0)` | own offset calibration: first `T_cal` s of the static phase (default `max(10, T_cal_rec)`), removes readings further than 4·MAD from the median, returns the mean of the rest |

The last cell shows a table with all results: `b0`, `b0_cal`, `T_cal_rec`, `d`, `slope_raw`, `theta_cal` after 3 min, `omega_max_pos` / `omega_max_neg`, `sigma`.

### README symbol → variable

| Symbol | Robot | Notebook |
|---|---|---|
| ω_z | `gz_cal` (mdeg/s) | `omega_z` (deg/s) |
| b0 | — | `b0`, `b0_cal` |
| d | — | `d` |
| θ (heading) | — | `theta_raw`, `theta_cal` |
| ω_max | — | `omega_max_pos`, `omega_max_neg` |
| σ | — | `sigma` |

---
