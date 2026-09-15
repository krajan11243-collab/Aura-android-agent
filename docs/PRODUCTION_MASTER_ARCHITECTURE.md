# 100% On-Device Standalone AI Agent — Production Master Architecture

## 0. Scope

This document defines the production architecture for a privacy-first Android voice agent running primarily on-device:

- No cloud dependency during normal conversation.
- No third-party application server.
- Audio, transcripts, model inference, task routing, and conversation state stay on-device unless the user explicitly enables an online feature.
- The design targets a persistent voice session implemented through supported Android foreground-service and lifecycle APIs.
- The system is optimized to keep heavyweight models out of memory while idle and to release them after inactivity.

> **Important platform boundary:** Android does not grant an ordinary app an unrestricted guarantee of 24/7 microphone access. Long-running microphone use must obey Android foreground-service, runtime permission, notification, app-standby, OEM battery-management, and background-execution rules. The product therefore implements a best-effort persistent voice session rather than claiming OS-independent permanent execution.

## 1. System Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                        Aura Android Agent                      │
├─────────────────────────────────────────────────────────────────┤
│ UI / Settings / Permissions / Session Controls                  │
├─────────────────────────────────────────────────────────────────┤
│ Voice Session Orchestrator                                     │
│  ├─ Lifecycle + state machine                                  │
│  ├─ audio focus / route management                             │
│  ├─ interruption / barge-in                                    │
│  └─ model memory policy                                        │
├─────────────────────────────────────────────────────────────────┤
│ Audio Frontend                                                 │
│  ├─ AudioRecord                                                │
│  ├─ 16 kHz PCM ring buffer                                     │
│  ├─ VAD                                                        │
│  └─ optional AEC / NS / AGC through Android audio path         │
├─────────────────────────────────────────────────────────────────┤
│ Speech Layer                                                   │
│  ├─ Offline STT adapter (Sherpa-ONNX)                         │
│  └─ Offline TTS adapter (Kokoro/Sherpa-ONNX where supported)  │
├─────────────────────────────────────────────────────────────────┤
│ Agent Layer                                                    │
│  ├─ Fast command router                                        │
│  ├─ Tool/Intent executor                                       │
│  ├─ LLM adapter (llama.cpp)                                   │
│  └─ Conversation memory                                       │
├─────────────────────────────────────────────────────────────────┤
│ Android Integration                                            │
│  ├─ Foreground service                                         │
│  ├─ notification                                                │
│  ├─ intents / package launch                                   │
│  ├─ accessibility only when explicitly enabled by user        │
│  └─ battery / wake policy                                      │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Model Stack

### 2.1 Voice Activity Detection

**Candidate:** Silero VAD v5, ONNX.

Responsibilities:
- Continuously inspect short PCM windows.
- Suppress unnecessary STT/LLM work while silence is present.
- Provide start-of-speech and end-of-speech events.

The exact RAM footprint depends on runtime, tensor arenas, sample rate, and buffering, so the figures must be measured on target devices rather than treated as contractual numbers.

### 2.2 Speech-to-Text

**Candidate:** Sherpa-ONNX streaming Zipformer Hindi/multilingual model.

Requirements:
- Streaming partial results.
- Final transcript event.
- Explicit language configuration.
- No network calls.
- Model assets stored in app-managed local storage.

Target behavior is low-latency partial decoding. The claimed 100–200 ms end-to-end STT number must be benchmarked per chipset and model; it is not guaranteed by model choice alone.

### 2.3 Local LLM

**Candidate:** Qwen2.5-1.5B-Instruct, GGUF Q4_K_M.

Runtime:
- llama.cpp through an Android NDK integration.
- Prefer memory-mapped model loading where supported.
- Keep context length bounded.
- Stream output tokens.
- Use a small system prompt and explicit tool schema to reduce token latency.

The effective RAM cost depends on model metadata, context size, KV-cache, backend, Vulkan buffers, and allocator behavior. Treat the 1.1–1.4 GB figures as initial planning estimates, not guaranteed values.

