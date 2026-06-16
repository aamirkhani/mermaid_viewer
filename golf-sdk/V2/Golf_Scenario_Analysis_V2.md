# Golf SDK - AR Glass Scenario Analysis

## Executive Summary

This document analyzes the data flow, sequence diagrams, and ML model triggering for the **Jinju AR Glass** golf application. The system uses **Gemma4** (initially) running on the phone, with plans to replace it with a custom ML model for improved battery life and latency.

---

## System Architecture Overview

```mermaid
block-beta
columns 2
  block:AR_Glass["AR Glass (Jinju)"]
    columns 1
    Camera["Camera"]
    IMU["IMU Sensor"]
    Mic["Microphone"]
    Spkr["Speaker"]
    MicroML["Micro ML (lightweight classification)"]
    MCU["MCU"]
  end
  block:Phone["Phone (S26 Ultra)"]
    columns 1
    Framework["Framework"]
    AmbientMem["Ambient Memory"]
    HeavyML["Heavy ML (Gemma4 → Custom)"]
    Gallery["Gallery Storage"]
    TTS["TTS Engine"]
  end
  AR_Glass <--> Phone
```

---

## Mode Entry Triggers

The system supports **4 mode entry options** before any feature activation:

| # | Method | Trigger | Priority |
|---|--------|---------|----------|
| 1 | **Manual** | PUI (Physical User Interface) or Gesture Input | P1 |
| 2 | **Audio** | Voice Command (Hotword + Request) | P2 |
| 3 | **Auto** | Visual Intelligence + Sensor Fusion | P0 |
| 4 | **Auto** | Audio Intelligence (Keyword Detection + Sounds) | P2 |

---

## P0/P1 CUJs (Complete User Journeys)

Based on the Golf_SDK_CUJ_Tasks.xlsx:

| CUJ | Type | Priority | ML Feasibility Notes |
|-----|------|----------|---------------------|
| **Erase Hat** | On-Round | P0 | Occlusion Detection Model (SRIB) |
| **Start/End Video Recording + Count Shots/Score** | On-Round | P0 | Visual cues (P1) + IMU (P1) + Audio (P2) |
| **Heads-up Live Coaching** | On-Round | P1 | Human Posture Detection |
| **Live Approach Strategy** | On-Round | P1 | Scene Understanding |
| **Warm-up Tips** | Pre-Round | P1 | Activity Recognition |

---

# SCENARIO 1: Golf Mode Entry (Auto - Visual + IMU)

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Golfer
    participant Glasses as Glasses<br/>(Cam/IMU/Mic/Spkr + MicroML)
    participant MCU as MCU
    participant PhoneApp as Phone App<br/>(Framework + Ambient Memory)
    participant HeavyML as Phone Heavy-ML<br/>(Gemma4 → Custom)
    participant Storage as Data Storage

    Golfer->>Glasses: Wears glasses
    activate Glasses
    
    Glasses->>Glasses: Capture frames (continuous)<br/>@ 1fps, 320x320x3
    Glasses->>Glasses: Collect IMU data<br/>(gyro/accel)
    
    Glasses->>Glasses: Vision+IMU Fusion<br/>[MicroML on-device]
    
    Glasses->>Glasses: Detect golf scene<br/>(confidence check)
    
    Glasses->>MCU: Gate① TRIGGERED
    activate MCU
    
    MCU->>PhoneApp: Send event via API
    activate PhoneApp
    
    PhoneApp->>HeavyML: Request ML inference
    activate HeavyML
    
    HeavyML->>HeavyML: Classify scene<br/>(lightweight model)
    HeavyML-->>PhoneApp: Return result
    deactivate HeavyML
    
    PhoneApp->>PhoneApp: Confirm Golf Mode
    PhoneApp-->>MCU: Mode confirmed
    deactivate PhoneApp
    
    MCU-->>Glasses: Mode active
    deactivate MCU
    
    Glasses->>Golfer: Haptic/Audio feedback
    Glasses->>Storage: Save session ID
    Glasses->>Storage: Log mode entry event
    
    deactivate Glasses
