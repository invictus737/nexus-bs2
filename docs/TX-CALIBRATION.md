# TX frequency plan and calibration — a simple, repeatable recipe

Nexus-BS 2.0 transmits every mode (TETRA, DMR, P25, FM) from **one** SDR, as
carriers offset from **one** TX local oscillator (LO). A direct-conversion
transmitter such as the SX1255 has two unavoidable artefacts:

- **LO leakage**: a spike exactly on the TX LO frequency. Uncorrected on an
  SX1255 board it is about as strong as a carrier (≈ 0 dBc).
- **I/Q images**: a weak mirror of every carrier on the other side of the LO,
  at `2·LO − f`.

PlutoSDR, LimeSDR and USRP boards usually calibrate LO leakage and I/Q balance
inside the device; look at the spectrum anyway and use the fields below only if
a spike or an image is visible. SX1255 boards need them.

This page is the whole procedure: where to put the LO, how to cancel both
artefacts from the dashboard, what to expect, and how to save the result.
It takes about 30 minutes and needs no code and no scripts.

## What you need

- A way to see the TX spectrum around 430–440 MHz with ≤ 3 kHz resolution:
  a spectrum analyzer, or an RTL-SDR/Airspy with a waterfall (SDR#, GQRX,
  SDR++). Absolute accuracy does not matter; you only compare levels.
- **A dummy load and attenuation.** Never connect the TX port straight to an
  analyzer or a dongle without enough attenuation for its input limit
  (an RTL-SDR needs 40–60 dB). Transmit into the dummy load, sample the signal
  through the attenuator.
- The dashboard, **Settings → SDR**. The fields `TX DC I`, `TX DC Q`,
  `TX I/Q gain (dB)` and `TX I/Q phase (°)` are applied **live** when saved:
  no restart, the station stays on the air.

## Step 0 — Start from a profile, if your board has one

