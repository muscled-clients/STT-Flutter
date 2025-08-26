# AI Prompting Strategy for Mental Math Voice

## The Art of Prompting for Development

This guide shows you how to effectively collaborate with AI assistants to build this project. Good prompting is the difference between struggling for hours and shipping features in minutes.

## Core Prompting Principles

### 1. Context is King
Always provide:
- What you're building
- What currently works
- What specific problem you're solving
- Any error messages

### 2. Be Specific, Not Vague
❌ **Bad**: "Help me with Flutter"
✅ **Good**: "Create a Flutter recording button that starts audio capture at 16kHz mono PCM"

### 3. Iterate in Small Steps
Break complex features into bite-sized tasks. Each prompt should have a clear, achievable goal.

## Phase-Specific Prompting Templates

### Phase 1: Foundation Setup

```
"Create a Flutter app structure with clean architecture:
- 4 layers: presentation, application, domain, infrastructure
- Riverpod for state management
- 3 main screens: Record, History, Settings
- Bottom navigation bar
Show me the folder structure and main.dart"
```

```
"Add Riverpod providers for:
- Audio recording state (idle, recording, processing)
- Transcript stream
- Settings (model selection, offline mode)
Use Riverpod code generation. Show complete code."
```

### Phase 2: Audio Pipeline

```
"Implement audio capture service:
- Use 'record' package
- 16kHz sample rate, mono channel
- Stream PCM audio buffers
- Include microphone permission handling
Show both the service and permission setup"
```

```
"Add simple VAD (Voice Activity Detection):
- Detect silence vs speech
- Use RMS energy threshold
- 500ms silence triggers end-of-speech
Include visualization widget for audio levels"
```

### Phase 3: Native Integration

```
"Create FFI bridge for whisper.cpp:
- Define Dart FFI types for whisper functions
- Handle model loading from assets
- Stream transcription results
- Memory-safe cleanup
Show whisper_ffi.dart with all bindings"
```

```
"Write platform-specific code to load whisper library:
- Android: load from jniLibs
- iOS: load from Frameworks
- Include error handling
Show method channel implementation"
```

### Phase 4: Math Parser

```
"Implement number word parser:
Input: 'twenty-five thousand three hundred forty-two'
Output: 25342

Handle:
- Basic numbers (one-nine)
- Teens (eleven-nineteen)  
- Tens (twenty-ninety)
- Hundreds, thousands, millions
- Compound numbers

Include comprehensive tests"
```

```
"Create math expression parser with these operations:
- Basic: plus, minus, times, divided by
- Advanced: percent, squared, square root
- Precedence: * / before + -
- Parentheses support

Parse 'twenty-five times four plus ten percent' → AST → evaluate to 110
Show parser, AST nodes, and evaluator"
```

### Phase 5: UI Development

```
"Build recording screen with:
- Circular record button (red when recording)
- Audio waveform visualization
- Live transcript display
- Result card showing calculation steps
Use Flutter animations for smooth transitions"
```

## Effective Debugging Prompts

### When You Hit Errors

```
"I'm getting this error in my Flutter whisper integration:
[paste full error]

Current code:
[paste relevant code]

Environment: Android/iOS, Flutter version X
What's wrong and how do I fix it?"
```

### For Performance Issues

```
"My whisper transcription has high latency:
- Model: base.en
- Device: [device specs]
- Current RTF: 1.2
- Buffer size: 16000 samples

How can I optimize for RTF < 0.5?"
```

## Code Review Prompts

```
"Review this math parser for correctness and edge cases:
[paste code]

Check for:
- Parsing accuracy
- Error handling
- Performance
- Missing test cases"
```

## Advanced Prompting Patterns

### The "Show Me" Pattern
```
"Show me complete, working code for [specific feature]
Include:
- All imports
- Error handling
- Comments for complex logic
- Example usage"
```

### The "Fix and Explain" Pattern
```
"This code doesn't work:
[paste broken code]

1. Fix the issue
2. Explain what was wrong
3. Show the corrected version
4. Add a test to prevent regression"
```

### The "Best Practice" Pattern
```
"What's the best practice for [specific task] in Flutter?
Consider:
- Performance
- Maintainability
- Testing
- Platform differences
Show implementation"
```

## Prompt Chaining Strategy

Build complex features by chaining simpler prompts:

1. **Architecture First**
   ```
   "Design the architecture for audio capture → STT → parsing pipeline"
   ```

2. **Implementation**
   ```
   "Implement the audio capture service from the architecture"
   ```

3. **Integration**
   ```
   "Connect audio service to STT using the stream from step 2"
   ```

4. **Testing**
   ```
   "Write tests for the integrated pipeline"
   ```

5. **Optimization**
   ```
   "Optimize the pipeline for < 150MB memory usage"
   ```

## Common Pitfalls to Avoid

### 1. Asking for Everything at Once
❌ "Build the entire app"
✅ "Build the record button UI"

### 2. Missing Context
❌ "Fix this error" [no error shown]
✅ "Fix this FFI error on Android: [full error]"

### 3. Skipping Prerequisites
❌ "Add whisper support" [no FFI setup]
✅ "Setup FFI first, then add whisper"

### 4. Ignoring Incremental Development
❌ "Make it perfect"
✅ "Make it work, then optimize"

## Testing-Driven Prompts

Always include testing:
```
"Write a Flutter widget that [does X]
Include:
1. The widget implementation
2. A widget test
3. A golden test
4. Example usage"
```

## Platform-Specific Prompts

### Android
```
"Setup Android native code for whisper:
- NDK configuration in build.gradle
- JNI bridge setup
- .so library loading
- ProGuard rules"
```

### iOS
```
"Setup iOS native code for whisper:
- Podspec configuration
- Swift/ObjC bridge
- Framework embedding
- Info.plist permissions"
```

## Optimization Prompts

```
"Profile and optimize this code:
[paste code]

Target metrics:
- Memory: < 150MB
- RTF: < 0.5
- Battery: < 2% per 5min

Suggest improvements with benchmarks"
```

## The Meta-Prompt

When stuck, use this:
```
"I'm building a Flutter app with offline STT using whisper.cpp.
Current status: [what works]
Trying to: [specific goal]
Blocked by: [specific issue]

What's the best approach to solve this?"
```

## Remember

- **Start simple**: Get basic functionality working first
- **Test constantly**: Verify each piece works
- **Document issues**: Include errors and context
- **Iterate quickly**: Small improvements over big rewrites
- **Learn patterns**: Reuse successful prompt structures

## Success Checklist

✅ Each prompt has a clear, single objective
✅ Context and constraints are specified
✅ Error messages are included in full
✅ Code samples are complete and runnable
✅ Expected behavior is described
✅ Test cases are requested when appropriate

Use these patterns and watch your development velocity soar!