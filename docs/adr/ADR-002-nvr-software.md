<!--
File: docs/adr/ADR-002-nvr-software.md

Architecture Decision Record for the homesec project (issue M1-3).
ADR format matches ADR-001 and remains provisional pending the shared
template/index in M1-5.
-->

# ADR-002: NVR software — Frigate

- **Status**: Accepted
- **Date**: 2026-07-14
- **Deciders**: @williamdemeo
- **Issue**: M1-3 (#4)
- **Related**: ADR-003 (server host, M1-4), ADR-004 (Frigate deployment mechanism, M4-1), M2-1/M2-2 (camera streams), M3-3 (Coral on NixOS), M4-2/M4-3 (decode + detector), M7-2 (desktop-GPU enrichments)

## Context

We need the software that ingests the three Reolink cameras, records them, runs
local object detection, and presents a web/mobile UI — the heart of the system.
The choice constrains almost everything downstream (deployment in M4, the
detector wiring in M3-3/M4-3, notifications in M5), so it is settled first.

Requirements that matter for this project:

- **Local object detection on the Google Coral** we already own — no cloud
  inference.
- **Works well with Reolink cameras**, including the models with finicky RTSP.
- **Hardware-accelerated decode** on the X1 Yoga's Intel iGPU (VAAPI), so three
  streams (incl. 4K H.265) don't melt the CPU.
- **Declarative deployment on NixOS.**
- A **web + phone UI** usable by non-technical family, with **notifications**.
- An **actively maintained** project (a surveillance system is long-lived).

## Decision

**Use [Frigate](https://frigate.video) as the NVR software.**  Frigate is a
purpose-built, local-first, open-source NVR whose entire design centers on
real-time object detection on modest hardware — exactly our use case.  It has
first-class Coral support, bundles [go2rtc](https://github.com/AlexxIT/go2rtc)
for restreaming, does VAAPI hardware decode, ships a good web/mobile UI with
built-in authentication and notifications, and is very actively developed.

**Version pin (feeds M4-1).**  Current stable is **Frigate 0.17.2** (July 2026).
M4-1 should pin a specific **0.17.x** release (by image digest if containerized)
and bump deliberately per the M6-3 update routine.  Note two relevant recent
changes when M4 configures recording/snapshots: 0.16 added **face
detection/recognition**, and **0.17 removed the `clean_copy` snapshot option**
(snapshots are now clean, unannotated WebP by default) — so M4-4 should not
reference `clean_copy`.

### What we are explicitly NOT choosing

- **Any cloud NVR** (Reolink Cloud, Ring, Nest, etc.) — contrary to the whole
  local-first, no-vendor-cloud premise (see ADR-001 and the HARDWARE.md privacy
  stance).
- **Running detection on the desktop GPU — for now.**  The plan is the Coral on
  the laptop.  Whether the reserve desktop's 8 GB GPU earns its power draw for
  Frigate's heavier *enrichment* features is a separate evaluation deferred to
  **M7-2**; it is not part of the day-one NVR.

## Alternatives considered

Criteria from the issue, scored for our context:

| Criterion | **Frigate (chosen)** | ZoneMinder | Shinobi | Scrypted |
|---|---|---|---|---|
| Coral Edge TPU detection | ✅ first-class (`edgetpu`, ~10 ms/inference); also OpenVINO/TensorRT/ONNX | ⚠️ only via external add-ons (zmeventnotification/hooks) | ⚠️ plugin-based, less polished | ✅ has Coral support, but less mature than Frigate's |
| Reolink E1 / RLC-811WA support | ✅ documented per-model stream guidance (see below); go2rtc restreaming | ⚠️ generic RTSP/ONVIF; no Reolink-specific guidance | ⚠️ generic | ✅ good, plugin-based |
| Hardware decode (VAAPI on Intel iGPU) | ✅ `preset-vaapi` and other presets built in | ⚠️ possible but manual | ⚠️ manual | ✅ supported |
| NixOS deployability | ✅ native `services.frigate` module **and** official OCI image (see below) | ⚠️ packaged but heavier (MySQL/Apache stack) | ⚠️ mostly Docker | ⚠️ mostly Docker/npm |
| Web + phone UI | ✅ polished responsive UI + PWA; built-in auth | ⚠️ dated UI | ⚠️ functional, buggy per reports | ✅ good, strong Apple/HomeKit story |
| Notifications | ✅ built-in WebPush + MQTT + HA integration | ⚠️ via add-ons | ⚠️ basic | ✅ via HA/HomeKit |
| Project activity / maintenance | ✅ very active (0.17.x, 2026) | ⚠️ mature but slow-moving | ⚠️ intermittent | ✅ active |

**Why not the others.**

- **ZoneMinder** is the veteran and rock-solid at plain recording, but object
  detection is a bolt-on, its stack (Apache/MySQL/Perl) is heavy to run
  declaratively, and it has no Coral or Reolink story to speak of.
- **Shinobi** is popular but community reports cite persistent glitches, and its
  detection/Coral integration is not in Frigate's league.
- **Scrypted** is excellent — especially if the household were Apple/HomeKit-
  centric — but for a Coral-based, Linux/NixOS, local-detection NVR, Frigate's
  Coral + go2rtc + Home Assistant integration is years more mature.
- **Viseron** (Docker, EdgeTPU-capable) and **Home-Assistant-centric** stacks
  (motionEye et al.) are viable but have smaller communities / are not
  purpose-built local-detection NVRs; noted for completeness, not chosen.

Frigate is the only option that is simultaneously **built around** local Coral
detection, has **explicit Reolink** support, does **VAAPI** decode, deploys
**natively on NixOS**, and is **actively maintained** with a family-friendly UI.

## Deployment options on NixOS (feeds ADR-004 / M4-1)

Listed here as required; the *choice* between them is ADR-004.

1. **Native `services.frigate` NixOS module.**  nixpkgs provides
   `services.frigate.{enable,package,settings,checkConfig,...}`, with the Frigate
   config expressed as a Nix attrset and an nginx vhost set up for the UI.  Most
   declarative; but the packaged version can lag upstream, and the
   ffmpeg/go2rtc/detector runtimes must come together from nixpkgs (hardware-
   accel and Coral wiring is more hands-on).
2. **Official OCI image via `virtualisation.oci-containers`** (docker or podman
   backend), pinned by digest.  Gives bit-for-bit what upstream ships and tests
   (bundled ffmpeg presets, go2rtc, detector runtimes), at the cost of a
   container layer and explicit device pass-through (`/dev/bus/usb` for the
   Coral, `/dev/dri` for VAAPI) and an adequate `--shm-size`.

Presumption to carry into ADR-004: the **pinned OCI image** for upstream parity,
with the config templated from Nix — but that is decided in M4-1, not here.

## Reolink stream guidance (feeds M2-1 / M2-2)

From Frigate's camera-specific documentation — "the http video streams seem to
be the most reliable for Reolink," while RTSP "isn't always reliable on all
hardware versions."  Recommendation by resolution/model:

| Our camera | Frigate's recommendation | Notes |
|------------|--------------------------|-------|
| **Reolink E1** (4MP, ≤5MP) | **`http-flv` (H.264)** via go2rtc | e.g. `ffmpeg:http://<ip>/flv?port=1935&app=bcs&stream=channel0_main.bcs&user=…&password=…#video=copy#audio=copy` |
| **RLC-811WA** (8MP, "RLC-8##" older 6MP+ model) | **`rtsp`** for the main stream | H.265 4K; still verify stability in M2-1 |

Camera-side settings Frigate recommends (feeds **M2-4**): enable **"On, fluency
first"** (CBR) and **"Interframe Space 1x"** (i-frame interval = frame rate)
where available.

This *refines* the more tentative "prefer HTTP-FLV for Reolink" note in
`HARDWARE.md`: HTTP-FLV for the 4MP E1s, but RTSP is Frigate's pick for the 8MP
811WA.  M2-1/M2-2 verify both against our actual firmware and record the final
URLs in `CAMERAS.md`.

## Detector note: Coral primary, OpenVINO fallback (feeds M3-3 / M4-3)

Frigate supports the Coral `edgetpu` detector and the `openvino` detector on
Intel iGPUs (Skylake/6th-gen and newer — the X1 Yoga's UHD 620 qualifies), but
**detectors cannot be mixed** for object detection.

For *this* hardware the Coral is the right **primary** detector: the UHD 620 will
already be busy doing VAAPI **decode** of three streams (including 4K H.265), so
putting **inference** on the same modest iGPU (OpenVINO) would make decode and
detection contend for it.  The Coral offloads inference to dedicated silicon
(~10 ms) and leaves the iGPU free to decode — which is the whole reason we own
it.

That said, community experience is that OpenVINO on an Intel iGPU is a viable
alternative with easier driver maintenance (Coral's kernel/driver support has
grown fussier over time).  So **OpenVINO on the UHD 620 is our documented
fallback** if the Coral proves troublesome under NixOS (a real risk flagged in
M3-3).  Choosing Frigate keeps both paths open with a one-line config change;
the primary-vs-fallback call is validated in M3-3/M4-3, not here.

## Consequences

**Positive**

- The NVR is built around exactly our use case (local Coral detection on modest
  hardware); most later decisions have well-trodden Frigate answers.
- go2rtc (bundled) solves Reolink restreaming and one-connection-per-camera fan-
  out; VAAPI decode is a built-in preset; WebPush/MQTT cover notifications (M5).
- Two clean NixOS deployment paths (native module or pinned OCI image).
- Coral **and** OpenVINO detector support gives a hardware-independent fallback.

**Negative / to watch**

- **Coral driver/kernel fragility on NixOS** — the top integration risk (M3-3);
  the OpenVINO fallback above is the mitigation.
- **Version churn** — Frigate moves fast and has breaking changes between minor
  versions (e.g. `clean_copy` removed in 0.17). Pin a version and bump
  deliberately (M6-3); re-run the M4-5 validation subset after each bump.
- **Still 0.x** — no 1.0 stability guarantee; acceptable given the very active
  maintenance and large user base, but it is why the pin-and-test discipline
  matters.

## Revisit conditions

- Frigate drops or breaks Coral support **and** OpenVINO on the UHD 620 proves
  inadequate → reconsider detector/host (interacts with ADR-003, M7-2).
- The project stalls (no releases / maintainer exit) → re-evaluate against
  whatever the active alternatives are at that time.
- We move NVR duty to the desktop GPU after the M7-2 evaluation → revisit
  detector config (not necessarily the software).

## References

- [Frigate](https://frigate.video/) and [Frigate releases](https://github.com/blakeblackshear/frigate/releases) (current stable 0.17.2).
- [Frigate — Object Detectors](https://docs.frigate.video/configuration/object_detectors/) (Coral `edgetpu`, OpenVINO, no mixing).
- [Frigate — Camera-specific setup (Reolink)](https://docs.frigate.video/configuration/camera_specific/).
- [Frigate — Recommended hardware](https://docs.frigate.video/frigate/hardware/).
- NixOS: [`services.frigate` options](https://search.nixos.org/options?query=services.frigate) and [`virtualisation.oci-containers`](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/virtualisation/oci-containers.nix).
