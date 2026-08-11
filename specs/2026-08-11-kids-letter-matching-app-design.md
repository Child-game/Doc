# Kids Letter-Matching App — Design

**Date:** 2026-08-11
**Status:** Approved for planning
**Stack:** Flutter (iOS-first, Android supported). Go backend deferred to v2.

## Goal

A learning game for children aged 3-5. One game type in v1: **Find the Same Letter**. A
target letter is shown; the child taps the matching letter(s) among several cards.

The app must work fully offline. Content must be replaceable from a backend later without
restructuring the app.

## Non-goals for v1

Explicitly out of scope, with the trigger for adding each:

| Dropped | Add when |
|---|---|
| Backend / any network call | Content needs to change without an App Store release |
| User accounts, parent dashboard | Parents ask where progress is |
| Second game type | The first one is validated with a real child |
| Riverpod / Bloc | State is genuinely shared across screens |
| `json_serializable` / `build_runner` | Models exceed ~5 classes |
| Hive / Isar | Progress outgrows a key-value blob |
| Redis, Clean Architecture layers, SDUI engine | Never, as specified — see rationale below |
| Image assets | A game type needs pictures |
| Recorded voice audio | TTS quality proves inadequate with a real child |

### Rationale for dropping SDUI

Server-Driven UI cannot ship a new *game type* without an App Store release, because the
Flutter widget must be compiled into the binary. It only ships new *content*. Since content
is just data, a versioned JSON bundle achieves the same thing with none of the machinery.

## Architecture

### Content is data on device; sync is an upgrade path

The whole local→remote seam is one function:

```dart
Future<Bundle> loadBundle() async {
  final downloaded = File('${dir.path}/bundle.json');
  final raw = await downloaded.exists()
      ? await downloaded.readAsString()
      : await rootBundle.loadString('assets/content/bundle.json');
  return Bundle.fromJson(jsonDecode(raw));
}
```

v1 exercises only the `rootBundle` branch. v2 adds a downloader that writes to that path.
No refactor.

### Content contract

```json
{
  "bundle_version": 1,
  "asset_base": "assets/content/",
  "sfx": { "correct": "sfx/yay.mp3", "wrong": "sfx/oops.mp3" },
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
```

Design decisions in the contract:

- **`correct: true` lives on the option**, not in a parallel `correct_option_ids` array. One
  source of truth; the two cannot drift apart.
- **Paths are relative, resolved against `asset_base`.** This is what makes local→remote free:
  v1 resolves against the asset bundle, v2 against a CDN URL or the download cache directory.
  Identical JSON, identical parser.
- **`sfx` is bundle-level**, not repeated per game.
- **No `{"success": true, "data": ...}` envelope.** HTTP status codes cover that in v2.
- **`prompt_audio` is optional and absent in v1.** When present, the app plays that file. When
  absent, the app speaks `prompt_text` via on-device TTS. This means v1 ships **zero recorded
  voice assets**, and recorded audio can be added later per-game with no code change.

### Audio

`flutter_tts` for prompts (AVSpeechSynthesizer on iOS, fully offline), `audioplayers` for the
two SFX files.

**Required iOS configuration:** set the `AVAudioSession` category to `playback` at startup.
By default Flutter audio respects the hardware mute switch, which would silently break an
audio-first app for every child whose iPad is muted.

### Files

```
lib/
  main.dart          # app entry, audio session setup
  bundle.dart        # models + hand-written fromJson
  content.dart       # loadBundle(), progress read/write
  audio.dart         # TTS + SFX helpers
  game_host.dart     # type -> widget registry, level sequencing
  games/
    find_same.dart   # the game screen (StatefulWidget)
assets/content/
  bundle.json
  sfx/yay.mp3
  sfx/oops.mp3
test/
  bundle_test.dart   # content validation
  game_test.dart     # tap logic
```

### Forward compatibility — required in v1

An old app receiving a future bundle must not crash:

```dart
final playable = bundle.games.where((g) => registry.containsKey(g.type));
```

Without this, a v2 bundle referencing a new game type crashes every v1 install in the field.
This must exist in v1 even though there is no backend yet.

This also enables **dark shipping**: compile a new game type's widget into a release, leave it
unreferenced by the bundle, then activate it later via a content update on installs already in
the wild.

## Game design

### Interaction

1. Screen loads, target letter renders large and centered at top.
2. Prompt plays automatically ("Find the letter A").
3. Option cards render below as oversized tappable areas (min 120pt square, generous padding).
4. **Correct tap:** card pops (scale), gains a persistent "found" highlight, `yay.mp3` plays,
   card locks.
5. **Wrong tap:** card shakes briefly, `oops.mp3` plays (gentle, not buzzer-like), nothing
   locks, no penalty.
6. **All correct found:** celebration animation, spoken praise, auto-advance after ~2s.
7. Tapping the target letter replays the prompt.

**No timer. No score. No lives. No fail state.** A 3-5 year old cannot recover from one, and
it teaches avoidance rather than letters.

Celebration uses built-in `AnimationController` (card pop plus a small star burst). No
`confetti` package unless it feels flat in the hand.

### Difficulty ladder

Ordered by **visual confusability**, which is the actual learning axis for letterforms:

| Levels | Content | Cards | Correct |
|---|---|---|---|
| 1-3 | Uppercase, highly distinct (A, O, X, S, T) | 3 | 1 |
| 4-6 | Uppercase, distinct | 4 | 1 |
| 7-9 | Uppercase confusables (M/W, E/F, P/R, C/G) | 4 | 2 |
| 10-12 | Lowercase, distinct (a, o, s, t) | 4 | 1 |
| 13+ | Lowercase confusables (b/d, p/q, n/u, m/w) | 4-6 | 2 |

Uppercase first: uppercase letterforms are more visually distinct from one another and are
conventionally learned first.

**Font:** a font whose lowercase `a` and `g` are single-story, matching how children are
taught to write them. The default system font is not. Matters from level 10 onward.

### Progress

`shared_preferences` holding a JSON blob: the set of completed game IDs and the current level.
Sequencing is "first game whose ID is not in the completed set." In v2 this blob is what gets
POSTed to the backend.

## Backend (v2, not built now)

The contract above already fixes its shape:

- `GET /bundle?since=<version>` → the same JSON, or `304 Not Modified`.
- `POST /progress` → the blob the app already stores locally.

`main.go` + `handlers.go` + `store.go`. Postgres with a `bundles` table holding a JSONB payload
and a version integer. No Redis — a weekly-changing bundle is a CDN's job, or an in-memory map.
No layered architecture until a second consumer exists.

## App Store constraints (Kids Category)

- Parent gate required on anything leaving the app (external links, purchases).
- No third-party analytics or ads.
- No data collection from children.

v1 has no network, so it complies by construction. These constrain v2's design.

## Verification

Two tests, both runnable with `flutter test`:

1. **`bundle_test.dart`** — loads the shipped bundle and asserts: every referenced asset path
   exists, every game has at least one `correct: true` option (an unwinnable level is the worst
   possible bug here), and every game's `type` is in the widget registry.
2. **`game_test.dart`** — the tap state machine: correct taps lock and accumulate, wrong taps
   change nothing, completion fires only when all correct options are found.

The bundle validator is written in Dart rather than Go so v1 needs no second toolchain. It
ports to Go when the backend validates uploaded content.

## Open items

None blocking. Deferred decisions are listed in Non-goals with their triggers.
