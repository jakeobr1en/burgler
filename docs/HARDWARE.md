# Burgler - Hardware Design

Companion to [ARCHITECTURE.md](./ARCHITECTURE.md). Budget target: $150-300 for the complete door unit.

## Summary

A single self-contained door unit: Raspberry Pi 5 with a Hailo NPU, a CSI camera with IR illumination, and an I2S amp driving a small speaker, in an IP65 box. One Ethernet run with PoE+ is the primary power/network plan; WiFi + mains is the fallback variant. No parts are owned yet, so everything below is on the bill of materials.

The full-system diagram (every component here plus power, network, and data flow) is in [ARCHITECTURE.md section 2](./ARCHITECTURE.md#2-hardware-summary-and-full-system-diagram).

## Compute: Raspberry Pi 5 (8GB) + Hailo-8L NPU

| Option | Price | Verdict |
|---|---|---|
| **Pi 5 8GB + AI HAT+ (Hailo-8L, 13 TOPS)** | ~$80 + ~$70 | **Chosen.** Runs the full vision pipeline (person det + face det + face embedding) on the NPU at well over real time, leaving the CPU nearly idle for decision/audio/recording/dashboard. |
| Jetson Orin Nano Super dev kit | $249 board alone | More headroom (TensorRT, bigger models), but board alone nearly consumes the budget; full build lands ~$400+. Right choice if the project later outgrows the Pi. |
| Pi 5 CPU-only | ~$80 | YOLO-nano + ArcFace on 4x A76 cores is achievable at low fps, but night-time detection wants better models and headroom. False economy for $70. |
| Coral USB/M.2 | ~$60 | Aging ecosystem, awkward face-embedding story, model conversion friction. Ruled out. |

Why Hailo specifically: the Hailo Model Zoo ships precompiled `.hef` binaries for exactly the models this pipeline needs (YOLOv8 person detection, SCRFD face detection, ArcFace-class recognition), and the Raspberry Pi + Hailo combination is the officially supported first-party AI path for the Pi 5. Minimal model-porting work, maximal community debugging surface.

13 vs 26 TOPS AI HAT+: 13 TOPS is comfortably sufficient for one camera at 10-15 inference fps across three small models. Save the $40.

## Camera: Pi Camera Module 3 NoIR Wide (~$35)

- **Interface**: CSI-2. Lowest latency path available (no network hop, no RTSP decode), direct into the ISP, and Picamera2 can emit a low-res stream for inference and an H.264 main stream for clip recording simultaneously from one sensor.
- **Low light**: NoIR variant (no IR-cut filter) + a separate 850nm IR illuminator board (~$10-15). 850nm has a faint red glow; 940nm is invisible but roughly halves sensor sensitivity. 850nm chosen - this is a doorbell, not a covert camera.
- **Wide angle**: the Wide variant gives 120 degrees diagonal FoV, appropriate for a porch at 0.5-3m.
- **Consequence**: CSI ribbon is ~30cm max, so **the Pi lives in the door enclosure**, not indoors.

Documented alternative: outdoor PoE RTSP camera + Pi indoors. Simpler weatherproofing and serviceability, but adds ~200-400ms of encode/network/decode latency, ~$60-100 of cost, and you lose the dual-stream trick. Worth revisiting only if the enclosure/thermal reality outdoors proves painful.

Daylight color is degraded on a NoIR sensor (pinkish cast). Acceptable: recognition runs on it fine, and clips are for review, not cinematography. A motorized IR-cut camera is a possible later upgrade.

## Audio: I2S amp + weatherized speaker (~$16)

- MAX98357A I2S mono amp breakout (~$6): digital audio straight off the Pi's I2S pins, no USB sound card, no analog noise, 3W into 4 ohm.
- 3W-5W 4/8-ohm mylar-cone or marine-style speaker (~$10), mounted behind a drainage-protected grille.
- 3W is genuinely loud at door distances. Volume in software (ALSA).

## Power and network

**Decision pending a wiring check at the door.** Both variants are designed; the primary is PoE.

### Variant A (primary): single Ethernet run, PoE+

One cable delivers power + network + NTP + dashboard access. Requires an 802.3at (PoE+, 25W) injector or switch port at the router end (~$20 injector if the switch is not PoE).

Pi 5 power caveat, stated plainly: the Pi 5 wants 5V/5A (25W), and the Pi 5 + Hailo + IR LEDs under load is realistically 12-18W. Two ways to deliver it:

1. **802.3at PoE+ splitter with USB-C PD output rated 5V/4A-5A** (~$20-25). Verify the 5V amperage rating before buying; many cheap splitters are 5V/2.4A and will brown out the Pi. With no USB peripherals in this build, 5V/4A plus `usb_max_current_enable` config is acceptable; 5V/5A is preferred.
2. Official Pi 5 PoE+ HAT (~$25). Mechanically conflicts with the AI HAT+ stack (both want the board top); only viable with a stacking header and careful standoff work. The splitter avoids the problem entirely - **splitter recommended**.

### Variant B (fallback): WiFi + mains

Pi 5 onboard WiFi 5 + outdoor-rated 27W USB-C supply from the nearest exterior outlet, in-enclosure or in a weatherproof outlet cover. All streaming stays on-device, so WiFi only carries dashboard views and MQTT chatter; bandwidth is a non-issue, but you gain a power brick to hide and lose wired reliability.

**Action item for you**: check whether an Ethernet pull to the door area is feasible (attic/basement/crawlspace route) and whether any exterior outlet is within ~2m of the door. That decides A vs B.

## Enclosure (~$25-35)

- IP65 ABS junction/project box, roughly 150x110x70mm to fit Pi + HAT + amp with airspace.
- Camera aperture: clear acrylic/glass window for the lens. **Mount the IR illuminator behind its own separate window** (or externally): a shared window internally reflects IR into the lens and whites out night frames.
- Speaker ports face downward with a mesh + drip path.
- Cable entry via IP68 cable glands (~$8), bottom entry only.
- Thermal: Pi 5 + Hailo in a sealed box in summer sun is the main risk. Include the official Pi Active Cooler (~$5), mount the box out of direct sun where possible, and treat sustained SoC >80C in `vcgencmd` as a redesign trigger (heatsink through-wall, vented double roof, or the RTSP-camera variant with the Pi indoors).

## Storage

- **SanDisk High Endurance 64GB microSD (~$12)** as the only storage. Clip recording is a continuous small write load; high-endurance (DVR-class) cards exist for exactly this.
- NVMe is **not** an option in this build: the AI HAT+ consumes the Pi 5's single PCIe lane. A dual-M.2 board exists but adds cost and height. If clip storage outgrows the card, add a USB 3 SSD (~$25) for the clip directory only.
- Retention policy (see ARCHITECTURE.md section 10) keeps usage bounded regardless.

## Bill of materials (primary variant)

| Item | Est. cost |
|---|---|
| Raspberry Pi 5, 8GB | $80 |
| Raspberry Pi AI HAT+ (Hailo-8L, 13 TOPS) | $70 |
| Pi Camera Module 3 NoIR Wide | $35 |
| 850nm IR illuminator board | $12 |
| MAX98357A I2S amp | $6 |
| 3-5W weatherized speaker | $10 |
| PoE+ splitter, USB-C 5V/4-5A | $22 |
| PoE+ injector (skip if switch has PoE+) | $20 |
| IP65 enclosure + glands + window | $30 |
| Pi Active Cooler | $5 |
| High Endurance 64GB microSD | $12 |
| CSI cable, standoffs, misc | $10 |
| **Total** | **~$312 (~$292 if your switch already does PoE+)** |

That lands right at the top of the $150-300 budget. Trims if needed: skip the injector (-$20 with a PoE+ switch), $20 enclosure (-$10), 32GB card (-$4): floor ~$278. Variant B (WiFi + mains) drops injector and splitter (-$42) and adds a 27W outdoor supply (+$15), landing ~$285.

Nice-to-have overflow (not in budget): USB SSD for clips ($25), 26 TOPS AI HAT+ (+$40), spare SD card for A/B update strategy ($12), PIR sensor for wake-on-motion power savings ($3, likely unnecessary since we run continuous detection anyway).