Dashboard **Settings → SDR → TX calibration profile** lists boards already
measured on a spectrum analyzer (see [Calibration profiles](#calibration-profiles)).
Choosing one writes its four values and the gains they were measured at, and
applies them live. Then run steps 3–4 once with the smallest steps: every board
differs a little. Editing any value by hand switches the profile to `custom`.

## Step 1 — Put the TX LO where it cannot hurt

Rules (all offsets are multiples of 25 kHz and at least 50 kHz apart):

1. **Never on a channel.** Offset 0 is refused on SX1255 boards.
2. **Keep the LO at least 50 kHz from the nearest carrier**, not 25 kHz: a
   residual LO spike 25 kHz from a carrier sits in its adjacent channel.
3. **Put every carrier on the same side of the LO.** Each image lands at
   `2·LO − f`, so with all carriers above the LO every image falls below it,
   in empty spectrum, and never on another channel.
4. **Stay inside the sample rate**: every `|offset| + 20 kHz` must be below
   half the sample rate, so ±275 kHz at the usual 600 kS/s.
5. **RX follows TX.** Each channel has one `offset_hz` for both directions,
   so set `rx_lo_hz = tx_lo_hz − duplex split` and the RX frequencies stay
   right.

Recipe: `TX LO = lowest carrier − 50 kHz`, then offsets `+50, +100, +150, …`.

Example (70 cm, 7 MHz split), every RF frequency unchanged:

| | TX LO | DMR | TETRA | P25 | FM |
|---|---|---|---|---|---|
| RF TX | **438.2625** | 438.3125 | 438.3625 | 438.4125 | 438.4625 |
| `offset_hz` | — | +50 000 | +100 000 | +150 000 | +200 000 |
| RF RX | **431.2625** | 431.3125 | 431.3625 | 431.4125 | 431.4625 |

Change `rx_lo_hz`, `tx_lo_hz` and every `offset_hz` **in one save** (Config
file tab), then restart the radio service. Saving the offsets one by one
fails validation half-way, because two carriers briefly come closer than
50 kHz.

## Step 2 — Leave the gains alone

- **On SX1255 boards: `MIXER` and `DAC` gain at maximum** (`MIXER = 30`, `DAC = 9`). The
  LO leakage does not shrink with the mixer gain, so lower gain only makes the
  carriers weaker *relative* to it.
- **Other SDRs:** use the TX gain at which the spectrum stays clean, then treat
  it like the SX1255 gains: fixed, and recalibrate if you change it.
- **Keep the default `tx_peak` values** (TETRA 0.8, DMR 0.5, P25 0.7, FM 0.7).
  Raising the digital level drove the SX1255 baseband into distortion: at 0.99
  the images rose ~10 dB while the carriers rose only 2–6 dB.
- Adjust output power with an external PA or attenuator, not by pushing the
  digital level.

## Step 3 — Cancel the LO leakage (TX DC I / Q)

Look at the TX LO frequency: span 50 kHz centred on it, RBW 1–3 kHz,
averaging on (≈ 10–20 sweeps, or a slow waterfall). Note the LO spike level.

1. Set `TX DC I` to **+0.01**, save, wait for the average to settle. Then
   **−0.01**. Keep the sign that lowered the spike.
2. Walk `TX DC I` in that direction while the spike keeps dropping; when it
   rises again, go back one step and halve the step
   (0.01 → 0.005 → 0.002 → 0.001 → 0.0005 → 0.0002).
3. Do the same for `TX DC Q`.
4. Go back to `TX DC I` once, with the smallest step, then `TX DC Q` once.

Done when the spike is **≥ 40 dB below the carriers**. Typical values are in
the 0.01–0.03 range. 0.0001 already moves the spike by a few dB, so stop at
0.0002 steps: below that you are chasing noise, because the residual
fluctuates by a few dB on its own.

## Step 4 — Cancel the images (TX I/Q phase / gain)

Look at the image of your strongest carrier at `2·LO − f`, with the same
settings as step 3. In the example above the TETRA image is at 438.1625 MHz.

1. `TX I/Q phase (°)` first: try **+0.5** and **−0.5**, keep the better sign,
   refine in 0.2° then 0.1° steps.
2. Then `TX I/Q gain (dB)`: try **+0.1** and **−0.1**, refine in 0.05 then
   0.02 dB steps.
3. One more pass of phase with the smallest step.

Done when the image is **≥ 50 dB below its carrier**, or you cannot see it
above the noise. Typical values are a few tenths of a degree and a few
hundredths of a dB. If the image is already invisible, leave both at 0.

## Step 5 — Check, then save the profile

- Let the station warm up **30 minutes** and repeat steps 3–4 once with the
  smallest steps.
- The values belong to **this board at these gains**. Redo steps 3–4 after
  changing the board, `MIXER`, `DAC` or the TX LO by more than a few hundred
  kHz. A 25 kHz LO move did not need a new calibration here.
- Save a profile so the next board swap costs one minute:
  `/opt/nexus-bs2/calibration/<board-name>.toml` with the four values, the
  gains, the LO and a line of results. See
  `Docs/evidence/calibration/Z32IT_sx3.0_amp.toml` for the format.
- Share it: the board name and the four values are useful to everyone with
  the same hardware.

## What to expect (measured, SX1255 HAT V3 with TQP3M9036 driver)

Analyzer RBW 3 kHz, PosPeak detector, 100 averages, carriers ≈ 0 dBm:

| | Uncorrected | After steps 1–4 |
|---|---|---|
| LO spike | ≈ 0 dBc, 25 kHz from DMR | ≈ −41 dBc, 50 kHz from DMR |
| TETRA / DMR image | ≈ −47 / −45 dBc | ≈ −57 / −55 dBc, in empty spectrum |
| Noise between carriers | ≈ −44 dBc | unchanged (from the SX1255 itself) |

About −41 dBc is the practical limit for the LO: the cancellation must match
the leakage to about 1 % and the leakage drifts with temperature and supply.
A filter cannot remove it at 25–50 kHz from a carrier, which is why step 1
matters.

## Troubleshooting

- **All carriers suddenly ~14 dB weaker, driver amplifier cold**: reseat the
  HAT. A poor contact on the header caused exactly this and cleared on
  reseating.
- **Station does not come back on the air after changing `tx_peak`**: update
  to a build that includes commit `e99c3f0`. Older builds accepted values up to
  0.99 in Settings but refused anything above 0.8 when starting the radio.
- **LO spike reappears after a restart**: normal drift; if it is more than
  ~5 dB worse, repeat step 3 with the smallest steps.
- **Spike or image does not react at all** to the fields: check that you are
  looking at the TX LO frequency (`tx_lo_hz`, not a carrier) and at
  `2·LO − f`, not `2·f − LO`.

- **FM looks as narrow as DMR**: a carrier with no audio is only the CTCSS
  tone, about 1 kHz wide. It widens with speech; if it stays narrow, check
  that the handheld is on a 25 kHz (wide) channel.

## Calibration profiles

Measured with a Multicomp MP700022 (OWON) spectrum analyzer over SCPI: PosPeak
detector, 100 averages, RBW 3 kHz unless noted. TX LO 438.2875 MHz during
calibration (re-checked at 438.2625), carriers DMR 438.3125, TETRA 438.3625,
P25 438.4125, FM 438.4625 MHz, 600 kS/s, SX1255 `DAC = 9`, `MIXER = 30`.

### Z32IT_sx3.0_amp

SX1255 RPi4-5 HAT V3 (Z32IT) **with** the TQP3M9036 driver amplifier.

| Setting | Value |
|---|---|
| `tx_profile` | `Z32IT_sx3.0_amp` |
| `tx_dc_i` / `tx_dc_q` | 0.01901 / 0.01648 |
| `tx_iq_gain_db` / `tx_iq_phase_deg` | −0.0305 / 0.3812 |
| Gains | `DAC = 9`, `MIXER = 30` |

| DC calibration | Before | After |
|---|---|---|
| LO spike, 50 kHz span, RBW 3 kHz | +0.1 dBm | −44.7 dBm (−45 dB) |
| LO spike, 1 MHz span | −3.8 dBm (≈ 0 dBc) | −42.5 dBm (≈ −41 dBc) |
| LO spike after moving the LO to 438.2625 | | −43.4 dBm (≈ −41 dBc), 50 kHz from DMR |

| I/Q calibration | Before | After |
|---|---|---|
| TETRA image, power in ±10 kHz, RBW 300 Hz | −36.6 dBm | −43.8 dBm (1 dB over the noise, ≤ −55 dBc) |
| TETRA image, 1 MHz span | −47.2 dBm | −56.3 dBm |
| DMR image, 1 MHz span | −44.8 dBm | −51.8 dBm |

| Output (zero span, RBW 30 kHz) | dBm | mW |
|---|---|---|
| TETRA channel | +1.8 | 1.5 |
| DMR channel | −1.7 | 0.68 |
| All four modes active (P25, FM calculated) | ≈ +6.8 | ≈ 4.8 |

MIXER 30 → 20 lowered every carrier 2 dB per 2 dB step: the driver is linear at
MIXER 30. Raising all `tx_peak` to 0.99 raised the images about 10 dB for
2–6 dB more carrier (SX1255 baseband distortion), so the defaults stay.

### Z32IT_sx3.0_noamp

The same HAT with the TQP3M9036 driver **bypassed**; TX port straight to the
analyzer (Att 26 dB).

| Setting | Value |
|---|---|
| `tx_profile` | `Z32IT_sx3.0_noamp` |
| `tx_dc_i` / `tx_dc_q` | 0.01896 / 0.00182 |
| `tx_iq_gain_db` / `tx_iq_phase_deg` | 0 / 0 (not needed, see below) |
| Gains | `DAC = 9`, `MIXER = 30` |

| DC calibration | Before | After |
|---|---|---|
| LO spike, 50 kHz span, RBW 3 kHz | −35.5 dBm (≈ 0 dBc) | −75.5 dBm (−40 dB, ≈ −41 dBc, 4.5 dB over the analyzer floor) |

| I/Q calibration | Result |
|---|---|
| TETRA image, power in ±10 kHz, RBW 300 Hz | −81.2 dBm against −83.6 dBm noise: 2.3 dB over the floor, ≤ −44 dBc |
| Decision | Left at 0: nothing measurable to cancel at this output level |

Carriers were about −34 dBm (peak in 3 kHz), so this board is a clean,
low-level source; the ≈ −44 dBc noise next to the carriers is the same as with
the driver, so it comes from the SX1255 signal itself.

### Adding your board

Measure it with steps 1–5, then send the board name, the four values, the gains
and the before/after numbers. Each measured board becomes a profile in the
dashboard.
