# Phase 2: Audio Pipeline - Capture & Processing

## Goal
Implement real audio capture with visualization, Voice Activity Detection (VAD), and prepare audio buffers for STT processing.

## Time Estimate: 3-4 hours

## Prerequisites
- ✅ Phase 1 complete (navigation & mocks working)
- ✅ Device or emulator with microphone
- ✅ Testing headphones (to avoid feedback)

## Step 1: Add Audio Dependencies (10 min)

### Update pubspec.yaml
```yaml
dependencies:
  # Audio capture
  record: ^5.0.0
  permission_handler: ^11.0.0
  
  # Audio processing  
  flutter_sound: ^9.2.0
  
  # Visualization
  audio_waveforms: ^1.0.0
  
  # Utilities
  rxdart: ^0.27.0
```

### Install
```bash
flutter pub get
```

## Step 2: Platform Configuration (20 min)

### Android (android/app/src/main/AndroidManifest.xml)
```xml
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

### iOS (ios/Runner/Info.plist)
```xml
<key>NSMicrophoneUsageDescription</key>
<string>This app needs microphone access to convert speech to text</string>
```

### AI Prompt
```
"Setup Flutter audio permissions for Android and iOS:
- Handle permission requests
- Show rationale dialog if denied
- Navigate to settings if permanently denied
Use permission_handler package"
```

## Step 3: Create Audio Service (45 min)

### audio_capture_service.dart
```dart
import 'dart:typed_data';
import 'package:record/record.dart';
import 'package:rxdart/rxdart.dart';

class AudioCaptureService {
  final AudioRecorder _recorder = AudioRecorder();
  final _audioStreamController = BehaviorSubject<Uint8List>();
  
  // Configuration
  static const sampleRate = 16000;  // 16kHz for whisper
  static const channels = 1;        // Mono
  static const bitRate = 16;        // 16-bit PCM
  
  Stream<Uint8List> get audioStream => _audioStreamController.stream;
  
  Future<bool> checkPermission() async {
    return await _recorder.hasPermission();
  }
  
  Future<void> startRecording() async {
    try {
      // Check permission
      if (!await checkPermission()) {
        throw Exception('Microphone permission denied');
      }
      
      // Configure recording
      final config = RecordConfig(
        encoder: AudioEncoder.pcm16bits,
        sampleRate: sampleRate,
        numChannels: channels,
      );
      
      // Start streaming
      final stream = await _recorder.startStream(config);
      
      // Convert and emit audio chunks
      stream.listen((chunk) {
        _audioStreamController.add(chunk);
      });
      
    } catch (e) {
      print('Error starting recording: $e');
      rethrow;
    }
  }
  
  Future<void> stopRecording() async {
    await _recorder.stop();
  }
  
  void dispose() {
    _recorder.dispose();
    _audioStreamController.close();
  }
}
```

### AI Prompt for Provider
```
"Create Riverpod provider for AudioCaptureService:
- Handle lifecycle (auto-dispose)
- Expose recording state
- Stream audio buffers
- Handle errors gracefully
Show complete provider with usage example"
```

## Step 4: Implement VAD (Voice Activity Detection) (45 min)

### vad_processor.dart
```dart
import 'dart:typed_data';
import 'dart:math' as math;

class VADProcessor {
  // VAD parameters
  static const energyThreshold = 0.01;  // RMS energy threshold
  static const silenceDuration = 500;   // ms of silence to trigger end
  
  int _silenceFrames = 0;
  final int _silenceThreshold;
  
  VADProcessor({int sampleRate = 16000}) 
    : _silenceThreshold = (silenceDuration * sampleRate / 1000).round();
  
  /// Process audio chunk and detect speech
  SpeechStatus processChunk(Uint8List audioData) {
    final rms = _calculateRMS(audioData);
    
    if (rms > energyThreshold) {
      _silenceFrames = 0;
      return SpeechStatus.speaking;
    } else {
      _silenceFrames += audioData.length ~/ 2; // 16-bit samples
      
      if (_silenceFrames > _silenceThreshold) {
        return SpeechStatus.endOfSpeech;
      }
      return SpeechStatus.silence;
    }
  }
  
  /// Calculate Root Mean Square energy
  double _calculateRMS(Uint8List audioData) {
    if (audioData.isEmpty) return 0.0;
    
    // Convert bytes to 16-bit PCM samples
    final samples = Int16List.view(audioData.buffer);
    
    double sum = 0;
    for (final sample in samples) {
      sum += sample * sample;
    }
    
    return math.sqrt(sum / samples.length) / 32768.0; // Normalize to 0-1
  }
  
  void reset() {
    _silenceFrames = 0;
  }
}

enum SpeechStatus {
  silence,
  speaking, 
  endOfSpeech,
}
```

### AI Prompt for Integration
```
"Integrate VAD with audio capture:
- Process audio chunks in real-time
- Auto-stop recording on end-of-speech
- Emit speech segments for STT
- Show visual feedback for speech detection
Include state management with Riverpod"
```

## Step 5: Create Audio Visualizer (40 min)

### audio_visualizer.dart
```dart
import 'package:flutter/material.dart';
import 'dart:typed_data';
import 'dart:math' as math;

