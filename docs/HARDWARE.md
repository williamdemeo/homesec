<!--
File: docs/HARDWARE.md

Hardware inventory and system requirements for the homesec project.
Tracked by issue M1-1.  This is the reference document to which later milestones
point for device capabilities (stream  protocols, codecs, port types).
Spec-sheet values are recorded here; fields that must be read off the physical
hardware are marked "TODO (unboxing)" and consolidated in the checklist at the end.
-->

# Hardware inventory and system requirements

**Status**: living document · **Date started**: 2026-07-14 · **Issue**: M1-1

This file records

1. what equipment we have and the capabilities later work depends on, and
2. what the finished system must actually do.

Two kinds of entry appear below:

+  **Spec-sheet values** — taken from the manufacturers' published specifications
   (sources listed at the end).  These are  reliable enough to design against, but a
   few are flagged **⚠ verify on device** where the product line has variants or
   where community reports contradict the spec; each such flag names the M2 issue
   that will confirm it.
+  **Per-unit values** — MAC addresses, firmware versions at unboxing, and the exact
   configuration of the two server candidates.  These can only be read off the
   hardware.  They are marked **TODO (unboxing)** inline and gathered into a single
   checklist in the last section.

---

## 1. Inventory summary

| #  | Device           | Model                                   | Role                      | Notes                                   |
|----|------------------|-----------------------------------------|---------------------------|-----------------------------------------|
| 2× | Indoor camera    | Reolink E1 (4MP, WiFi 6, pan-tilt)      | Indoor monitoring         | microSD slot; 2-way audio               |
| 1× | Outdoor camera   | Reolink RLC-811WA                       | Outdoor monitoring        | 4K, 5× optical zoom; microSD slot; IP67 |
| 1× | ML accelerator   | Google Coral USB Accelerator            | Local object detection    | Edge TPU; wants USB 3.0                 |
| 1× | Server (chosen)  | Lenovo ThinkPad X1 Yoga (3rd gen, 2018) | 24/7 NVR host             | Intel iGPU decode; battery = UPS        |
| 1× | Server (reserve) | Linux desktop                           | Reserve / stretch compute | 8 GB GPU; higher idle draw              |
| 1× | Router / uplink  | Starlink                                | WiFi + internet           | CGNAT (no inbound); see `NETWORK.md`    |

The server choice (laptop vs. desktop) is decided in **ADR-003 / issue M1-4**;
the desktop is documented here for completeness and held in reserve for the
[M7-2] GPU-enrichment evaluation.

---

## 2. Cameras

All three are Reolink WiFi cameras with a local microSD slot.  For the NVR we
care about four things per camera:

1.  the **stream sources** (protocol + URL + codec + resolution/fps for a full-quality
    *main* stream and a low-cost *sub* stream),
2.  **night-vision** behavior,
3.  **microSD** capacity for camera-local failover recording, and
4.  (indoors) **pan/tilt**.

> **Stream-protocol note.**  Reolink cameras typically expose RTSP (port 554),
> ONVIF (port 8000), and an HTTP-FLV stream (port 1935, `bcs` app).  Frigate's
> Reolink guidance recommends the **HTTP-FLV** stream for several Reolink
> models because their RTSP implementation can be unstable.  Which source we
> actually use per camera is **verified and recorded in M2** (M2-1 outdoor,
> M2-2 indoor) — the URL templates below are starting points, not confirmed
> facts.

Reference URL templates (to be confirmed in M2, using the dedicated `nvr`
account created in M2-3, with credentials injected from the secrets store):

```
RTSP  main:  rtsp://<user>:<pass>@<ip>:554/h264Preview_01_main   (or h265Preview_01_main)
RTSP  sub:   rtsp://<user>:<pass>@<ip>:554/h264Preview_01_sub
FLV   main:  http://<ip>/flv?port=1935&app=bcs&stream=channel0_main.bcs&user=<user>&password=<pass>
FLV   sub:   http://<ip>/flv?port=1935&app=bcs&stream=channel0_ext.bcs&user=<user>&password=<pass>
```

