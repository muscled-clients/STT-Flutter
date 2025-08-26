# Phase 1: Foundation - Flutter Setup & Navigation

## Goal
Build the app skeleton with navigation, state management, and mock services. This phase gets you a working UI that you can test immediately.

## Time Estimate: 2-3 hours

## Step 1: Create Flutter Project (15 min)

### Command Sequence
```bash
# Create project with proper structure
flutter create . --project-name mental_math_voice --org com.yourdomain --platforms android,ios

# Verify setup
flutter doctor
flutter run
```

### Expected Output
A basic Flutter app running with counter demo.

## Step 2: Add Dependencies (10 min)

### Edit pubspec.yaml
```yaml
dependencies:
  flutter:
    sdk: flutter
  
  # State management
  flutter_riverpod: ^2.4.0
  riverpod_annotation: ^2.3.0
  
  # Navigation
  go_router: ^12.0.0
  
  # UI helpers
  google_fonts: ^6.0.0
  flutter_animate: ^4.2.0
  
  # Storage
  shared_preferences: ^2.2.0
  path_provider: ^2.1.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.0
  riverpod_generator: ^2.3.0
  flutter_lints: ^3.0.0
```

### Install
```bash
flutter pub get
```

## Step 3: Create Folder Structure (20 min)

### AI Prompt
```
"Create Flutter clean architecture folder structure:
lib/
- main.dart (app entry)
- presentation/ (UI layer)
  - screens/ (record, history, settings)
  - widgets/ (shared widgets)
  - providers/ (Riverpod state)
- application/ (use cases)
- domain/ (business logic)
- infrastructure/ (external services)

Show main.dart with router setup and basic navigation"
```

### Expected Structure
```
lib/
├── main.dart
├── presentation/
│   ├── screens/
│   │   ├── record/
│   │   │   └── record_screen.dart
│   │   ├── history/
│   │   │   └── history_screen.dart
│   │   └── settings/
│   │       └── settings_screen.dart
│   ├── widgets/
│   │   └── app_bottom_nav.dart
│   └── providers/
│       └── app_providers.dart
├── application/
│   └── services/
│       └── mock_stt_service.dart
├── domain/
│   ├── entities/
│   │   └── transcript.dart
│   └── repositories/
│       └── stt_repository.dart
└── infrastructure/
    └── mock/
        └── mock_stt_impl.dart
```

## Step 4: Implement Navigation (30 min)

### main.dart
```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'presentation/screens/record/record_screen.dart';
import 'presentation/screens/history/history_screen.dart';
import 'presentation/screens/settings/settings_screen.dart';

void main() {
  runApp(
    ProviderScope(
      child: MentalMathVoiceApp(),
    ),
  );
}

class MentalMathVoiceApp extends ConsumerWidget {
  MentalMathVoiceApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp.router(
      title: 'Mental Math Voice',
      theme: _buildTheme(),
      routerConfig: _router,
      debugShowCheckedModeBanner: false,
    );
  }

  ThemeData _buildTheme() {
    return ThemeData(
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(
        seedColor: Colors.deepPurple,
        brightness: Brightness.light,
      ),
    );
  }

  final GoRouter _router = GoRouter(
    initialLocation: '/',
    routes: [
      ShellRoute(
        builder: (context, state, child) => AppShell(child: child),
        routes: [
          GoRoute(
            path: '/',
            name: 'record',
            builder: (context, state) => RecordScreen(),
          ),
          GoRoute(
            path: '/history',
            name: 'history',
            builder: (context, state) => HistoryScreen(),
          ),
          GoRoute(
            path: '/settings',
            name: 'settings',
            builder: (context, state) => SettingsScreen(),
          ),
        ],
      ),
    ],
  );
}

class AppShell extends StatelessWidget {
  final Widget child;
  
  const AppShell({Key? key, required this.child}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: child,
      bottomNavigationBar: AppBottomNav(),
    );
  }
}
```

## Step 5: Create Basic Screens (30 min)

### AI Prompt for Each Screen
```
"Create RecordScreen for Flutter with:
- Centered microphone button (FAB style)
- State: idle, recording, processing
- Placeholder for transcript display
- Placeholder for result card
Use Riverpod for state management"
```

### Record Screen Example
```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class RecordScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isRecording = ref.watch(recordingStateProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: Text('Mental Math Voice'),
        centerTitle: true,
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Recording button
            GestureDetector(
              onTap: () => ref.read(recordingStateProvider.notifier).toggle(),
              child: AnimatedContainer(
                duration: Duration(milliseconds: 200),
                width: isRecording ? 100 : 80,
                height: isRecording ? 100 : 80,
                decoration: BoxDecoration(
                  shape: BoxShape.circle,
                  color: isRecording ? Colors.red : Colors.deepPurple,
                  boxShadow: [
                    BoxShadow(
                      color: isRecording 
                        ? Colors.red.withOpacity(0.5)
                        : Colors.deepPurple.withOpacity(0.3),
                      blurRadius: 20,
                      spreadRadius: 5,
                    ),
                  ],
                ),
                child: Icon(
                  isRecording ? Icons.stop : Icons.mic,
                  color: Colors.white,
                  size: 40,
                ),
              ),
            ),
            SizedBox(height: 40),
            Text(
              isRecording ? 'Recording...' : 'Tap to record',
              style: Theme.of(context).textTheme.titleMedium,
            ),
            SizedBox(height: 40),
            // Transcript placeholder
            Container(
              margin: EdgeInsets.symmetric(horizontal: 20),
              padding: EdgeInsets.all(20),
              decoration: BoxDecoration(
                color: Colors.grey.shade100,
                borderRadius: BorderRadius.circular(12),
              ),
              child: Text(
                ref.watch(transcriptProvider),
                style: Theme.of(context).textTheme.bodyLarge,
                textAlign: TextAlign.center,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

## Step 6: Setup Riverpod Providers (30 min)

### app_providers.dart
```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'app_providers.g.dart';

