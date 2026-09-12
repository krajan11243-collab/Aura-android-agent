# Offline Android Voice Agent

Base Android app for a privacy-first voice assistant that can run an on-device language model, speech recognition, and text-to-speech without requiring an internet connection during normal conversation.

## Planned architecture

- Android app: Kotlin + Jetpack Compose
- Conversation UI designed around a persistent voice session rather than repeatedly opening/closing the app
- On-device LLM runtime abstraction so a compatible quantized model can be bundled/downloaded locally
- Offline speech-to-text abstraction
- Android Text-to-Speech for spoken replies
- Foreground service for an active voice session, subject to Android OS permissions and battery/background limits
- Explicit online/offline mode indicator

## Important limitation

The app can keep the voice session active while the user is using it, but Android does not allow an ordinary app to stay permanently active without OS restrictions. A foreground service, microphone permission, notification, and device-specific battery settings may be required for a long-running session.

## Next steps

1. Add the Android Gradle project structure.
2. Add the voice-session state machine.
3. Add offline STT/TTS adapters.
4. Add an on-device LLM adapter and model setup instructions.
5. Add wake/stop controls and lifecycle handling.
