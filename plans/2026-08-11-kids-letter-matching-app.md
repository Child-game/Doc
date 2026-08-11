# Kids Letter-Matching App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an offline Flutter app for children aged 3-5 with one game type — tap the letter that matches the target — driven by a JSON content bundle that a Go backend can serve later without app changes.

**Architecture:** Content lives in `assets/content/bundle.json` and is parsed into plain Dart models. A single `loadBundle()` function prefers a downloaded bundle file over the shipped asset, which is the entire seam for a future backend. Game state is a pure Dart class tested independently of the UI; the widget renders it. No state-management library, no code generation, no network.

**Tech Stack:** Flutter (stable), Dart. Packages: `flutter_tts`, `audioplayers`, `shared_preferences`, `path_provider`. No others.

## Global Constraints

- **Spec:** `docs/superpowers/specs/2026-08-11-kids-letter-matching-app-design.md`. Read it before Task 1.
- **Prerequisite:** Flutter is not installed on this machine. Install the stable channel and confirm `flutter doctor` before Task 1.
- **Platform:** iOS is the priority target (min iOS 13), Android supported (minSdk 21). iOS builds require macOS/Xcode — develop and test on Android emulator or `flutter run -d linux`; the iOS build is done on a Mac or macOS CI runner.
- **No network calls anywhere in v1.** No analytics, no ads, no third-party SDKs beyond the four packages listed.
- **No new dependencies** beyond the four listed. If a task seems to need a fifth, stop and ask.
- **No code generation.** No `build_runner`, no `json_serializable`. `fromJson` is hand-written.
- **No state-management library.** No Riverpod, no Bloc, no Provider.
- **No timers, no scores, no lives, no fail states** anywhere in the UI.
- **Minimum tap target 120pt square.**
- **All text shown to a child is rendered in the bundled Andika font** (single-story `a` and `g`).
- Run `flutter analyze` before every commit; it must be clean.

---

## File Structure

| File | Responsibility |
|---|---|
| `lib/bundle.dart` | Data models (`Bundle`, `Game`, `Option`) and their `fromJson` parsers. No I/O. |
| `lib/content.dart` | Loading the bundle from disk/assets, filtering to playable games, reading/writing progress. |
| `lib/audio.dart` | Audio session setup, TTS prompts, SFX playback. |
| `lib/games/find_same_state.dart` | Pure game state machine. No Flutter imports. |
| `lib/games/find_same.dart` | The game screen widget. |
| `lib/game_host.dart` | Game-type registry, level sequencing, progress marking. |
| `lib/main.dart` | App entry, theme, font, audio init. |
| `assets/content/bundle.json` | The content. |
| `test/bundle_test.dart` | Model parsing + content validation. |
| `test/find_same_state_test.dart` | Game state machine. |
| `test/game_host_test.dart` | Sequencing and type filtering. |

---

### Task 1: Project scaffold and dependencies

**Files:**
- Create: `pubspec.yaml` (via `flutter create`), `lib/main.dart`
- Create: `assets/fonts/Andika-Regular.ttf`
- Create: `assets/content/sfx/yay.wav`, `assets/content/sfx/oops.wav`

**Interfaces:**
- Consumes: nothing.
- Produces: a runnable Flutter app; asset paths `content/...` and font family `Andika` available to all later tasks.

- [ ] **Step 1: Verify Flutter is installed**

Run: `flutter doctor`
Expected: Flutter stable listed with no blocking errors for Android/Linux. iOS toolchain errors are expected on Linux and can be ignored.

- [ ] **Step 2: Create the project in place**

The repo root `/home/ved/Desktop/Games` already contains `docs/` and `.git`. Create the Flutter project into it:

```bash
cd /home/ved/Desktop/Games
flutter create --org com.example --project-name kidsletters --platforms=ios,android,web .
```

`web` is included because Chrome is the only working visual target on this
machine — the Linux desktop toolchain (clang/cmake/ninja/pkg-config) and the
Android SDK cmdline-tools are both absent. Scaffolding `ios/` and `android/`
works fine on Linux; only *building* them needs their toolchains.

- [ ] **Step 3: Add the four dependencies**

```bash
flutter pub add flutter_tts audioplayers shared_preferences path_provider
```

Do not pin versions by hand — `pub add` resolves current ones.

- [ ] **Step 4: Download the Andika font**

Andika is a SIL font designed for beginning readers; its `a` and `g` are single-story, matching how children are taught to write. Licensed OFL.

```bash
mkdir -p assets/fonts assets/content/sfx
curl -L -o assets/fonts/Andika-Regular.ttf \
  "https://github.com/google/fonts/raw/main/ofl/andika/Andika-Regular.ttf"
```

Verify it is a real font file, not an HTML error page:

Run: `file assets/fonts/Andika-Regular.ttf`
Expected: output mentions `TrueType` or `OpenType`. If it says `HTML`, the download failed — fetch Andika manually from fonts.google.com and place it at that path.

- [ ] **Step 5: Verify the sound effects are present**

The two sounds already exist — they are generated by `tools/gen_sfx.py` (stdlib
`wave` module, no ffmpeg and no downloads), and both the generator and the
resulting `.wav` files are already committed:

- `assets/content/sfx/yay.wav` — a rising major arpeggio, ~0.7s.
- `assets/content/sfx/oops.wav` — two soft descending notes, ~0.4s. Deliberately
  not a buzzer; a wrong tap must never feel like punishment to a 3 year old.

Run: `python3 tools/gen_sfx.py && file assets/content/sfx/*.wav`
Expected: both regenerate and report `RIFF ... WAVE audio, Microsoft PCM, 16 bit, mono 44100 Hz`.

Do not replace these with silent placeholders. If a real child finds either
sound harsh, tune the note lists in `tools/gen_sfx.py` and re-run it.

- [ ] **Step 6: Register assets and font in pubspec.yaml**

In `pubspec.yaml`, under the existing `flutter:` key, replace the commented-out `assets:`/`fonts:` sections with:

```yaml
flutter:
  uses-material-design: true

  assets:
    - assets/content/
    - assets/content/sfx/

  fonts:
    - family: Andika
      fonts:
        - asset: assets/fonts/Andika-Regular.ttf
```

- [ ] **Step 7: Replace lib/main.dart with a minimal shell**

```dart
import 'package:flutter/material.dart';

void main() => runApp(const KidsLettersApp());

class KidsLettersApp extends StatelessWidget {
  const KidsLettersApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Letters',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        fontFamily: 'Andika',
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF4C6EF5)),
        useMaterial3: true,
      ),
      home: const Scaffold(body: Center(child: Text('Letters', style: TextStyle(fontSize: 48)))),
    );
  }
}
```

- [ ] **Step 8: Verify it builds**

Interactive `flutter run` needs a terminal this agent does not have. Verify with a
headless build instead:

Run: `flutter build web --debug`
Expected: `✓ Built build/web`

Do not attempt `flutter run` — it will hang waiting for input. The visual and
audio check happens once, at Task 9 Step 5, driven by the human.

- [ ] **Step 9: Verify analysis is clean**