### 2.4 Text-to-Speech

**Candidate:** Kokoro-82M ONNX or another fully offline TTS engine supported by the chosen runtime.

Requirements:
- Incremental/chunked synthesis.
- PCM streaming to AudioTrack/AudioOutput.
- Voice selection stored locally.
- No cloud TTS fallback in strict offline mode.

Natural female voice quality is model/voice dependent and should be validated on-device. ChatTTS is not assumed to be the default mobile production engine merely because it is available; latency, memory, licensing, and Android runtime maturity must be benchmarked.

## 3. Android Persistent Voice Session

### 3.1 Foreground service

Use a dedicated foreground service for an active voice session and declare the microphone-related foreground service type required by the Android target SDK and device policy.

Core requirements:
- `RECORD_AUDIO` runtime permission.
- Foreground-service notification.
- Correct foreground-service declaration for the target Android version.
- Explicit start/stop controls.
- Service lifecycle recovery.
- Clear user-visible status while microphone capture is active.

Do **not** claim that adding `phoneCall` automatically grants unrestricted microphone access. Phone-call audio roles and foreground-service types have platform-specific requirements and are not a generic bypass for background restrictions.

### 3.2 Audio mode

Use Android communication audio APIs only where they are appropriate for the actual use case.

- `AudioManager.MODE_IN_COMMUNICATION` can be evaluated for conversational audio behavior.
- Prefer platform-supported acoustic processing such as `AcousticEchoCanceler`, `NoiseSuppressor`, and device-provided voice-processing paths when available.
- AEC/NS are not guaranteed to erase arbitrary YouTube/game audio perfectly. Real-world performance depends on speaker, mic geometry, routing, volume, DSP implementation, and OEM firmware.

### 3.3 Battery and sleep

Do not silently bypass battery policy.

- Use normal Android power-management APIs first.
- Use `PARTIAL_WAKE_LOCK` only for brief, justified critical sections and release it promptly.
- If continuous operation genuinely requires exemption, explain the tradeoff to the user and take them to the appropriate system setting.
- `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` should not be treated as a universal solution and should only be used where policy and Play requirements permit it.

## 4. Three-Tier Memory Lifecycle

### Tier A — Idle listening

Loaded:
- audio pipeline
- VAD
- small ring buffers
- lightweight command metadata

Unloaded:
- LLM
- heavyweight STT decoder state when not needed
- TTS model

Goal: minimize RAM and battery while maintaining the voice-session state.

### Tier B — Fast command routing

Before loading the LLM, classify known deterministic commands with a local intent router.

Examples:
- Open YouTube.
- Search YouTube for a phrase.
- Open a known app package.
- Start/stop an audio session.
- Read a local setting.

Use explicit Android intents where supported. For example, YouTube search can use a web/search intent or app-specific intent only when the target app exposes it. Do not assume an exact intent contract for every third-party app.

### Tier C — Conversation mode

Load:
- STT streaming decoder
- LLM runtime
- TTS runtime

Policy:
- Reuse loaded runtimes for the active session.
- Limit context/KV cache size.
- Stream LLM output.
- Start TTS as soon as a safe sentence chunk is available.
- Release heavyweight resources after an inactivity timer.

The proposed 15-second flush is a tuning parameter. Production code should make this configurable and measure whether reloading costs more energy than retaining the runtime for likely follow-up turns.

## 5. End-to-End Streaming Pipeline

```text
Mic PCM
  ↓
Ring Buffer
  ↓
VAD
  ├─ silence → remain in idle state
  └─ speech → STT
               ↓
          partial transcript
               ↓
        local fast-router
          ├─ deterministic command → Android executor
          └─ conversational/task request → llama.cpp
                                               ↓
                                      streamed response text
                                               ↓
                                      sentence chunker
                                               ↓
                                             TTS
                                               ↓
                                          AudioTrack
```

### Latency budget

