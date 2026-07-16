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
   configuration of the server candidates.  These can only be read off the
   hardware.  They are marked **TODO (unboxing)** inline and gathered into a single
   checklist in the last section.

---

## 1. Inventory summary

| #  | Device            | Model                                   | Role                            | Notes                                                 |
|----|-------------------|-----------------------------------------|---------------------------------|-------------------------------------------------------|
| 2× | Indoor camera     | Reolink E1 (4MP, WiFi 6, pan-tilt)      | Indoor monitoring               | microSD slot; 2-way audio                             |
| 1× | Outdoor camera    | Reolink RLC-811WA                       | Outdoor monitoring              | 4K, 5× optical zoom; microSD slot; IP67               |
| 1× | ML accelerator    | Google Coral USB Accelerator            | Local object detection          | Edge TPU; wants USB 3.0; role now host-dependent (§3) |
| 1× | Server (down)     | Lenovo ThinkPad X1 Yoga (3rd gen, 2018) | Intended low-power NVR host     | ⚠ currently won't boot — see §4.1                     |
| 1× | Server (interim)  | Linux desktop                           | Near-term NVR host              | 8 GB GPU; higher idle draw                            |
| 1× | Server (new)      | NVIDIA Jetson AGX Orin 64 GB Dev Kit    | NVR-host candidate / AI offload | aarch64, Ampere GPU + DLA; can run whole NVR (§4.3)   |
| 1× | Router / uplink   | Starlink                                | WiFi + internet                 | CGNAT (no inbound); see `NETWORK.md`                  |

The server-host choice (**ADR-003 / issue M1-4**) is now **reopened with three
candidates** — the laptop (currently down), the desktop, and the newly-added
Jetson AGX Orin — plus a NixOS-integration tradeoff that did not exist before.
See **§4.0** for the current status and the near-term plan.

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

> **The Coral's role is now host-dependent.**  On an x86 host (laptop or desktop)
> the Coral is the detection accelerator, as designed.  But if the **Jetson AGX
> Orin** (§4.3) becomes the host, its GPU + DLA run detection directly (Frigate's
> TensorRT detector) and the Coral would be **redundant** — kept as a spare.  So
> whether we use the Coral at all is downstream of the ADR-003 host decision.

---

## 4. Server candidates

### 4.0 Status and near-term plan (revised 2026-07-16)

Two developments changed the server picture after the initial inventory:

1.  **The X1 Yoga is currently down.**  Its failing 512 GB SSD (bad sectors) was
    replaced with a new **2 TB SSD**, but the machine now fails to install Linux
    (the Ubuntu installer stalls from USB).  Until it is revived it cannot be the
    NVR host — though it remains the preferred *low-power* target if repaired
    (§4.1).
2.  **A Jetson AGX Orin 64 GB is available** and unused — a much more powerful
    machine that can run the entire NVR by itself (§4.3).

**Near-term plan (tight schedule — a one-week absence starts soon):**

-  **Camera first, server later.**  Getting the outdoor **RLC-811WA** recording
   to its own **microSD** and viewable/alerting through the Reolink app needs
   **no server at all**, and is the realistic goal before departure.  This means
   a *temporary*, time-boxed reliance on the Reolink app/cloud for the week — an
   explicit exception to the no-vendor-cloud stance in §6.7, to be undone once
   the NVR is live (M2-3).
-  **The NVR-host decision is not on the critical path** and should not be rushed
   under the deadline.  The **desktop** (§4.2) is the low-friction interim host;
   the **Jetson** (§4.3) is the strongest permanent candidate and is what M1-4
   should evaluate.
-  The host decision is **ADR-003 / issue #5**, still open — now weighing three
   machines plus a NixOS-vs-vendor-OS tradeoff.

### 4.1 Lenovo ThinkPad X1 Yoga, 3rd gen (2018) — preferred low-power host (currently DOWN)

