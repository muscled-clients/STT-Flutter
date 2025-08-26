# Mental Math Voice

A Flutter mobile app that performs offline speech-to-text using whisper.cpp and parses spoken mental-math narration to compute and display results.

## Features

- 🎤 Real-time speech recognition (offline by default)
- 🧮 Mental math narration parsing ("twenty-five times four equals...")
- 📱 Cross-platform (Android & iOS)
- 🔒 Privacy-first: on-device inference
- 📊 Performance metrics & RTF display
- 📝 Session history with timestamps
- ⚡ Optional online mode (toggled in settings)

## Quick Start

```bash
# Clone and setup
git clone <repo>
cd mental_math_voice

# Install dependencies
flutter pub get

# Download whisper model
scripts/download_models.sh

# Run app
flutter run
```

See [SETUP.md](SETUP.md) for detailed build instructions.

## Architecture

Clean architecture with clear separation:
- **Presentation**: UI screens, widgets, state management
- **Application**: Use cases, service orchestration
- **Domain**: Core business logic, math parser, entities
- **Infrastructure**: Native FFI, audio capture, storage

See [ARCHITECTURE.md](ARCHITECTURE.md) for details.

## Development Backlog

### Phase 1: Foundation ✅
- [x] Project planning & documentation
- [ ] Flutter scaffold with Riverpod
- [ ] Basic navigation (Record/History/Settings)

### Phase 2: Audio Pipeline
- [ ] Microphone capture (16kHz mono)
- [ ] Simple VAD/endpointing
- [ ] Mock STT service for UI development

### Phase 3: Native Integration
- [ ] Whisper.cpp submodule setup
- [ ] Android JNI bridge (.so)
- [ ] iOS static library (.a)
- [ ] dart:ffi bindings
- [ ] Model management & download

### Phase 4: Core Logic
- [ ] Math narration parser
- [ ] Number word mapping
- [ ] Expression evaluator (decimal arithmetic)
- [ ] Parser unit tests (≥40 examples)

### Phase 5: User Interface
- [ ] Record screen with live transcript
- [ ] Step-by-step result display
- [ ] History list with timestamps
- [ ] Settings (model selection, offline toggle)
- [ ] Accessibility improvements

### Phase 6: Polish
- [ ] Performance metrics (RTF display)
- [ ] Debug panel
- [ ] Export/share sessions
- [ ] Multi-language support (stretch)
- [ ] Online STT fallback (stretch)

## Testing

- **Unit tests**: Parser coverage ≥90%
- **Integration tests**: Audio pipeline, FFI bridge
- **Golden tests**: UI screens
- See [TEST_PLAN.md](docs/TEST_PLAN.md)

## Performance

Target metrics:
- Whisper base.en: RTF < 0.5 on mid-range devices
- Memory usage: < 150MB peak
- Battery drain: < 2% per 5min session

## Privacy

- Default: All processing on-device
- No network requests without explicit opt-in
- Clear indicator when online mode enabled
- No telemetry or analytics

## License

MIT - See [LICENSE](LICENSE)