# Phase 3: Native Integration - Whisper.cpp Setup

## Goal
Integrate whisper.cpp for offline speech-to-text on both Android and iOS platforms using Flutter FFI.

## Time Estimate: 4-6 hours

## Prerequisites
- ✅ Phase 2 complete (audio capture working)
- ✅ Android NDK installed
- ✅ Xcode installed (for iOS)
- ✅ CMake 3.18+
- ✅ 2GB free space for models

## Step 1: Setup Native Directory (20 min)

### Create Structure
```bash
# In project root
mkdir -p native/whisper
cd native

# Clone whisper.cpp
git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp
git checkout v1.5.4  # Use stable version
```

### Project Structure
```
native/
├── whisper.cpp/          # Cloned repository
├── whisper_dart.c        # C wrapper for Dart
├── CMakeLists.txt        # Build configuration
└── models/               # Model files
    └── ggml-base.en.bin
```

## Step 2: Download Whisper Models (15 min)

### Download Script (scripts/download_models.sh)
```bash
#!/bin/bash

MODELS_DIR="assets/models"
mkdir -p $MODELS_DIR

# Download base.en model (recommended for mobile)
echo "Downloading base.en model (74MB)..."
curl -L "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin" \
     -o "$MODELS_DIR/ggml-base.en.bin"

# Optional: smaller model for testing
echo "Downloading tiny.en model (39MB)..."
curl -L "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-tiny.en.bin" \
     -o "$MODELS_DIR/ggml-tiny.en.bin"

echo "Models downloaded successfully!"
ls -lh $MODELS_DIR
```

### Run Download
```bash
chmod +x scripts/download_models.sh
./scripts/download_models.sh
```

### Update pubspec.yaml
```yaml
flutter:
  assets:
    - assets/models/
```

## Step 3: Create C Wrapper (30 min)

### native/whisper_dart.c
```c
#include "whisper.cpp/whisper.h"
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

#ifdef __cplusplus
extern "C" {
#endif

// Initialize whisper context
void* whisper_dart_init(const char* model_path) {
    struct whisper_context_params cparams = whisper_context_default_params();
    cparams.use_gpu = false;  // Disable GPU for mobile
    
    struct whisper_context* ctx = whisper_init_from_file_with_params(model_path, cparams);
    if (ctx == NULL) {
        fprintf(stderr, "Failed to initialize whisper from: %s\n", model_path);
    }
    return ctx;
}

// Transcribe audio
char* whisper_dart_transcribe(
    void* ctx,
    const float* pcm,
    int n_samples,
    const char* language
) {
    if (ctx == NULL || pcm == NULL) {
        return strdup("Error: Invalid context or audio");
    }
    
    struct whisper_full_params wparams = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);
    
    // Configure parameters for mobile
    wparams.n_threads = 4;  // Adjust based on device
    wparams.language = language ? language : "en";
    wparams.translate = false;
    wparams.no_context = true;
    wparams.single_segment = false;
    wparams.print_progress = false;
    wparams.print_realtime = false;
    
    // Process audio
    int result = whisper_full(ctx, wparams, pcm, n_samples);
    
    if (result != 0) {
        return strdup("Error: Transcription failed");
    }
    
    // Get segments and build result
    const int n_segments = whisper_full_n_segments(ctx);
    if (n_segments == 0) {
        return strdup("");
    }
    
    // Calculate total text length
    size_t total_len = 0;
    for (int i = 0; i < n_segments; i++) {
        const char* text = whisper_full_get_segment_text(ctx, i);
        total_len += strlen(text) + 1; // +1 for space
    }
    
    // Build complete transcript
    char* transcript = (char*)malloc(total_len + 1);
    transcript[0] = '\0';
    
    for (int i = 0; i < n_segments; i++) {
        const char* text = whisper_full_get_segment_text(ctx, i);
        strcat(transcript, text);
        if (i < n_segments - 1) {
            strcat(transcript, " ");
        }
    }
    
    return transcript;
}

// Get processing info
typedef struct {
    int64_t t_sample_us;
    int64_t t_encode_us;
    int64_t t_decode_us;
    int64_t t_total_us;
} whisper_timing_info;

whisper_timing_info whisper_dart_get_timing(void* ctx) {
    whisper_timing_info info = {0};
    if (ctx) {
        struct whisper_context* wctx = (struct whisper_context*)ctx;
        info.t_total_us = whisper_full_get_segment_t1(wctx, 0);
    }
    return info;
}

// Free context
void whisper_dart_free(void* ctx) {
    if (ctx) {
        whisper_free((struct whisper_context*)ctx);
    }
}

// Free string allocated by transcribe
void whisper_dart_free_string(char* str) {
    if (str) {
        free(str);
    }
}

#ifdef __cplusplus
}
#endif
```