**⚠ Current status: will not install/boot.**  After swapping the failing 512 GB
SSD for a new 2 TB SSD, the Ubuntu installer stalls when booted from USB.  Things
worth trying, in rough priority: switch the BIOS SATA/NVMe mode from **RAID /
Intel RST** to **AHCI** (Linux often can't see or install to the disk under RST);
disable **Secure Boot**; add the **`nomodeset`** kernel parameter at the
installer boot menu (the UHD 620 can hang the graphical installer); **re-write
the USB stick** and verify the ISO checksum, using a USB-A port; and update the
BIOS.  Until it boots it is out of the running — but if revived it is still the
best *low-power* host, so it is worth a second pass (tracked under M1-4 / M3-1).

The reason it was the initial choice, and still the low-power ideal if repaired
(ADR-003 / M1-4): low idle draw, silent, an Intel iGPU that decodes H.264 **and**
HEVC/H.265 in hardware (essential for the 4K outdoor stream), USB-A 3.0 for the
Coral and external storage, and a battery that rides through short power cuts as
a built-in UPS.

| Property | Value | Confidence |
|----------|-------|------------|
| CPU | 8th-gen Intel Core (i5-8250U/8350U or i7-8550U/8650U, Kaby Lake-R, 4C/8T, 15 W) | Spec range — ⚠ confirm exact CPU |
| iGPU | Intel UHD Graphics 620 — VAAPI / Quick Sync decode of H.264, HEVC 8-bit (and 10-bit), VP8/9 | Spec |
| RAM | Up to 16 GB LPDDR3-2133, **soldered** (not upgradeable) | Spec — ⚠ confirm amount |
| Storage | **New 2 TB M.2 NVMe** (replaced a failing 512 GB); OS + Frigate DB, recordings on external drive | ⚠ OS install pending — see status above |
| USB-A (for Coral + drive) | 2× USB-A 3.1 Gen 1 (= USB 3.0) | Spec — ⚠ confirm on unit |
| USB-C | 2× Thunderbolt 3 (USB-C) | Spec |
| Other ports | HDMI, microSD reader, mini-Ethernet (dongle), 3.5 mm | Spec |
| Battery | ~54 Wh (UPS role; charge thresholds set in M3-1) | Spec — ⚠ confirm health |
| Networking | WiFi + (adapter) Gigabit Ethernet — **prefer wired to the router if reachable** | — |

**Port budget check (feeds M1-4 acceptance):** two USB-A 3.0 ports cover the
Coral + one external USB drive directly, no hub required.  Confirm both ports
are USB 3.0 (they should be) on the actual unit.

Per-unit (TODO, once it boots):

- [ ] **Revive it** (see the AHCI / Secure Boot / `nomodeset` / USB-stick steps above)
- [ ] Exact CPU model (`lscpu`)
- [ ] Installed RAM (`free -h`)
- [ ] NVMe capacity and free space (`lsblk`, `df -h`)
- [ ] Battery health / design vs. full capacity
- [ ] BIOS: is "Power On with AC Attach" present? (recorded in ADR-003)
- [ ] Confirm both Type-A ports negotiate USB 3.0 with the Coral (`lsusb -t`)

### 4.2 Linux desktop — interim NVR host

With the laptop down, the desktop is the **low-friction interim host**: a clean
x86 target that runs NixOS + Frigate + the Coral without the Jetson's OS
complications.  Its higher idle draw — the original reason it was only "reserve"
— is the main cost; for an interim (or even permanent) host that may be
acceptable, and it is quantified in the ADR-003 power comparison.  It also
remains the machine for the separate **M7-2** GPU-enrichment evaluation.  Specs
are still TODO (fill in once it is powered up).

| Property | Value |
|----------|-------|
| CPU | _TODO — model_ |
| GPU | _TODO — model + VRAM (user reports "powerful 8 GB GPU")_ |
| RAM | _TODO_ |
| Storage | _TODO_ |
| PSU | _TODO — wattage (user reports "large power supply")_ |
| Idle / load power | _TODO — measure with a plug meter for the ADR-003 cost comparison_ |

### 4.3 NVIDIA Jetson AGX Orin 64 GB Developer Kit — new candidate (evaluate in M1-4)

Available and currently unused.  On raw capability this is by far the strongest
of the three machines, and it can run the **entire NVR by itself** — object
detection on its GPU/DLA and video decode on its media engine — which would make
the Coral unnecessary (§3).

| Property | Value | Confidence |
|----------|-------|------------|
| CPU | 12-core Arm Cortex-A78AE (Armv8.2, **64-bit / aarch64**) | Spec |
| GPU / accel | 2048-core NVIDIA Ampere GPU + 64 Tensor Cores; 2× NVDLA v2 | Spec |
| AI performance | up to ~275 TOPS (INT8) — vast overkill for three cameras | Spec |
| RAM | 64 GB LPDDR5 | Spec |
| Storage | 64 GB eMMC on-module + M.2 NVMe slot (add an SSD for recordings) | ⚠ confirm an NVMe is fitted |
| Video decode | NVDEC media engine — HW decode of H.264/H.265 incl. 4K (Orin has **no** HW *encoder*, which Frigate does not need for our setup) | Spec |
| Networking / ports | onboard Ethernet (RJ45), USB 3.2 (A/C), DisplayPort | ⚠ confirm Ethernet link speed |
| Power | configurable `nvpmodel`: 15 W / 30 W / 50 W / MAXN (~60 W) — a low mode is ample for 3 cameras | Spec |

> **"ARMv7" in the listing is a mislabel.**  The AGX Orin is 64-bit **aarch64**
> (Armv8.2, Cortex-A78AE); "ARMv7" would be 32-bit.  This matters because
> software — including Frigate's image — is built for arm64.

**Frigate on the Jetson — supported.**  Frigate ships a **`stable-tensorrt-jp6`**
image for JetPack 6+.  Detection runs on the GPU/DLA via the **TensorRT** (or
ONNX) detector — roughly **20–40 ms/inference**, slower per frame than the
Coral's ~10 ms but able to run larger/better models and many streams
effortlessly — and video decode uses the Jetson's **NVDEC** media engine.  One
box does everything the laptop-plus-Coral plan does, and more.

**The catch — NixOS integration.**  The Jetson's clean, vendor-supported path is
NVIDIA **JetPack** (an Ubuntu-based L4T image) + Docker, **not** NixOS.  NixOS is
possible via the community **[`jetpack-nixos`](https://github.com/anduril/jetpack-nixos)**
overlay — it supports `orin-agx` and is actively maintained — but it is advanced:
documented display-console quirks on Orin (no HDMI/DP console; serial for
troubleshooting), UEFI-boot caveats, and a vendor-BSP flashing process.  So the
Jetson trades some of our **declarative-NixOS reproducibility goal** (project
goal #3) for raw power and use of idle hardware.

**Working assessment (for ADR-003 / M1-4):** the Jetson is the most capable and
the most compute-per-watt option, and it is free (already owned, idle).  Its one
real downside is the NixOS friction.  Recommendation: evaluate it seriously as
the *permanent* host — likely JetPack + Docker first, `jetpack-nixos` later —
while the **desktop** stands up an NVR sooner if we want one running this week.
Measure its wall-power in a low `nvpmodel` for the ADR-003 comparison.

---

## 5. Consumables and purchases still needed

Sizes are finalized by the **M3-2** retention math; rough guidance here.

| Item | Qty | Guidance | Decided in |
|------|-----|----------|------------|
| microSD cards (camera-local failover) | 3 | High-endurance ("surveillance"/"endurance" rated) cards; size to the camera's max — 128–256 GB each is plenty for failover.  **One is needed now** for the 811WA camera-first bring-up (§4.0). | M2-4 |
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
| X1 Yoga: **revive it** (AHCI / Secure Boot / `nomodeset`), then CPU/RAM/SSD, USB 3.0, BIOS AC-attach | M1-4 / M3-1 |
| Desktop full specs + measured power draw | M1-4 (ADR-003 cost comparison) |
| Jetson: Frigate-on-JetPack vs `jetpack-nixos`; NVMe fitted; wall-power in low `nvpmodel` | M1-4 (ADR-003) |
| Host decision among laptop / desktop / Jetson (+ NixOS tradeoff) | ADR-003 / #5 |

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

**X1 Yoga (currently down — revive first):**
- [ ] Get it to install/boot: try AHCI (not Intel RST), disable Secure Boot, `nomodeset`, re-flash the USB stick
- [ ] Once booting: `lscpu`, `free -h`, `lsblk`, `df -h` — CPU, RAM, storage
- [ ] `lsusb -t` — confirm both Type-A ports are USB 3.0
- [ ] Battery health; BIOS "Power On with AC Attach" present?

**Desktop (interim host + ADR-003 comparison):**
- [ ] CPU, GPU + VRAM, RAM, PSU wattage
- [ ] Idle and under-load power draw (plug meter)

**Jetson AGX Orin (evaluate as permanent host):**
- [ ] Confirm it boots; note the JetPack version; is an M.2 NVMe fitted?
- [ ] Wall-power draw in a low `nvpmodel` (for the ADR-003 comparison)
- [ ] Decide path: JetPack + Docker (fast) vs `jetpack-nixos` (declarative)

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