```

## Data Analysis for Scenario 1

| Data Type | Generated | Shared | Saved | Retention |
|-----------|-----------|--------|-------|-----------|
| IMU Raw Data | ✓ (continuous) | → Phone API | ✗ | Volatile (buffer only) |
| Camera Frames | ✓ (1fps) | → Phone API | ✗ | Volatile (preprocessing only) |
| Vision+IMU Fusion Result | ✓ | Internal | ✗ | Volatile |
| Gate Trigger Event | ✓ | → MCU → Phone | ✓ | Session log |
| ML Classification Result | ✓ (on phone) | → App | ✓ | Session log |
| Mode State | ✓ | Internal | ✓ | Until mode exit |
| Session ID | ✓ | Internal | ✓ | Until session end |

---

# SCENARIO 2: Hit Detection & Auto Video Recording

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Golfer
    participant Glasses as Glasses<br/>(Cam/IMU/Mic/Spkr + MicroML)
    participant MCU as MCU
    participant PhoneApp as Phone App<br/>(Framework + Ambient Memory)
    participant HeavyML as Phone Heavy-ML<br/>(Gemma4 → Custom)
    participant Gallery as Gallery Storage

    Note over Golfer,Gallery: Golf Mode Already Active
    
    Golfer->>Glasses: Prepare for shot
    activate Glasses
    
    Glasses->>Glasses: High-fps IMU sampling
    Glasses->>Glasses: Detect SWING/HIT event<br/>[IMU spike detection]
    
    Glasses->>MCU: Gate① HIT DETECTED
    activate MCU
    
    MCU->>PhoneApp: Start Recording Command
    activate PhoneApp
    
    PhoneApp->>Glasses: Enable pre-buffer (5 sec)
    Glasses->>Glasses: Ring buffer active
    
    Glasses->>MCU: Record 10 sec video
    MCU->>PhoneApp: Transfer video stream
    PhoneApp->>Gallery: Save to Gallery
    activate Gallery
    
    PhoneApp->>HeavyML: Trigger analysis (async)
    activate HeavyML
    
    HeavyML->>HeavyML: Analyze video context
    HeavyML->>HeavyML: Generate insights
    HeavyML-->>PhoneApp: Return analysis
    deactivate HeavyML
    
    PhoneApp->>PhoneApp: Generate TTS
    PhoneApp->>Glasses: Send audio to speaker
    Glasses->>Golfer: Audio feedback
    
    PhoneApp->>Gallery: Save shot metadata
    Gallery-->>PhoneApp: Confirmed
    deactivate Gallery
    
    deactivate PhoneApp
    deactivate MCU
    deactivate Glasses
```

## Data Analysis for Scenario 2

| Data Type | Generated | Shared | Saved | Retention |
|-----------|-----------|--------|-------|-----------|
| IMU Hit Detection | ✓ | → Phone | ✗ | Volatile |
| Pre-buffer Video | ✓ (ring) | → Phone | ✗ | 5 sec overwrite |
| **Recorded Video** | ✓ (10 sec) | → Phone | ✓ **Gallery** | Permanent |
| Video Metadata | ✓ | Internal | ✓ | With video file |
| Shot Count | ✓ | Internal | ✓ | Session + Persistent |
| Gemma4 Analysis | ✓ | Internal | ✓ | Session cache |
| TTS Audio Stream | ✓ | → Glasses | ✗ | Streamed only |
| Session Statistics | ✓ | Internal | ✓ | Persistent |

---

