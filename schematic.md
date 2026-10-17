[OSC1: MASTER CLOCK]
- Crystal or RC oscillator at F_MASTER (e.g., 1 MHz or 3.579545 MHz).
- Implemented with Q1 (NPN) in Pierce/Colpitts configuration.
- Output: CLK_MASTER.

[DIVIDER CHAIN]
- Q2, Q3, Q4, Q5 as transistor flip-flops / multivibrators.
- Divide CLK_MASTER down to:
  - FRAME_CLK (e.g., 30 Hz / 100 Hz / 1000 Hz via jumper-selectable taps).
  - PHASE_CLK (e.g., ~15 kHz virtual horizontal).

[OSC2: ENVELOPE OSCILLATOR]
- Q6 as low-frequency RC oscillator (0.1–2 Hz).
- Output: ENV_CLK (drives breathing / drift cycles).

[PROFILE SELECT / GATING]
- Q7, Q8 as simple transistor switches:
  - Select profile: Breathing / Chromatic Drift / Ghostwave / Static.
  - Route ENV_CLK or fixed DC into luma envelope network.

[LUMA ENVELOPE NETWORK]
- RC ladder + Q9:
  - Integrate ENV_CLK into a smooth ramp (Breathing Field).
  - Or step levels for Ghostwave (pre/mid/post blink).
- Output: LUMA_CTRL (0–1 normalized analog).

[CHROMA / MODE SELECT]
- If using LED dot:
  - Q10, Q11, Q12 drive different LED colors (white/cyan/magenta/yellow).
  - Selection based on PHASE_CLK or profile state.
- If using CRT:
  - Chroma is symbolic; you mainly modulate brightness (LUMA_CTRL).

[DOT DRIVER / OUTPUT]
- Q11 (or Q12) as main current driver:
  - Input: LUMA_CTRL gated by FRAME_CLK and PHASE_CLK.
  - Output:
    - LED current (for physical dot), or
    - Video input signal (for CRT center spot).

- Optional:
  - Mix small sync-like pulses from FRAME_CLK into video line
    if you want a CRT to lock to it.

