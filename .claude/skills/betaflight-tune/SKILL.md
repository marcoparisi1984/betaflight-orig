---
name: betaflight-tune
description: Betaflight blackbox log analysis and PID/filter tuning advice. Triggered when the user sends .bbl/.bfl blackbox logs from this Betaflight build for PID, FFT, filter, or hardware analysis.
---

# Betaflight Tune (stune --platform betaflight)

Betaflight-only tuning advisor, powered by the SmartTune CLI (`stune`). Scoped to this
project: blackbox logs (`.bbl` / `.bfl`) only. ArduPilot/PX4 features are out of scope —
use the generic `smarttune` skill for those.

## Installation

```bash
cd ~/cli-tools/smarttune-cli
pip install -e ".[all]"
```

## Quick Reference

```bash
# Comprehensive analysis (recommended) — PID + FFT + Filter
stune analyze -i log.bbl --platform betaflight

# Log quality scoring — data completeness / excitation / sample rate
stune quality -i log.bbl --platform betaflight

# Individual analyses
stune pid -i log.bbl --platform betaflight -a roll
stune fft -i log.bbl --platform betaflight
stune filter -i log.bbl --platform betaflight --visual
stune hardware -i log.bbl --platform betaflight

# List supported platforms / confirm Betaflight capabilities
stune platforms
```

`.bbl`/`.bfl` files auto-detect as Betaflight, so `--platform betaflight` is optional —
keep it explicit anyway to fail loudly if a non-Betaflight log is passed by mistake.

## Betaflight Capability Matrix

| Feature | Supported |
|---------|-----------|
| `analyze` | ✅ |
| `quality` | ✅ |
| `pid` | ✅ |
| `fft` | ✅ |
| `filter` | ✅ |
| `hardware` | ✅ |
| `sysid` | ❌ (no ARX model for Betaflight) |
| `magfit` | ❌ (no magnetometer data in blackbox logs) |

Do not call `stune sysid` or `stune magfit` on Betaflight logs — they are ArduPilot-only.

## Usage Rules

1. **Check help first**: `stune <command> --help` when unsure about flags.
2. **Terminal output by default**: no `-o` flag → stdout only.
3. **Optional charts**: add `--visual` for matplotlib plots.
4. **Cleanup after analysis**: delete raw `.bbl/.bfl` files after processing.
5. **⚠️ MANDATORY: Validate parameters before recommending**. After analysis generates
   tuning suggestions, you MUST verify each parameter exists in Betaflight by running:
   ```bash
   stune params --validate <PARAM_NAME> <VALUE> -p betaflight
   ```
   If validation fails (exit code 1), do NOT recommend that parameter — it may not exist
   in the firmware version, or the value is out of valid range. Find an alternative or
   tell the user the parameter is not available.

   This is critical because Betaflight 4.5+ renamed many parameters:
   - `d_min_roll` → `d_max_roll`
   - `gyro_lowpass_hz` → `gyro_lpf1_static_hz`
   - `dterm_lowpass_hz` → `dterm_lpf1_static_hz`

   **Never** blindly output parameter recommendations without validation.

⚠️ Analysis done = output results. No residual files needed.

## Workflow

```bash
# 1. Hardware config check
stune hardware -i flight.bbl --platform betaflight

# 2. Log quality scoring
stune quality -i flight.bbl --platform betaflight

# 3. Comprehensive analysis
stune analyze -i flight.bbl --platform betaflight --visual

# 4. Targeted tuning
stune pid -i flight.bbl --platform betaflight -a roll
stune fft -i flight.bbl --platform betaflight --visual
stune filter -i flight.bbl --platform betaflight --visual

# 5. Validate every recommended parameter before presenting results
stune params --validate p_roll 50 -p betaflight
stune params --validate d_max_roll 44 -p betaflight
stune params --validate gyro_lpf1_static_hz 200 -p betaflight
```

**Rule: One `stune params --validate` call per recommended parameter before presenting results.**

## Betaflight Parameter Categories (45 total)

```bash
stune params bf                              # full table
stune params --category pid -p betaflight    # p_roll, i_roll, d_roll, f_roll, d_max_*, anti_gravity_gain, iterm_relax_cutoff, feedforward_transition
stune params --category filter -p betaflight # gyro_lpf1/2_*, dterm_lpf1/2_*, gyro_notch1/2_*, dyn_notch_*, rpm_filter_*
stune params --category rate -p betaflight   # *_rc_rate, *_expo, *_srate
stune params --search feedforward            # find current BF 4.5+ names
```

## Command Reference

### stune analyze
PID + FFT + filter recommendations in one pass.
```bash
stune analyze -i flight.bbl --platform betaflight --visual
stune analyze -i flight.bbl --platform betaflight -a roll
stune analyze -i flight.bbl --platform betaflight -o report.md --report md
```

### stune pid
Step response: rise time, overshoot, settling time, oscillation count.
```bash
stune pid -i flight.bbl --platform betaflight             # all axes
stune pid -i flight.bbl --platform betaflight -a roll --visual
```

### stune fft
Vibration spectrum, dominant frequencies, notch filter suggestions.
```bash
stune fft -i flight.bbl --platform betaflight --visual
```

### stune filter
Filter transfer function (Bode plot). Auto-derives from `gyro_lowpass_hz` / notch params,
or override manually with `--gyro-filter` / `--notch-freq`.
```bash
stune filter -i flight.bbl --platform betaflight                       # auto-derive
stune filter -i flight.bbl --platform betaflight --no-auto --gyro-filter 200 --visual
```

### stune hardware
IMU, filter, and PID parameters at a glance.
```bash
stune hardware -i flight.bbl --platform betaflight
```

### stune quality
Data completeness / excitation / sample rate scoring before trusting the analysis.
```bash
stune quality -i flight.bbl --platform betaflight
```

### stune params
```bash
stune params bf                                              # list all 45 BF params
stune params p_roll                                          # detail for one param
stune params --search notch
stune params --validate d_max_roll 44 -p betaflight           # exit 0 valid / 1 invalid
```

## Further Help

**Rule: Always check `--help` before using a command you haven't run recently.**

| Scenario | Action |
|----------|--------|
| Full parameter list | `stune <command> --help` |
| Parameter meaning reference | `smarttune/knowledge/rules/betaflight/` and `smarttune/knowledge/params/betaflight.json` |

## Response Style

* Start with whether the log is usable (quality score + rating).
* Explain the main fault signals in plain language.
* Give conservative tuning advice; recommendations are capped at ±25% by default.
* Recommend one change at a time and ask for a new log after changes.
* Do not claim the aircraft is safe to fly — only describe what the log supports.