A total response latency such as ~600 ms should be a benchmark target, not a guaranteed specification. Measure the following separately:

1. voice onset detection latency
2. STT partial/final latency
3. router classification latency
4. model first-token latency
5. TTS first-audio latency
6. audio output startup latency

Optimize the slowest stage rather than hard-coding a single end-to-end promise.

## 6. Barge-In / Interruption

When the agent is speaking:

1. Keep VAD active on a low-cost path.
2. Detect user speech onset.
3. Immediately stop or duck the current TTS playback.
4. Cancel the current TTS generation job.
5. Cancel or deprioritize the current LLM stream when the user asks a new request.
6. Return the audio pipeline to STT capture.

`AudioTrack.flush()` alone is not a complete cancellation mechanism; the architecture must also cancel the producer coroutine/thread and prevent stale audio buffers from being written afterward.

## 7. State Machine

```text
STOPPED
  ↓ startSession()
STARTING
  ↓
IDLE_LISTENING
  ↓ speech detected
LISTENING
  ↓ transcript finalized
ROUTING
  ├─ fast command → EXECUTING_TOOL → IDLE_LISTENING
  └─ LLM task → THINKING
                  ↓ streamed tokens
                SPEAKING
                  ├─ barge-in → LISTENING
                  └─ finished → IDLE_LISTENING

Any state → STOPPING → STOPPED
```

All transitions must be serialized. Avoid simultaneous microphone, TTS, and service teardown operations.

## 8. Project Structure

```text
app/src/main/java/<package>/
├── MainActivity.kt
├── agent/
│   ├── AgentOrchestrator.kt
│   ├── AgentState.kt
│   ├── CommandRouter.kt
│   ├── ToolExecutor.kt
│   └── ConversationMemory.kt
├── audio/
│   ├── AudioCapture.kt
│   ├── AudioOutput.kt
│   ├── AudioProcessing.kt
│   └── AudioFocusManager.kt
├── service/
│   └── VoiceAgentForegroundService.kt
├── speech/
│   ├── SttEngine.kt
│   ├── SherpaOnnxSttEngine.kt
│   ├── TtsEngine.kt
│   └── KokoroTtsEngine.kt
├── llm/
│   ├── LlmEngine.kt
│   ├── LlamaCppEngine.kt
│   └── ModelManager.kt
├── lifecycle/
│   ├── ModelLifecycleManager.kt
│   └── ResourcePolicy.kt
├── permissions/
│   └── PermissionController.kt
├── ui/
│   ├── HomeScreen.kt
│   ├── AgentSettingsScreen.kt
│   └── SessionStatus.kt
└── util/
    ├── CoroutineDispatchers.kt
    └── LogStore.kt

app/src/main/assets/
├── models/
│   ├── vad/
│   ├── stt/
│   ├── llm/
│   └── tts/
└── prompts/
```

## 9. Model Manager Contract

The model manager exposes a single lifecycle contract:

```kotlin
interface ModelManager {
    suspend fun ensureVadLoaded()
    suspend fun ensureSttLoaded()
    suspend fun ensureLlmLoaded()
    suspend fun ensureTtsLoaded()

    suspend fun unloadLlm()
    suspend fun unloadTts()
    suspend fun unloadStt()

    fun currentMemoryMode(): MemoryMode
}

enum class MemoryMode {
    IDLE,
    FAST_ROUTE,
    CONVERSATION
}
```

All heavy model load/unload operations should run outside the main thread and be guarded against duplicate concurrent initialization.

## 10. Local Privacy Boundary

Strict offline mode must enforce:

- No HTTP client invocation for agent conversation.
- No analytics upload containing audio or transcript.
- No automatic crash-report payload containing conversation text.
- No remote logging of model prompts/responses.
- Model and transcript storage encrypted or access-controlled where appropriate.
- A visible offline indicator in the UI.

Create a central `NetworkPolicy`/`OfflineGuard` abstraction so accidental network use is detectable during development and test builds.

