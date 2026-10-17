This is an analog television system to reduce the manufacturing process of displays and make it more practical. Same color burst, 3.579545MHz.

Channels 1-999999
Potential subchannels 1-999

24k Max on Cathode Ray Televisions [CRT]. Unknown if True Black.

www.thingiverse.com/thing:6267817 This would be the cutout for this project.

"[LVDS] Possible as hybrid DIY. Schematic's pure analog dividers lock to NTSC burst, but an MCU on one IO synced to ~60 Hz field rate could switch modes or envelopes, holding a continuous single-dot 480i pulse while flipping illusion classes. Bypass forces the stream past native display timing. Adds programmable flexibility at cost of analog purity; still one spot, so real multi-pixel detail stays unreachable. Fun experiment path."

This work is licensed under a Creative Commons Attribution–NonCommercial–NoDerivatives 4.0 License.

# AHD-TV-1 Nano - Summary

Working on a full analog/digital CRT expansion system that turns any 480i input into a 24k‑class illusion engine. The Arduino Nano firmware in this repo locks to real VSYNC, then fires ultra‑precise luma/chroma pulses that the analog core shapes into HD → 4K → 8K → 24k virtual resolution. No pixels, no framebuffer — just pure timing, RC networks, and a single dot that becomes an entire image through temporal rendering. Retro hardware, modern illusion.

# AHD-TV-1 CRT Expansion Board — Technology Summary

The AHD-TV-1 CRT Expansion Board is a hybrid analog/digital timing engine that produces
ultra‑high‑resolution temporal illusions (HD → 4K → 8K → 24k virtual lines) on any CRT or
composite‑input display. Instead of rendering pixels, the system renders time: a single bright
dot is flashed with extreme precision, synchronized to 480i vertical sync, and the human visual
system integrates these pulses into a virtual raster far beyond the physical resolution of the
display.

## Analog Timing Core
- 3.579545 MHz NTSC crystal oscillator
- Transistor divider chain generating:
  - 30 Hz (NTSC frame)
  - 100 Hz (AHD mode)
  - 1000 Hz (AHD turbo mode)
  - 59.94 Hz field sync
  - 15.734 kHz horizontal reference
- These signals form the backbone of the illusion engine.

## Illusion Classes (L0–L6)
Each class corresponds to a virtual resolution tier:
- L0: LDTV (~400 lines)
- L1: SD 480
- L2: HD 720
- L3: Full HD 1080
- L4: 4K
- L5: 8K
- L6: 16K–24K illusion class

Each class is implemented as a pulse‑width RC network selected by the MCU.

## Illusion Profiles
Profiles modify the dot’s envelope:
- Breathing Field (sinusoidal luma modulation)
- Chromatic Drift (phase‑shifted chroma pulses)
- Ghostwave (blink‑synced scene cuts)
- Whiteout (full‑field flash)
- Static Field (constant brightness)

## Dot Driver
A single NPN transistor drives the CRT’s composite input. The dot’s brightness and timing encode
the illusion. The CRT still receives a valid 480i signal from the source; the illusion engine
overlays a precisely timed pulse that the brain interprets as high‑resolution structure.

## MCU + Firmware Behavior
The Arduino Nano does not generate video or timing. It performs safe switching of analog networks
and outputs a single pulse per frame synchronized to VSYNC.

### MCU Responsibilities
- Reads VSYNC from LM1881
- Waits for field boundary
- Selects:
  - Mode (30 / 100 / 1000 fps)
  - Illusion Class (L0–L6)
  - Profile (Breathing, Drift, Ghostwave, etc.)
- Generates illusion pulses using Timer1 (OC1A/OC1B)

### Firmware Modes
1. Multi‑Button Illusion Class Selector  
   Seven buttons choose L0–L6 in real time.

2. Serial‑Controlled Illusion Engine  
   Send “0–6” over USB serial to select illusion class.

3. Dual‑Pulse Chroma Drift Mode  
   Outputs:
   - Luma pulse on D9
   - Chroma drift pulse on D10
   with phase‑shifted chroma cycles.

## 24k Illusion Capability
The board achieves 24k virtual resolution through:
- Sub‑millisecond pulse timing
- High‑frequency mode (1000 fps)
- Ultra‑short pulse widths (L6 class)
- Phase‑aligned chroma drift
- Field‑synchronized switching

The CRT does not physically draw 24,000 lines — the illusion is created by temporal precision,
phase coherence, envelope modulation, and field‑locked pulses.

## Wiring Summary
480i Composite → LM1881 → Nano D2 (VSYNC)  
Nano D9 → 1k → TV Composite In  
Nano D10 → 1k → TV Composite In (dual‑pulse mode)  
Nano GND → TV Composite Ground  

Buttons (mode 1):  
D3–D11 → buttons → GND  

Power:  
Nano +5V → +5V rail  
Nano GND → GND rail  
LM1881 +5V → +5V rail  
LM1881 GND → GND rail

## Summary
The AHD-TV-1 CRT Expansion Board is a temporal illusion engine that transforms any CRT or
composite‑input display into a 24k‑class perceptual system using a single dot and the human
visual system’s ability to integrate time into space.