# SCENARIO 3: Club Detection (Gemma4 ML Triggering)

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Golfer
    participant Glasses as Glasses<br/>(Cam/IMU/Mic/Spkr + MicroML)
    participant MCU as MCU
    participant PhoneApp as Phone App<br/>(Framework + Ambient Memory)
    participant HeavyML as Phone Heavy-ML<br/>(Gemma4 → Custom)
    participant Storage as Data Storage

    Note over Golfer,Storage: Golf Mode Active
    
    Glasses->>Glasses: Continuous camera streaming<br/>@ 1fps
    activate Glasses
    
    Glasses->>Glasses: Preprocess image<br/>- Resize<br/>- Crop<br/>- Grayscale<br/>(320x320x3)
    
    Glasses->>MCU: Send frame + metadata
    activate MCU
    
    MCU->>PhoneApp: Forward to framework
    activate PhoneApp
    
    PhoneApp->>PhoneApp: Queue for ML
    PhoneApp->>PhoneApp: Prepare prompt
    
    PhoneApp->>HeavyML: Send for inference
    activate HeavyML
    
    HeavyML->>HeavyML: Gemma4 INFERENCE<br/>[4.5GB model]<br/>Target: <1 sec
    
    HeavyML->>HeavyML: Detect:<br/>- Club present?<br/>- Club type?<br/>- Player posture?
    
    HeavyML-->>PhoneApp: Return text analysis
    deactivate HeavyML
    
    PhoneApp->>PhoneApp: Parse response
    PhoneApp->>MCU: Send display info
    deactivate PhoneApp
    
    MCU->>Glasses: Display AR overlay
    Glasses->>Golfer: Show club info (AR)
    
    PhoneApp->>Storage: Log inference result
    PhoneApp->>Storage: Log latency metrics
    PhoneApp->>Storage: Monitor battery impact
    
    deactivate MCU
    deactivate Glasses
```

## Data Analysis for Scenario 3

| Data Type | Generated | Shared | Saved | Retention |
|-----------|-----------|--------|-------|-----------|
| Raw Camera Frame | ✓ | → Phone | ✗ | Volatile |
| Preprocessed Image | ✓ | Internal | ✗ | Processing buffer |
| ML Prompt | ✓ | Internal | ✗ | Temporary |
| **Gemma4 Response** | ✓ | Internal | ✓ | Session cache |
| Club Detection | ✓ | → App UI | ✓ | Shot log |
| Inference Metrics | ✓ | Internal | ✓ | Performance log |
| Battery Stats | ✓ | Internal | ✓ | Session monitoring |

---

# SCENARIO 4: Occlusion Detection (Erase Hat - P0)

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Golfer
    participant Glasses as Glasses<br/>(Cam/IMU/Mic/Spkr + MicroML)
    participant MCU as MCU
    participant PhoneApp as Phone App<br/>(Framework + Ambient Memory)
    participant HeavyML as Phone Heavy-ML<br/>(Gemma4 → Custom)
    participant Storage as Data Storage

    Note over Golfer,Storage: Golf Mode Active
    
    Glasses->>Glasses: Continuous monitoring<br/>for hat/visor
    activate Glasses
    
    Glasses->>Glasses: Detect occlusion<br/>[SRIB occlusion model]
    
    Glasses->>Glasses: Check confidence threshold
    
    Glasses->>MCU: Send alert event
    activate MCU
    
    MCU->>PhoneApp: Forward occlusion alert
    activate PhoneApp
    
    PhoneApp->>PhoneApp: Generate user prompt
    
    PhoneApp->>PhoneApp: TTS: "Adjust hat"
    
    PhoneApp->>MCU: Send audio cue
    deactivate PhoneApp
    
    MCU->>Glasses: Play audio
    Glasses->>Golfer: Audio cue via speaker
    
    Glasses->>MCU: Log occlusion event
    deactivate MCU
    
    PhoneApp->>Storage: Save occlusion event<br/>(timestamp, duration)
    activate Storage
    Storage-->>PhoneApp: Confirmed
    deactivate Storage
    
    deactivate Glasses
```

## Data Analysis for Scenario 4

| Data Type | Generated | Shared | Saved | Retention |
|-----------|-----------|--------|-------|-----------|
| Occlusion Detection | ✓ | → Phone | ✓ | Session log |
| Confidence Score | ✓ | Internal | ✓ | Event log |
| User Prompt | ✓ | → TTS | ✗ | Temporary |
| TTS Audio | ✓ | → Glasses | ✗ | Streamed |
| **Occlusion Events** | ✓ | Internal | ✓ | Session report |

