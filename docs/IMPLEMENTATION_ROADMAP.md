# Aura Android Agent — Implementation Roadmap

## Milestone 1: Offline voice loop

- [ ] Kotlin/Compose application shell
- [ ] Microphone permission flow
- [ ] Foreground voice-session service
- [ ] AudioRecord PCM capture
- [ ] VAD adapter
- [ ] Offline STT adapter
- [ ] Offline TTS adapter
- [ ] Session state machine
- [ ] Stop/restart controls

## Milestone 2: Local intelligence

- [ ] llama.cpp JNI bridge
- [ ] GGUF model manager
- [ ] Streaming token API
- [ ] Conversation context manager
- [ ] Cancellation support
- [ ] Memory-pressure handling

## Milestone 3: Fast tools

- [ ] Typed command registry
- [ ] App launcher
- [ ] Search intent adapter
- [ ] URL intent adapter
- [ ] Confirmation policy for sensitive actions

## Milestone 4: Performance

- [ ] Sentence chunker
- [ ] Streaming TTS queue
- [ ] Barge-in cancellation
- [ ] Model load/unload scheduler
- [ ] RAM telemetry
- [ ] Thermal-aware policy
- [ ] Battery benchmark

## Milestone 5: Privacy hardening

- [ ] Strict offline mode
- [ ] Network guard in debug builds
- [ ] Local transcript policy
- [ ] No remote analytics for audio/transcripts
- [ ] Model checksum verification
- [ ] Privacy documentation

## Milestone 6: Device validation

- [ ] Snapdragon CPU backend
- [ ] Snapdragon Vulkan backend
- [ ] Dimensity CPU backend
- [ ] Dimensity Vulkan backend
- [ ] 6 GB RAM device
- [ ] 8 GB+ RAM device
- [ ] Screen-off test
- [ ] Bluetooth test
- [ ] YouTube/game audio AEC test
- [ ] Overnight soak test

## Acceptance criteria

A milestone is complete only after automated tests and at least one real-device validation pass. Performance figures are measured results, not assumptions copied from model specifications.