### 2.1 Reolink E1 — indoor (×2)

Retail description: *"E1 (2 Pack), Smart Human/Pet Detection · 4MP HD Plug-in
Indoor WiFi 6 Pan Tilt Pet Camera, Baby Monitor, Night Vision, 2-Way Talk,
Local microSD Card Storage."*

| Property            | Value                                                   | Confidence      |
|---------------------|---------------------------------------------------------|-----------------|
| Sensor / resolution | 4 MP, 2560 × 1440 ("Super HD")                          | Spec            |
| Main-stream codec   | H.264 (confirm whether H.265 is offered)                | ⚠ verify — M2-2 |
| Sub-stream          | Lower-res "fluent" stream (e.g. 640×480 / 15 fps class) | ⚠ verify — M2-2 |
| WiFi band           | **Listing says "WiFi 6."** Reolink's classic E1 spec lists **2.4 GHz-only**; E1 Pro/Zoom add dual-band. Listing and legacy spec disagree. | ⚠ verify on device — see note below, feeds M1-2 |
| Pan / tilt          | ≈355° pan / ≈50° tilt (marketed as 360° horizontal) | ⚠ verify in-app |
| Night vision        | Infrared, ~8 IR LEDs, range ~12 m (40 ft) | Spec |
| Two-way audio       | Yes (mic + speaker) | Spec |
| microSD             | Up to 256 GB per classic-E1 spec (some variants list 512 GB) | ⚠ verify max accepted |
| Protocols           | RTSP :554, ONVIF :8000, RTMP :1935, media :9000 (per E1 spec) — **RTSP reliability varies by E1 revision/firmware** | ⚠ verify — M2-2 |
| Smart detection     | Human / pet (on-camera); Frigate will do its own detection on the Coral | Spec |

**⚠ WiFi-band question (matters for M1-2).**  The box advertises "WiFi 6," but
Reolink's long-standing E1 specification lists 2.4 GHz-only radio (the "WiFi 6"
claim on some listings refers to compatibility with WiFi-6 *routers*, not a
5 GHz-capable camera radio).  Do not assume 5 GHz on the E1s: check
**Settings → Network** in the Reolink app for a 5 GHz option on each unit and
record the answer.  The outdoor RLC-811WA *is* confirmed dual-band; the indoor
E1s may not be, which changes the 2.4 GHz channel plan in `NETWORK.md`.

Per-unit (TODO, unboxing — one row per camera):

| Unit | Location (planned)   | MAC    | Firmware @ unboxing | Assigned IP                  |
|------|----------------------|--------|---------------------|------------------------------|
| E1-a | front entry landing  | _TODO_ | _TODO_              | _TODO (per NETWORK.md plan)_ |
| E1-b | second floor landing | _TODO_ | _TODO_              | _TODO_                       |

### 2.2 Reolink RLC-811WA — outdoor (×1)

Retail description: *"4K 8MP Outdoor Security Camera with 5X Optical Zoom,
Wi-Fi 6, H.265 Recording, Color Night Vision, Human/Vehicle/Animal Detection."*

| Property            | Value                                   | Confidence |
|---------------------|-----------------------------------------|------------|
| Sensor / resolution | 8 MP, 3840 × 2160 @ 15 fps; 1/2.8″ CMOS | Spec |
| Main-stream codec   | **H.265** (also H.264) — H.265 halves 4K storage and the X1 Yoga decodes it in hardware | Spec |
| Sub-stream          | Lower-res "fluent" stream for detection | ⚠ verify exact res/fps — M2-1 |
| Optical zoom / lens | 5× optical, f = 2.7–13.5 mm, F1.6; FOV ~100°–31° (wide→tele) | Spec |
| WiFi band           | 2.4 / 5 GHz, WiFi 6 (dual-band confirmed) | Spec |
| Pan / tilt          | None (fixed mount; zoom only) | Spec |
| Night vision        | IR to ~30 m **plus** color night vision via 5× spotlights (4 W, 6500 K) | Spec |
| Two-way audio       | Yes | Spec |
| Weatherproof        | IP67 | Spec |
| microSD             | microSD slot (confirm max, typically up to 256 GB) | ⚠ verify max accepted |
| Protocols           | RTSP :554 + ONVIF :8000 (standard on Reolink 8-series) | ⚠ confirm paths — M2-1 |
| Smart detection     | Human / vehicle / animal (on-camera) | Spec |