## Step 4: Android Build Setup (45 min)

### android/app/build.gradle
```gradle
android {
    ...
    defaultConfig {
        ...
        externalNativeBuild {
            cmake {
                cppFlags ""
                arguments "-DANDROID_STL=c++_shared"
            }
        }
        ndk {
            abiFilters 'armeabi-v7a', 'arm64-v8a', 'x86_64'
        }
    }
    
    externalNativeBuild {
        cmake {
            path "../../native/CMakeLists.txt"
            version "3.18.1"
        }
    }
}
```

### native/CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.18.1)
project(whisper_dart)

set(CMAKE_CXX_STANDARD 11)

# Add whisper.cpp
add_subdirectory(whisper.cpp)

# Create shared library
add_library(whisper_dart SHARED
    whisper_dart.c
)

# Link with whisper
target_link_libraries(whisper_dart
    whisper
    ${CMAKE_DL_LIBS}
)

# Set properties
set_target_properties(whisper_dart PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/../android/app/src/main/jniLibs/${ANDROID_ABI}"
)
```

### Build for Android
```bash
# Build for all ABIs
cd android
./gradlew assembleDebug

# Or build manually for specific ABI
cd native
mkdir build-android && cd build-android

cmake .. \
  -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-21 \
  -DCMAKE_BUILD_TYPE=Release

make -j8
```

## Step 5: iOS Build Setup (45 min)

### ios/Runner.xcodeproj Configuration

1. Open in Xcode:
```bash
open ios/Runner.xcworkspace
```

2. Add Build Phase:
- Select Runner target
- Build Phases → + → New Run Script Phase
- Add script:

```bash
# Build whisper for iOS
cd "$SRCROOT/../../native"

# Build for device
xcodebuild -project whisper.cpp/whisper.cpp.xcodeproj \
  -scheme whisper \
  -configuration Release \
  -sdk iphoneos \
  -arch arm64 \
  ONLY_ACTIVE_ARCH=NO \
  BUILD_DIR="$SRCROOT/build"

# Copy library
cp "$SRCROOT/build/Release-iphoneos/libwhisper.a" "$SRCROOT/Frameworks/"
```

### ios/Runner/bridge.h
```objc
#ifndef bridge_h
#define bridge_h

#import <Foundation/Foundation.h>

@interface WhisperBridge : NSObject
+ (NSString *)libraryPath;
@end

#endif
```

### ios/Runner/bridge.m
```objc
#import "bridge.h"

@implementation WhisperBridge

+ (NSString *)libraryPath {
    return [[NSBundle mainBundle] pathForResource:@"whisper_dart" 
                                            ofType:@"framework"];
}

@end
```

## Step 6: Dart FFI Bindings (45 min)

### lib/infrastructure/stt/whisper_ffi.dart
```dart
import 'dart:ffi';
import 'dart:io';
import 'dart:typed_data';
import 'package:ffi/ffi.dart';
import 'package:flutter/services.dart';
import 'package:path_provider/path_provider.dart';

// Native function signatures
typedef WhisperInitNative = Pointer<Void> Function(Pointer<Utf8>);
typedef WhisperInit = Pointer<Void> Function(Pointer<Utf8>);

typedef WhisperTranscribeNative = Pointer<Utf8> Function(
  Pointer<Void>, Pointer<Float>, Int32, Pointer<Utf8>);
typedef WhisperTranscribe = Pointer<Utf8> Function(
  Pointer<Void>, Pointer<Float>, int, Pointer<Utf8>);

typedef WhisperFreeNative = Void Function(Pointer<Void>);
typedef WhisperFree = void Function(Pointer<Void>);

typedef WhisperFreeStringNative = Void Function(Pointer<Utf8>);
typedef WhisperFreeString = void Function(Pointer<Utf8>);

