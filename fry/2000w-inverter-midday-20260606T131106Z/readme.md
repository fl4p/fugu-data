* tracking is disturbed by wrong notch filter freq

## de-tuned INA226 on fry

fry's Vout/Iout come from an INA226 that converts *faster* than its nominal conversion time implies:
~512 sps vs the nominal `1e6/(2·1100µs)` = 454 sps (it IDs as a genuine 0x2260 but the shunt channel
is noisy — likely a counterfeit/out-of-spec part). Because both the scope client and the firmware only
*assume* the channel rate, the absolute frequencies here are off: the inverter ripple shows at ~126 Hz
in this capture and the on-device detector (using nominal 454 sps) called it ~89 Hz, but the true tone
is ~100 Hz (50 Hz inverter, 2×line) — same tone, scaled by each tool's sample-rate error. Trust the
ripple *shape*, not the labeled Hz. Fixed by the auto-tuned notch (immune, uses normalized freq) and
`CONFIG_FUGU_INA226_MEASURED_RATE` (reports the chip's measured rate).