Run: `flutter analyze`
Expected: `No issues found!`

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "chore: scaffold Flutter app with Andika font and audio assets"
```

---

### Task 2: Bundle models and JSON parsing

**Files:**
- Create: `lib/bundle.dart`
- Test: `test/bundle_test.dart`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `class Bundle { int bundleVersion; String assetBase; Map<String,String> sfx; List<Game> games; Bundle.fromJson(Map<String,dynamic>) }`
  - `class Game { String id; String type; int level; String promptText; String? promptAudio; String targetText; List<Option> options; Game.fromJson(Map<String,dynamic>); Set<String> get correctIds }`
  - `class Option { String id; String text; bool correct; Option.fromJson(Map<String,dynamic>) }`

- [ ] **Step 1: Write the failing test**

Create `test/bundle_test.dart`:

```dart
import 'dart:convert';
import 'package:flutter_test/flutter_test.dart';
import 'package:kidsletters/bundle.dart';

const _sample = '''
{
  "bundle_version": 1,
  "asset_base": "assets/content/",
  "sfx": { "correct": "sfx/yay.wav", "wrong": "sfx/oops.wav" },
  "games": [
    {
      "id": "L1_A",
      "type": "find_same",
      "level": 1,
      "prompt_text": "Find the letter A",
      "target": { "text": "A" },
      "options": [
        { "id": "1", "text": "O" },
        { "id": "2", "text": "A", "correct": true },
        { "id": "3", "text": "X" }
      ]
    }
  ]
}
''';

