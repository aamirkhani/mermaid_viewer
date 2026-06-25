# 03 · State & Transition Machines

> Every FSM here is grounded in the `GolfCues` POC source (`com.golfcues.app.*`). Enum names,
> thresholds, timers, and transition conditions are **verbatim from code** unless tagged 🅡 (roadmap)
> or 🅐 (assumption). Diagrams are Mermaid `stateDiagram-v2`.

**Contents**
1. [Golf-Mode lifecycle (top level)](#1-golf-mode-lifecycle-top-level)
2. [Ambient power-tier FSM](#2-ambient-power-tier-fsm)
3. [Mode-entry FSM (4 entry paths)](#3-mode-entry-fsm-4-entry-paths)
4. [Foreground-service lifecycle](#4-foreground-service-lifecycle)
5. [Shot-detection FSM (code-accurate)](#5-shot-detection-fsm-code-accurate)
6. [Motion-state FSM](#6-motion-state-fsm)
7. [Audio-status FSM](#7-audio-status-fsm)
8. [Vision-status FSM](#8-vision-status-fsm)
9. [Recording / pre-buffer FSM](#9-recording--pre-buffer-fsm)
10. [Glasses camera connection & firmware FSM](#10-glasses-camera-connection--firmware-fsm)
11. [On-demand glass-streaming FSM](#11-on-demand-glass-streaming-fsm)
12. [Session & shot-feedback lifecycle](#12-session--shot-feedback-lifecycle)

---

## 1. Golf-Mode lifecycle (top level)

The mode-first principle (Penke): *"for any feature, the very first step is to get to the right
mode."* Everything else is a sub-state of `GolfModeActive`.

```mermaid
stateDiagram-v2
    [*] --> Off
    Off --> Arming: entry trigger (manual / voice / visual+IMU / audio)
    Arming --> GolfModeActive: capabilities negotiated + pipelines started
    Arming --> Off: permission denied / no mic

    state GolfModeActive {
        [*] --> Sensing
        Sensing --> Primed: about-to-hit suspected (vision stance / head-pose)
        Primed --> Sensing: timeout / false alarm
        Primed --> Capturing: hit confirmed (audio ∧ motion-gate)
        Sensing --> Capturing: hit confirmed directly (audio-only path)
        Capturing --> Sensing: clip done + cooldown
    }

    GolfModeActive --> Off: user stop / exitGolfMode() / onDestroy
    GolfModeActive --> Suspended: phone backgrounded / battery critical
    Suspended --> GolfModeActive: resumed
    Suspended --> Off: killed
```

| From | Event | To | Source |
|------|-------|----|--------|
| Off | entry trigger | Arming | mode-entry FSM §3 |
| Arming | caps OK + pipelines up | GolfModeActive | `startGolfMode()` |
| Arming | mic permission missing | Off | `onStartCommand()` stops service |
| Sensing | stance/head-pose says "about to hit" | Primed | vision pipeline |
| Primed | hit confirmed | Capturing | `ShotDetectionEngine.Logged` |
| Capturing | clip finalized + cooldown | Sensing | recorder finalize §9 |
| GolfModeActive | user stop | Off | `GolfModeForegroundService.stop()` |

---

## 2. Ambient power-tier FSM

The power model from [`01 §7`](01_System_Architecture.md#7-power--data-tiering-model). Default is
Tier 0; the system briefly escalates and falls back.

```mermaid
stateDiagram-v2
    [*] --> Tier0_Ambient

    Tier0_Ambient: TIER 0 — Ambient (default)
    Tier0_Ambient: ASP always-on · micro-gates only
    Tier0_Ambient: NO raw streaming · phone AP may doze

    Tier1_Primed: TIER 1 — Primed
    Tier1_Primed: phone wakes pipelines
    Tier1_Primed: vision interval ↑ · still no glass raw stream

    Tier2_Capture: TIER 2 — Active capture
    Tier2_Capture: on-demand glass camera stream
    Tier2_Capture: commit pre-buffer + post-roll

    Tier0_Ambient --> Tier1_Primed: ASP event {human ∧ near-idle ∧ pre-impact sound}
    Tier1_Primed --> Tier2_Capture: hit confirmed
    Tier1_Primed --> Tier0_Ambient: timeout / false alarm
    Tier2_Capture --> Tier0_Ambient: clip synced + cooldown
    Tier2_Capture --> LowPower: battery critical
    LowPower --> Tier0_Ambient: recovered
    LowPower: degraded — capture suspended, interval maxed (30s)
```

> 🅡 The Tier-0 ASP gating is the **target**. In today's POC the phone always runs the audio + motion
> pipelines while in Golf Mode; "Tier 0" maps to *idle listening* with vision at the 3 s interval.

---

## 3. Mode-entry FSM (4 entry paths)

Penke's four entry options. All converge on `GolfModeActive`.

```mermaid
stateDiagram-v2
    [*] --> Idle

    Idle --> Manual: PUI tap / gesture
    Idle --> Voice: hotword detected → command
    Idle --> VisualIMU: scene looks like golf + near-idle
    Idle --> AudioCue: keyword / characteristic sounds

    Manual --> ConfirmEntry
    Voice --> ConfirmEntry: intent = "start golf"
    VisualIMU --> ConfirmEntry: Gemma scene/stance + motion gate
    AudioCue --> ConfirmEntry: acoustic keyword/sound gate

    ConfirmEntry --> GolfModeActive: requestGolfMode(entry)
    ConfirmEntry --> Idle: rejected / low confidence

    state GolfModeActive {
        [*] --> running
    }
    GolfModeActive --> Idle: exit
```

| Entry path | Trigger source | Confidence gate | Today |
|-----------|----------------|-----------------|-------|
| **Manual** | PUI / gesture | n/a (explicit) | ✅ (Home → Start Golf Mode) |
| **Voice** | hotword → ASR intent | intent conf | 🅡 |
| **Visual + IMU** | Gemma scene/stance + `MotionState` gate | stance + near-idle | partial (vision exists; auto-entry 🅡) |
| **Audio** | keyword / sound classifier | acoustic gate | 🅡 |

---

## 4. Foreground-service lifecycle

`GolfModeForegroundService` — the orchestrator. Code-accurate from `onCreate`/`onStartCommand`/
`startGolfMode`/`onDestroy`.

```mermaid
stateDiagram-v2
    [*] --> Created: onCreate()\n(NotificationManager, ShotDetectionEngine, stop receiver)
    Created --> CheckPermission: onStartCommand()
    CheckPermission --> Stopped: mic permission missing → stopSelf
    CheckPermission --> Starting: startGolfMode() in serviceScope

    state Starting {
        [*] --> s1: load settings
        s1 --> s2: sessionRepository.startSession()
        s2 --> s3: MotionSensorManager + MotionStateDetector (1s eval)
        s3 --> s4: AudioRecorder + AudioClassifier (TFLite or Fake)
        s4 --> s5: if visionEnabled → GolfVisionAnalyzer
        s5 --> s6: elapsed-time timer (500ms) + observe settings
    }

    Starting --> Running: all pipelines live
    Running --> Running: sensor events → fusion → SessionState.update()
    Running --> Reconfig: settings changed (observe)
    Reconfig --> Running: apply new DetectionSettings
    Running --> Destroying: stop broadcast / system kill
    Destroying --> [*]: onDestroy()\nendSessionDirect(), tear down pipelines, reset state
```

**Service facts:** `serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)`; ongoing
notification via `GolfModeNotificationManager`; elapsed-time tick every **500 ms**; event buffer in
`SessionState` capped at **50** events.

---

## 5. Shot-detection FSM (code-accurate)

The heart of the P0 "count shots" CUJ — `ShotDetectionEngine.evaluate(audioHitConfidence,
motionState, config)`. The engine itself is nearly stateless: its only state is
`lastLoggedShotTimestamp`. The FSM below makes the decision flow explicit.

```mermaid
stateDiagram-v2
    [*] --> Listening

    Listening --> Eval: new audio window classified (P(golf_hit))

    state Eval {
        [*] --> chkConf
        chkConf --> belowFloor: conf < threshold*0.85 → NoAction
        chkConf --> lowConf: threshold*0.85 ≤ conf < threshold → Ignored("low confidence")
        chkConf --> chkMotion: conf ≥ threshold
        chkMotion --> motionReject: motion ∉ {IDLE,NEAR_IDLE} → Ignored("motion")
        chkMotion --> chkCooldown: motion ∈ {IDLE,NEAR_IDLE}
        chkCooldown --> cooldownReject: now-lastShot < cooldownMillis → Ignored("cooldown")
        chkCooldown --> pass: now-lastShot ≥ cooldownMillis
    }

    belowFloor --> Listening
    lowConf --> Listening: log IGNORED_AUDIO_HIT (reason)
    motionReject --> Listening: log IGNORED_AUDIO_HIT (reason)
    cooldownReject --> Listening: log IGNORED_AUDIO_HIT (reason)
    pass --> ShotLogged: Logged(PROBABLE_SHOT, conf, motion)

    ShotLogged --> Cooldown: set lastLoggedShotTimestamp = now
    Cooldown --> Listening: after cooldownMillis (default 5000)
    ShotLogged --> VisionKick: requestImmediateRun() (off-cadence vision)
    VisionKick --> Listening
```

**Thresholds (verbatim)**

| Gate | Rule | Constant |
|------|------|----------|
| Audio threshold | `P(golf_hit) ≥ sensitivity.threshold()` | LOW `0.90` · NORMAL `0.80` (default) · HIGH `0.70` |
| Low-confidence band | `threshold*0.85 ≤ conf < threshold` ⇒ `Ignored("low confidence")` | `0.85` factor |
| Below floor | `conf < threshold*0.85` ⇒ `NoAction` (not even logged) | — |
| Motion gate | must be `IDLE` or `NEAR_IDLE` (if `motionGateEnabled`, default true) | — |
| Cooldown | `now - lastLoggedShotTimestamp ≥ cooldownMillis` | default `5000 ms` |
| Success | emit `Logged(EventType.PROBABLE_SHOT, conf, motionState)` | — |

**Result types:** `NoAction` · `Ignored(reason)` (persists `IGNORED_AUDIO_HIT` with the reason
string) · `Logged(...)` (persists `PROBABLE_SHOT`, increments shot count, kicks vision).

---

## 6. Motion-state FSM

`MotionSensorManager.classify(accelVar, gyroVar, recentSteps, msSinceLastStep)` — phone IMU
(accelerometer + gyroscope + step detector), evaluated every **1 s** by `MotionStateDetector`. This
is the **gate** that suppresses false hits while the golfer walks.

```mermaid
stateDiagram-v2
    [*] --> UNKNOWN
    note left of UNKNOWN: evaluated every 1000ms from 64-sample ring buffers

    UNKNOWN --> IDLE: accelVar<0.05 ∧ gyroVar<0.02 ∧ steps==0
    UNKNOWN --> NEAR_IDLE: accelVar<0.15 ∧ gyroVar<0.08 ∧ steps==0\n(or accelVar<0.3 ∧ gyroVar<0.15)
    UNKNOWN --> WALKING: accelVar>1.5 ∨ steps>0 ∨ msSinceStep<2000
    UNKNOWN --> RUNNING: accelVar>4 ∧ gyroVar>1.5 ∧ steps>2
    UNKNOWN --> VEHICLE: accelVar>0.8 ∧ gyroVar>0.5

    IDLE --> NEAR_IDLE: slight movement
    NEAR_IDLE --> IDLE: settles
    NEAR_IDLE --> WALKING: steps / accel rise
    IDLE --> WALKING: steps / accel rise
    WALKING --> RUNNING: high accel+gyro+steps
    WALKING --> NEAR_IDLE: stops (steps stale >2s)
    RUNNING --> WALKING: slows
    WALKING --> VEHICLE: smooth sustained motion
    VEHICLE --> NEAR_IDLE: stops

    state "SHOT-GATE OPEN" as GateOpen
    IDLE --> GateOpen
    NEAR_IDLE --> GateOpen
    GateOpen: only IDLE/NEAR_IDLE let a hit through (§5)
```

**Classification thresholds (verbatim, evaluation order = RUNNING → WALKING → VEHICLE → IDLE →
NEAR_IDLE → UNKNOWN)**

| State | Condition |
|-------|-----------|
| `RUNNING` | `accelVar > 4 ∧ gyroVar > 1.5 ∧ recentSteps > 2` |
| `WALKING` | `accelVar > 1.5 ∨ recentSteps > 0 ∨ msSinceLastStep < 2000` |
| `VEHICLE` | `accelVar > 0.8 ∧ gyroVar > 0.5` |
| `IDLE` | `accelVar < 0.05 ∧ gyroVar < 0.02 ∧ recentSteps == 0` |
| `NEAR_IDLE` | `accelVar < 0.15 ∧ gyroVar < 0.08 ∧ recentSteps == 0` **or** `accelVar < 0.3 ∧ gyroVar < 0.15` |
| `UNKNOWN` | otherwise |

Windows: steps "recent" if `< 3000 ms` old; step counter reset if `> 60000 ms` since last step.
Ring-buffer capacity 64 samples per axis.

---

## 7. Audio-status FSM

`AudioStatus` drives the live UI badge. Derived from classifier confidence vs. the active threshold.

```mermaid
stateDiagram-v2
    [*] --> LISTENING
    LISTENING --> POSSIBLE_HIT_DETECTED: P(golf_hit) ≥ threshold
    LISTENING --> NOISE_IGNORED: threshold*0.7 ≤ P < threshold (audible but rejected)
    POSSIBLE_HIT_DETECTED --> LISTENING: window passes / logged
    NOISE_IGNORED --> LISTENING: next window
    LISTENING --> MUTED_OR_PERMISSION_MISSING: mic error / permission lost
    MUTED_OR_PERMISSION_MISSING --> LISTENING: mic restored
```

| State | Meaning |
|-------|---------|
| `LISTENING` | baseline, capturing 1 s windows @16 kHz (250 ms hop) |
| `POSSIBLE_HIT_DETECTED` | confidence cleared threshold (may still be gated by motion/cooldown) |
| `NOISE_IGNORED` | loud but sub-threshold |
| `MUTED_OR_PERMISSION_MISSING` | AudioRecord error / permission revoked |

---

## 8. Vision-status FSM

`VisionStatus` — lifecycle of the Gemma4 pipeline (`GolfVisionAnalyzer`).

```mermaid
stateDiagram-v2
    [*] --> DISABLED: settings.visionEnabled == false
    DISABLED --> MODEL_MISSING: enabled but model file absent
    DISABLED --> WAITING_FOR_FRAME: enabled + model present
    MODEL_MISSING --> WAITING_FOR_FRAME: model side-loaded (adb push)
    WAITING_FOR_FRAME --> INFERRING: fresh frame available
    INFERRING --> READY: inference parsed (GolfStanceResult)
    READY --> INFERRING: next interval (3s, clamp 2–30s) or requestImmediateRun()
    READY --> WAITING_FOR_FRAME: frame went stale
    INFERRING --> ERROR: exception / parse fail
    ERROR --> INFERRING: retry next interval
    note right of INFERRING: Mutex single-flight — one inference at a time
```

---

## 9. Recording / pre-buffer FSM

Two layers: **today's** on-phone CameraX clip recorder (`GolfAddressVideoRecorder`, 5 s), and the
**target** glass-side circular pre-buffer (commit-on-hit).

```mermaid
stateDiagram-v2
    state "TODAY — GolfAddressVideoRecorder (phone CameraX)" as Today {
        [*] --> Idle
        Idle --> Recording: requestClip() [attached ∧ source ready ∧ !active ∧ cooldown elapsed]
        Recording --> Finalizing: auto-stop after 5000ms
        Finalizing --> Saved: file ≥ 1024 bytes → MP4 (SD quality)
        Finalizing --> Discarded: file < 1024 bytes → delete
        Saved --> Idle: set lastClipStartedMs (12s cooldown)
        Discarded --> Idle
    }

    state "TARGET 🅡 — Glass circular pre-buffer" as Target {
        [*] --> Buffering
        Buffering --> Buffering: encode low-bitrate into 5–10s RAM ring
        Buffering --> Commit: phone says 'hit confirmed'
        Commit --> PostRoll: backfill pre-swing + record 6–8s post-impact
        PostRoll --> Encode: H.265 ~1080p/30fps (HW)
        Encode --> Sync: deferred transfer 'when golfer walks to next shot'
        Sync --> Buffering
    }
```

**Today's constants:** clip `5000 ms`, cooldown `12000 ms`, min size `1024 B`, quality `SD` with
fallback; precondition `isVideoSourceReady` (first frame received or preview streaming). Stance JPEG
snapshots use a separate `12000 ms` cooldown, quality 88, min 512 B.

---

## 10. Glasses camera connection & firmware FSM

Captures the **HaeAn/GG firmware fragility** (D10) and the Projected-API bind path
(`GolfCameraBindState`, `GlassesContextHelper`, `GolfCameraPreview`).

```mermaid
stateDiagram-v2
    [*] --> PHONE: default / glasses off

    PHONE --> Binding: useGlassesCamera ∧ API≥34 (UPSIDE_DOWN_CAKE)
    Binding --> GLASSES_CONNECTED: ProjectedContext + permission OK + frames stream
    Binding --> GLASSES_UNAVAILABLE: ProjectedContext null / no device
    Binding --> PermissionDenied: projected camera permission denied

    GLASSES_CONNECTED --> BlackPreview: streaming but black frames (bad HaeAn build! D10)
    BlackPreview --> GLASSES_CONNECTED: pin known-good build / reconnect
    GLASSES_CONNECTED --> GLASSES_DISCONNECTED: previewStreamState STREAMING→IDLE
    GLASSES_DISCONNECTED --> Binding: rebind (bump projectedCameraBindNonce)
    GLASSES_DISCONNECTED --> PHONE: fallBackToPhoneCamera()
    GLASSES_UNAVAILABLE --> PHONE: fallBackToPhoneCamera()
    PermissionDenied --> PHONE: fallBackToPhoneCamera()

    note right of BlackPreview
      Slack: "device connected but black" ⇒ firmware build.
      Known-good = ~1-month-old HaeAn build.
    end note
```

| State | Meaning |
|-------|---------|
| `PHONE` | using the phone camera (default / fallback) |
| `GLASSES_CONNECTED` | Projected device bound, frames streaming |
| `GLASSES_UNAVAILABLE` | no projected context / device not present |
| `GLASSES_DISCONNECTED` | was streaming, link dropped (STREAMING→IDLE) → dialog + rebind |
| *BlackPreview* (🅐 op-state) | "connected" but black frames ⇒ suspect firmware build |

`isVideoSourceReady` flips true only after the first frame (or preview streaming) — it gates clip
recording (§9).

---

## 11. On-demand glass-streaming FSM

The **D3 power contract** as a state machine: the glass radio link is dark by default and opens only
for a specific consumer, minimal modality, for a bounded time. This is the engine behind the critical
"phone triggers N-sec capture" flow ([`04 §7`](04_Sequence_Diagrams.md)).

```mermaid
stateDiagram-v2
    [*] --> Dark: no raw stream (ASP-local only)

    Dark --> Requested: consumer needs modality M for duration N
    Requested --> Negotiate: check capabilities + power budget
    Negotiate --> Streaming: open link for {M} only (camera/IMU/MIC)
    Negotiate --> Denied: budget/caps insufficient → degrade

    Streaming --> Streaming: deliver frames/samples to the one consumer
    Streaming --> Closing: N elapsed / consumer done / battery low
    Closing --> Dark: tear down link, return to ASP-local

    Denied --> Dark
    note right of Streaming
      Only the specific data the consuming model needs (Khani's rule).
      Prefer phone IMU/MIC; reserve glass link for camera + head-orientation.
    end note
```

---

## 12. Session & shot-feedback lifecycle

The data lifecycle of a round and the human-in-the-loop correction of detected shots
(`GolfSession`, `ShotEvent`, `FeedbackType`).

```mermaid
stateDiagram-v2
    [*] --> SessionActive: startSession() (UUID, startTime, zeros)

    state SessionActive {
        [*] --> Accumulating
        Accumulating --> Accumulating: PROBABLE_SHOT → totalProbableShots++
        Accumulating --> Accumulating: IGNORED_AUDIO_HIT logged (reason)
        Accumulating --> Accumulating: PREPARED_TO_HIT (stance JPEG)
    }

    SessionActive --> SessionEnded: endSessionDirect()\n(set endTime, durationMillis)
    SessionEnded --> [*]

    state "Per-shot feedback (user)" as FB {
        [*] --> UNMARKED
        UNMARKED --> CORRECT: user confirms → confirmedShots++
        UNMARKED --> NOT_A_SHOT: user rejects → rejectedShots++
        [*] --> MISSED_SHOT: addMissedShot() → MANUAL_MISSED_SHOT, missedShots++
    }
```

**Event types:** `PROBABLE_SHOT` · `POSSIBLE_SHOT` (reserved) · `IGNORED_AUDIO_HIT` ·
`MANUAL_MISSED_SHOT` · `PREPARED_TO_HIT`.
**Feedback types:** `UNMARKED` · `CORRECT` · `NOT_A_SHOT` · `MISSED_SHOT`.
Session counters: `totalProbableShots`, `confirmedShots`, `rejectedShots`, `missedShots`.

> 🅡 **Score** (vs. raw shot count) and **per-hole** segmentation are the natural extensions: bind
> shot events to a hole via GPS (`04 §11`) to turn the shot log into a scorecard.
