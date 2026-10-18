# RetroTINK 4K CE → LG C4: EDID, VRR and modeline findings

Notes from troubleshooting VRR + BFI from a RetroTINK 4K CE (RT4K) to an LG C4 OLED,
October 2026. The goal: SNES (60.0988 Hz) with the RT4K's software BFI at 2× output
(~120.2 Hz), VRR on, sound on, at 3840x1080.

## Result

Final modeline (RT4K format: h_active, h_fp, h_sync, h_bp, h_pol, v_active, v_fp, v_sync, v_bp, v_pol, refresh, name):

```
3840, 8, 32, 72, 1, 1080, 3, 5, 61, 0, 130, "3840x1080p130"
```

| | Value |
|---|---|
| Total (H × V) | 3952 × 1149 |
| Pixel clock | 590.3 MHz (9.7 MHz under the 600 MHz TMDS limit) |
| Line rate | 149.4 kHz |
| Horizontal blanking | 112 px (minimum for audio is 101–104 px; 12 px of safety margin) |
| Vertical blanking | 69 lines, ~462 µs (the TV's own 144 Hz-class vertical shape) |
| Margin over 120.2 Hz content | 8.2% |
| VRR lock time | ~1 s |
| Sound | yes |

TV setting: **144Hz mode ON**. RT4K firmware: the release that added "HDMI 2.1 style
extended EDID" support (see below). The lowest-clock mode that still has audio is
`3840, 8, 32, 64, 1, 1080, 3, 5, 61, 0, 130` (589.1 MHz), but it has only 4 px of
margin above the audio limit.

Still to do: a long session on the final mode (artifacts, dropouts, sound cut-outs), with
the RT4K's highest audio sample rate or channel count, and with a couple of other consoles.

## Setup

- RetroTINK 4K CE: TMDS-only HDMI source, assumed ~600 MHz pixel-clock ceiling.
- LG C4 OLED on HDMI 1 (source physical address 1.0.0.0). EDID model string "LG TV SSCR2",
  manufacturer GSM, product 33485, week 1 of 2024.

## Finding 1: the "144Hz mode" setting changes the EDID

The TV reports two different EDIDs depending on its 144Hz mode setting. It doesn't change the
EDID when the source changes resolution: no hot-plug events and an identical EDID across
mode switches.

| Field | 144Hz mode ON (`c4_144on.bin`) | 144Hz mode OFF (`c4_144off.bin`) |
|---|---|---|
| Size | 512 bytes, 4 blocks | 256 bytes, 2 blocks |
| Extension count | 3 | 1 |
| Block 1 | Block Map (tag 0xF0) | CTA-861 (tag 0x02) |
| CTA-861 block | block 2 | block 1 |
| DisplayID 1.2 block | block 3: 3840x2160, 2560x1440, 1920x1080 @ ~144 Hz | absent |
| Range limits, max vertical | 144 Hz | 120 Hz |
| Range limits, max pixel clock | 1340 MHz | 1190 MHz |
| HDMI Forum VSDB VRRmax | 144 Hz | 120 Hz |
| AMD FreeSync VSDB max | 144 Hz | 120 Hz |
| VRRmin / FreeSync min | 40 Hz | 40 Hz |

Identical in both: the video modes, HDMI Forum VSDB otherwise (600 MHz TMDS, FRL up to
12 Gbps × 4 lanes, DSC 1.2a, ALLM), audio, HDR, Dolby Vision.

The 144Hz-on layout (Block Map at block 1, CTA-861 at block 2) is valid for an EDID 1.3
display, but a source has to read every extension block to find the VRR data. That's what
the RT4K firmware update fixed (see below).

## Finding 2: why VRR and 120.2 Hz failed (before the RT4K update)

- **144Hz mode ON:** the RT4K reported "no VRR." The likely cause: it read only blocks 0–1,
  and with 144Hz mode on block 1 is just the Block Map, so it never reached the VRR data in
  block 2. This was inferred from the layout, and the dev's fix (below) is consistent with it.
- **144Hz mode OFF:** the RT4K saw VRR, but the EDID caps VRR and range limits at 120 Hz.
  Sending VRR at 120.2 Hz caused repeated sync loss (screen and RT4K menus blanking several
  times a second).
- **Fixed refresh (no VRR), either mode:** 120.2 Hz synced, but with occasional BFI flicker
  (a black frame skipped or doubled every few seconds).

## RT4K firmware update

The RT4K dev released a fix. Release notes:

> Added support for HDMI 2.1 style extended EDID
> VRR gating no longer uses the strict VICs declared by the display's EDID
> True VRR pacing will engage as long as the frame rate is within the display's declared minimum and maximum
> However, this may result in some displays rejecting the modelines if they strictly enforce the declared VIC

After the update, the RT4K detects VRR with 144Hz mode ON.

## Finding 3: VRR needs ~4.6% or more margin above the content rate

With VRR on and 144Hz mode on, sync loss (full blanking, BFI on or off) still happened
until the modeline was well above 120.2 Hz:

| Modeline | Margin over 120.2 Hz | Result |
|---|---|---|
| 120 Hz | below the content rate | sync loss |
| 121 Hz | 0.7% | sync loss, faster |
| 122 Hz | 1.5% | sync loss, faster still |
| up to 125 Hz | 4.0% | sync loss, ~4–5 per second |
| 126 Hz | 4.8% | clean |

Interpretation: under VRR, the modeline sets the *maximum* rate; the source paces slower
content by stretching vertical blanking. The dropout rate grew with
(modeline − 120.2 Hz) (0.8, 1.8 … 4.8 per second predicted), which fits the output running
near the modeline rate and making a correction the TV can't follow once per beat cycle,
until there's enough margin for true per-frame pacing. Whether this threshold is in the RT4K
or the TV is unconfirmed. It's worth asking the dev, since the release notes suggest pacing should
work anywhere inside the TV's 40–144 Hz range.

**VRR lock time** (the "locking" phase when VRR starts) also depends on margin:

| Modeline | Margin | Lock time |
|---|---|---|
| 126 Hz | 4.8% | ~10 s |
| 128 Hz | 6.5% | ~3–5 s (A128 felt slightly slower than EF128/F128; possibly noise) |
| 130 Hz | 8.2% | ~1 s |

Content slower than SNES (59.94, 60, 50 Hz) has more margin at the same modeline. Assuming the
~4.6% threshold holds, a 130 Hz modeline should handle content up to about 62 Hz at 2× BFI.

## Finding 4: pixel clock and blanking limits

- The online RT4K mode generator's mode, `3840, 8, 32, 80, 1, 1080, 53, 8, 67, 0, 126`
  (3960 × 1208), needs **602.7 MHz at 126 Hz**, which is over 600 MHz. It showed RT4K menu artifacts.
  Its 128 lines of vertical blanking are where most of the clock went. The TV's own
  DisplayID 1080p144 timing uses 3/5/69 (77 lines), and that works.
- **Horizontal blanking minimums (at 126–130 Hz, 3840 wide):**
  - 80 px (8/32/40, the CVT-RB2 standard): **"No Signal"** on the TV
  - 100 px (8/32/60): syncs, **no audio**
  - 104, 108, 112, 116, 120 px: syncs **with audio**

  So audio needs more than 100 px; the minimum is 101–104 px. Likely cause: HDMI carries audio
  in data-island packets during blanking, and the transmitter or TV needs room for
  them. This is inferred, not confirmed. Every timing the TV itself advertises uses 160 px.
- **Vertical blanking:** 68–77 lines (~460–528 µs, around the 460 µs CVT-RB minimum)
  works at 3840 wide.

### All modelines tested

| Name | Modeline | Pixel clock | Result |
|---|---|---|---|
| generator 126 | `3840, 8, 32, 80, 1, 1080, 53, 8, 67, 0, 126` | 602.7 MHz | works, menu artifacts |
| A (126) | `3840, 8, 32, 80, 1, 1080, 3, 5, 69, 0, 126` | 577.3 MHz | works, ~10 s lock |
| B (126) | `3840, 8, 32, 40, 1, 1080, 3, 5, 69, 0, 126` | 571.5 MHz | No Signal |
| C (128) | `3840, 8, 32, 40, 1, 1080, 3, 5, 69, 0, 128` | 580.5 MHz | No Signal |
| D (126) | `3840, 8, 32, 40, 1, 1080, 3, 5, 101, 0, 126` | 587.3 MHz | No Signal |
| A128 | `3840, 8, 32, 80, 1, 1080, 3, 5, 69, 0, 128` | 586.5 MHz | works, sound, ~4–5 s lock |
| E128 | `3840, 8, 32, 60, 1, 1080, 3, 5, 69, 0, 128` | 583.5 MHz | works, ~3 s lock (sound not checked) |
| F128 | `3840, 8, 32, 80, 1, 1080, 3, 5, 60, 0, 128` | 581.9 MHz | works, sound, ~3–4 s lock |
| EF128 | `3840, 8, 32, 60, 1, 1080, 3, 5, 60, 0, 128` | 579.0 MHz | works, **no sound**, ~3–4 s lock |
| EF130 | `3840, 8, 32, 60, 1, 1080, 3, 5, 61, 0, 130` | 588.5 MHz | works, **no sound**, ~1 s lock |
| H104 | `3840, 8, 32, 64, 1, 1080, 3, 5, 61, 0, 130` | 589.1 MHz | works, sound |
| H108 | `3840, 8, 32, 68, 1, 1080, 3, 5, 61, 0, 130` | 589.7 MHz | works, sound |
| H112 | `3840, 8, 32, 72, 1, 1080, 3, 5, 61, 0, 130` | 590.3 MHz | **chosen**: H108 and H116 both work; long session pending |
| H116 | `3840, 8, 32, 76, 1, 1080, 3, 5, 61, 0, 130` | 590.9 MHz | works, sound |

Where the values come from: the vertical 3/5/69 is from the EDID (DisplayID 1080p144 timing). The horizontal
8/32/80 is from the mode generator, and 8/32/40 is CVT-RB2. The refresh targets come from testing. No
3840x1080 mode appears in the EDID; every mode above is inside the EDID's limits
(VRR 40–144 Hz, range limits 24–144 Hz / 30–255 kHz, 600 MHz TMDS).

## Applying this to another LG TV (e.g. a C3)

Only the C4 was tested. Things to check on another model:

- **Refresh ceiling.** The C3 is a 120 Hz TV with no 144Hz mode, so its EDID will most
  likely cap VRR at 120 Hz, like the C4 with 144Hz mode OFF. In that state, VRR at 120.2 Hz
  (SNES 2× BFI) caused repeated sync loss here, and Finding 3 suggests a modeline needs about
  4.6% headroom above the content rate, which a 120 Hz cap doesn't leave. Fixed refresh (VRR
  off) at 120.2 Hz did sync on the C4, with occasional BFI flicker. Content at 59.94/60 Hz
  (≤ 120 Hz at 2×) may behave differently; untested.
- **Dump and compare the EDID.** Read it (commands below) and check the fields in the
  Finding 1 table: block count/Block Map, range limits max V and pixel clock, HDMI Forum
  VRRmax, FreeSync max.
- **Pixel clock and blanking.** The 600 MHz TMDS limit and the audio blanking minimum
  (more than 100 px horizontal) are most likely RT4K/HDMI-side, so they should carry over.
  Start from the modelines in Finding 4.
- **RT4K firmware.** Use a release with "HDMI 2.1 style extended EDID" support.

## Files

| File | What it is |
|---|---|
| `c4_144on.bin` / `.txt` | TV EDID with 144Hz mode ON (512 bytes) + edid-decode 1.32.0 output |
| `c4_144off.bin` / `.txt` | TV EDID with 144Hz mode OFF (256 bytes) + edid-decode 1.32.0 output |

Useful commands:

```sh
cat /sys/class/drm/card1-HDMI-A-1/edid > dump.bin   # Linux; connector name varies
edid-decode -c dump.bin
```