class WhisperFFI {
  late final DynamicLibrary _lib;
  late final WhisperInit _init;
  late final WhisperTranscribe _transcribe;
  late final WhisperFree _free;
  late final WhisperFreeString _freeString;
  
  Pointer<Void>? _context;
  
  WhisperFFI() {
    _lib = _loadLibrary();
    _init = _lib.lookupFunction<WhisperInitNative, WhisperInit>(
      'whisper_dart_init');
    _transcribe = _lib.lookupFunction<WhisperTranscribeNative, WhisperTranscribe>(
      'whisper_dart_transcribe');
    _free = _lib.lookupFunction<WhisperFreeNative, WhisperFree>(
      'whisper_dart_free');
    _freeString = _lib.lookupFunction<WhisperFreeStringNative, WhisperFreeString>(
      'whisper_dart_free_string');
  }
  
  static DynamicLibrary _loadLibrary() {
    if (Platform.isAndroid) {
      return DynamicLibrary.open('libwhisper_dart.so');
    } else if (Platform.isIOS) {
      return DynamicLibrary.process();
    } else {
      throw UnsupportedError('Platform not supported');
    }
  }
  
  Future<void> initialize(String modelName) async {
    // Copy model from assets to file system
    final modelPath = await _copyModelToFile(modelName);
    
    final pathPtr = modelPath.toNativeUtf8();
    _context = _init(pathPtr);
    malloc.free(pathPtr);
    
    if (_context == nullptr) {
      throw Exception('Failed to initialize Whisper model');
    }
  }
  
  Future<String> _copyModelToFile(String modelName) async {
    final dir = await getApplicationDocumentsDirectory();
    final file = File('${dir.path}/$modelName');
    
    if (!file.existsSync()) {
      final data = await rootBundle.load('assets/models/$modelName');
      await file.writeAsBytes(data.buffer.asUint8List());
    }
    
    return file.path;
  }
  
  String transcribe(Float32List audioData) {
    if (_context == null) {
      throw Exception('Whisper not initialized');
    }
    
    // Allocate memory for audio data
    final pcmPtr = malloc.allocate<Float>(audioData.length * sizeOf<Float>());
    final pcmList = pcmPtr.asTypedList(audioData.length);
    pcmList.setAll(0, audioData);
    
    // Set language
    final langPtr = 'en'.toNativeUtf8();
    
    // Call native transcribe
    final resultPtr = _transcribe(_context!, pcmPtr, audioData.length, langPtr);
    
    // Get result string
    final transcript = resultPtr.toDartString();
    
    // Clean up
    _freeString(resultPtr);
    malloc.free(pcmPtr);
    malloc.free(langPtr);
    
    return transcript;
  }
  
  void dispose() {
    if (_context != null) {
      _free(_context!);
      _context = null;
    }
  }
}
```

## Step 7: Integration Service (30 min)

### whisper_service.dart
```dart
import 'dart:typed_data';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class WhisperService {
  final WhisperFFI _whisper = WhisperFFI();
  bool _isInitialized = false;
  
  Future<void> initialize({String model = 'ggml-base.en.bin'}) async {
    if (_isInitialized) return;
    
    try {
      await _whisper.initialize(model);
      _isInitialized = true;
      print('Whisper initialized with model: $model');
    } catch (e) {
      print('Failed to initialize Whisper: $e');
      rethrow;
    }
  }
  
  Stream<String> transcribeStream(Stream<Uint8List> audioStream) async* {
    if (!_isInitialized) {
      await initialize();
    }
    
    // Buffer audio chunks
    final buffer = <int>[];
    const chunkSize = 16000 * 2; // 2 seconds of audio
    
    await for (final chunk in audioStream) {
      buffer.addAll(chunk);
      
      // Process when we have enough data
      if (buffer.length >= chunkSize) {
        final audioData = _convertToFloat32(Uint8List.fromList(buffer));
        final transcript = _whisper.transcribe(audioData);
        
        if (transcript.isNotEmpty) {
          yield transcript;
        }
        
        // Keep last 0.5 seconds for context
        buffer.removeRange(0, buffer.length - 8000);
      }
    }
    
    // Process remaining audio
    if (buffer.isNotEmpty) {
      final audioData = _convertToFloat32(Uint8List.fromList(buffer));
      final transcript = _whisper.transcribe(audioData);
      
      if (transcript.isNotEmpty) {
        yield transcript;
      }
    }
  }
  
  Float32List _convertToFloat32(Uint8List pcm16) {
    final samples = Int16List.view(pcm16.buffer);
    final float32 = Float32List(samples.length);
    
    for (int i = 0; i < samples.length; i++) {
      float32[i] = samples[i] / 32768.0;
    }
    
    return float32;
  }
  
  void dispose() {
    _whisper.dispose();
  }
}
```

### AI Prompt for Provider
```
"Create Riverpod provider for WhisperService:
- Initialize on first use
- Connect to audio stream from Phase 2
- Stream transcription updates
- Handle errors and retry
Show complete integration"
```

## Step 8: Test Integration (30 min)

### test_whisper.dart
```dart
void testWhisper() async {
  print('Testing Whisper integration...');
  
  // Initialize
  final whisper = WhisperService();
  await whisper.initialize(model: 'ggml-tiny.en.bin');
  
  // Load test audio
  final testAudio = await loadTestAudio();
  
  // Transcribe
  final result = whisper.transcribe(testAudio);
  print('Transcription: $result');
}