class AudioVisualizer extends StatefulWidget {
  final Stream<Uint8List> audioStream;
  final bool isRecording;
  
  const AudioVisualizer({
    Key? key,
    required this.audioStream,
    required this.isRecording,
  }) : super(key: key);
  
  @override
  _AudioVisualizerState createState() => _AudioVisualizerState();
}

class _AudioVisualizerState extends State<AudioVisualizer>
    with SingleTickerProviderStateMixin {
  
  List<double> waveform = List.filled(50, 0.0);
  late AnimationController _animationController;
  
  @override
  void initState() {
    super.initState();
    _animationController = AnimationController(
      vsync: this,
      duration: Duration(milliseconds: 100),
    );
    
    widget.audioStream.listen((audioData) {
      if (widget.isRecording) {
        _updateWaveform(audioData);
      }
    });
  }
  
  void _updateWaveform(Uint8List audioData) {
    // Calculate amplitude
    final samples = Int16List.view(audioData.buffer);
    double maxAmplitude = 0;
    
    for (final sample in samples) {
      maxAmplitude = math.max(maxAmplitude, sample.abs().toDouble());
    }
    
    setState(() {
      waveform.removeAt(0);
      waveform.add(maxAmplitude / 32768.0); // Normalize
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Container(
      height: 100,
      padding: EdgeInsets.symmetric(horizontal: 20),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        crossAxisAlignment: CrossAxisAlignment.center,
        children: List.generate(waveform.length, (index) {
          final height = waveform[index] * 80 + 5;
          
          return AnimatedContainer(
            duration: Duration(milliseconds: 100),
            width: 4,
            height: widget.isRecording ? height : 5,
            decoration: BoxDecoration(
              color: widget.isRecording 
                ? Colors.deepPurple.withOpacity(0.8)
                : Colors.grey.withOpacity(0.3),
              borderRadius: BorderRadius.circular(2),
            ),
          );
        }),
      ),
    );
  }
  
  @override
  void dispose() {
    _animationController.dispose();
    super.dispose();
  }
}
```

## Step 6: Update Recording Screen (30 min)

### Updated record_screen.dart
```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class RecordScreen extends ConsumerStatefulWidget {
  @override
  _RecordScreenState createState() => _RecordScreenState();
}

class _RecordScreenState extends ConsumerState<RecordScreen> {
  @override
  Widget build(BuildContext context) {
    final audioState = ref.watch(audioStateProvider);
    final transcript = ref.watch(transcriptProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: Text('Mental Math Voice'),
        actions: [
          // VAD indicator
          Padding(
            padding: EdgeInsets.only(right: 16),
            child: Icon(
              Icons.graphic_eq,
              color: audioState.isSpeaking 
                ? Colors.green 
                : Colors.grey,
            ),
          ),
        ],
      ),
      body: Column(
        children: [
          Spacer(),
          
          // Audio visualizer
          AudioVisualizer(
            audioStream: ref.watch(audioStreamProvider),
            isRecording: audioState.isRecording,
          ),
          
          SizedBox(height: 40),
          
          // Recording button with animation
          _buildRecordButton(audioState),
          
          SizedBox(height: 20),
          
          // Status text
          Text(
            _getStatusText(audioState),
            style: Theme.of(context).textTheme.titleMedium,
          ),
          
          SizedBox(height: 40),
          
          // Transcript card
          _buildTranscriptCard(transcript),
          
          Spacer(),
        ],
      ),
    );
  }
  
  Widget _buildRecordButton(AudioState state) {
    return GestureDetector(
      onTapDown: (_) => _startRecording(),
      onTapUp: (_) => _stopRecording(),
      child: AnimatedContainer(
        duration: Duration(milliseconds: 200),
        width: state.isRecording ? 100 : 80,
        height: state.isRecording ? 100 : 80,
        decoration: BoxDecoration(
          shape: BoxShape.circle,
          color: state.isRecording ? Colors.red : Colors.deepPurple,
          boxShadow: [
            BoxShadow(
              color: state.isRecording 
                ? Colors.red.withOpacity(0.5)
                : Colors.deepPurple.withOpacity(0.3),
              blurRadius: state.isRecording ? 30 : 20,
              spreadRadius: state.isRecording ? 10 : 5,
            ),
          ],
        ),
        child: Icon(
          state.isRecording ? Icons.stop : Icons.mic,
          color: Colors.white,
          size: 40,
        ),
      ),
    );
  }
  
  String _getStatusText(AudioState state) {
    if (state.isRecording) {
      return state.isSpeaking ? "Listening..." : "Speak now...";
    }
    return "Hold to record";
  }
  
  Widget _buildTranscriptCard(String transcript) {
    if (transcript.isEmpty) return SizedBox.shrink();
    
    return Container(
      margin: EdgeInsets.symmetric(horizontal: 20),
      padding: EdgeInsets.all(20),
      decoration: BoxDecoration(
        color: Colors.white,
        borderRadius: BorderRadius.circular(16),
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.1),
            blurRadius: 10,
            offset: Offset(0, 5),
          ),
        ],
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            'Transcript',
            style: Theme.of(context).textTheme.labelMedium,
          ),
          SizedBox(height: 8),
          Text(
            transcript,
            style: Theme.of(context).textTheme.bodyLarge,
          ),
        ],
      ),
    );
  }
  
  void _startRecording() async {
    try {
      await ref.read(audioServiceProvider).startRecording();
    } catch (e) {
      _showError(e.toString());
    }
  }
  
  void _stopRecording() async {
    await ref.read(audioServiceProvider).stopRecording();
  }
  
  void _showError(String message) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(message)),
    );
  }
}
```

## Step 7: Wire Everything Together (30 min)

### audio_providers.dart
```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'audio_providers.g.dart';

