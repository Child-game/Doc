# Critter Camp — Mini Design Doc (v0.2)

Educational iOS app, ages 3–6. One module for v1: **Animals** — ten levels, one hero mascot.
Stack: **Flutter + Rive** (2D). No backend, no third-party SDKs, no text the child must read.

---

## 1. How the game will be

**The map.** One vertical, scrollable winding road (Candy-Crush style) with ten nodes,
split into 2–3 themed chapters (the farm, the pond, the forest). The map opens already
scrolled to the current node — the only pulsing thing on screen, so a non-reader knows
where to tap. Completed nodes show a check + sticker, future nodes sit visible but locked.

**A level (2–4 minutes).** Enter from map → mascot speaks the task ("Where is the cow?")
→ child taps among 3–4 illustrated choices → celebrate or gently retry → back to the map,
node marked done, next node pops open (~1.5s ritual: mascot walks the road, node colours in,
sticker flies on).

**No-fail rules (non-negotiable):**
- No lives, timers, streaks, scores, or stars. Progress only goes up.
- Wrong answer → the level gets *easier*: fewer choices, stronger audio hint. Success guaranteed within a few tries.
- Any level replayable forever (replay is how this age masters).
- Everything for adults (settings, progress report, links) sits behind a gate:
  arithmetic written in words — "what is fourteen plus seven?"

**Ergonomics:** tap is the only gesture. Hit areas ≥ 120pt, ≥ 100pt apart, taps debounced
~600ms. Orientation locked. Autosave on every completion.

## 2. How character animation will look

One hero: a **bipedal animal** (bear/bunny/fox standing on two legs), built as a **Rive
rig** — vector art, bones + mesh deform, one state machine. Start by remixing a CC-BY
marketplace rig (keep rig + state machine, redraw the vectors), not from scratch.

**Idle = alive.** The character is never a statue:
- **Breathing loop**, 3–4s: chest rises 1–2% scale, shoulders follow slightly *later* —
  the offset between belly/chest/head is what reads as alive. First and last frame identical (no loop pop).
- **Blink** every 3–5s on a *randomised* timer (never synced to the breath loop).
- **Fidgets** every 30–60s: ear flick, head tilt, look-around. 3–5 of them, blended in over ~0.2s.

**Reactions (state machine inputs, Duolingo-style):**

| Trigger | Animation |
|---|---|
| `tap` on mascot | bounce + giggle pose |
| `correct` | big celebration — jump, arms up, eyes closed smile |
| `retry` | warm encouraging nod — never a sad/fail pose |
| `isSpeaking` (bool) | jaw opens/closes, slight head bob |
| idle timeout | fidget from the pool |

Every state blends back to idle; breathing runs *under* all states as a layer, so it is
authored once.

## 3. How voice will work

**Audio is the interface** — there is no text fallback anywhere.

- **Mascot voice = one recorded human voice** (warm, exaggerated, slow). Record it
  ourselves first; upgrade to pro VO ($100–500) only if kids don't respond.
- Every tappable thing **names itself when touched** ("Cow!", "Level three!", "Stickers!").
- Line types per level: task line, 2 hint lines (progressively more helpful), praise lines
  (varied, 4–5 so they don't repeat), chapter title line.
- **Mixing:** 3 channels — VO / SFX / music. VO ducks music, is never cut off by SFX.
  All VO for the current chapter preloaded; a late voice line feels broken to a child.
- **Mouth sync:** amplitude-driven — while a clip plays, `isSpeaking` is on and jaw
  openness follows loudness. No phoneme lip-sync in v1 (invisible at this art size).
- Files: plain `.m4a` in assets, named `vo_<level>_<line>.m4a`, mapped from the level JSON.

## 4. Build order (unchanged)

1. Mascot remixed in Rive, idle+blink+tap working in a Flutter screen on iPhone
2. Audio spine (3 channels, ducking) + one voiced interaction
3. Map with 10 nodes + JSON progress
4. One full level loop → then a real 3–6yo tests it (the true milestone)
5. Duplicate to ten levels, record VO, polish, TestFlight

Full research doc: `~/Desktop/critter-camp-design-doc.md`