Future<Float32List> loadTestAudio() async {
  // Generate 3 seconds of silence (for testing)
  return Float32List(16000 * 3);
}
```

## Testing Checkpoints

### ✅ Checkpoint 1: Library Loads
```dart
// Should not crash
final lib = WhisperFFI();
print('Library loaded successfully');
```

### ✅ Checkpoint 2: Model Initializes
```dart
final whisper = WhisperService();
await whisper.initialize();
// Check logs for "Whisper initialized"
```

### ✅ Checkpoint 3: Transcription Works
```dart
// Record audio
// Should see transcription appear
```

### ✅ Checkpoint 4: Performance Check
```dart
// Measure RTF (Real Time Factor)
// Should be < 0.5 for base.en model
```

## Platform-Specific Debugging

### Android
```bash
# Check library is built
ls android/app/src/main/jniLibs/arm64-v8a/

# View logs
adb logcat | grep whisper

# Check symbols
nm -D android/app/src/main/jniLibs/arm64-v8a/libwhisper_dart.so
```

### iOS
```bash
# Check framework
ls ios/Frameworks/

# View device logs
xcrun simctl spawn booted log stream | grep whisper

# Test on simulator first
flutter run -d iPhone
```

## Common Issues & Solutions

### Issue: "Failed to load dynamic library"
**Solution**:
- Android: Check .so file in jniLibs
- iOS: Check library search paths in Xcode

### Issue: "Model file not found"
**Solution**:
```yaml
# Verify in pubspec.yaml
flutter:
  assets:
    - assets/models/
```

### Issue: Crash on initialization
**Solution**: Use smaller model (tiny.en) for testing

### Issue: Poor transcription quality
**Solution**: 
- Ensure 16kHz sample rate
- Check audio normalization
- Try base.en model instead of tiny

## Performance Optimization

1. **Model Selection**:
   - tiny.en: Fast but less accurate
   - base.en: Good balance
   - small.en: Best accuracy, slower

2. **Threading**:
   ```c
   wparams.n_threads = 4;  // Adjust based on device
   ```

3. **Chunking**:
   - Process 2-3 second chunks
   - Keep 0.5s overlap for context

4. **Memory**:
   - Load model once, reuse context
   - Free transcription strings immediately

## Success Criteria

✅ Whisper library builds for both platforms
✅ Model loads from assets
✅ Audio transcribes successfully
✅ RTF < 0.5 on target devices
✅ Memory usage < 150MB
✅ No crashes during extended use

## Next Phase

With Whisper integrated, move to [PHASE4_PARSER.md](PHASE4_PARSER.md) to implement math expression parsing.

## Quick Integration Test

```dart
// Add to record_screen.dart
final whisperService = ref.watch(whisperServiceProvider);
final audioStream = ref.watch(audioStreamProvider);

whisperService.transcribeStream(audioStream).listen((transcript) {
  print('Whisper says: $transcript');
  ref.read(transcriptProvider.notifier).update(transcript);
});
```

## AI Prompt for Issues

```
"Debug Whisper FFI integration:
Platform: [Android/iOS]
Error: [full error with stack trace]
Model: [tiny/base/small]
Code: [FFI binding code]
What's wrong and how to fix?"
```

Remember: Native integration is complex - test each step!