Per-unit (TODO, unboxing):

| Unit  | Location (planned) | MAC    | Firmware @ unboxing | Assigned IP |
|-------|--------------------|--------|---------------------|-------------|
| 811WA | NE corner of house | _TODO_ | _TODO_              | _TODO_      |

---

## 3. Compute — Google Coral USB Accelerator

The Coral offloads Frigate's object detection from the CPU to a dedicated Edge
TPU, which is what makes running three cameras on a low-power laptop feasible.

| Property | Value |
|----------|-------|
| Type | Edge TPU ML coprocessor (Google Coral USB Accelerator) |
| Throughput | ~4 TOPS (int8) at ~2 W; ~10 ms/inference on Frigate's default MobileDet model |
| Interface | USB 3.0 (Type-C on the device; ships with USB-C-to-USB-C cable) — **wants a USB 3.0 host port**; USB 2.0 works but roughly triples inference latency |
| Host software | `libedgetpu` runtime + udev rules; used by Frigate as the `edgetpu` detector — set up in M3-3 |
| USB enumeration quirk | Enumerates as `1a6e:089a` (Global Unichip) before firmware upload, then re-enumerates as `18d1:9302` (Google Inc.) after first use — **udev rules must cover both IDs** |
| Thermal | Runs hot under sustained load; needs ventilation and cable strain relief |

No per-unit data to collect beyond confirming it enumerates (done in M3-3).

---

## 4. Server candidates

### 4.1 Lenovo ThinkPad X1 Yoga, 3rd gen (2018) — chosen host

Why it's the presumptive choice (ADR-003 / M1-4): low idle draw, silent, an
Intel iGPU that decodes H.264 **and** HEVC/H.265 in hardware (essential for the
4K outdoor stream), USB-A 3.0 for the Coral and external storage, and a battery
that rides through short power cuts as a built-in UPS.

| Property | Value | Confidence |
|----------|-------|------------|
| CPU | 8th-gen Intel Core (i5-8250U/8350U or i7-8550U/8650U, Kaby Lake-R, 4C/8T, 15 W) | Spec range — ⚠ confirm exact CPU |
| iGPU | Intel UHD Graphics 620 — VAAPI / Quick Sync decode of H.264, HEVC 8-bit (and 10-bit), VP8/9 | Spec |
| RAM | Up to 16 GB LPDDR3-2133, **soldered** (not upgradeable) | Spec — ⚠ confirm amount |
| Storage | M.2 NVMe SSD (OS + Frigate DB; recordings go to external drive) | 2 Tb confirm capacity |
| USB-A (for Coral + drive) | 2× USB-A 3.1 Gen 1 (= USB 3.0) | Spec — ⚠ confirm on unit |
| USB-C | 2× Thunderbolt 3 (USB-C) | Spec |
| Other ports | HDMI, microSD reader, mini-Ethernet (dongle), 3.5 mm | Spec |
| Battery | ~54 Wh (UPS role; charge thresholds set in M3-1) | Spec — ⚠ confirm health |
| Networking | WiFi + (adapter) Gigabit Ethernet — **prefer wired to the router if reachable** | — |

**Port budget check (feeds M1-4 acceptance):** two USB-A 3.0 ports cover the
Coral + one external USB drive directly, no hub required.  Confirm both ports
are USB 3.0 (they should be) on the actual unit.

Per-unit (TODO, unboxing / inspection):