## 11. Permissions and User Controls

The first-run experience should request only what is necessary.

Minimum voice session flow:

1. Explain that microphone access is required.
2. Request microphone permission.
3. Start a visible foreground-service notification.
4. Let the user explicitly enable/disable persistent voice mode.
5. Show whether the agent is idle, listening, thinking, or speaking.
6. Provide an immediate stop-microphone control.
7. Provide battery-policy guidance instead of silently changing system settings.

Optional capabilities, such as accessibility-based UI automation, must be separately explained and enabled by the user.

## 12. Tool / Intent Security

The LLM should never receive unrestricted shell or arbitrary Android capability.

Use a typed tool registry:

```text
open_app(package)
search_youtube(query)
open_url(url)
set_volume(stream, level)
read_local_setting(key)
```

Validate every argument before execution. For sensitive actions, require explicit user confirmation.

Do not implement unrestricted background interaction with other apps through hidden or deceptive mechanisms.

## 13. Llama.cpp Android Integration

Recommended build direction:

- Android NDK C++ layer.
- JNI wrapper with a small stable API.
- GGUF model file in app-private storage.
- Optional Vulkan backend when device/runtime compatibility is verified.
- CPU fallback for devices where Vulkan is unavailable or unstable.

Keep the Java/Kotlin layer independent of llama.cpp internals so the backend can later be replaced.

Example Kotlin abstraction:

```kotlin
interface LlmEngine {
    suspend fun load(modelPath: String)
    fun stream(prompt: List<ChatMessage>): Flow<String>
    suspend fun cancel()
    suspend fun unload()
}
```

## 14. Sherpa-ONNX Integration

Use a dedicated adapter that owns:

- model paths
- tokenizer/config paths
- recognizer lifecycle
- PCM input conversion
- partial/final callback events
- thread/coroutine confinement

Do not let UI code call the native recognizer directly.

## 15. TTS Chunking

The chunker should flush on:

- sentence punctuation (`.`, `!`, `?`, `।`)
- a safe word-count threshold
- an explicit newline
- an upper latency timeout

Avoid splitting inside abbreviations or numbers when practical.

Target:

```text
LLM token stream
      ↓
text accumulator
      ↓
chunk detector
      ├─ chunk ready → TTS queue
      └─ keep buffering
```

The TTS queue must be cancellable so barge-in can stop future chunks as well as current playback.

## 16. Battery and Thermal Strategy

Production optimization priorities:

1. Keep VAD small and low duty-cycle.
2. Avoid keeping the LLM loaded while idle.
3. Reduce LLM context size where quality permits.
4. Prefer streaming and bounded buffers.
5. Release native allocations deterministically.
6. Observe thermal state and reduce concurrency under sustained load.
7. Do not run a busy polling loop for the microphone; use blocking/efficient audio reads.

24/7 use is an operational goal, not a guarantee of unlimited battery life or zero thermal impact.

## 17. Benchmark Matrix

Every release should be tested on at least:

- Snapdragon mid/high-tier device.
- Dimensity mid/high-tier device.
- 6 GB RAM device.
- 8 GB+ RAM device.
- Android screen-on.
- Android screen-off.
- Bluetooth headset.
- Wired headset where supported.
- Phone speaker.
- YouTube/game audio playing.
- Device locked.
- Battery saver enabled.

Measure:

| Metric | Target | Status |
|---|---:|---|
| VAD wake latency | < 200 ms | benchmark required |
| STT partial latency | device dependent | benchmark required |
| LLM first token | < 500 ms desired | benchmark required |
| TTS first audio | < 300 ms desired | benchmark required |
| End-to-end response | ~600–1200 ms desired | benchmark required |
| Idle RAM | as low as practical | benchmark required |
| Conversation RAM | device dependent | benchmark required |
| 1-hour active battery drain | measured % | benchmark required |

## 18. Failure Recovery

The service must recover from:

