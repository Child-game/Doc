# Docs

Design and planning documents for the kids learning app.

Code lives in [Child-game/App](https://github.com/Child-game/App).

## What's here

**[`specs/`](specs/)** — the design. What is being built and why, the decisions
and the reasoning behind them. Written before any code, and it is the document
to read first.

The 2026-08-11 spec covers: the JSON content contract between app and backend,
why server-driven UI was rejected, the offline-first constraint that drove the
architecture, the difficulty ladder for letter learning, and an explicit list of
what was left out of v1 with the trigger for adding each.

**[`plans/`](plans/)** — the implementation plans specs get turned into. Task by
task, with the code and the tests, written so the work can be picked up cold.

## The load-bearing decisions

Recorded here so they don't have to be rediscovered:

**Content is data, not UI.** Levels live in a JSON bundle the app parses. Adding
levels needs no code change and, once the backend exists, no App Store release.
Adding a new *kind* of game does need a release — the widget has to be compiled
in — and no architecture avoids that, which is why server-driven UI was dropped.

**The app must work offline.** Children use it in cars and on planes. That single
constraint means content is on-device, which is what turns "the server drives the
UI" into "the server ships a content bundle."

**The local-to-remote seam is one function.** `loadBundle()` prefers a downloaded
file over the one shipped in the app. Nothing writes that file yet. When the
backend arrives it only has to drop a file at that path — no refactor.

**Old apps must survive new bundles.** The app skips any game type it doesn't
recognise. Without that, shipping a new type would crash every install already
in the field.

## Backend

Not built yet. Its shape is already fixed by the contract in the spec:

- `GET /bundle?since=<version>` → the same JSON the app already parses, or `304`
- `POST /progress` → the set of completed level ids the app already stores

Content validation belongs on the server: port the `shipped bundle` test group
from the app's `test/bundle_test.dart` and run it on upload. Those invariants —
every level winnable, every correct card matching its target, no distractor
secretly matching — are what stop a broken level reaching a child.
