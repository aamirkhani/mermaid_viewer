# 06 · Glossary & Traceability

> So every claim in this set is auditable: a **glossary** of the domain/tech terms, and a
> **traceability map** from each diagram back to its source (Tech Overview / CUJ docs / Slack /
> POC code). Tags: 🅡 roadmap · 🅐 assumption.

**Contents**
1. [Glossary](#1-glossary)
2. [Source inventory](#2-source-inventory)
3. [Diagram → source traceability](#3-diagram--source-traceability)
4. [Code component → doc map](#4-code-component--doc-map)
5. [Roadmap vs. shipped ledger](#5-roadmap-vs-shipped-ledger)
6. [Open questions](#6-open-questions)

---

## 1. Glossary

| Term | Meaning |
|------|---------|
| **Jinju** | The display-less AR glasses (single camera, IMU, MIC, speaker) — a **phone accessory**, the target device. |
| **HaeAn / GG** | Glasses firmware builds. Camera streaming over Projected is **build-sensitive**: too-new builds bricked devices; a ~1-month-old build fixed black preview (Slack). |
| **ASP** | Application/Always-on Sensor Processor — the glass low-power island that runs always-on ambient sensing + micro-models while the AP sleeps. |
| **AP** | Application Processor — the heavier compute core (on glass it deep-sleeps; the **phone AP** is the main hub). |
| **Projected API** | `androidx.xr.projected` (alpha04). Phone obtains the glasses camera as a *projected device context* (`ProjectedContext.createProjectedDeviceContext`); needs Android 14 / API 34. |
| **Ambient (Low-Power) Sensing** | Always-on, low-power sensing on the ASP that wakes heavier compute only on an impact-class event (Tech Overview). |
| **Power tiering** | Tier 0 ambient → Tier 1 primed → Tier 2 active-capture; the system minimizes time in Tier 2 (`01 §7`, `03 §2`). |
| **Sensor fusion** | Combining time-stamped, confidence-tagged events from glass/phone/watch on a fusion bus (`01 §6`). |
| **Gemma 4 (E2B-it)** | The on-device VLM (`gemma-4-E2B-it.litertlm`) run via LiteRT-LM that answers visual questions via prompting — the bootstrap "brain". |
| **LiteRT-LM** | `com.google.ai.edge.litertlm` runtime (v0.10.0) used to run Gemma4 on device (`Backend.CPU()` today). |
| **TFLite** | TensorFlow Lite (2.16.1) — runs the 8-class audio hit classifier on the phone. |
| **Log-mel spectrogram** | The audio feature (FFT 1024, 64 mel bins, 25 ms window / 10 ms hop) feeding the hit classifier. |
| **Hit detection** | Detecting ball impact by fusing audio transient + IMU + motion gate (`02 §6.1`, `03 §5`). |
| **Motion gate** | Requiring `IDLE`/`NEAR_IDLE` motion before accepting a hit — rejects walking false-positives. |
| **Cooldown** | Min interval (default 5 s) between logged shots — de-bounce. |
| **About-to-Hit / Address** | Golfer is set up over the ball (`preparedToHit = club ∧ ball`; `addressPosition = prepared ∧ aboutToSwing`). |
| **Pre-buffer** | Glass-side circular RAM buffer (5–10 s) committed on hit so the *pre-swing* is captured (Tech Overview). 🅡 |
| **Occlusion / Erase Hat** | P0 CUJ: detect (and inpaint) occluders (hat/hands/grass/sun) over ball/club; built by **SRIB**. |
| **Strokes Gained (SG)** | Golf performance metric comparing a shot's outcome to a baseline; computed in cloud post-round. |
| **Tempo** | Backswing-to-downswing timing ratio — the swing's "rhythm" (not raw speed). |
| **CUJ** | Critical User Journey — the UX team's prioritized use-cases (the `Golf_SDK_CUJ_Tasks.xlsx` spine). |
| **SDK consumer / host app** | A 3rd-party (or GolfCues) app that integrates the Golf-SDK via its public API. |
| **Watch7 PoC** | UX team's Galaxy Watch experiment streaming IMU/audio over UDP for swing/shot counting. |

---

## 2. Source inventory

| Source | Type | What it gave us |
|--------|------|-----------------|
| `Golf-SDK_Tech_Overview.docx` | doc | goals, use-cases, ambient sensing, pre-buffer, ML model intents, feasibility |
| `0528_XRUXLab_Golf_CUJs.pdf` | doc (image) | CUJ definitions (UX) |
| `Golf-UI-Flow.pdf` | doc | UI flow / mode-entry options |
| `Golf_SDK_CUJ_Tasks.xlsx` | sheet | **CUJ priorities (P0/P1/P2)**, P0 = Erase Hat + Record/Count; SRIB occlusion note; single-camera limits |
| `0617_Special_Modes_for_Post_Jinju_Final_Share.pdf` | deck | post-Jinju special modes; slide 22 IMU+audio demos; watch IMU swing |
| `GolfCues-Sequence-Diagram-v1.png` | diagram | the **partial** v1 sequence (CUJ #10/#11) this set completes |
| `Screen_Recording_…GolfCues.mp4` | video | POC behaviour reference |
| `Siva meeting golf.m4a` | audio | meeting context |
| `slack-channel__golf-ideation.txt` | chat | **the decisions**: phone-as-hub, micro-vs-heavy ML split, power/streaming rule, capability negotiation, fusion, Projected API, HaeAn build saga, watch fusion |
| `ARG-MSV-Apps/GolfCues` (POC source) | code | **ground truth** for components, enums, thresholds, pipelines |

---

## 3. Diagram → source traceability

| Diagram | Doc | Primary source(s) |
|---------|-----|-------------------|
| System context (C4-L1) | 01 §2 | Tech Overview goals + Slack (SDK for 3rd-party) |
| Device topology (C4-L2) | 01 §3 | Slack (phone hub, Projected, BLE events, watch UDP) |
| On-phone component model (C4-L3) | 01 §4 | **POC code** (`GolfModeForegroundService` et al.) |
| Fusion contract | 01 §4.1, 03 §5 | `ShotDetectionEngine.evaluate()` |
| Glasses ASP/AP split | 01 §5 | Tech Overview (ambient) + Slack (micro vs heavy) |
| Sensor-fusion bus | 01 §6 | Slack (Penke fusion ask, Khani glass-IMU note) |
| Power & data tiering | 01 §7, 03 §2 | Slack (Khani streaming rule) + Tech Overview (ambient) |
| Transport stack | 01 §8 | Slack (Projected/BLE/UDP) + POC deps |
| Capability negotiation | 01 §9, 04 §1 | Slack (Penke "query capabilities") |
| Concurrency model | 01 §10 | **POC code** (serviceScope, flows, Mutex) |
| Failure/degradation | 01 §11 | Slack (HaeAn builds) + POC (`GolfCameraBindState`, `FakeAudioClassifier`) |
| Deployment | 01 §12 | Slack (build pinning) + POC (API 34 gate) |
| Three-tier ML | 02 §1 | Slack (Penke micro/heavy/cloud) |
| Per-CUJ model map | 02 §2 | CUJ xlsx + POC + Tech Overview |
| Gemma→in-house migration | 02 §3 | user's stated goal + POC seam (`GolfVisionLlmClient`) |
| Vision pipeline internals | 02 §5 | **POC code** (`GolfVisionAnalyzer/LlmClient/Prompts`) |
| Hit/club/occlusion design | 02 §6 | Tech Overview + CUJ xlsx (SRIB) + POC |
| All FSMs | 03 | **POC code** enums/thresholds (verbatim) |
| Hit→record→count (completes v1) | 04 §7 | **POC code** + v1 PNG |
| ⭐ Phone→glass capture | 04 §8 | Slack (Penke "very important" flow) + Tech Overview pre-buffer |
| Occlusion/Erase Hat | 04 §11 | CUJ xlsx (SRIB) + Tech Overview |
| Watch fusion | 04 §13 | Slack (Waghulde Watch7 PoC) |
| ER model / enums / API | 05 | **POC code** (`data.*`, `Models.kt`) + D9/D11 |

---

## 4. Code component → doc map

| POC class (`com.golfcues.app.*`) | Architectural role | Docs |
|----------------------------------|--------------------|------|
| `service.GolfModeForegroundService` | Session Engine / orchestrator | 01 §4,§10 · 03 §4 · 04 §5 |
| `service.GolfModeSessionState` | live state (StateFlow) | 03 §7,§8 · 05 §6 |
| `domain.detection.ShotDetectionEngine` | fusion/decision core | 01 §4.1 · 03 §5 · 04 §7,§9 |
| `domain.motion.MotionSensorManager` / `MotionStateDetector` | motion gate | 03 §6 · 04 §7 |
| `domain.audio.AudioRecorder` | PCM windows | 02 §6.1 · 04 §7 |
| `ml.AudioFeatureExtractor` | log-mel features | 02 §6.1 · 04 §7 |
| `ml.TfliteAudioClassifier` (+`FakeAudioClassifier`) | 8-class hit model | 02 §5,§6.1,§8 · 04 §7,§9 |
| `vision.GolfVisionAnalyzer` | vision interval loop | 02 §5 · 03 §8 · 04 §6 |
| `vision.GolfVisionLlmClient` | Gemma4/LiteRT-LM seam | 02 §3,§5 |
| `vision.GolfStancePrompts` | prompt + parse seam | 02 §3,§5 |
| `vision.GolfVisionSpeakHelper` | TTS tiers | 04 §6 |
| `xr.GlassesContextHelper` | Projected API access | 01 §3 · 03 §10 · 04 §15 |
| `camera.GolfCameraBindState` | glasses/phone camera FSM | 03 §10 · 04 §15 |
| `camera.GolfAddressVideoRecorder` | 5 s clip recorder | 03 §9 · 04 §7,§8 |
| `camera.GolfStanceSnapshotStore` | stance JPEG | 03 §9 · 04 §6 |
| `camera.FramePreprocessor` | 320² frame prep | 02 §5 |
| `model.ModelStorage` | model path resolve | 02 §7 |
| `data.*` (entities/dao/db/repos/Mappers) | persistence | 05 §1,§6 |
| `domain.model.Models` | enums + domain types | 05 §2 |
| `ui.live.LiveGolfModeViewModel` | live UI binding | 04 §14 · 05 §6 |

---

## 5. Roadmap vs. shipped ledger

A single honest list of **what exists in the POC** vs. **what the architecture proposes**.

| Capability | Status | Notes |
|------------|:------:|-------|
| Manual Golf-Mode entry | ✅ | Home → Start Golf Mode |
| Phone audio hit detection (TFLite 8-class) | ✅ | `golf_audio_classifier.tflite` |
| Motion gate (phone IMU) | ✅ | `MotionStateDetector` |
| Cooldown / sensitivity / ignored-audit | ✅ | `ShotDetectionEngine` |
| Vision stance (Gemma4 on CPU) | ✅ | side-loaded `.litertlm` |
| TTS cues over speaker | ✅ | `TtsManager` + tiers |
| 5 s address clip (phone CameraX) | ✅ | `GolfAddressVideoRecorder` |
| Glasses camera via Projected | ✅ | build-sensitive (HaeAn) |
| Phone/glasses camera fallback | ✅ | `GolfCameraBindState` |
| Session/shot persistence + feedback | ✅ | Room v4 |
| Glass ASP micro-model gate tier | 🅡 | today everything heavy is on phone |
| On-demand glass raw streaming (D3) | 🅡 | proposed contract (`03 §11`, `04 §8`) |
| Glass circular pre-buffer commit-on-hit | 🅡 | Tech Overview design |
| IMU decel-spike + watch swing confirm | 🅡 | fusion hardening |
| Occlusion / Erase Hat (P0) | 🅡 | **SRIB** building |
| Club detection net | 🅡 | Gemma4 cue today |
| Hole/pin localization + scorecard | 🅡 | needs GPS + glass-IMU orientation |
| Voice / audio auto-entry | 🅡 | manual + vision today |
| Cloud SG / coaching | 🅡 | opt-in design |
| In-house net migration | 🅡 | seam ready (`02 §3`) |
| NPU/GPU backend for Gemma4 | 🅡 | `Backend.CPU()` today |

---

## 6. Open questions

Worth resolving with the team (sourced from Slack threads + gaps in the source material):

1. **ASP micro-model contract** — exact list, formats, and event rates the Jinju ASP can run (🅐 we
   assumed human/motion/transient/hotword). Drives `01 §5`, `02 §1`, `03 §2`.
2. **Glass IMU streaming** — is on-demand head-orientation streaming feasible for hole localization
   (the one case Khani says it's truly needed)? (`04 §12`)
3. **Pre-buffer on Jinju** — does the glass support a RAM circular video buffer + commit, or must
   capture start cold on trigger? Determines whether `04 §8` can backfill the pre-swing.
4. **Bulk clip transport** — WiFi-Direct vs. companion-app bulk; resumable? Affects clip latency.
5. **Watch integration** — adopt the UX Watch7 UDP path as-is, or normalize onto the BLE fusion bus?
6. **Score vs. shot count** — when do we bind shots to holes (GPS) to produce a real scorecard?
7. **Known-good firmware pin** — which exact HaeAn/GG build is the supported baseline? (`01 §12`)
8. **Cloud boundary** — what leaves the device (frames? features only?) and the consent UX (D8).

---

*End of the Golf-SDK V4 architecture set. Start at [`README.md`](README.md).*