// Recording state
@riverpod
class RecordingState extends _$RecordingState {
  @override
  bool build() => false;
  
  void toggle() => state = !state;
  void start() => state = true;
  void stop() => state = false;
}

// Transcript stream
@riverpod
class Transcript extends _$Transcript {
  @override
  String build() => "Tap to start recording";
  
  void update(String text) => state = text;
  void clear() => state = "";
}

// Mock STT Service
@riverpod
Stream<String> sttStream(SttStreamRef ref) async* {
  final isRecording = ref.watch(recordingStateProvider);
  
  if (!isRecording) return;
  
  // Simulate progressive transcription
  await Future.delayed(Duration(seconds: 1));
  yield "twenty";
  await Future.delayed(Duration(milliseconds: 500));
  yield "twenty five";
  await Future.delayed(Duration(milliseconds: 500));
  yield "twenty five times";
  await Future.delayed(Duration(milliseconds: 500));
  yield "twenty five times four";
}
```

### Generate Providers
```bash
dart run build_runner build --delete-conflicting-outputs
```

## Step 7: Add Bottom Navigation (20 min)

### app_bottom_nav.dart
```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

class AppBottomNav extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final currentRoute = GoRouter.of(context).routerDelegate.currentConfiguration.uri.path;
    
    return NavigationBar(
      selectedIndex: _getSelectedIndex(currentRoute),
      onDestinationSelected: (index) => _onItemTapped(context, index),
      destinations: [
        NavigationDestination(
          icon: Icon(Icons.mic),
          label: 'Record',
        ),
        NavigationDestination(
          icon: Icon(Icons.history),
          label: 'History',
        ),
        NavigationDestination(
          icon: Icon(Icons.settings),
          label: 'Settings',
        ),
      ],
    );
  }
  
  int _getSelectedIndex(String route) {
    switch (route) {
      case '/': return 0;
      case '/history': return 1;
      case '/settings': return 2;
      default: return 0;
    }
  }
  
  void _onItemTapped(BuildContext context, int index) {
    switch (index) {
      case 0: context.go('/');
      case 1: context.go('/history');
      case 2: context.go('/settings');
    }
  }
}
```

## Step 8: Create Mock Services (20 min)

### mock_stt_service.dart
```dart
class MockSTTService {
  Stream<TranscriptUpdate> transcribe() async* {
    // Simulate real-time transcription
    final phrases = [
      "twenty",
      "twenty five", 
      "twenty five times",
      "twenty five times four",
      "twenty five times four equals",
      "twenty five times four equals one hundred"
    ];
    
    for (final phrase in phrases) {
      await Future.delayed(Duration(milliseconds: 600));
      yield TranscriptUpdate(
        text: phrase,
        isFinal: phrase == phrases.last,
      );
    }
  }
}
```

## Testing Checkpoints

### ✅ Checkpoint 1: Navigation Works
```bash
flutter run
# Verify you can tap between Record, History, Settings screens
```

### ✅ Checkpoint 2: Recording State
```bash
# Tap mic button
# Should change color and show "Recording..."
```

### ✅ Checkpoint 3: Mock Transcription
```bash
# Start recording
# Should see progressive text updates
```

## Common Issues & Solutions

### Issue: Provider not found
**Solution**: Ensure ProviderScope wraps MaterialApp

### Issue: Navigation not working
**Solution**: Check GoRouter paths match exactly

### Issue: Build runner fails
**Solution**: 
```bash
flutter clean
flutter pub get
dart run build_runner build --delete-conflicting-outputs
```

## Success Criteria

✅ App runs without errors
✅ Three screens with navigation
✅ Recording button changes state
✅ Mock transcription displays
✅ Clean architecture structure in place

## Next Phase

With foundation complete, move to [PHASE2_AUDIO.md](PHASE2_AUDIO.md) to implement real audio capture.

## Quick Win Tips

1. **Use hot reload** - Make changes and see instantly
2. **Start with UI** - Get visual feedback quickly  
3. **Mock everything** - Real implementation comes later
4. **Test on device** - Simulator is fine for now
5. **Commit often** - Save progress frequently

## AI Prompting for Issues

If stuck, use:
```
"I'm building Flutter mental math app, Phase 1 foundation.
Current issue: [specific problem]
Error: [error message]
Code: [relevant code]
How do I fix this?"
```

Remember: The goal is a working skeleton, not perfection!