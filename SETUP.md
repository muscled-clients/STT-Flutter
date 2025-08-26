# Setup Guide

## Prerequisites

### Common
- Flutter SDK ≥3.0.0
- Git
- 8GB+ RAM recommended

### Android
- Android Studio
- Android SDK (API 21+)
- NDK r25c or later
- CMake 3.18+

### iOS
- Xcode 14+
- iOS 12.0+ deployment target
- CocoaPods

## Environment Setup

### 1. Flutter Installation

```bash
# Verify Flutter installation
flutter doctor -v

# Should show:
# [✓] Flutter
# [✓] Android toolchain (for Android)
# [✓] Xcode (for iOS)
```

### 2. Native Toolchain Setup

#### Android NDK Setup

```bash
# In Android Studio:
# Tools → SDK Manager → SDK Tools → NDK (Side by side)

# Or via command line:
sdkmanager "ndk;25.2.9519653"
sdkmanager "cmake;3.22.1"

# Set environment variables
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/25.2.9519653
export PATH=$PATH:$ANDROID_NDK_HOME
```

#### iOS Setup

```bash
# Install Xcode command line tools
xcode-select --install

# Install CocoaPods
sudo gem install cocoapods

# For M1/M2 Macs
sudo gem install ffi
```

## Building Native Libraries

### Whisper.cpp Integration

#### 1. Clone whisper.cpp

```bash
# In project root
mkdir -p native
cd native
git clone https://github.com/ggml-org/whisper.cpp.git
cd whisper.cpp
git checkout v1.5.4  # Use stable release
```

#### 2. Android Build (.so files)

```bash
cd native/whisper.cpp

# Build for all Android ABIs
./scripts/build-android.sh

# Or manual build for specific ABI:
mkdir build-android && cd build-android
cmake .. \
  -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-21 \
  -DCMAKE_BUILD_TYPE=Release \
  -DWHISPER_BUILD_TESTS=OFF \
  -DWHISPER_BUILD_EXAMPLES=OFF

make -j8

# Copy to Flutter project
cp libwhisper.so ../../../android/app/src/main/jniLibs/arm64-v8a/
```

Repeat for other ABIs: `armeabi-v7a`, `x86_64`

#### 3. iOS Build (.a static library)

```bash
cd native/whisper.cpp

# Build for iOS
./scripts/build-ios.sh

# Or manual build:
mkdir build-ios && cd build-ios

# For device
cmake .. \
  -DCMAKE_SYSTEM_NAME=iOS \
  -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0 \
  -DCMAKE_BUILD_TYPE=Release \
  -DWHISPER_BUILD_TESTS=OFF \
  -DWHISPER_BUILD_EXAMPLES=OFF

make -j8

# For simulator (x86_64 + arm64)
cmake .. \
  -DCMAKE_SYSTEM_NAME=iOS \
  -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" \
  -DCMAKE_OSX_SYSROOT=iphonesimulator \
  -DCMAKE_BUILD_TYPE=Release

make -j8

# Create universal binary
lipo -create build-ios/libwhisper.a build-ios-sim/libwhisper.a \
  -output ../../../ios/native/whisper/libwhisper.a

# Copy headers
cp include/*.h ../../../ios/native/whisper/
```

## Model Download

### Download Whisper Models

```bash
# Create models directory
mkdir -p assets/models

# Download base.en model (default)
cd assets/models
wget https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin

# Optional: Download other models
wget https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-tiny.en.bin
wget https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-small.en.bin
```

Model sizes:
- tiny.en: 39 MB
- base.en: 74 MB (recommended)
- small.en: 150 MB

## FFI Bridge Setup

### 1. Create C Wrapper

Create `native/whisper_dart.c`:

```c
#include "whisper.h"
#include <string.h>

void* whisper_dart_init(const char* model_path, int32_t threads) {
    struct whisper_context_params cparams = whisper_context_default_params();
    cparams.n_threads = threads;
    return whisper_init_from_file_with_params(model_path, cparams);
}

int whisper_dart_transcribe(void* ctx, const float* pcm, int32_t frames, 
                            char* out_text, int32_t out_cap) {
    // Implementation details...
    return 0; // success
}

void whisper_dart_free(void* ctx) {
    whisper_free((struct whisper_context*)ctx);
}
```

### 2. Update CMakeLists.txt

Add to `native/whisper.cpp/CMakeLists.txt`:

```cmake
# Dart FFI wrapper
add_library(whisper_dart SHARED
    ../whisper_dart.c
)
target_link_libraries(whisper_dart whisper)
```

### 3. Dart FFI Bindings

See `lib/infrastructure/stt/whisper_ffi.dart` for implementation.

## Flutter Project Setup

```bash
# Install dependencies
flutter pub get

# Generate code (if using build_runner)
flutter pub run build_runner build --delete-conflicting-outputs

# iOS specific
cd ios && pod install && cd ..

# Verify setup
flutter doctor
flutter test
```

## Running the App

### Development

```bash
# Run with hot reload
flutter run

# Specific device
flutter run -d <device_id>

# Release mode (faster)
flutter run --release
```

### Building

```bash
# Android APK
flutter build apk --release

# Android App Bundle
flutter build appbundle --release

# iOS
flutter build ios --release
# Then archive in Xcode
```

## Troubleshooting

### Android Issues

**NDK not found:**
```bash
# Add to android/local.properties
ndk.dir=/path/to/android-ndk-r25c
```

**UnsatisfiedLinkError:**
- Verify .so files in correct jniLibs folders
- Check ABI compatibility
- Ensure CMake build succeeded

### iOS Issues

**Library not found:**
- Check libwhisper.a in ios/native/whisper/
- Verify "Library Search Paths" in Xcode
- Clean build: `flutter clean && cd ios && pod deintegrate && pod install`

**Bitcode errors:**
- Disable bitcode in Xcode project settings
- Or rebuild whisper with `-fembed-bitcode`

### Model Loading

**Model not found:**
- Check assets declaration in pubspec.yaml
- Verify model file exists in assets/models/
- Use correct path with `rootBundle.load()`

### Performance

**Slow inference:**
- Use smaller model (tiny.en)
- Reduce number of threads
- Enable GPU acceleration (Metal on iOS)
- Profile with Flutter DevTools

## CI/CD Setup

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test
```

### Pre-commit Hooks

```bash
# Install pre-commit
pip install pre-commit

# Setup hooks
pre-commit install

# .pre-commit-config.yaml configured
```