- [ ] Exact CPU model (`lscpu`)
- [ ] Installed RAM (`free -h`)
- [ ] NVMe capacity and free space (`lsblk`, `df -h`)
- [ ] Battery health / design vs. full capacity
- [ ] BIOS: is "Power On with AC Attach" present? (recorded in ADR-003)
- [ ] Confirm both Type-A ports negotiate USB 3.0 with the Coral (`lsusb -t`)

### 4.2 Linux desktop — reserve

Held in reserve for the M7-2 evaluation (does the 8 GB GPU earn its power draw
for Frigate's enrichment features — semantic search, face/plate recognition,
generative descriptions?).  Not part of the day-one build.

| Property | Value |
|----------|-------|
| CPU | _TODO — model_ |
| GPU | _TODO — model + VRAM (user reports "powerful 8 GB GPU")_ |
| RAM | _TODO_ |
| Storage | _TODO_ |
| PSU | _TODO — wattage (user reports "large power supply")_ |
| Idle / load power | _TODO — measure with a plug meter for the ADR-003 cost comparison_ |

---

## 5. Consumables and purchases still needed

Sizes are finalized by the **M3-2** retention math; rough guidance here.

| Item | Qty | Guidance | Decided in |
|------|-----|----------|------------|
| microSD cards (camera-local failover) | 3 | High-endurance ("surveillance"/"endurance" rated) cards; size to the camera's max — 128–256 GB each is plenty for failover | M2-4 |
| External USB drive (NVR recordings) | 1 | Powered USB 3.0 drive or SSD preferred over a bus-powered spinner for 24/7 duty; ~2 TB is a sensible start (≈2 weeks continuous or months of motion-only for 3 cameras) | M3-2 |
| Camera mounting hardware | as needed | Outdoor 811WA: mount ≥ 2.5 m, aimed at the approach, within WiFi range; indoor E1s: shelf/wall near outlets | M2-1 / M2-2 |
| (Possible) USB hub — powered, USB 3.0 | 0–1 | Only if the two laptop Type-A ports prove insufficient; the port-budget check says not needed | M3-2 |
| (Possible) Ethernet adapter for the laptop | 0–1 | If wiring the server to the router (preferred over WiFi for the NVR) | M3-1 |

---

## 6. System requirements

What the finished system must do.  Written so that **M4-5 (end-to-end
validation)** has concrete, testable targets.

### 6.1 What we monitor

- **Outdoor (RLC-811WA):** the approach / driveway and entry — the primary
  "who's coming to the house" camera; 5× zoom for detail at distance.
- **Indoor (2× E1):** interior common areas / entry points and pet areas;
  optional baby-monitor duty.  Pan-tilt widens coverage, but detection zones
  (M4-3) assume a defined home position.

### 6.2 Detection expectations

- **Person** detection on every camera, **day and night**, at ranges that
  matter for each location (recorded per-camera during M4-5 walk tests).
- **Vehicle** detection in the outdoor driveway zone (arrival and departure).
- **Pet** (cat/dog) detection indoors that does **not** raise person alerts.
- Detection runs **locally on the Coral** (not CPU fallback, not any cloud).

### 6.3 Day / night image quality

- Usable color image by day on all cameras.
- At night: the 811WA uses color night vision (spotlight) with IR fallback; the
  E1s use IR (~12 m).  The bar is **detection-relevant** image quality at the
  distances that matter — not merely a bright-looking picture.

### 6.4 Recording and retention

- **Initial target: 14 days** of motion/alert events retained, all cameras.
- Continuous (24/7) recording only if the storage envelope allows — decided in
  **M4-4**, sized in **M3-2**.
- Frigate's retention must fill-and-rotate gracefully; the filesystem must
  never hit 100 % (monitored in M6-1).

### 6.5 Notifications

- **Person/vehicle-in-zone → phone push in under 30 seconds**, measured
  end-to-end over cellular (M5-2).
- Notifications scoped to *alerts*, not every detection — notification fatigue
  is a first-class failure mode.

### 6.6 Availability

- Headless, unattended, 24/7; recovers after reboot; rides short power cuts on
  the laptop battery.