void main() {
  test('parses a bundle', () {
    final b = Bundle.fromJson(jsonDecode(_sample) as Map<String, dynamic>);

    expect(b.bundleVersion, 1);
    expect(b.assetBase, 'assets/content/');
    expect(b.sfx['correct'], 'sfx/yay.wav');
    expect(b.games, hasLength(1));
  });

  test('parses a game and its options', () {
    final b = Bundle.fromJson(jsonDecode(_sample) as Map<String, dynamic>);
    final g = b.games.single;

    expect(g.id, 'L1_A');
    expect(g.type, 'find_same');
    expect(g.level, 1);
    expect(g.promptText, 'Find the letter A');
    expect(g.promptAudio, isNull);
    expect(g.targetText, 'A');
    expect(g.options.map((o) => o.text), ['O', 'A', 'X']);
  });

  test('correct defaults to false and correctIds collects the true ones', () {
    final g = Bundle.fromJson(jsonDecode(_sample) as Map<String, dynamic>).games.single;

    expect(g.options[0].correct, isFalse);
    expect(g.options[1].correct, isTrue);
    expect(g.correctIds, {'2'});
  });

  test('prompt_audio is read when present', () {
    final json = jsonDecode(_sample) as Map<String, dynamic>;
    (json['games'] as List)[0]['prompt_audio'] = 'audio/a.mp3';

    final g = Bundle.fromJson(json).games.single;

    expect(g.promptAudio, 'audio/a.mp3');
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `flutter test test/bundle_test.dart`
Expected: FAIL — `Target of URI doesn't exist: 'package:kidsletters/bundle.dart'`

- [ ] **Step 3: Write the models**

Create `lib/bundle.dart`:

```dart
/// Data models for the content bundle. Pure Dart — no I/O, no Flutter imports.

class Bundle {
  const Bundle({
    required this.bundleVersion,
    required this.assetBase,
    required this.sfx,
    required this.games,
  });

  final int bundleVersion;
  final String assetBase;
  final Map<String, String> sfx;
  final List<Game> games;

  factory Bundle.fromJson(Map<String, dynamic> json) => Bundle(
        bundleVersion: json['bundle_version'] as int,
        assetBase: json['asset_base'] as String,
        sfx: Map<String, String>.from(json['sfx'] as Map),
        games: (json['games'] as List)
            .map((g) => Game.fromJson(g as Map<String, dynamic>))
            .toList(growable: false),
      );
}

class Game {
  const Game({
    required this.id,
    required this.type,
    required this.level,
    required this.promptText,
    required this.promptAudio,
    required this.targetText,
    required this.options,
  });

  final String id;
  final String type;
  final int level;
  final String promptText;

  /// Optional recorded prompt. When null the app speaks [promptText] via TTS,
  /// which is why v1 ships no voice assets.
  final String? promptAudio;

  final String targetText;
  final List<Option> options;

  factory Game.fromJson(Map<String, dynamic> json) => Game(
        id: json['id'] as String,
        type: json['type'] as String,
        level: json['level'] as int,
        promptText: json['prompt_text'] as String,
        promptAudio: json['prompt_audio'] as String?,
        targetText: (json['target'] as Map)['text'] as String,
        options: (json['options'] as List)
            .map((o) => Option.fromJson(o as Map<String, dynamic>))
            .toList(growable: false),
      );

  Set<String> get correctIds =>
      options.where((o) => o.correct).map((o) => o.id).toSet();
}

class Option {
  const Option({required this.id, required this.text, required this.correct});

  final String id;
  final String text;
  final bool correct;

  factory Option.fromJson(Map<String, dynamic> json) => Option(
        id: json['id'] as String,
        text: json['text'] as String,
        correct: json['correct'] as bool? ?? false,
      );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `flutter test test/bundle_test.dart`
Expected: `+4: All tests passed!`

- [ ] **Step 5: Analyze and commit**

```bash
flutter analyze
git add lib/bundle.dart test/bundle_test.dart
git commit -m "feat: content bundle models and JSON parsing"
```

---

### Task 3: Author the content bundle and validate it

**Files:**
- Create: `assets/content/bundle.json`
- Modify: `test/bundle_test.dart` (append a validation group)

**Interfaces:**
- Consumes: `Bundle.fromJson` from Task 2.
- Produces: `assets/content/bundle.json` containing 15 `find_same` games, levels 1-15.

- [ ] **Step 1: Write the failing validation test**

Append to `test/bundle_test.dart` (add `import 'dart:io';` at the top):

```dart
  group('shipped bundle', () {
    late Bundle bundle;

    setUpAll(() {
      final raw = File('assets/content/bundle.json').readAsStringSync();
      bundle = Bundle.fromJson(jsonDecode(raw) as Map<String, dynamic>);
    });

    test('every game has at least one correct option', () {
      // An unwinnable level is the worst bug in this app: a child taps forever.
      for (final g in bundle.games) {
        expect(g.correctIds, isNotEmpty, reason: '${g.id} has no correct option');
      }
    });

    test('every correct option matches the target letter', () {
      for (final g in bundle.games) {
        for (final o in g.options.where((o) => o.correct)) {
          expect(o.text, g.targetText, reason: '${g.id} option ${o.id}');
        }
      }
    });

    test('no incorrect option matches the target letter', () {
      for (final g in bundle.games) {
        for (final o in g.options.where((o) => !o.correct)) {
          expect(o.text, isNot(g.targetText), reason: '${g.id} option ${o.id}');
        }
      }
    });

    test('option ids are unique within a game', () {
      for (final g in bundle.games) {
        expect(g.options.map((o) => o.id).toSet(), hasLength(g.options.length),
            reason: '${g.id} has duplicate option ids');
      }
    });

    test('game ids are unique and levels are contiguous from 1', () {
      expect(bundle.games.map((g) => g.id).toSet(), hasLength(bundle.games.length));

      final levels = bundle.games.map((g) => g.level).toList()..sort();
      expect(levels, List.generate(bundle.games.length, (i) => i + 1));
    });

    test('every referenced asset file exists', () {
      for (final path in bundle.sfx.values) {
        expect(File('${bundle.assetBase}$path').existsSync(), isTrue, reason: path);
      }
      for (final g in bundle.games.where((g) => g.promptAudio != null)) {
        expect(File('${bundle.assetBase}${g.promptAudio}').existsSync(), isTrue,
            reason: g.promptAudio);
      }
    });

    test('cards stay within the limit a young child can scan', () {
      for (final g in bundle.games) {
        expect(g.options.length, inInclusiveRange(3, 6), reason: g.id);
      }
    });
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `flutter test test/bundle_test.dart`
Expected: FAIL — `PathNotFoundException` / cannot open `assets/content/bundle.json`

- [ ] **Step 3: Write the content bundle**

Create `assets/content/bundle.json`. The ladder is ordered by **visual confusability**, which is the real learning axis for letterforms: distinct uppercase → confusable uppercase → distinct lowercase → confusable lowercase. Card count and number of correct answers both grow.

```json
{
  "bundle_version": 1,
  "asset_base": "assets/content/",
  "sfx": { "correct": "sfx/yay.wav", "wrong": "sfx/oops.wav" },
  "games": [
    { "id": "L01_A", "type": "find_same", "level": 1, "prompt_text": "Find the letter A", "target": { "text": "A" },
      "options": [ { "id": "1", "text": "O" }, { "id": "2", "text": "A", "correct": true }, { "id": "3", "text": "X" } ] },

    { "id": "L02_O", "type": "find_same", "level": 2, "prompt_text": "Find the letter O", "target": { "text": "O" },
      "options": [ { "id": "1", "text": "T" }, { "id": "2", "text": "O", "correct": true }, { "id": "3", "text": "S" } ] },

    { "id": "L03_S", "type": "find_same", "level": 3, "prompt_text": "Find the letter S", "target": { "text": "S" },
      "options": [ { "id": "1", "text": "A" }, { "id": "2", "text": "S", "correct": true }, { "id": "3", "text": "T" } ] },

    { "id": "L04_T", "type": "find_same", "level": 4, "prompt_text": "Find the letter T", "target": { "text": "T" },
      "options": [ { "id": "1", "text": "O" }, { "id": "2", "text": "T", "correct": true }, { "id": "3", "text": "X" }, { "id": "4", "text": "S" } ] },

    { "id": "L05_X", "type": "find_same", "level": 5, "prompt_text": "Find the letter X", "target": { "text": "X" },
      "options": [ { "id": "1", "text": "X", "correct": true }, { "id": "2", "text": "A" }, { "id": "3", "text": "O" }, { "id": "4", "text": "T" } ] },

    { "id": "L06_B", "type": "find_same", "level": 6, "prompt_text": "Find the letter B", "target": { "text": "B" },
      "options": [ { "id": "1", "text": "S" }, { "id": "2", "text": "B", "correct": true }, { "id": "3", "text": "T" }, { "id": "4", "text": "X" } ] },

    { "id": "L07_M", "type": "find_same", "level": 7, "prompt_text": "Find the letter M", "target": { "text": "M" },
      "options": [ { "id": "1", "text": "W" }, { "id": "2", "text": "M", "correct": true }, { "id": "3", "text": "M", "correct": true }, { "id": "4", "text": "W" } ] },

    { "id": "L08_E", "type": "find_same", "level": 8, "prompt_text": "Find the letter E", "target": { "text": "E" },
      "options": [ { "id": "1", "text": "F" }, { "id": "2", "text": "E", "correct": true }, { "id": "3", "text": "E", "correct": true }, { "id": "4", "text": "F" } ] },

    { "id": "L09_P", "type": "find_same", "level": 9, "prompt_text": "Find the letter P", "target": { "text": "P" },
      "options": [ { "id": "1", "text": "R" }, { "id": "2", "text": "P", "correct": true }, { "id": "3", "text": "P", "correct": true }, { "id": "4", "text": "R" } ] },

    { "id": "L10_a", "type": "find_same", "level": 10, "prompt_text": "Find the letter a", "target": { "text": "a" },
      "options": [ { "id": "1", "text": "o" }, { "id": "2", "text": "a", "correct": true }, { "id": "3", "text": "s" }, { "id": "4", "text": "t" } ] },

    { "id": "L11_o", "type": "find_same", "level": 11, "prompt_text": "Find the letter o", "target": { "text": "o" },
      "options": [ { "id": "1", "text": "t" }, { "id": "2", "text": "o", "correct": true }, { "id": "3", "text": "a" }, { "id": "4", "text": "s" } ] },

    { "id": "L12_s", "type": "find_same", "level": 12, "prompt_text": "Find the letter s", "target": { "text": "s" },
      "options": [ { "id": "1", "text": "a" }, { "id": "2", "text": "s", "correct": true }, { "id": "3", "text": "o" }, { "id": "4", "text": "t" } ] },

    { "id": "L13_b", "type": "find_same", "level": 13, "prompt_text": "Find the letter b", "target": { "text": "b" },
      "options": [ { "id": "1", "text": "d" }, { "id": "2", "text": "b", "correct": true }, { "id": "3", "text": "b", "correct": true }, { "id": "4", "text": "d" } ] },

    { "id": "L14_p", "type": "find_same", "level": 14, "prompt_text": "Find the letter p", "target": { "text": "p" },
      "options": [ { "id": "1", "text": "q" }, { "id": "2", "text": "p", "correct": true }, { "id": "3", "text": "p", "correct": true }, { "id": "4", "text": "q" } ] },

    { "id": "L15_n", "type": "find_same", "level": 15, "prompt_text": "Find the letter n", "target": { "text": "n" },
      "options": [ { "id": "1", "text": "u" }, { "id": "2", "text": "n", "correct": true }, { "id": "3", "text": "n", "correct": true }, { "id": "4", "text": "u" } ] }
  ]
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `flutter test test/bundle_test.dart`
Expected: all tests pass.

If the sfx existence test fails, the placeholder files from Task 1 Step 5 are missing — create them.

- [ ] **Step 5: Analyze and commit**

```bash
flutter analyze
git add assets/content/bundle.json test/bundle_test.dart
git commit -m "feat: 15-level letter bundle with content validation tests"
```

---

### Task 4: Bundle loading and playable-game filtering

**Files:**
- Create: `lib/content.dart`
- Test: `test/game_host_test.dart`

**Interfaces:**
- Consumes: `Bundle`, `Game` from Task 2.
- Produces:
  - `const Set<String> kSupportedTypes`
  - `Future<Bundle> loadBundle()`
  - `List<Game> playableGames(Bundle bundle)` — supported types only, sorted by level
  - `Future<Set<String>> loadCompleted()`
  - `Future<void> markCompleted(String gameId)`
  - `Future<void> resetProgress()`

- [ ] **Step 1: Write the failing test**

Create `test/game_host_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:kidsletters/bundle.dart';
import 'package:kidsletters/content.dart';

Game _game(String id, int level, {String type = 'find_same'}) => Game(
      id: id,
      type: type,
      level: level,
      promptText: 'p',
      promptAudio: null,
      targetText: 'A',
      options: const [Option(id: '1', text: 'A', correct: true)],
    );

Bundle _bundle(List<Game> games) => Bundle(
      bundleVersion: 1,
      assetBase: 'assets/content/',
      sfx: const {},
      games: games,
    );

void main() {
  test('playableGames sorts by level', () {
    final games = playableGames(_bundle([_game('c', 3), _game('a', 1), _game('b', 2)]));

    expect(games.map((g) => g.id), ['a', 'b', 'c']);
  });

  test('playableGames drops unknown types so an old app survives a new bundle', () {
    final games = playableGames(_bundle([
      _game('known', 1),
      _game('future', 2, type: 'trace_letter'),
    ]));

    expect(games.map((g) => g.id), ['known']);
  });

  test('playableGames returns empty rather than throwing when nothing is supported', () {
    expect(playableGames(_bundle([_game('future', 1, type: 'trace_letter')])), isEmpty);
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `flutter test test/game_host_test.dart`
Expected: FAIL — `Target of URI doesn't exist: 'package:kidsletters/content.dart'`

- [ ] **Step 3: Write content.dart**

```dart
import 'dart:convert';
import 'dart:io';

import 'package:flutter/services.dart' show rootBundle;
import 'package:path_provider/path_provider.dart';
import 'package:shared_preferences/shared_preferences.dart';

import 'bundle.dart';

/// Game types this build can render. A bundle may reference types added in a
/// later app version; those are filtered out rather than crashing.
const Set<String> kSupportedTypes = {'find_same'};

const _completedKey = 'completed_game_ids';

/// Prefers a downloaded bundle over the one shipped in the app.
///
/// v1 never writes the downloaded file — this branch is the entire seam for a
/// future backend, which only has to drop a file at this path.
Future<Bundle> loadBundle() async {
  final dir = await getApplicationSupportDirectory();
  final downloaded = File('${dir.path}/bundle.json');

  final raw = await downloaded.exists()
      ? await downloaded.readAsString()
      : await rootBundle.loadString('assets/content/bundle.json');

  return Bundle.fromJson(jsonDecode(raw) as Map<String, dynamic>);
}

/// Games this build can render, in level order.
List<Game> playableGames(Bundle bundle) {
  final games = bundle.games.where((g) => kSupportedTypes.contains(g.type)).toList();
  games.sort((a, b) => a.level.compareTo(b.level));
  return games;
}

Future<Set<String>> loadCompleted() async {
  final prefs = await SharedPreferences.getInstance();
  return (prefs.getStringList(_completedKey) ?? const <String>[]).toSet();
}

Future<void> markCompleted(String gameId) async {
  final prefs = await SharedPreferences.getInstance();
  final completed = (prefs.getStringList(_completedKey) ?? const <String>[]).toSet()
    ..add(gameId);
  await prefs.setStringList(_completedKey, completed.toList());
}

Future<void> resetProgress() async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.remove(_completedKey);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `flutter test test/game_host_test.dart`
Expected: `+3: All tests passed!`

- [ ] **Step 5: Analyze and commit**

```bash
flutter analyze
git add lib/content.dart test/game_host_test.dart
git commit -m "feat: bundle loading, type filtering, progress storage"
```

**Deliberately not tested:** the downloaded-file branch of `loadBundle`, because nothing writes that file in v1. It gets a test when the v2 downloader exists.

---

### Task 5: Game state machine

**Files:**
- Create: `lib/games/find_same_state.dart`
- Test: `test/find_same_state_test.dart`

**Interfaces:**
- Consumes: `Game`, `Option` from Task 2.
- Produces:
  - `class FindSameState { FindSameState(Game game); Game get game; Set<String> get found; bool tap(String optionId); bool isFound(String optionId); bool get isComplete; }`
  - `tap` returns `true` when the tapped option was correct.

- [ ] **Step 1: Write the failing test**

Create `test/find_same_state_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:kidsletters/bundle.dart';
import 'package:kidsletters/games/find_same_state.dart';

Game _twoCorrect() => const Game(
      id: 'g',
      type: 'find_same',
      level: 1,
      promptText: 'Find the letter M',
      promptAudio: null,
      targetText: 'M',
      options: [
        Option(id: '1', text: 'W', correct: false),
        Option(id: '2', text: 'M', correct: true),
        Option(id: '3', text: 'M', correct: true),
        Option(id: '4', text: 'W', correct: false),
      ],
    );

void main() {
  test('starts incomplete with nothing found', () {
    final state = FindSameState(_twoCorrect());

    expect(state.found, isEmpty);
    expect(state.isComplete, isFalse);
  });

  test('a correct tap reports true and is remembered', () {
    final state = FindSameState(_twoCorrect());

    expect(state.tap('2'), isTrue);
    expect(state.isFound('2'), isTrue);
    expect(state.found, {'2'});
  });

  test('a wrong tap reports false and changes nothing', () {
    final state = FindSameState(_twoCorrect());

    expect(state.tap('1'), isFalse);
    expect(state.isFound('1'), isFalse);
    expect(state.found, isEmpty);
    expect(state.isComplete, isFalse);
  });

  test('completes only when every correct option is found', () {
    final state = FindSameState(_twoCorrect());

    state.tap('2');
    expect(state.isComplete, isFalse);

    state.tap('3');
    expect(state.isComplete, isTrue);
  });

  test('tapping the same correct option twice does not double-count', () {
    final state = FindSameState(_twoCorrect());

    state.tap('2');
    state.tap('2');

    expect(state.found, {'2'});
    expect(state.isComplete, isFalse);
  });

  test('wrong taps never block completion — there is no fail state', () {
    final state = FindSameState(_twoCorrect());

    state.tap('1');
    state.tap('4');
    state.tap('1');
    state.tap('2');
    state.tap('3');

    expect(state.isComplete, isTrue);
  });

  test('an unknown option id is treated as a wrong tap', () {
    final state = FindSameState(_twoCorrect());

    expect(state.tap('nope'), isFalse);
    expect(state.found, isEmpty);
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `flutter test test/find_same_state_test.dart`
Expected: FAIL — `Target of URI doesn't exist: 'package:kidsletters/games/find_same_state.dart'`

- [ ] **Step 3: Write the state machine**

Create `lib/games/find_same_state.dart`:

```dart
import '../bundle.dart';

/// Tap state for one "find the same letter" round.
///
/// Pure Dart — no Flutter imports — so the rules are tested without a widget.
/// There is no fail state: wrong taps are simply ignored.
class FindSameState {
  FindSameState(this.game) : _correctIds = game.correctIds;

  final Game game;
  final Set<String> _correctIds;
  final Set<String> _found = {};

  Set<String> get found => Set.unmodifiable(_found);

  bool isFound(String optionId) => _found.contains(optionId);

  /// Records a tap. Returns true when [optionId] was a correct option.
  bool tap(String optionId) {
    if (!_correctIds.contains(optionId)) return false;
    _found.add(optionId);
    return true;
  }

  bool get isComplete =>
      _correctIds.isNotEmpty && _found.length == _correctIds.length;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `flutter test test/find_same_state_test.dart`
Expected: `+7: All tests passed!`

- [ ] **Step 5: Analyze and commit**

```bash
flutter analyze
git add lib/games/find_same_state.dart test/find_same_state_test.dart
git commit -m "feat: find-same game state machine"
```

---

### Task 6: Audio service

**Files:**
- Create: `lib/audio.dart`
- Modify: `lib/main.dart`
- Modify: `ios/Runner/Info.plist`

**Interfaces:**
- Consumes: `Bundle` from Task 2.
- Produces:
  - `Future<void> initAudio()` — call once at startup, before `runApp`
  - `Future<void> speak(String text)`
  - `Future<void> stopSpeaking()`
  - `Future<void> playSfx(String assetPath)` — path relative to `assets/`, e.g. `content/sfx/yay.wav`
  - `String sfxAsset(Bundle bundle, String key)` — maps a bundle sfx key to an `audioplayers` asset path

- [ ] **Step 1: Write audio.dart**

There is no unit test for this task — it is a thin wrapper over two plugins and has no logic worth asserting. It is verified by hand in Step 4 and again in Task 9. The one piece of real logic, `sfxAsset`, is covered by the test in Step 2.

```dart
import 'package:audioplayers/audioplayers.dart';
import 'package:flutter_tts/flutter_tts.dart';

import 'bundle.dart';

final FlutterTts _tts = FlutterTts();
final AudioPlayer _sfxPlayer = AudioPlayer();

/// Configures audio so prompts play even when the iOS hardware mute switch is on.
///
/// Without this an audio-first app is silently broken for every child whose
/// iPad is muted — which is most of them.
Future<void> initAudio() async {
  // Android is left at its defaults (usageType: media, contentType: music) —
  // correct here, because the volume rocker controls the media stream and the
  // game's audio IS its content, not a UI beep.
  await _sfxPlayer.setAudioContext(
    AudioContext(
      iOS: AudioContextIOS(
        category: AVAudioSessionCategory.playback,
        options: const {},
      ),
    ),
  );
  await _sfxPlayer.setReleaseMode(ReleaseMode.stop);

  await _tts.setIosAudioCategory(
    IosTextToSpeechAudioCategory.playback,
    [IosTextToSpeechAudioCategoryOptions.duckOthers],
  );
  await _tts.setLanguage('en-US');
  await _tts.setSpeechRate(0.4); // slow — 3-to-5 year olds need it
  await _tts.setPitch(1.1);
}

Future<void> speak(String text) async {
  await _tts.stop();
  await _tts.speak(text);
}

Future<void> stopSpeaking() => _tts.stop();

/// [assetPath] is relative to `assets/`, e.g. `content/sfx/yay.wav`.
Future<void> playSfx(String assetPath) async {
  await _sfxPlayer.stop();
  await _sfxPlayer.play(AssetSource(assetPath));
}

/// Bundle paths are relative to `asset_base`, but `audioplayers` resolves
/// relative to `assets/`. This strips the leading `assets/`.
String sfxAsset(Bundle bundle, String key) {
  final path = '${bundle.assetBase}${bundle.sfx[key] ?? ''}';
  return path.startsWith('assets/') ? path.substring('assets/'.length) : path;
}
```

- [ ] **Step 2: Write the test for the one piece of real logic**

Create `test/audio_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:kidsletters/audio.dart';
import 'package:kidsletters/bundle.dart';

void main() {
  test('sfxAsset strips the assets/ prefix audioplayers adds back', () {
    const bundle = Bundle(
      bundleVersion: 1,
      assetBase: 'assets/content/',
      sfx: {'correct': 'sfx/yay.wav'},
      games: [],
    );

    expect(sfxAsset(bundle, 'correct'), 'content/sfx/yay.wav');
  });
}
```

- [ ] **Step 3: Run the test**

Run: `flutter test test/audio_test.dart`
Expected: `+1: All tests passed!`

- [ ] **Step 4: Call initAudio from main**

In `lib/main.dart`, replace the `void main()` line with:

```dart
import 'package:flutter/material.dart';

import 'audio.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initAudio();
  runApp(const KidsLettersApp());
}
```

- [ ] **Step 5: Allow background-capable audio on iOS**

In `ios/Runner/Info.plist`, add inside the top-level `<dict>`:

```xml
	<key>UIBackgroundModes</key>
	<array>
		<string>audio</string>
	</array>
```

- [ ] **Step 6: Verify by hand**

Add a temporary button to the `home:` scaffold in `lib/main.dart` that calls `speak('Find the letter A')`, then:

Run: `flutter run -d linux` (or an Android emulator — TTS is unavailable on Linux desktop, so use the emulator if Linux produces no sound)
Expected: the phrase is spoken slowly and clearly.

Remove the temporary button before committing.

The mute-switch behaviour can only be confirmed on a physical iPhone/iPad: flip the side switch to silent and confirm the prompt still plays. Record this as a manual check to run on the first Mac build.

- [ ] **Step 7: Analyze and commit**

```bash
flutter analyze
git add lib/audio.dart lib/main.dart test/audio_test.dart ios/Runner/Info.plist
git commit -m "feat: TTS prompts and SFX with iOS playback session"
```

---

### Task 7: The find-same game screen

**Files:**
- Create: `lib/games/find_same.dart`
- Test: `test/find_same_widget_test.dart`

**Interfaces:**
- Consumes: `Game` (Task 2), `FindSameState` (Task 5), `speak`/`playSfx`/`sfxAsset` (Task 6).
- Produces: `class FindSameGame extends StatefulWidget { const FindSameGame({required Game game, required String correctSfx, required String wrongSfx, required VoidCallback onComplete, Key? key}); }`

- [ ] **Step 1: Write the failing widget test**

Create `test/find_same_widget_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:kidsletters/bundle.dart';
import 'package:kidsletters/games/find_same.dart';

const _game = Game(
  id: 'g',
  type: 'find_same',
  level: 1,
  promptText: 'Find the letter M',
  promptAudio: null,
  targetText: 'M',
  options: [
    Option(id: '1', text: 'W', correct: false),
    Option(id: '2', text: 'M', correct: true),
    Option(id: '3', text: 'M', correct: true),
    Option(id: '4', text: 'W', correct: false),
  ],
);

Widget _harness({required VoidCallback onComplete}) => MaterialApp(
      home: FindSameGame(
        game: _game,
        correctSfx: 'content/sfx/yay.wav',
        wrongSfx: 'content/sfx/oops.wav',
        onComplete: onComplete,
      ),
    );

void main() {
  testWidgets('renders the target and one card per option', (tester) async {
    await tester.pumpWidget(_harness(onComplete: () {}));

    expect(find.byKey(const Key('target')), findsOneWidget);
    expect(find.byType(OptionCard), findsNWidgets(4));
  });

  testWidgets('every card meets the 120pt minimum tap target', (tester) async {
    await tester.pumpWidget(_harness(onComplete: () {}));

    for (final element in find.byType(OptionCard).evaluate()) {
      final size = tester.getSize(find.byWidget(element.widget));
      expect(size.width, greaterThanOrEqualTo(120));
      expect(size.height, greaterThanOrEqualTo(120));
    }
  });

  testWidgets('completes only after both correct cards are tapped', (tester) async {
    var completed = false;
    await tester.pumpWidget(_harness(onComplete: () => completed = true));

    await tester.tap(find.byKey(const Key('option_2')));
    await tester.pumpAndSettle();
    expect(completed, isFalse);

    await tester.tap(find.byKey(const Key('option_3')));
    await tester.pumpAndSettle();
    await tester.pump(const Duration(seconds: 3));
    expect(completed, isTrue);
  });

  testWidgets('a wrong tap does not complete and does not lock the card', (tester) async {
    var completed = false;
    await tester.pumpWidget(_harness(onComplete: () => completed = true));

    await tester.tap(find.byKey(const Key('option_1')));
    await tester.pumpAndSettle();

    expect(completed, isFalse);
    final card = tester.widget<OptionCard>(find.byKey(const Key('option_1')));
    expect(card.found, isFalse);
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `flutter test test/find_same_widget_test.dart`
Expected: FAIL — `Target of URI doesn't exist: 'package:kidsletters/games/find_same.dart'`

- [ ] **Step 3: Write the widget**

Create `lib/games/find_same.dart`:

```dart
import 'dart:math' as math;

import 'package:flutter/material.dart';

import '../audio.dart';
import '../bundle.dart';
import 'find_same_state.dart';

class FindSameGame extends StatefulWidget {
  const FindSameGame({
    super.key,
    required this.game,
    required this.correctSfx,
    required this.wrongSfx,
    required this.onComplete,
  });

  final Game game;
  final String correctSfx;
  final String wrongSfx;
  final VoidCallback onComplete;

  @override
  State<FindSameGame> createState() => _FindSameGameState();
}

class _FindSameGameState extends State<FindSameGame> {
  late FindSameState _state;
  late List<Option> _shuffled;
  String? _shakingId;

  @override
  void initState() {
    super.initState();
    _start();
  }

  @override
  void didUpdateWidget(FindSameGame old) {
    super.didUpdateWidget(old);
    if (old.game.id != widget.game.id) _start();
  }

  void _start() {
    _state = FindSameState(widget.game);
    // Shuffled so a child learns the letterform rather than memorising a position.
    _shuffled = List.of(widget.game.options)..shuffle();
    _shakingId = null;
    _speakPrompt();
  }

  void _speakPrompt() => speak(widget.game.promptText);

  Future<void> _onTap(Option option) async {
    if (_state.isFound(option.id)) return;

    if (_state.tap(option.id)) {
      setState(() {});
      await playSfx(widget.correctSfx);
      if (_state.isComplete) {
        await speak('Great job!');
        await Future<void>.delayed(const Duration(seconds: 2));
        if (mounted) widget.onComplete();
      }
      return;
    }

    // Wrong tap: a gentle nudge, never a penalty.
    setState(() => _shakingId = option.id);
    await playSfx(widget.wrongSfx);
    await Future<void>.delayed(const Duration(milliseconds: 500));
    if (mounted) setState(() => _shakingId = null);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFFFFF8E7),
      body: SafeArea(
        child: Column(
          children: [
            const SizedBox(height: 16),
            GestureDetector(
              key: const Key('target'),
              onTap: _speakPrompt, // tap the target to hear the prompt again
              child: Container(
                width: 180,
                height: 180,
                alignment: Alignment.center,
                decoration: BoxDecoration(
                  color: Colors.white,
                  borderRadius: BorderRadius.circular(32),
                  boxShadow: const [
                    BoxShadow(color: Color(0x22000000), blurRadius: 12, offset: Offset(0, 4)),
                  ],
                ),
                child: Text(
                  widget.game.targetText,
                  style: const TextStyle(fontSize: 110, height: 1.1),
                ),
              ),
            ),
            const SizedBox(height: 24),
            Expanded(
              child: Center(
                child: Wrap(
                  alignment: WrapAlignment.center,
                  spacing: 20,
                  runSpacing: 20,
                  children: [
                    for (final option in _shuffled)
                      OptionCard(
                        key: Key('option_${option.id}'),
                        option: option,
                        found: _state.isFound(option.id),
                        shaking: _shakingId == option.id,
                        onTap: () => _onTap(option),
                      ),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class OptionCard extends StatelessWidget {
  const OptionCard({
    super.key,
    required this.option,
    required this.found,
    required this.shaking,
    required this.onTap,
  });

  final Option option;
  final bool found;
  final bool shaking;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    final card = AnimatedScale(
      scale: found ? 1.12 : 1.0,
      duration: const Duration(milliseconds: 220),
      curve: Curves.easeOutBack,
      child: AnimatedContainer(
        duration: const Duration(milliseconds: 220),
        width: 140,
        height: 140,
        alignment: Alignment.center,
        decoration: BoxDecoration(
          color: found ? const Color(0xFFD3F9D8) : Colors.white,
          borderRadius: BorderRadius.circular(28),
          border: Border.all(
            color: found ? const Color(0xFF37B24D) : const Color(0x22000000),
            width: found ? 5 : 2,
          ),
          boxShadow: const [
            BoxShadow(color: Color(0x1A000000), blurRadius: 10, offset: Offset(0, 4)),
          ],
        ),
        child: Text(option.text, style: const TextStyle(fontSize: 84, height: 1.1)),
      ),
    );

    return GestureDetector(
      onTap: onTap,
      behavior: HitTestBehavior.opaque,
      child: shaking ? _Shake(child: card) : card,
    );
  }
}

/// A short side-to-side wobble. Built-in animation — no package needed.
class _Shake extends StatefulWidget {
  const _Shake({required this.child});

  final Widget child;

  @override
  State<_Shake> createState() => _ShakeState();
}

class _ShakeState extends State<_Shake> with SingleTickerProviderStateMixin {
  late final AnimationController _controller = AnimationController(
    vsync: this,
    duration: const Duration(milliseconds: 400),
  )..forward();

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) => Transform.translate(
        offset: Offset(math.sin(_controller.value * math.pi * 6) * 10, 0),
        child: child,
      ),
      child: widget.child,
    );
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `flutter test test/find_same_widget_test.dart`
Expected: `+4: All tests passed!`

If the tests hang or error on plugin channels, the TTS/audio calls are being invoked in the test environment. Fix by guarding at the top of `test/find_same_widget_test.dart`'s `main()`:

```dart
  TestWidgetsFlutterBinding.ensureInitialized();
  // Plugins have no implementation in tests; swallow their channel calls.
  const channels = [
    MethodChannel('flutter_tts'),
    MethodChannel('xyz.luan/audioplayers'),
  ];
  setUp(() {
    for (final channel in channels) {
      TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
          .setMockMethodCallHandler(channel, (call) async => null);
    }
  });
```

(add `import 'package:flutter/services.dart';`)

- [ ] **Step 5: Analyze and commit**

```bash
flutter analyze
git add lib/games/find_same.dart test/find_same_widget_test.dart
git commit -m "feat: find-same game screen with shake and found states"
```

---

### Task 8: Game host — sequencing and progress

**Files:**
- Create: `lib/game_host.dart`
- Modify: `lib/main.dart`
- Modify: `test/game_host_test.dart`

**Interfaces:**
- Consumes: `loadBundle`, `playableGames`, `loadCompleted`, `markCompleted`, `resetProgress` (Task 4); `FindSameGame` (Task 7); `sfxAsset` (Task 6).
- Produces:
  - `Game? nextGame(List<Game> games, Set<String> completed)` — first game not yet completed, or null when all are done
  - `class GameHost extends StatefulWidget` — loads the bundle, plays games in order, persists progress

- [ ] **Step 1: Write the failing test**

Append to `test/game_host_test.dart`:

```dart
  group('nextGame', () {
    final games = [_game('a', 1), _game('b', 2), _game('c', 3)];

    test('returns the first game when nothing is completed', () {
      expect(nextGame(games, {})?.id, 'a');
    });

    test('skips completed games', () {
      expect(nextGame(games, {'a', 'b'})?.id, 'c');
    });

    test('skips completed games out of order', () {
      expect(nextGame(games, {'b'})?.id, 'a');
    });

    test('returns null when everything is completed', () {
      expect(nextGame(games, {'a', 'b', 'c'}), isNull);
    });

    test('returns null for an empty game list', () {
      expect(nextGame([], {}), isNull);
    });
  });
```

Also append this group, which closes the one real coverage gap left by Task 4 —
progress persistence is a read-modify-write, and if it breaks a child silently
never advances past level 1 or loses their place on every relaunch:

```dart
  group('progress persistence', () {
    setUp(() => SharedPreferences.setMockInitialValues({}));

    test('starts with nothing completed', () async {
      expect(await loadCompleted(), isEmpty);
    });

    test('markCompleted persists and accumulates', () async {
      await markCompleted('a');
      await markCompleted('b');

      expect(await loadCompleted(), {'a', 'b'});
    });

    test('markCompleted is idempotent', () async {
      await markCompleted('a');
      await markCompleted('a');

      expect(await loadCompleted(), {'a'});
    });

    test('resetProgress clears everything', () async {
      await markCompleted('a');
      await resetProgress();

      expect(await loadCompleted(), isEmpty);
    });
  });
```

Add these imports at the top of the file:

```dart
import 'package:shared_preferences/shared_preferences.dart';
import 'package:kidsletters/game_host.dart';
```

`TestWidgetsFlutterBinding.ensureInitialized();` must be the first line of
`main()` for `setMockInitialValues` to work. Add it if it is not already there.

- [ ] **Step 2: Run test to verify it fails**

Run: `flutter test test/game_host_test.dart`
Expected: FAIL — `Target of URI doesn't exist: 'package:kidsletters/game_host.dart'`

- [ ] **Step 3: Write game_host.dart**

```dart
import 'package:flutter/material.dart';

import 'audio.dart';
import 'bundle.dart';
import 'content.dart';
import 'games/find_same.dart';

/// The first game the child has not finished, or null when all are done.
Game? nextGame(List<Game> games, Set<String> completed) {
  for (final game in games) {
    if (!completed.contains(game.id)) return game;
  }
  return null;
}

class GameHost extends StatefulWidget {
  const GameHost({super.key});

  @override
  State<GameHost> createState() => _GameHostState();
}

class _GameHostState extends State<GameHost> {
  Bundle? _bundle;
  List<Game> _games = const [];
  Set<String> _completed = const {};
  Object? _error;

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    try {
      final bundle = await loadBundle();
      final completed = await loadCompleted();
      if (!mounted) return;
      setState(() {
        _bundle = bundle;
        _games = playableGames(bundle);
        _completed = completed;
      });
    } catch (error) {
      if (mounted) setState(() => _error = error);
    }
  }

  Future<void> _onComplete(Game game) async {
    await markCompleted(game.id);
    if (!mounted) return;
    setState(() => _completed = {..._completed, game.id});
  }

  Future<void> _playAgain() async {
    await resetProgress();
    if (!mounted) return;
    setState(() => _completed = const {});
  }

  @override
  Widget build(BuildContext context) {
    final bundle = _bundle;

    if (_error != null) {
      return const _Message(emoji: '🐛', label: 'Something went wrong');
    }
    if (bundle == null) {
      return const Scaffold(body: Center(child: CircularProgressIndicator()));
    }
    if (_games.isEmpty) {
      return const _Message(emoji: '📦', label: 'No games yet');
    }

    final game = nextGame(_games, _completed);
    if (game == null) {
      return _AllDone(onPlayAgain: _playAgain);
    }

    return FindSameGame(
      key: ValueKey(game.id),
      game: game,
      correctSfx: sfxAsset(bundle, 'correct'),
      wrongSfx: sfxAsset(bundle, 'wrong'),
      onComplete: () => _onComplete(game),
    );
  }
}

class _Message extends StatelessWidget {
  const _Message({required this.emoji, required this.label});

  final String emoji;
  final String label;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFFFFF8E7),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text(emoji, style: const TextStyle(fontSize: 96)),
            const SizedBox(height: 16),
            Text(label, style: const TextStyle(fontSize: 28)),
          ],
        ),
      ),
    );
  }
}

/// End of the ladder. No score, no stars — just a big friendly way to start over.
class _AllDone extends StatelessWidget {
  const _AllDone({required this.onPlayAgain});

  final VoidCallback onPlayAgain;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFFFFF8E7),
      body: Center(
        child: GestureDetector(
          key: const Key('play_again'),
          onTap: onPlayAgain,
          behavior: HitTestBehavior.opaque,
          child: Container(
            width: 220,
            height: 220,
            alignment: Alignment.center,
            decoration: BoxDecoration(
              color: const Color(0xFFD3F9D8),
              shape: BoxShape.circle,
              border: Border.all(color: const Color(0xFF37B24D), width: 6),
            ),
            child: const Text('▶', style: TextStyle(fontSize: 96)),
          ),
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `flutter test test/game_host_test.dart`
Expected: `+12: All tests passed!` (3 filtering + 5 sequencing + 4 progress)

- [ ] **Step 5: Make GameHost the home screen**

In `lib/main.dart`, add `import 'game_host.dart';` and replace the `home:` line with:

```dart
      home: const GameHost(),
```

- [ ] **Step 6: Run the full suite**

Run: `flutter test`
Expected: all tests pass.

- [ ] **Step 7: Analyze and commit**

```bash
flutter analyze
git add lib/game_host.dart lib/main.dart test/game_host_test.dart
git commit -m "feat: game host with level sequencing and progress"
```

---

### Task 9: Celebration, orientation lock, and end-to-end verification

**Files:**
- Modify: `lib/games/find_same.dart`
- Modify: `lib/main.dart`
- Modify: `ios/Runner/Info.plist`, `android/app/src/main/AndroidManifest.xml`
- Create: `docs/manual-checks.md`

**Interfaces:**
- Consumes: everything from Tasks 1-8.
- Produces: no new public API.

- [ ] **Step 1: Add a star burst on completion**

In `lib/games/find_same.dart`, add this widget at the bottom of the file:

```dart
/// A ring of stars that flies outward once. Built-in animation — no package.
class _StarBurst extends StatefulWidget {
  const _StarBurst();

  @override
  State<_StarBurst> createState() => _StarBurstState();
}

class _StarBurstState extends State<_StarBurst> with SingleTickerProviderStateMixin {
  static const _count = 12;

  late final AnimationController _controller = AnimationController(
    vsync: this,
    duration: const Duration(milliseconds: 1200),
  )..forward();

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return IgnorePointer(
      child: AnimatedBuilder(
        animation: _controller,
        builder: (context, _) {
          final t = Curves.easeOut.transform(_controller.value);
          return Stack(
            alignment: Alignment.center,
            children: [
              for (var i = 0; i < _count; i++)
                Transform.translate(
                  offset: Offset(
                    math.cos(i * 2 * math.pi / _count) * 220 * t,
                    math.sin(i * 2 * math.pi / _count) * 220 * t,
                  ),
                  child: Opacity(
                    opacity: (1 - t).clamp(0.0, 1.0),
                    child: const Text('⭐', style: TextStyle(fontSize: 44)),
                  ),
                ),
            ],
          );
        },
      ),
    );
  }
}
```

- [ ] **Step 2: Show the burst when the round completes**

In `_FindSameGameState`, add a field:

```dart
  bool _celebrating = false;
```

In `_onTap`, inside the `if (_state.isComplete)` branch, set it before speaking:

```dart
      if (_state.isComplete) {
        setState(() => _celebrating = true);
        await speak('Great job!');
        await Future<void>.delayed(const Duration(seconds: 2));
        if (mounted) widget.onComplete();
      }
```

In `_start()`, reset it:

```dart
    _celebrating = false;
```

In `build`, wrap the existing `Scaffold`'s `body` content in a `Stack`:

```dart
      body: Stack(
        alignment: Alignment.center,
        children: [
          SafeArea(child: /* the existing Column stays here unchanged */),
          if (_celebrating) const _StarBurst(),
        ],
      ),
```

- [ ] **Step 3: Run the full suite to confirm nothing regressed**

Run: `flutter test`
Expected: all tests pass. The completion test in Task 7 still passes because the burst is decorative and `IgnorePointer` keeps it out of hit testing.

- [ ] **Step 4: Lock to landscape**

Small hands hold a tablet in landscape, and the card row is laid out for width.

In `lib/main.dart`, add `import 'package:flutter/services.dart';` and set orientations in `main()` before `runApp`:

```dart
  await SystemChrome.setPreferredOrientations([
    DeviceOrientation.landscapeLeft,
    DeviceOrientation.landscapeRight,
  ]);
```

In `ios/Runner/Info.plist`, ensure both `UISupportedInterfaceOrientations` and
`UISupportedInterfaceOrientations~ipad` contain only:

```xml
	<array>
		<string>UIInterfaceOrientationLandscapeLeft</string>
		<string>UIInterfaceOrientationLandscapeRight</string>
	</array>
```

In `android/app/src/main/AndroidManifest.xml`, add to the `<activity>` tag:

```xml
android:screenOrientation="sensorLandscape"
```

- [ ] **Step 5: Run the app end to end**

Run: `flutter run -d <android-emulator-or-device>`

Walk through and confirm each of these:

1. Level 1 loads, the prompt "Find the letter A" is spoken automatically.
2. Tapping the target letter replays the prompt.
3. Tapping a wrong card wobbles it, plays the soft sound, and leaves it unlocked.
4. Tapping the correct card enlarges it, turns it green, and plays the bright sound.
5. On the last correct card, stars burst, "Great job!" is spoken, and level 2 loads after ~2s.
6. Level 7 requires two taps before advancing.
7. Kill and relaunch the app — it resumes at the level you reached, not level 1.
8. Finish all 15 — the big green play button appears and restarts the ladder.

- [ ] **Step 6: Record the checks that need real hardware**

Create `docs/manual-checks.md`:

```markdown
# Manual checks

Automated tests cannot cover these. Run them on a physical device before each release.

## iPad / iPhone (requires a Mac build)

- [ ] Hardware mute switch ON — the spoken prompt still plays. This is the single
      most important check; if it fails the app is silent for most children.
- [ ] Music playing in another app — starting a level ducks or stops it, and the
      prompt is audible.
- [ ] Screen locks and unlocks mid-level — audio recovers, state is intact.
- [ ] Landscape only; the app does not rotate to portrait.
- [ ] Card tap targets are comfortable for a small hand on the smallest supported
      iPad, with no accidental double-registration.

## Content

- [ ] `oops.wav` sounds neutral and gentle to a real child, never like an error
      buzzer. Tune `tools/gen_sfx.py` and re-run it if not.
- [ ] `yay.wav` is audible over a noisy room without being piercing.

## Before App Store submission (Kids Category)

- [ ] No network calls in the build.
- [ ] No third-party analytics or ad SDKs.
- [ ] Parent gate on anything leaving the app — not applicable while there are no
      external links, but re-check whenever one is added.
```

- [ ] **Step 7: Analyze and commit**

```bash
flutter analyze
flutter test
git add -A
git commit -m "feat: celebration burst, landscape lock, manual check list"
```

---

## What v2 adds, and why nothing above has to change

Recorded here so the seams are not accidentally removed during v1.

1. **Backend content sync.** A Go service serving `GET /bundle?since=<version>` writes its response to `${applicationSupportDirectory}/bundle.json`. `loadBundle()` already prefers that file. No other v1 file changes.
2. **New content.** New games in the bundle appear without an app release.
3. **New game types.** Requires an app release — the widget must be compiled in. Add the type to `kSupportedTypes` and the widget to `GameHost`'s dispatch. Existing installs already skip unknown types (Task 4), so they will not crash on the new bundle.
4. **Dark shipping.** Ship a game-type widget in a release while no bundle references it, then activate it later with a content update.
5. **Progress sync.** `loadCompleted`/`markCompleted` are the only progress touchpoints; POSTing the same set is additive.
6. **Content validation server-side.** Port `test/bundle_test.dart`'s `shipped bundle` group to Go and run it on upload.

---

## Self-Review

**Spec coverage**

| Spec requirement | Task |
|---|---|
| Offline, no network | Global Constraints; nothing imports `http` |
| `loadBundle()` local/remote seam | 4 |
| Content contract with `correct` on the option | 2, 3 |
| Relative paths + `asset_base` | 2, 3, 6 (`sfxAsset`) |
| Bundle-level `sfx` | 2, 3 |
| Optional `prompt_audio` with TTS fallback | 2, 6, 7 |
| iOS `AVAudioSession` playback category | 6 |
| File structure | Tasks 2, 4, 5, 6, 7, 8 |
| Forward-compat unknown-type filter | 4 |
| Interaction rules 1-7 | 7 |
| No timer/score/lives/fail state | 5 (no fail path), 7, 8 (`_AllDone`) |
| Celebration via built-in animation | 9 |
| Difficulty ladder, uppercase→lowercase by confusability | 3 |
| Single-story `a`/`g` font | 1 |
| Progress via `shared_preferences` | 4, 8 |
| Min 120pt tap target | 7 (asserted in test) |
| Kids Category constraints | 9 (`docs/manual-checks.md`) |
| `bundle_test.dart` validation | 3 |
| `game_test.dart` tap state machine | 5 |

Two deliberate deviations from the spec, both simplifications in the same direction:
- Progress is stored with `setStringList` rather than a hand-encoded JSON blob. Same data, less code, still POSTable in v2.
- The spec named two test files; the plan has five, because widget, audio, and host logic each earned their own. Both spec-named files exist with their specified contents.

**Placeholder scan:** none. Every code step contains runnable code, and every asset the plan references exists on disk before Task 1 begins — there are no stubs.

**Type consistency:** `Bundle`/`Game`/`Option` field names are used identically in Tasks 2-8. `FindSameState.tap` returns `bool` everywhere. `sfxAsset(Bundle, String)` is defined in Task 6 and called in Task 8. `playableGames`/`nextGame`/`markCompleted` signatures match between definition and call sites. `OptionCard` is exported from `find_same.dart` because the Task 7 test references it by type.
