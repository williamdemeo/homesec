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
- **Related**: ADR-003 (server host, M1-4), ADR-004 (Frigate deployment mechanism, M4-1), M2-1/M2-2 (camera streams), M3-3 (Coral / detector setup), M4-2/M4-3 (decode + detector), M7-2 (desktop-GPU enrichments)

## Context

We need the software that ingests the three Reolink cameras, records them, runs
local object detection, and presents a web/mobile UI — the heart of the system.
The choice constrains almost everything downstream (deployment in M4, the
detector wiring in M3-3/M4-3, notifications in M5), so it is settled first.

Requirements that matter for this project:

- **Local object detection on the Google Coral** we already own — no cloud
  inference.
- **Works well with Reolink cameras**, including the models with finicky RTSP.
- **Hardware-accelerated decode** on the host's GPU (VAAPI on an Intel iGPU, or
  NVDEC on the Jetson), so three streams (incl. 4K H.265) don't melt the CPU.
- **Reproducible, easily-redeployed configuration** — Docker on an Ubuntu-based
  OS is fine; NixOS is optional, not required.
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
- **Running detection on the desktop GPU — for now.**  Detection runs on the
  chosen host — the Coral on an x86 host, or the Orin GPU if ADR-003 selects the
  Jetson.  Whether the reserve desktop's 8 GB GPU earns its power draw for
  Frigate's heavier *enrichment* features is a separate evaluation deferred to
  **M7-2**; it is not part of the day-one NVR.

## Alternatives considered

Criteria from the issue, scored for our context:

| Criterion | **Frigate (chosen)** | ZoneMinder | Shinobi | Scrypted |
|---|---|---|---|---|
| Coral Edge TPU detection | ✅ first-class (`edgetpu`, ~10 ms/inference); also OpenVINO/TensorRT/ONNX | ⚠️ only via external add-ons (zmeventnotification/hooks) | ⚠️ plugin-based, less polished | ✅ has Coral support, but less mature than Frigate's |
| Reolink E1 / RLC-811WA support | ✅ documented per-model stream guidance (see below); go2rtc restreaming | ⚠️ generic RTSP/ONVIF; no Reolink-specific guidance | ⚠️ generic | ✅ good, plugin-based |
| Hardware decode (VAAPI on Intel iGPU) | ✅ `preset-vaapi` and other presets built in | ⚠️ possible but manual | ⚠️ manual | ✅ supported |
| Deployability (Docker / declarative) | ✅ official Docker image (runs on any host OS) + optional native NixOS module | ⚠️ packaged but heavier (MySQL/Apache stack) | ⚠️ mostly Docker | ⚠️ mostly Docker/npm |
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
  centric — but for a Coral-based, Linux, local-detection NVR, Frigate's
  Coral + go2rtc + Home Assistant integration is years more mature.
- **Viseron** (Docker, EdgeTPU-capable) and **Home-Assistant-centric** stacks
  (motionEye et al.) are viable but have smaller communities / are not
  purpose-built local-detection NVRs; noted for completeness, not chosen.

Frigate is the only option that is simultaneously **built around** local
detection (Coral, or the Jetson's GPU), has **explicit Reolink** support, does
**hardware decode** (VAAPI / NVDEC), deploys **cleanly on Linux** (Docker, or an
optional native NixOS module), and is **actively maintained** with a
family-friendly UI.

## Deployment options (feeds ADR-004 / M4-1)

Listed here as required; the *choice* is ADR-004.  With NixOS no longer a
requirement, the host OS is likely Ubuntu-based (JetPack on the Jetson — see
HARDWARE.md §4.3), which makes the Docker path the natural default.

1. **Official Docker image** (via `docker compose`, or `virtualisation.oci-containers`
   on NixOS), pinned by digest — the `stable-tensorrt-jp6` image on the Jetson.
   Gives bit-for-bit what upstream ships and tests (bundled ffmpeg presets,
   go2rtc, detector runtimes) and runs on any host OS, at the cost of a container
   layer and explicit device pass-through (the GPU/Coral and the decode device)
   plus an adequate `--shm-size`.  **Primary path.**
2. **Native `services.frigate` NixOS module** — optional, only if a host runs
   NixOS.  nixpkgs provides `services.frigate.{enable,package,settings,...}` with
   the config as a Nix attrset and an nginx vhost; most declarative, but the
   packaged version can lag upstream and the runtimes come together from nixpkgs.

Presumption to carry into ADR-004: the **pinned Docker image** for upstream
parity — decided for real in M4-1, not here.

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
fallback** if the Coral proves troublesome (a driver/kernel risk flagged in
M3-3).  Choosing Frigate keeps both paths open with a one-line config change;
the primary-vs-fallback call is validated in M3-3/M4-3, not here.

**Host caveat.**  This Coral-vs-OpenVINO analysis assumes an **x86 host** (the
laptop or desktop).  If ADR-003 selects the **Jetson** (now the frontrunner —
HARDWARE.md §4.3), detection instead runs on the Orin GPU/DLA via **TensorRT**,
the Coral is unnecessary, and this section is moot.

## Consequences

**Positive**

- The NVR is built around exactly our use case (local Coral detection on modest
  hardware); most later decisions have well-trodden Frigate answers.
- go2rtc (bundled) solves Reolink restreaming and one-connection-per-camera fan-
  out; VAAPI decode is a built-in preset; WebPush/MQTT cover notifications (M5).
- A clean Docker deployment path (runs on Ubuntu/JetPack/NixOS), plus an optional native NixOS module.
- Coral **and** OpenVINO detector support gives a hardware-independent fallback.

**Negative / to watch**

- **Coral driver/kernel fragility** (on any Linux) — an integration risk *if the
  host uses the Coral* (M3-3); mitigated by the OpenVINO fallback, or avoided
  entirely if the Jetson host is chosen (GPU detection, no Coral).
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