- Alerts us on the failure modes that matter — service down, camera offline,
  disk filling, Coral failure (M6-1).

### 6.7 Privacy and access stance

- **No vendor cloud once the NVR is live:** Reolink UID/P2P disabled on every
  camera (M2-3).  Trade-off accepted: the Reolink phone app then works only on
  the LAN; remote viewing is the NVR's job over our own Tailscale overlay (M5).
- Video stays on-premises; it leaves the house only over our encrypted
  Tailscale network, never via port-forwarding (Starlink CGNAT prevents that
  anyway, and we would not want it).
- Object detection is local (Coral), not a third-party service.

---

## 7. "Verify on device" items → where they're resolved

Satisfies M1-1's acceptance criterion that every verify-on-device question is
tracked by an M2 (or later) task.

| Question | Resolved in |
|----------|-------------|
| E1 WiFi band (2.4 GHz-only vs. dual-band) | M1-2 (network/band plan) — check in Reolink app |
| E1 usable stream source (RTSP vs. HTTP-FLV) + exact main/sub res/fps/codec | M2-2 |
| E1 pan/tilt exact ranges; H.265 availability; microSD max | M2-2 / M2-4 |
| 811WA exact sub-stream res/fps; RTSP/ONVIF stream paths | M2-1 |
| 811WA microSD max | M2-1 / M2-4 |
| Coral enumerates and hits expected inference latency | M3-3 |
| X1 Yoga exact CPU/RAM/SSD, USB 3.0 confirmation, BIOS AC-attach | M1-4 / M3-1 |
| Desktop full specs + measured power draw | M1-4 (ADR-003 cost comparison) |

---

## 8. Consolidated unboxing checklist

Read these off the hardware as it comes out of the box and fill in the tables
above.

**Cameras (each of the 3):**
- [ ] Record model, MAC address, firmware version *before* updating
- [ ] Update firmware; record the new version
- [ ] Confirm WiFi band options (esp. the E1s — 5 GHz available?)
- [ ] Confirm microSD max capacity accepted

**Coral:**
- [ ] Confirm it's the USB Accelerator (Type-C) and locate the USB-C-to-USB-C cable

**X1 Yoga (chosen server):**
- [ ] `lscpu`, `free -h`, `lsblk`, `df -h` — CPU, RAM, storage
- [ ] `lsusb -t` — confirm both Type-A ports are USB 3.0
- [ ] Battery health; BIOS "Power On with AC Attach" present?

**Desktop (reserve — for the ADR-003 comparison):**
- [ ] CPU, GPU + VRAM, RAM, PSU wattage
- [ ] Idle and under-load power draw (plug meter)

---

## Sources

Spec-sheet values above are drawn from the manufacturers' official product
pages and corroborating spec listings (accessed 2026-07-14):

- Reolink E1 — [official product page](https://reolink.com/product/e1/) and
  [E1 series overview](https://store.reolink.com/e1-series-cameras/).
- Reolink RLC-811WA — [official product page](https://reolink.com/product/rlc-811wa/)
  and [Amazon listing](https://www.amazon.com/REOLINK-Security-Recording-Detection-RLC-811WA/dp/B0CDPX99X2).
- Lenovo ThinkPad X1 Yoga (3rd Gen) —
  [Lenovo PSREF platform specifications](https://psref.lenovo.com/syspool/Sys/PDF/ThinkPad/ThinkPad_X1_Yoga_3rd_Gen/ThinkPad_X1_Yoga_3rd_Gen_Spec.PDF)
  and [LaptopMedia specs](https://laptopmedia.com/series/lenovo-thinkpad-x1-yoga-3rd-gen-2018/).
- Google Coral USB Accelerator — [coral.ai datasheet](https://coral.ai/products/accelerator/).

Where a listing's marketing (e.g. the E1 "WiFi 6" claim) conflicts with the
detailed specification, the conflict is flagged **⚠ verify on device** above
rather than resolved on paper.
