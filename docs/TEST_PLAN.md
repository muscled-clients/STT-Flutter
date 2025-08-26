# Test Plan

## Overview

Comprehensive testing strategy for the Mental Math Voice app, covering unit, integration, widget, and end-to-end tests.

## Test Coverage Goals

- **Overall**: ≥80%
- **Parser**: ≥90%
- **Domain Logic**: ≥85%
- **Critical Paths**: 100%

## Test Categories

### 1. Unit Tests

#### Domain Layer

**Math Parser** (`test/domain/math/parser_test.dart`)
- Number word recognition (20 tests)
- Basic operations (10 tests)
- Complex expressions (10 tests)
- Percentage calculations (5 tests)
- Powers and roots (5 tests)
- Parentheses handling (5 tests)
- Error cases (5 tests)

**Math Evaluator** (`test/domain/math/evaluator_test.dart`)
- AST evaluation (10 tests)
- Operator precedence (5 tests)
- Decimal precision (5 tests)
- Division by zero handling (2 tests)
- Overflow detection (3 tests)

**Number Words** (`test/domain/math/number_words_test.dart`)
- Cardinal numbers (10 tests)
- Compound numbers (5 tests)
- Large numbers (5 tests)
- Decimal numbers (5 tests)
- Fractions (5 tests)

#### Application Layer

**Transcription Service** (`test/application/stt/transcription_service_test.dart`)
- Stream processing (5 tests)
- Partial updates (3 tests)
- Finalization (3 tests)
- Error handling (4 tests)

**Calculation Service** (`test/application/math/calculation_service_test.dart`)
- Parse and evaluate pipeline (5 tests)
- Step tracking (3 tests)
- History management (3 tests)

### 2. Integration Tests

**Audio Pipeline** (`test/integration/audio_pipeline_test.dart`)
```dart
testWidgets('audio capture to transcription flow', (tester) async {
  // Mock microphone input
  // Verify VAD detection
  // Check buffer management
  // Validate transcription trigger
});
```

**FFI Bridge** (`test/integration/whisper_ffi_test.dart`)
```dart
test('whisper model initialization', () async {
  // Load test model
  // Verify memory allocation
  // Check thread configuration
});

test('transcription accuracy', () async {
  // Load test audio
  // Process through whisper
  // Verify transcript
});
```

**Storage Layer** (`test/integration/storage_test.dart`)
```dart
test('session persistence', () async {
  // Create session
  // Save to storage
  // Retrieve and verify
  // Test migration
});
```

### 3. Widget Tests

**Record Screen** (`test/widgets/record_screen_test.dart`)
```dart
testWidgets('recording flow', (tester) async {
  await tester.pumpWidget(RecordScreen());
  
  // Find record button
  await tester.tap(find.byIcon(Icons.mic));
  await tester.pump();
  
  // Verify recording state
  expect(find.text('Recording...'), findsOneWidget);
  
  // Stop recording
  await tester.tap(find.byIcon(Icons.stop));
  await tester.pumpAndSettle();
  
  // Check transcript display
  expect(find.byType(TranscriptDisplay), findsOneWidget);
});
```

**Settings Screen** (`test/widgets/settings_screen_test.dart`)
```dart
testWidgets('model selection', (tester) async {
  // Test model dropdown
  // Verify selection persistence
  // Check download trigger
});

testWidgets('offline mode toggle', (tester) async {
  // Toggle offline mode
  // Verify UI update
  // Check service switch
});
```

### 4. Golden Tests

**UI Screenshots** (`test/golden/`)
- Record screen states
- History list
- Settings page
- Result cards
- Error states

```dart
testWidgets('record screen golden', (tester) async {
  await tester.pumpWidget(RecordScreen());
  await expectLater(
    find.byType(RecordScreen),
    matchesGoldenFile('golden/record_screen.png'),
  );
});
```

### 5. Performance Tests

**Memory Profiling** (`test/performance/memory_test.dart`)
```dart
test('memory usage during recording', () async {
  final baseline = getMemoryUsage();
  
  // Start recording
  await startRecording();
  
  // Record for 60 seconds
  await Future.delayed(Duration(seconds: 60));
  
  final peak = getMemoryUsage();
  expect(peak - baseline, lessThan(150 * 1024 * 1024)); // <150MB
});
```

**RTF Measurement** (`test/performance/rtf_test.dart`)
```dart
test('realtime factor', () async {
  final audio = loadTestAudio('samples/test_30s.wav');
  final start = DateTime.now();
  
  await whisper.transcribe(audio);
  
  final elapsed = DateTime.now().difference(start);
  final rtf = elapsed.inSeconds / 30;
  
  expect(rtf, lessThan(0.5)); // RTF < 0.5
});
```

