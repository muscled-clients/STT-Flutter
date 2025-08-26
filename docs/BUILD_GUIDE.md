# Mental Math Voice - Complete Build Guide

## Overview

This guide walks through building a Flutter mobile app with offline speech-to-text and mental math parsing capabilities. The project uses **vibe coding** - an iterative, AI-assisted development approach where you build incrementally with continuous testing.

## What We're Building

A privacy-first mobile app that:
- Captures voice input and converts it to text offline
- Parses mental math narration ("twenty-five times four")
- Displays step-by-step calculations
- Works completely offline using whisper.cpp

## Development Philosophy: Vibe Coding

**Vibe coding** means:
- Start simple, iterate constantly
- Test each piece as you build
- Use AI assistance effectively
- Keep the development flow smooth
- Focus on working code over perfect code initially

## Project Structure

```
📁 STT Mobile/
├── 📄 BUILD_GUIDE.md (this file)
├── 📄 PROMPTING_STRATEGY.md - How to prompt AI effectively
├── 📄 PHASE1_FOUNDATION.md - Flutter setup & navigation
├── 📄 PHASE2_AUDIO.md - Audio capture pipeline
├── 📄 PHASE3_NATIVE.md - Whisper.cpp integration
├── 📄 PHASE4_PARSER.md - Math expression parsing
└── 📄 DEBUGGING_TIPS.md - Common issues & solutions
```

## Development Phases

### Phase 1: Foundation (2-3 hours)
- Flutter project setup
- Basic navigation structure
- Riverpod state management
- Mock services for testing

### Phase 2: Audio Pipeline (3-4 hours)
- Microphone permissions
- Audio capture service
- Voice Activity Detection (VAD)
- Audio visualization

### Phase 3: Native Integration (4-6 hours)
- Whisper.cpp setup
- FFI bridge creation
- Model management
- Platform-specific builds

### Phase 4: Math Parser (3-4 hours)
- Number word recognition
- Expression parsing
- AST evaluation
- Result formatting

### Phase 5: UI Polish (2-3 hours)
- Live transcription display
- Result animations
- History management
- Settings screen

### Phase 6: Testing & Optimization (2-3 hours)
- Performance profiling
- Memory optimization
- Edge case handling
- Final polish

## Quick Start Commands

```bash
# Initial setup
git init
flutter create . --project-name mental_math_voice --org com.example

# Add dependencies
flutter pub add riverpod flutter_riverpod 
flutter pub add permission_handler record
flutter pub add ffi path_provider

# Run development
flutter run --debug

# Test continuously
flutter test --watch
```

## Key Success Factors

1. **Start with mocks** - Build UI with fake data first
2. **Test early** - Write tests as you code
3. **Incremental integration** - Add native code gradually
4. **Keep it simple** - Don't over-engineer early
5. **Use AI wisely** - Let AI handle boilerplate

## Architecture Overview

```
┌─────────────────────────┐
│   Presentation Layer    │ ← Flutter UI & Widgets
├─────────────────────────┤
│   Application Layer     │ ← Use Cases & Services  
├─────────────────────────┤
│     Domain Layer        │ ← Business Logic & Parser
├─────────────────────────┤
│  Infrastructure Layer   │ ← FFI, Audio, Storage
└─────────────────────────┘
```

## Tools You'll Need

- **Flutter SDK** 3.0+
- **Android Studio** or **VS Code**
- **Git** for version control
- **Android NDK** (for Android builds)
- **Xcode** (for iOS builds)
- **CMake** 3.18+ (for native compilation)

## Estimated Timeline

- **Solo Developer**: 20-30 hours total
- **With AI Assistance**: 15-20 hours
- **Experienced Flutter Dev**: 10-15 hours

## Getting Help

When stuck:
1. Check DEBUGGING_TIPS.md
2. Review the prompting strategies
3. Look at test examples
4. Simplify and isolate the problem

## Success Metrics

You'll know you're successful when:
- ✅ App records audio and shows waveform
- ✅ Transcription appears in real-time
- ✅ Math expressions evaluate correctly
- ✅ Works offline on real device
- ✅ Memory usage stays under 150MB
- ✅ RTF < 0.5 (processes faster than real-time)

## Next Steps

1. Read [PROMPTING_STRATEGY.md](PROMPTING_STRATEGY.md) to understand AI collaboration
2. Start with [PHASE1_FOUNDATION.md](PHASE1_FOUNDATION.md) to build the base
3. Progress through phases sequentially
4. Test continuously and iterate

## Remember

> "Perfect is the enemy of good. Get it working, then make it better."

The goal is a working app that solves the problem, not perfect code on the first try. Embrace the vibe coding approach: build, test, iterate, ship!