- AudioRecord initialization failure.
- Mic resource loss because another app owns audio input.
- Bluetooth route change.
- Screen lock/unlock.
- Activity process recreation.
- Model load OOM.
- Native runtime exception.
- TTS synthesis cancellation.
- LLM cancellation.
- Temporary audio focus loss.

For OOM:

1. stop TTS
2. unload TTS
3. unload LLM
4. reduce context / concurrency
5. return to idle mode
6. show a recoverable status instead of crashing the service

## 19. Release Phases

### Phase 1 — Core offline conversation

- foreground service
- VAD
- offline STT
- offline TTS
- simple local LLM
- conversation UI

### Phase 2 — Fast tools

- app launch
- search intents
- local device commands
- typed tool registry

### Phase 3 — Production memory policy

- load/unload scheduler
- memory/thermal telemetry
- cancellation correctness
- barge-in optimization

### Phase 4 — Device compatibility

- Snapdragon
- Dimensity
- Vulkan/CPU fallback
- OEM battery behavior

### Phase 5 — Hardening

- offline network guard
- secure local storage
- automated tests
- crash-safe service restart
- release checklist

## 20. Test Plan

### Unit tests

- command router
- sentence chunker
- state transitions
- model lifecycle
- cancellation semantics
- permission state mapping

### Instrumentation tests

- foreground service start/stop
- notification state
- screen off/on transitions
- Bluetooth route changes
- audio focus changes

### Soak tests

- 1 hour
- 4 hour
- overnight

Track:

- memory growth
- native allocation growth
- battery drain
- audio dropouts
- thermal throttling
- service termination
- transcript failures

## 21. Production Truths / Corrections to the Initial Blueprint

1. **Android cannot promise unrestricted 24/7 background microphone access.** A foreground service and correct permissions are required, and OEMs can still impose restrictions.
2. **`phoneCall` is not a generic bypass.** Use the correct foreground-service type and role for the actual product behavior.
3. **AEC/NS are not perfect.** They can substantially help, but game/YouTube audio rejection must be tested on real hardware.
4. **RAM numbers are estimates.** Native backends, KV cache, context size, Vulkan buffers, and allocator behavior materially change memory use.
5. **~600 ms is a performance target, not a universal guarantee.** Benchmark the full pipeline.
6. **mmap does not mean instant 300 ms model startup on every phone.** Page faults, storage speed, GPU backend initialization, and context setup matter.
7. **`flush()` is not enough for barge-in.** Producer cancellation is also required.
8. **Battery-optimization exemption should be user-visible and policy-compliant.** Avoid silently claiming a bypass.

## 22. Definition of Done

The project is production-ready only when all of the following are true:

- Offline conversation works with network disabled.
- Audio and transcripts remain local in strict offline mode.
- Foreground service status is visible and correct.
- Session survives ordinary Activity recreation.
- Barge-in stops TTS reliably.
- Heavy models unload after the configured idle period.
- Tool execution is typed and validated.
- OOM and audio-route failures recover gracefully.
- Target devices meet the benchmark thresholds.
- No feature depends on an undocumented cloud endpoint.

## 23. Suggested Repository Roadmap

```text
/docs/PRODUCTION_MASTER_ARCHITECTURE.md   ← this specification
/app                                    ← Android application
/native                                 ← llama.cpp / native bridge
/models                                 ← download/setup docs only; large binaries excluded from Git
/benchmarks                             ← latency/RAM/battery test notes
/scripts                                ← reproducible model/setup helpers
/.github/workflows                      ← CI checks
```

Large model binaries should normally **not** be committed to Git. Store model acquisition instructions, checksums, licenses, and expected file names in documentation, and let the application install verified model assets locally.

---

## Final Architecture Principle

**VAD-first + offline STT + local intent router + on-demand local LLM + streaming offline TTS + cancellable foreground voice session** is the production architecture.

The system should optimize for four simultaneous properties:

**Privacy → Reliability → Low latency → Controlled resource usage.**