## Test Data

### Audio Fixtures
```
test/fixtures/audio/
├── samples/
│   ├── numbers_basic.wav      # "one two three four five"
│   ├── math_simple.wav        # "five plus three"
│   ├── math_complex.wav       # "twenty-five times four plus ten percent"
│   └── edge_cases.wav         # Various edge cases
├── noise/
│   ├── silence.wav
│   ├── background_noise.wav
│   └── music.wav
└── accents/
    ├── us_english.wav
    ├── uk_english.wav
    └── non_native.wav
```

### Test Transcripts
```dart
const testCases = [
  // Basic
  ("five", 5),
  ("twenty", 20),
  ("hundred", 100),
  
  // Compound
  ("twenty-five", 25),
  ("three hundred forty-two", 342),
  
  // Operations
  ("five plus three", 8),
  ("ten minus four", 6),
  ("six times seven", 42),
  
  // Complex
  ("twenty-five times four plus ten percent", 110),
  ("square root of sixteen", 4),
  
  // Edge cases
  ("divided by zero", ParseError),
  ("", EmptyInput),
];
```

## Mock Implementations

### Mock STT Service
```dart
class MockSTTService implements STTService {
  final bool shouldFail;
  final Duration delay;
  
  @override
  Stream<Transcript> transcribe(AudioBuffer audio) async* {
    if (shouldFail) throw STTException('Mock failure');
    
    await Future.delayed(delay);
    
    // Simulate progressive transcription
    yield Transcript(text: "twenty", isFinal: false);
    await Future.delayed(Duration(milliseconds: 500));
    yield Transcript(text: "twenty five", isFinal: false);
    await Future.delayed(Duration(milliseconds: 500));
    yield Transcript(text: "twenty five times four", isFinal: true);
  }
}
```

### Mock Audio Service
```dart
class MockAudioService implements AudioService {
  @override
  Stream<AudioBuffer> capture() async* {
    // Generate sine wave
    final samples = Float32List(16000); // 1 second
    for (int i = 0; i < samples.length; i++) {
      samples[i] = sin(2 * pi * 440 * i / 16000); // 440Hz
    }
    yield AudioBuffer(samples);
  }
}
```

## Test Execution

### Local Development
```bash
# All tests
flutter test

# Specific category
flutter test test/domain/
flutter test test/integration/

# With coverage
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html

# Golden tests
flutter test --update-goldens
```

### CI Pipeline
```yaml
test:
  steps:
    - name: Unit Tests
      run: flutter test test/domain/ test/application/
      
    - name: Widget Tests
      run: flutter test test/widgets/
      
    - name: Integration Tests
      run: flutter test test/integration/
      
    - name: Coverage Check
      run: |
        flutter test --coverage
        lcov --remove coverage/lcov.info 'lib/generated/*' -o coverage/lcov.info
        if [ $(lcov --summary coverage/lcov.info | grep lines | cut -d ' ' -f 4 | cut -d '%' -f 1) -lt 80 ]; then
          echo "Coverage below 80%"
          exit 1
        fi
```

## Test Environment

### Device Testing
- **Android**: API 21, 28, 33
- **iOS**: iOS 12, 14, 16
- **Tablets**: iPad, Android tablet

### Accessibility Testing
- Screen reader compatibility
- Large text support
- High contrast mode
- Reduced motion

## Regression Testing

### Critical Paths
1. Record → Transcribe → Parse → Display
2. Settings change → Service update
3. Model download → Load → Use
4. History save → Retrieve → Display

### Test Scenarios
```gherkin
Feature: Mental Math Calculation
  
  Scenario: Simple Addition
    Given the app is on the record screen
    When I say "five plus three"
    Then I should see "5 + 3"
    And the result should be "8"
  
  Scenario: Complex Expression
    Given the app is on the record screen
    When I say "twenty-five times four plus ten percent"
    Then I should see the steps
    And the final result should be "110"
```

## Bug Reporting Template

```markdown
### Description
Brief description of the issue

### Steps to Reproduce
1. Step one
2. Step two
3. ...

### Expected Behavior
What should happen

### Actual Behavior
What actually happens

### Environment
- Device: [e.g., iPhone 14]
- OS: [e.g., iOS 16.5]
- App Version: [e.g., 1.0.0]
- Model: [e.g., base.en]

### Logs
```
Relevant logs or errors
```

### Screenshots
If applicable
```

## Test Metrics

### KPIs
- Test execution time: < 5 minutes
- Flaky test rate: < 1%
- Bug escape rate: < 5%
- Test coverage trend: Increasing

### Reporting
- Daily: Test pass rate
- Weekly: Coverage report
- Sprint: Bug metrics
- Release: Quality gate