---

# SCENARIO 5: Audio Trigger (Voice Command)

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Golfer
    participant Glasses as Glasses<br/>(Cam/IMU/Mic/Spkr + MicroML)
    participant MCU as MCU
    participant PhoneApp as Phone App<br/>(Framework + Ambient Memory)
    participant HeavyML as Phone Heavy-ML<br/>(Gemma4 → Custom)
    participant Storage as Data Storage

    Golfer->>Glasses: "Hey Golf" (hotword)
    activate Glasses
    
    Glasses->>Glasses: Continuous audio sampling
    Glasses->>Glasses: Keyword Detection<br/>[MicroML on-device]
    
    Glasses->>MCU: HOTWORD DETECTED
    activate MCU
    
    Golfer->>Glasses: "Start recording"
    
    Glasses->>MCU: Send audio snippet (~3 sec)
    MCU->>PhoneApp: Forward command
    activate PhoneApp
    
    PhoneApp->>PhoneApp: Parse voice command
    
    PhoneApp->>PhoneApp: Execute action
    
    PhoneApp->>PhoneApp: Generate TTS confirmation
    
    PhoneApp->>MCU: "Recording started"
    deactivate PhoneApp
    
    MCU->>Glasses: Send audio
    Glasses->>Golfer: Audio confirmation
    
    PhoneApp->>Storage: Log command
    PhoneApp->>Storage: Save action result
    
    deactivate MCU
    deactivate Glasses
```

## Data Analysis for Scenario 5

| Data Type | Generated | Shared | Saved | Retention |
|-----------|-----------|--------|-------|-----------|
| Audio Stream | ✓ (continuous) | → Phone | ✗ | Volatile buffer |
| Hotword Event | ✓ | → Phone | ✓ | Event log |
| Audio Snippet | ✓ (~3 sec) | → Phone | ✗ | Processing only |
| Parsed Command | ✓ | Internal | ✓ | Session log |
| Action Result | ✓ | Internal | ✓ | Session log |
| TTS Confirmation | ✓ | → Glasses | ✗ | Streamed |

---

# ML Model Strategy: Gemma4 → Custom Model

## Current Implementation (Gemma4)

```mermaid
block-beta
columns 1
  block:Gemma4["GEMMA4 ON-DEVICE (Current)"]
    columns 1
    Size["Model Size: 4.5 GB"]
    Platform["Platform: S26 Ultra (Phone)"]
    Inference["Inference Time: <1 second"]
    Input["Input: Preprocessed image (320x320x3, grayscale)"]
    Output["Output: Text description/analysis"]
    UseCases["Use Cases: Club detection, Scene understanding,<br/>Shot analysis, Context-aware coaching"]
  end
  block:Limitations["LIMITATIONS"]
    columns 1
    L1["• High battery consumption (4.5GB model load)"]
    L2["• Latency bottleneck for real-time feedback"]
    L3["• Memory pressure on phone"]
    L4["• Not optimized for golf-specific scenarios"]
  end
```

## Future Implementation (Custom Model)

```mermaid
block-beta
columns 1
  block:Custom["CUSTOM ML MODEL (Planned)"]
    columns 1
    C1["Target Improvements:"]
    C2["• Model Size: <500 MB (10x reduction)"]
    C3["• Platform: Optimized for S26 Ultra + potential edge"]
    C4["• Inference Time: <200ms (5x improvement)"]
    C5["• Input: Same (320x320x3) + IMU fusion"]
    C6["• Output: Structured data (club type, confidence, action)"]
    C7["Use Cases: Golf-specific object detection,<br/>Swing analysis, Shot classification,<br/>Multi-modal fusion (vision + IMU + audio)"]
  end
  block:Benefits["BENEFITS"]
    columns 1
    B1["✓ Improved battery life (smaller model, faster inference)"]
    B2["✓ Lower latency (real-time feedback possible)"]
    B3["✓ Reduced memory footprint"]
    B4["✓ Golf-optimized accuracy"]
    B5["✓ Potential for on-glass inference (future)"]
  end
