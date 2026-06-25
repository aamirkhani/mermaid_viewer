# 04 · Sequence Diagrams (Complete Catalog)

> **This chapter completes the incomplete v1 diagram.** The original `GolfCues-Sequence-Diagram-v1.png`
> (Luo) covered CUJ #10/#11 — *Live count shots / record* — **partially**, in one pass. This catalog
> re-derives that flow end-to-end **and** adds every other entry path and P0/P1 CUJ, the critical
> *phone-triggers-N-sec-glass-capture-and-retrieves-it* flow Penke flagged, capability negotiation,
> watch fusion, occlusion/erase-hat, post-round cloud analysis, and the error/recovery paths.
> Participant names map to POC classes. Tags: 🅡 roadmap · 🅐 assumption.

**Index**

| # | Sequence | CUJ | Status |
|---|----------|-----|:------:|
| 1 | [Pairing & capability negotiation](#1-pairing--capability-negotiation) | setup | 🅡 |
| 2 | [Golf-Mode entry — Manual (PUI/gesture)](#2-golf-mode-entry--manual-puigesture) | entry P0 | ✅ |
| 3 | [Golf-Mode entry — Voice / hotword](#3-golf-mode-entry--voice--hotword) | entry P1 | 🅡 |
| 4 | [Golf-Mode entry — Auto (Visual + IMU)](#4-golf-mode-entry--auto-visual--imu) | entry P0 | partial |
| 5 | [Service startup — pipeline bring-up](#5-service-startup--pipeline-bring-up) | infra | ✅ |
| 6 | [About-to-Hit prelude (vision priming + cue)](#6-about-to-hit-prelude-vision-priming--cue) | live coaching P1 | ✅ |
| 7 | [Hit detection → auto-record → count shot (completes v1)](#7-hit-detection--auto-record--count-shot-completes-v1) | **P0 #10/#11** | ✅ |
| 8 | [⭐ Phone-triggered N-sec glass capture → retrieve](#8--phone-triggered-n-sec-glass-capture--retrieve) | **critical** | 🅡 |
| 9 | [False-positive rejection paths](#9-false-positive-rejection-paths) | P0 robustness | ✅ |
| 10 | [Club detection](#10-club-detection) | P1 | partial |
| 11 | [Occlusion detection / Erase Hat](#11-occlusion-detection--erase-hat) | **P0** | 🅡 |
| 12 | [Hole / pin localization](#12-hole--pin-localization) | P1 | 🅡 |
| 13 | [Watch sensor-fusion confirm](#13-watch-sensor-fusion-confirm) | P1 | 🅡 |
| 14 | [User feedback & manual missed shot](#14-user-feedback--manual-missed-shot) | scoring | ✅ |
| 15 | [Glasses camera fault & fallback](#15-glasses-camera-fault--fallback) | robustness | ✅ |
| 16 | [End of round → post-round cloud analysis](#16-end-of-round--post-round-cloud-analysis) | P1/P2 | 🅡 |

---

## 1. Pairing & capability negotiation

Per Penke: *"Mobile device queries the capabilities of Golf-supported features from wearables."*

```mermaid
sequenceDiagram
    autonumber
    actor U as Golfer
    participant App as Host App / SDK
    participant SE as Session Engine
    participant G as Jinju Glasses
    participant W as Watch (optional)

    U->>App: open app (glasses paired)
    App->>SE: queryDevices()
    SE->>G: queryCapabilities()
    G-->>SE: {camera:1(mono), imu:~200Hz, mic:1, speaker:1,<br/>asp_microModels:[human,motion,transient] 🅡}
    alt watch paired
        SE->>W: queryCapabilities()
        W-->>SE: {imu:1, swingModel:true 🅡}
    else no watch
        SE-->>SE: degrade graph (drop swing-confirm)
    end
    SE->>SE: build fusion graph + power plan from caps
    SE-->>App: capabilities{activeProducers, plan}
    App-->>U: show available Golf features
```

---

## 2. Golf-Mode entry — Manual (PUI/gesture)

The path that **works today** in the POC (Home → "Start Golf Mode").

```mermaid
sequenceDiagram
    autonumber
    actor U as Golfer
    participant UI as HomeScreen / HomeViewModel
    participant Perm as Permissions
    participant SE as GolfModeForegroundService
    participant SS as GolfModeSessionState

    U->>UI: tap "Start Golf Mode"
    UI->>Perm: ensure(MIC, CAMERA, LOCATION)
    alt mic granted
        UI->>SE: startForegroundService(START)
        SE->>SE: onStartCommand() → check mic permission
        SE->>SE: startGolfMode() (serviceScope)
        SE->>SS: update(isActive=true, session=…)
        SS-->>UI: LiveSessionState flows → navigate LiveGolfMode
    else mic denied
        SE->>SE: stopSelf()
        UI-->>U: explain mic required
    end
```

---

## 3. Golf-Mode entry — Voice / hotword

```mermaid
sequenceDiagram
    autonumber
    actor U as Golfer
    participant GMIC as Glass MIC + ASP hotword gate 🅡
    participant SE as Session Engine
    participant ASR as Intent/Keyword (phone) 🅡
    participant SPK as Glass Speaker (TTS)

    U->>GMIC: "Hey <hotword>… start golf"
    GMIC->>GMIC: ASP hotword gate fires (low-power)
    GMIC-->>SE: event{hotword, conf, ts} (BLE)
    SE->>SE: wake phone pipeline (Tier 0→1)
    SE->>ASR: resolve intent from short audio window
    ASR-->>SE: intent = START_GOLF (conf)
    alt conf high
        SE->>SE: requestGolfMode(VOICE)
        SE->>SPK: speak "Golf mode on"
    else low conf
        SE->>SPK: speak "Sorry, didn't catch that"
    end
```

---

## 4. Golf-Mode entry — Auto (Visual + IMU)

Scene + sensor fusion. Vision exists today; **auto-entry** is roadmap. Maps to SCENARIO 1 of the
Cline-generated deck.

```mermaid
sequenceDiagram
    autonumber
    participant ASP as Glass ASP (human gate) 🅡
    participant PI as Phone IMU (MotionStateDetector)
    participant SE as Session Engine
    participant FH as FrameHolder
    participant VA as GolfVisionAnalyzer (Gemma4)
    participant SS as SessionState

    ASP-->>SE: event{human_present, conf} (BLE) 🅡
    PI-->>SE: motionState = NEAR_IDLE (gate hint)
    SE->>SE: candidate golf context → sample vision
    SE->>FH: copyLatest()
    SE->>VA: analyze(frame) "golf scene?"
    VA-->>SE: GolfStanceResult{clubVisible, ballVisible, summary}
    alt scene looks like golf (club/ball + near-idle)
        SE->>SE: requestGolfMode(VISUAL_IMU)
        SE->>SS: isActive=true
    else not golf
        SE->>SE: stay ambient (Tier 0)
    end
```

---

## 5. Service startup — pipeline bring-up

Code-accurate `startGolfMode()` fan-out (see [`03 §4`](03_State_Machines.md)).

```mermaid
sequenceDiagram
    autonumber
    participant SE as GolfModeForegroundService
    participant SET as SettingsRepository
    participant SR as SessionRepository
    participant MSM as MotionSensorManager
    participant MSD as MotionStateDetector
    participant AR as AudioRecorder
    participant AC as AudioClassifier (TFLite/Fake)
    participant VA as GolfVisionAnalyzer
    participant SS as SessionState

    SE->>SET: getSettings() → DetectionSettings
    SE->>SR: startSession() → GolfSession(UUID)
    SE->>MSM: init (accel+gyro+step)
    SE->>MSD: start(1s eval) → onUpdate(motionState)
    SE->>AR: start(onWindow) 16kHz·1s·250ms hop
    SE->>AC: create (golf_audio_classifier.tflite or Fake)
    opt settings.visionEnabled
        SE->>VA: start() (interval=visionIntervalMs)
    end
    SE->>SE: launch elapsed-timer (500ms) + observe settings
    SE->>SS: update(isActive=true)
    Note over SE,SS: Running — sensor events now flow into fusion (§6,§7)
```

---

## 6. About-to-Hit prelude (vision priming + cue)

The **live-coaching** path: Gemma4 detects the golfer is set up and speaks an early cue. This *primes*
the engine (Tier 0→1) but is **not** the hard hit trigger.

```mermaid
sequenceDiagram
    autonumber
    participant CAM as Camera (glass via Projected)
    participant FP as FramePreprocessor (320²)
    participant FH as FrameHolder
    participant VA as GolfVisionAnalyzer
    participant LC as GolfVisionLlmClient (Gemma4 CPU)
    participant SH as SpeakHelper (tiers)
    participant SPK as TtsManager → glass speaker
    participant SS as SessionState

    loop every 3s (clamp 2–30s)
        CAM->>FP: frame
        FP->>FH: onFrame(320² bitmap)
        VA->>FH: copyLatest()
        VA->>LC: generate(bitmap, stancePrompt)
        LC-->>VA: JSON {club_visible, ball_visible, about_to_swing, summary}
        VA->>SS: stanceReady/clubVisible/ballVisible/aboutToSwing
        VA->>SH: resolveVisionSpeakTier()
        alt tier rises or 10s cooldown elapsed
            SH->>SPK: speak("Golf club seen."/"Golf ball seen."/"Prepared to hit. {summary}")
        end
        opt headPoseGate && motion ∉ {IDLE,NEAR_IDLE}
            VA->>VA: discard (head not steady)
        end
        opt preparedToHit
            VA->>SS: snapshot stance JPEG (12s cooldown)
        end
    end
```

---

## 7. Hit detection → auto-record → count shot (completes v1)

**This is the v1 diagram, completed and made code-accurate.** It is the P0 #10/#11 loop:
*sense → detect hit → record clip → count shot → ready for next.*

```mermaid
sequenceDiagram
    autonumber
    actor U as Golfer
    participant MIC as MIC (phone/glass)
    participant AR as AudioRecorder
    participant FE as AudioFeatureExtractor (log-mel)
    participant AC as TfliteAudioClassifier (8-class)
    participant PI as MotionStateDetector
    participant SDE as ShotDetectionEngine
    participant VA as GolfVisionAnalyzer
    participant VR as GolfAddressVideoRecorder
    participant SR as Repositories (Room)
    participant SS as SessionState
    participant UI as LiveGolfMode UI

    U->>MIC: swing → ball impact "crack"
    MIC->>AR: PCM (16kHz)
    AR->>FE: 1s window (250ms hop)
    FE->>AC: log-mel (FFT1024/64-mel)
    AC-->>SDE: P(golf_hit) [+7 other classes]
    PI-->>SDE: motionState (gate)
    SDE->>SDE: evaluate(conf, motion, cfg)
    alt conf≥threshold ∧ motion∈{IDLE,NEAR_IDLE} ∧ cooldown ok
        SDE-->>SS: Logged(PROBABLE_SHOT, conf) → shotCount++
        SDE->>VA: requestImmediateRun() (off-cadence vision)
        opt recordAddressVideo
            SDE->>VR: requestClip() (5s, SD)
            VR->>VR: record 5s → finalize
            alt file ≥ 1KB
                VR-->>SR: save clip path → ShotEvent.stanceImagePath/clip
            else < 1KB
                VR->>VR: discard
            end
        end
        SR-->>SS: persist ShotEvent + session counts
        SS-->>UI: shotCount, event row, clip thumbnail
        UI-->>U: "Stroke logged" (and TTS earcon)
    else gated
        SDE-->>SR: IGNORED_AUDIO_HIT(reason: low-confidence/motion/cooldown)
        SR-->>SS: ignored event (audit)
    end
    Note over U,UI: cooldown (5s) → loop ready for next stroke
```

> **What v1 was missing and this adds:** the audio feature path (log-mel), the **8-class** classifier
> (not just "hit/no-hit"), the explicit **motion gate** and **cooldown**, the **ignored-event audit
> trail**, the **vision off-cadence kick**, the clip **finalize/discard** branch, and the persistence
> + UI propagation.

---

## 8. ⭐ Phone-triggered N-sec glass capture → retrieve

The flow Penke flagged: *"event from mobile → triggers a video capture on Glasses (N secs) → and
retrieve it onto the Phone. This is very important and we need to nail asap."* This is the **D3
on-demand streaming contract** (see [`03 §11`](03_State_Machines.md)) in action.

```mermaid
sequenceDiagram
    autonumber
    participant SE as Session Engine (phone)
    participant CAP as Capture API (SDK)
    participant GAP as Glass AP (woken)
    participant ASP as Glass ASP (pre-buffer)
    participant ENC as Glass HW Encoder (H.265)
    participant XfR as Bulk transfer (WiFi-Direct)
    participant DB as Clip store (Room + file)
    participant UI as Phone UI

    Note over SE: trigger = hit confirmed (or app triggerClip(N))
    SE->>CAP: triggerClip(durationSec=N, reason=HIT)
    CAP->>GAP: wake + open camera stream for N s (camera modality only)
    GAP->>ASP: commit circular pre-buffer (backfill pre-swing) 🅡
    ASP-->>ENC: pre-buffer frames + live post-roll (6–8s)
    ENC->>ENC: encode H.265 ~1080p/30fps (HW)
    Note over GAP,XfR: link kept minimal — only camera, only for N s
    GAP-->>SE: clip ready {id, size, ts}
    alt golfer idle / walking to next shot (deferred)
        SE->>XfR: pull clip (resumable bulk)
        XfR-->>DB: store clip + path
        DB-->>UI: clip available in session
    else immediate small clip
        GAP-->>SE: stream clip inline
        SE->>DB: store
    end
    SE->>GAP: close stream → glass link goes Dark again (Tier 2→0)
```

**Design contract:** open the link for **only the camera modality**, for **only N seconds**, to
**one consumer**; backfill from the ASP **pre-buffer** so the *pre-swing* is captured; **defer** the
bulk transfer to when the golfer walks ("not latency-critical"); then return the glass to the dark,
ASP-local ambient state.

---

## 9. False-positive rejection paths

P0 robustness — the course is noisy (carts, wind, footsteps, chatter). Shows all three rejection
branches and the audit trail.

```mermaid
sequenceDiagram
    autonumber
    participant AC as TfliteAudioClassifier
    participant PI as MotionStateDetector
    participant SDE as ShotDetectionEngine
    participant SR as Repositories

    rect rgb(255,245,245)
    Note over AC,SDE: A) Loud-but-not-a-hit (cart/wind/footsteps)
    AC-->>SDE: P(golf_hit)=0.74 (NORMAL threshold 0.80)
    SDE->>SDE: 0.68 ≤ 0.74 < 0.80 (0.85·thr=0.68)
    SDE-->>SR: Ignored("low confidence") → IGNORED_AUDIO_HIT
    end

    rect rgb(245,248,255)
    Note over AC,SDE: B) Hit-like sound WHILE walking
    AC-->>SDE: P(golf_hit)=0.88
    PI-->>SDE: motionState = WALKING
    SDE-->>SR: Ignored("WALKING") → IGNORED_AUDIO_HIT
    end

    rect rgb(245,255,248)
    Note over AC,SDE: C) Double-trigger within cooldown
    AC-->>SDE: P(golf_hit)=0.92 (2.1s after last shot)
    SDE-->>SR: Ignored("cooldown active") → IGNORED_AUDIO_HIT
    end
```

> The ignored events are **persisted with reasons** so the team can tune `sensitivity`,
> `motionGateEnabled`, and `cooldownMillis` from real-course data — and so a future fusion net has
> labeled negatives.

---

## 10. Club detection

```mermaid
sequenceDiagram
    autonumber
    participant VA as GolfVisionAnalyzer (Gemma4 ▶ club net 🅡)
    participant IMU as IMU swing-arc 🅡
    participant BLE as BLE club tag (optional) 🅡
    participant CLS as Club classifier
    participant SS as SessionState

    Note over VA: at address (preparedToHit)
    VA-->>CLS: silhouette / club_visible cue
    IMU-->>CLS: swing-arc tempo signature 🅡
    BLE-->>CLS: grip RFID/BLE id (if present) 🅡
    CLS->>CLS: fuse → P(club) (single-camera limited)
    CLS-->>SS: clubId + confidence
    Note over CLS: per-user calibration over first few rounds 🅡
    SS->>SS: gate the per-club swing model
```

---

## 11. Occlusion detection / Erase Hat (P0)

Built by **SRIB**. Two outcomes: gate unreliable frames, or inpaint the hat.

```mermaid
sequenceDiagram
    autonumber
    participant CAM as Glass camera
    participant FH as FrameHolder
    participant SEG as Occlusion/Seg net (SRIB) 🅡
    participant DS as Downstream analysis
    participant UI as Phone UI / gallery

    CAM->>FH: frame
    FH->>SEG: frame
    SEG->>SEG: segment hat/hands/body/grass/sun over ball·club
    alt occludes ball/club
        SEG-->>DS: flag frame UNRELIABLE → skip/interpolate trajectory
    else hat enters frame (Erase-Hat CUJ)
        SEG->>SEG: inpaint occluder
        SEG-->>UI: cleaned frame / clip
    else clear
        SEG-->>DS: frame usable
    end
```

---

## 12. Hole / pin localization

The one case where the **glass IMU** is genuinely needed (Khani): head orientation + GPS.

```mermaid
sequenceDiagram
    autonumber
    participant GI as Glass IMU (head orientation) 🅡
    participant GPS as Phone GPS
    participant SE as Session Engine
    participant VA as Vision (pin detect) 🅡
    participant SS as SessionState

    SE->>GPS: fine location (Tier 2)
    GPS-->>SE: lat/lon → hole # (course map)
    SE->>GI: request head-orientation stream (on-demand) 🅡
    GI-->>SE: yaw/pitch (where golfer looks)
    opt vision available
        SE->>VA: detect pin/flag in frame 🅡
        VA-->>SE: pin bearing
    end
    SE->>SE: fuse orientation + GPS (+pin) → target/yardage
    SE-->>SS: hole #, distance-to-pin context
    Note over SE,SS: binds subsequent shot events to this hole → scorecard 🅡
```

---

## 13. Watch sensor-fusion confirm

The UX team's Watch7 PoC streams IMU/audio over UDP; here it acts as an extra **hit confirm**.

```mermaid
sequenceDiagram
    autonumber
    participant W as Watch (swing model) 🅡
    participant SE as Session Engine (fusion bus)
    participant AC as Audio hit conf
    participant SDE as ShotDetectionEngine

    par parallel producers
        W-->>SE: swing-segment {backswing→impact, conf} (BLE/UDP) 🅡
    and
        AC-->>SE: P(golf_hit)
    end
    SE->>SDE: evaluate(audio, motion) [+watch confirm 🅡]
    alt audio ∧ motion-gate ∧ watch-swing agree
        SDE-->>SE: high-precision PROBABLE_SHOT
    else only audio
        SDE-->>SE: PROBABLE_SHOT (audio-primary today)
    else only watch (no audio)
        SDE-->>SE: candidate → wait for audio window
    end
    Note over SE: no watch ⇒ graceful degrade (audio stays primary)
```

---

## 14. User feedback & manual missed shot

Human-in-the-loop scoring correction (`LiveGolfModeViewModel`).

```mermaid
sequenceDiagram
    autonumber
    actor U as Golfer
    participant UI as LiveGolfMode UI
    participant VM as LiveGolfModeViewModel
    participant SR as Repositories
    participant SS as SessionState

    alt confirm a detected shot
        U->>UI: tap ✓ on event
        UI->>VM: submitFeedback(event, CORRECT)
        VM->>SR: update event + session.confirmedShots++
    else reject false positive
        U->>UI: tap ✗ on event
        UI->>VM: submitFeedback(event, NOT_A_SHOT)
        VM->>SR: update event + session.rejectedShots++
    else add a missed shot
        U->>UI: tap "+ missed shot"
        UI->>VM: addMissedShot()
        VM->>SR: insert MANUAL_MISSED_SHOT + session.missedShots++
    end
    SR-->>SS: updated counts
    SS-->>UI: refreshed scorecard
```

---

## 15. Glasses camera fault & fallback

The HaeAn/GG firmware reality (D10) — graceful fallback to the phone camera.

```mermaid
sequenceDiagram
    autonumber
    participant SE as Session Engine
    participant GC as GlassesContextHelper
    participant BS as GolfCameraBindState
    participant PV as GolfCameraPreview (CameraX)
    participant UI as UI

    SE->>GC: getProjectedCameraContext() (API≥34)
    alt projected context present
        GC-->>PV: ProjectedContext + camera permission
        PV->>PV: bindToLifecycle (glasses)
        alt frames stream
            PV->>BS: setVideoSourceReady(true) → GLASSES_CONNECTED
        else black/no frames (bad build!)
            PV->>BS: still not ready → 🅐 BlackPreview
            BS-->>UI: hint "check glasses firmware build"
        end
        Note over PV,BS: previewStreamState STREAMING→IDLE
        PV->>BS: onGlassesDisconnected() → GLASSES_DISCONNECTED
        BS->>UI: show reconnect dialog
        BS->>PV: rebind (bump nonce) OR fallBackToPhoneCamera()
    else context null / permission denied
        GC-->>BS: GLASSES_UNAVAILABLE / PermissionDenied
        BS->>PV: fallBackToPhoneCamera() → PHONE
    end
    Note over SE,UI: audio+motion hit detection unaffected by camera state
```

---

## 16. End of round → post-round cloud analysis

On-device first (D8); cloud is opt-in (slide 22). Closes Penke's "full gaming experience" loop.

```mermaid
sequenceDiagram
    autonumber
    actor U as Golfer
    participant UI as UI
    participant SE as Session Engine
    participant SR as Repositories (Room)
    participant CL as Cloud AI (opt-in) 🅡
    participant CO as Coach/Peer

    U->>UI: end round
    UI->>SE: exitGolfMode()
    SE->>SR: endSessionDirect() (endTime, duration, counts)
    SR-->>UI: session summary (shots, ignored, clips)
    alt cloud opt-in + network
        UI->>CL: upload session (shots, clips, features)
        CL->>CL: Strokes-Gained, swing pattern, coaching notes 🅡
        CL-->>UI: coaching notes + SG analytics
        UI-->>U: post-round insights
        opt share
            U->>CO: shareable summary
        end
    else offline / declined
        UI-->>U: on-device summary only (queue upload) 🅡
    end
```

---

## Appendix · Cross-cutting legend

| Symbol | Meaning |
|--------|---------|
| solid arrow `->>` | synchronous call / event |
| dashed arrow `-->>` | return / async event |
| `par` / `alt` / `opt` | parallel / branch / optional Mermaid blocks |
| 🅡 | roadmap (not in POC yet) |
| 🅐 | assumption / operational state (not a code enum) |
| ⭐ | the critical "phone-triggers-glass-capture" flow |

> **Traceability:** every participant maps to a class in `com.golfcues.app.*` (see
> [`06_Glossary_and_Traceability.md`](06_Glossary_and_Traceability.md)); every 🅡 maps to a Slack
> decision or Tech-Overview statement, not to shipped code.