@riverpod
class AudioService extends _$AudioService {
  late final AudioCaptureService _captureService;
  late final VADProcessor _vadProcessor;
  
  @override
  AudioState build() {
    _captureService = AudioCaptureService();
    _vadProcessor = VADProcessor();
    
    // Listen to audio stream and process with VAD
    _captureService.audioStream.listen((chunk) {
      final status = _vadProcessor.processChunk(chunk);
      state = state.copyWith(
        isSpeaking: status == SpeechStatus.speaking,
      );
      
      if (status == SpeechStatus.endOfSpeech) {
        // Auto-stop on silence
        stopRecording();
      }
    });
    
    return AudioState.initial();
  }
  
  Future<void> startRecording() async {
    state = state.copyWith(isRecording: true);
    await _captureService.startRecording();
  }
  
  Future<void> stopRecording() async {
    state = state.copyWith(isRecording: false, isSpeaking: false);
    await _captureService.stopRecording();
    _vadProcessor.reset();
  }
}

@freezed
class AudioState with _$AudioState {
  const factory AudioState({
    required bool isRecording,
    required bool isSpeaking,
    String? error,
  }) = _AudioState;
  
  factory AudioState.initial() => AudioState(
    isRecording: false,
    isSpeaking: false,
  );
}
```

## Testing Checkpoints

### ✅ Checkpoint 1: Permissions
```dart
// Test permission request appears
// Grant permission
// Verify no crashes
```

### ✅ Checkpoint 2: Audio Capture
```dart
// Hold record button
// Speak into mic
// Release button
// Check console for audio data
```

### ✅ Checkpoint 3: VAD Working
```dart
// Start recording
// Speak, then pause
// Should auto-stop after 500ms silence
```

### ✅ Checkpoint 4: Visualization
```dart
// Record and speak
// Waveform bars should move
// Louder = taller bars
```

## Debugging Commands

### Check Permissions
```dart
final hasPermission = await Permission.microphone.status;
print('Mic permission: $hasPermission');
```

### Test Audio Data
```dart
audioStream.listen((chunk) {
  print('Audio chunk size: ${chunk.length}');
  print('First 10 bytes: ${chunk.take(10).toList()}');
});
```

### Monitor VAD
```dart
print('RMS: $rms, Speaking: $isSpeaking');
```

## Common Issues & Solutions

### Issue: Permission denied on iOS
**Solution**: 
```bash
cd ios && pod install
flutter clean && flutter run
```

### Issue: No audio data
**Solution**: Check recorder configuration matches:
- Sample rate: 16000
- Channels: 1
- Encoding: PCM 16-bit

### Issue: VAD too sensitive
**Solution**: Adjust energyThreshold (try 0.02 or 0.005)

### Issue: Visualization laggy
**Solution**: Reduce waveform array size or update frequency

## Performance Tips

1. **Buffer Management**: Pool audio buffers to reduce GC
2. **Processing**: Run VAD in separate isolate for smooth UI
3. **Visualization**: Throttle updates to 10-15 fps
4. **Memory**: Clear old audio data after processing

## Success Criteria

✅ Microphone permission handled properly
✅ Audio captures at 16kHz mono
✅ VAD detects speech vs silence
✅ Auto-stops on extended silence
✅ Waveform visualization working
✅ No memory leaks during recording

## Next Phase

With audio pipeline complete, move to [PHASE3_NATIVE.md](PHASE3_NATIVE.md) to integrate whisper.cpp.

## Quick Test Script

```dart
// Quick test in main.dart
void testAudio() async {
  final service = AudioCaptureService();
  await service.startRecording();
  
  service.audioStream.listen((chunk) {
    print('Got ${chunk.length} bytes');
  });
  
  await Future.delayed(Duration(seconds: 5));
  await service.stopRecording();
}
```

## AI Prompt for Issues

```
"Debug Flutter audio capture issue:
Platform: [Android/iOS]
Error: [full error message]
Code: [relevant code]
Expected: 16kHz mono PCM audio stream
Getting: [actual behavior]
Fix?"
```

Remember: Audio must work perfectly before adding STT!