```

## ML Model Comparison

| Aspect | Gemma4 (Current) | Custom Model (Target) |
|--------|------------------|----------------------|
| **Size** | 4.5 GB | <500 MB |
| **Inference** | <1 sec | <200 ms |
| **Battery Impact** | High | Low |
| **Accuracy (Golf)** | General | Optimized |
| **Output Format** | Text | Structured |
| **Multi-modal** | Vision only | Vision + IMU + Audio |
| **Deployment** | Phone only | Phone + Edge |

---

# Data Flow Summary

## Data Types and Destinations

```mermaid
flowchart TD
    subgraph Glasses["Glasses (Cam/IMU/Mic/Spkr + MicroML)"]
        G1[IMU Raw]
        G2[Camera Raw]
        G3[Preprocessed Frames]
        G4[Audio Stream]
    end
    
    subgraph Phone["Phone App (Framework + Ambient Memory)"]
        P1[Video Recording]
        P2[ML Inference Result]
        P3[TTS Audio]
        P4[Session Stats]
        P5[Event Logs]
    end
    
    subgraph Storage["Data Storage"]
        S1[Gallery]
        S2[Cache]
        S3[Logs]
        S4[Settings]
    end
    
    G1 -->|API| Phone
    G2 -->|Stream| Phone
    G3 -->|API| Phone
    G4 -->|Buffer| Phone
    
    P1 --> S1
    P2 --> S2
    P3 -->|Stream| Glasses
    P4 --> S3
    P5 --> S3
```

## Data Flow Matrix

| Data Type | Generated | Shared | Saved | Where |
|-----------|-----------|--------|-------|-------|
| IMU Raw | Glasses | → Phone API | ✗ | Buffer |
| Camera Raw | Glasses | → Phone | ✗ | Stream |
| Camera Preprocessed | Glasses | → Phone | ✗ | Buffer |
| Video Recording | Glasses | → Phone | ✓ | Gallery |
| Audio Stream | Glasses | → Phone | ✗ | Buffer |
| ML Inference Result | Phone | → App | ✓ | Cache |
| TTS Audio | Phone | → Glasses | ✗ | Stream |
| Session Stats | Phone | Internal | ✓ | Storage |
| Event Logs | Both | → Phone | ✓ | Logs |
| User Preferences | Phone | Internal | ✓ | Settings |

## Privacy & Security Considerations

| Data Category | Sensitivity | Encryption | User Control |
|---------------|-------------|------------|--------------|
| Video Recordings | High | ✓ (at rest) | Delete/Export |
| Audio Recordings | High | ✓ (in transit) | Delete/Export |
| IMU Data | Low | ✓ (in transit) | Session only |
| ML Results | Medium | ✓ (at rest) | Delete |
| Session Logs | Medium | ✓ (at rest) | Auto-purge |
| User Preferences | Low | ✓ (at rest) | Edit/Delete |

---

# Appendix: Timing Constraints

| Operation | Target | Current (Gemma4) | Target (Custom) |
|-----------|--------|------------------|-----------------|
| Mode Detection | <5 sec | ~5 sec | <2 sec |
| Hit Detection | <100 ms | ~50 ms | ~30 ms |
| Video Start | <500 ms | ~300 ms | ~200 ms |
| ML Inference | <1 sec | ~800 ms | <200 ms |
| TTS Response | <2 sec | ~1.5 sec | <1 sec |
| End-to-End Shot Analysis | <3 sec | ~2.5 sec | <1 sec |

---

# Document Information

- **Created**: June 16, 2026
- **Author**: ML Team Analysis
- **Version**: 2.0 (Mermaid Format)
- **Status**: Draft - For Review
- **Related Documents**:
  - Golf_SDK_CUJ_Tasks.xlsx
  - 0528_XRUXLab_Golf_CUJs.pdf
  - Golf-UI-Flow.pdf
  - golf-ideation.txt (Slack transcript)
  - 42.jpg (Architecture Diagram)

---

*Note: This document is based on available project documentation and the 42.jpg architecture diagram. Some details may require validation with the development team.*
