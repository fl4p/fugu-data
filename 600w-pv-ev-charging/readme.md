# 600 W PV, EV-charging load

A ~600 W PV array feeding a battery while an **EV charger** is the dominant load — a very dynamic,
broadband current draw. ~442 s, raw ADC counts, **per-channel rates in the filenames**
(`iout/vout` 452 Hz, `vin` 392 Hz, `ntc` 196 Hz). `vout_filt` is the device's own filtered Vout.

## what it is

| chan | mean | σ | σ% | character |
|------|------|----|-----|-----------|
| iout | 1777 | **584** | **33%** | violent broadband + occasional ADC rails |
| vout | 2523 | 38 | 1.5% | stable, ~100 Hz ripple |
| vin  | 2487 | 56 | 2.3% | slow drift |

- **iout rails the ADC** (0 and 4095) — but only **39 events in 200 k samples (0.02%), every one a
  single sample**. No sustained saturation; just isolated clips on top of the huge real swing.
- Spectrum: a **~100 Hz tone** (2×line) carries ~20 % of iout's AC energy (and ~26 % of vout's);
  the rest is **broadband + slow drift** — the EV charger's genuine current dynamics.

## filter finding — the existing pipeline already copes

Replaying `notch → median5 → EWM` (and the live `notch → despike → EWM`) on the real iout:

- All variants land on the **same ~0 bias and ~414 σ** power-path estimate. That 414 σ is the
  **real broadband current** — it is signal, not noise, and no time-domain filter should remove it
  (the lever for smoothing it is the EWM span / P&O speed, not clipping).
- The **median already handles the rails**: single-sample 0/4095 spikes are removed from the
  estimate *and* from OC protection, which reads `med3.get()` for current limits (`mppt.cpp`), so the
  rails cannot false-trip overcurrent. The ~100 Hz tone is removed by the adaptive notch.
- A pre-notch **saturation guard** (hold the previous valid sample on a rail) was tried: it changes
  the power path by ~0.3 of 42 — negligible, because the rails are 39 samples vs 200 k of real swing.

## why despike does NOT help here (contrast with the China capture)

The glitch-safe median (`sensor.conf::despike`) uses a **relative** threshold (8× the channel's own
running deviation). EV iout is so broadband (σ 584) that the threshold becomes ~3700 counts, so the
despike **catches only 2 of the 39 rails** and otherwise mis-clips real fast swings, slightly *raising*
the residual (42→58, σ 414→416). Lesson: `despike` is for **dense periodic current pulses**
(`china-inverter-1kw-load`) and sparse impulse glitches — not for a broadband load like an EV charger,
where the right tool is averaging, and the plain median already neutralises the rare ADC rails.
