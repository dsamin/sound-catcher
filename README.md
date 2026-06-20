# Sound Catcher

> *"Which one says ssss?"* Listen, tap the picture, catch the sound — no letters needed.

**App 1 of 5 — BUILD FIRST.** Sound Catcher is the cheapest app in the slate, it's letter-free by design (so written letters never gate the activity), and it establishes and validates the **shared chassis** every later app reuses.

---

## What it is

Sound Catcher is a calm, audio-first iPad game that builds **oral phonemic awareness** — the ear-skill of hearing, isolating, rhyming, and segmenting individual speech sounds. It is the *oral bedrock of reading*: young children develop the ability to hear `/s/`–`/u/`–`/n/` before printed letters can map onto sounds for them. There are **no letters on screen, ever.** A warm human voice asks a question, two or three big picture cards appear, and the child taps the one that matches the sound.

It is designed to be played for ~90 seconds at a time, and it is built so the child essentially cannot fail.

---

## Who it's for

**Early learners (preschool / pre-K, roughly ages 3–5).** Most "learning to read" apps put letters front-and-center. Sound Catcher deliberately removes letters entirely and works the *ear* first, which matches how young children develop phonemic awareness before they can read. Every win is a pure win.

This is a fully on-device app: no ads, no purchases, no accounts, no network. It runs entirely on the iPad.

---

## What it teaches

The primary goal is **oral phonemic awareness**, developed along the correct developmental ladder:

1. **Isolation** — hearing and picking out a single sound ("which one *starts* with `/sss/`?"). This is the MVP.
2. **Rhyme** — hearing that *cat* and *hat* end the same.
3. **Segmenting** — breaking a word into its sounds (`c-a-t`) and blending them back.
4. **Discrimination** — odd-one-out ("which one does *not* start with `/m/`?").

Phonemes are taught in a **continuous-consonants-first order** (`m, s, f, n, l, r` + short `a`, then the stop consonants `t, p, b, d, k, g`). Continuous sounds can be stretched and held ("mmmmm", "sssss"), which makes them far easier for a young child to hear and imitate — and this order deliberately matches the later reading/handwriting apps so the skills line up.

---

## How it works (the core loop)

1. The child taps the mascot on the start screen → an adaptive session begins.
2. The mascot **speaks one prompt**: *"Which one says sssss… like sun?"* A large replay button lets the child hear it again, infinitely.
3. Two or three **big picture cards** slide in (e.g. sun / moon / fish), generously spaced. Illustrations only — no word text.
4. The child taps a card. Tapping any card first **says its word** ("moon").
 - **Correct** → the word is said clearly, the card does a small calm grow-and-settle, the mascot smiles, a soft chime plays, and the session advances.
 - **Wrong** → **never a buzzer, never a fail screen.** The target sound is re-modeled *stretched* ("listen… ssss… ssssun"), the card gently wiggles back, and the child retries immediately.
5. After three misses on a round, the correct card is gently auto-assisted (it pulses brighter, a soft "here it is" plays) so the child **never dead-ends**.
6. ~6–8 winnable rounds make up a ~90-second session. A quiet token accrues every few rounds — never a prize-wheel, never a score the child sees.

Two algorithms make or break this loop:

- **Distractor-fairness governor.** Early rounds only contrast *maximally different* sounds (`/s/` vs `/m/` vs `/f/`), never near-minimal pairs (`/s/` vs `/f/`, `/m/` vs `/n/`) until much later. If distractors are too close, the game degrades into guessing. The MVP uses a small **hand-authored confusability table** (≈20 sounds) — e.g. `safeDistractors[/s/] = [m, n, l, r, d, b, g]` — rather than computing articulatory distance generically.
- **Difficulty governor.** Targets **~75% success** over a rolling window of the last ~8 rounds. Above ~85% it introduces a 3rd card / advances the phoneme / tightens distractor distance one notch; below ~60% it drops to 2 cards, widens distractor distance, repeats the phoneme, and surfaces hints sooner.

---

## Screens

1. **StartView** — full-bleed calm scene, centered tappable mascot (gentle idle animation), a spoken "Tap Ollie to play!", and a dim parent-gate cog in the top-trailing corner. One tap starts a session.
2. **RoundView** — top: small mascot + a large circular **replay** button + the prompt shown only as a speech-bubble *waveform* (never readable text). Center: 2–3 `PictureCardView`s in an `HStack` with big hit targets (≥120pt) and generous spacing. Bottom: a quiet progress strip of dots.
3. **FeedbackOverlay** — non-modal; correct = grow + chime + mascot smile; wrong = stretched re-model + wiggle, never blocking the retry.
4. **ParentGateView** — press-and-hold the cog ~1.5s, then drag a slider to unlock. A single stray child tap can never open it.
5. **ParentDashboardView** — for the adult: mode toggles, active phoneme sets, per-sound mastery (green/amber/red), session length, voice volume, and reset progress. This is the only screen with reading text, and it's behind the gate.

`PictureCardView` is illustration-only — **no word text ever appears on the child's screen.**

---

## Key design decisions

- **Letter-free by design.** Written letters never gate the activity here — the app works the ear and shows only pictures, matching how phonemic awareness develops before reading.
- **Errorless, always.** A wrong tap re-models the correct answer and invites a retry, so children never hit a fail state. No buzzers, no fail states, no losing. Preschool motivation is fragile; we protect it.
- **The voice is the product.** A real, warm, slightly-slowed human voice — not a synth — because phoneme modelling has to be trustworthy and human to imitate. (`AVSpeechSynthesizer` is wired in first as a swappable dev placeholder so the loop is testable on day one; recorded clips swap in file-by-file behind the same API.)
- **Distractor fairness over cleverness.** The single biggest correctness risk is unfair distractors. Hand-author the confusability table; do not over-engineer articulatory-distance math for the MVP.
- **Calm by default.** Muted palette, soft transitions, one warm voice. No confetti, no streaks, no timers, no visible scores.
- **The parent is invisible to the child.** All settings live behind a hold-then-drag gate.

---

## Shared chassis

The five apps in the slate are **not independent** — they share one core, and Sound Catcher is where that core is born and proven. Build these three engines now, in their own group, with **no Sound Catcher-specific imports**, so they can be lifted into a local Swift package (`LearningKit`) when app #2 starts:

- **`ContentLibrary`** — loads the tagged word/sound/picture database (`words.json` + asset-catalog lookups). The shared content DB.
- **`AudioEngine`** — the real-human-voice audio engine (protocol + `AVAudioPlayer` impl + TTS placeholder impl), with ducking/queuing so prompt → card-word → feedback never overlap.
- **`MasteryService`** — per-phoneme mastery backed by SwiftData; the seed of the cross-app "what needs review" spaced-repetition service.

Design every one of these public APIs as if it already lives in `LearningKit`. Sound Catcher is then "just" a mode on top of that core, and Stretchy Sounds / Trace & Read / Story Time / the memory app reuse the same DB, voice, and review layer.

```text
SoundCatcher/
 App/SoundCatcherApp.swift
 Models/
 Phoneme.swift WordCard.swift GameMode.swift Round.swift MasteryRecord.swift
 Chassis/ ← lift into LearningKit package when building app #2
 ContentLibrary.swift ← loads words.json + asset lookups
 AudioEngine.swift ← protocol + AVAudioPlayer impl + TTS placeholder impl
 MasteryService.swift ← read/update SwiftData, "what needs review" queries
 ParentGate.swift ← hold-then-drag gesture + state
 Engine/
 RoundGenerator.swift ← round selection + confusability table (distractor fairness)
 DifficultyGovernor.swift ← ~75% success target
 HintLadder.swift ← errorless re-model → pulse → auto-assist
 ViewModels/RoundViewModel.swift
 Views/
 StartView.swift RoundView.swift PictureCardView.swift
 MascotView.swift FeedbackOverlay.swift ProgressStrip.swift
 ParentGateView.swift ParentDashboardView.swift
 Resources/
 words.json
 Assets.xcassets/ ← one illustration per word + mascot states
 Audio/ ← phoneme (clean + stretched), word, cue clips
```

---

## Tech

- **Native SwiftUI**, iPad-only, **landscape-locked**, **iPadOS 17+** (`@Observable`, SwiftData, modern animation APIs).
- **Audio:** `AVAudioPlayer` with pre-recorded human-voice clips; `AVSpeechSynthesizer` as a swappable dev placeholder behind an identical `AudioEngine` API. Light `CoreHaptics` paired with feedback.
- **Persistence:** SwiftData (`MasteryRecord`) — small data, future-proofs the cross-app review layer.
- **Content:** bundled `words.json` + asset catalogs. No backend.
- **Architecture:** MVVM-lite with `@Observable`; one `RoundViewModel` drives the loop; engine objects are injectable so they lift cleanly into `LearningKit`.
- **Dependencies:** **none.** Fully offline; airplane-mode launch is fully functional.

---

## Build plan / MVP

Ship **only Find-the-Sound (initial sound)** first. Keep `GameMode` an enum from day one so the generator and views branch cleanly for later modes.

Ordered first-build cut:

1. Models + `ContentLibrary` loading `words.json` (start with ~5 words).
2. `AudioEngine` with the **TTS placeholder** — get sound out of the device immediately.
3. `RoundGenerator` (initial-sound only) + `RoundViewModel` + a bare `RoundView` with tappable cards → **playable end-to-end here.**
4. Hint ladder + errorless feedback + `FeedbackOverlay`.
5. `MasteryService` + `DifficultyGovernor` wired to round selection.
6. `StartView` + `MascotView` + `ProgressStrip` polish.
7. `ParentGate` + `ParentDashboardView`.
8. Swap TTS → recorded human-voice clips, word by word; expand to **~20 words** across the continuous-consonant set (sun, sock, sea / man, moon, mug / fan, fish, fox / net, nut / leg, lip / red, rug …), each with a clear, unambiguous illustration.
9. **Playtest with children** and tune distractor distance + governor thresholds from real behavior.

**Deferred (designed-in, built later):** rhyme-match (do next — cheap, high engagement), final-sound, odd-one-out, segmenting (tap-per-sound then blend), voice-echo capture, the richer sticker book. None block the MVP; all reuse the same engine and DB.

---

## Acceptance criteria

- [ ] Launches to StartView; one tap on the mascot begins a session with a spoken instruction. No reading required anywhere in the child flow.
- [ ] A round presents 2–3 illustration-only cards; the prompt is spoken and replayable on demand, infinitely.
- [ ] Tapping the correct card → spoken word + calm grow animation + chime + advance.
- [ ] Tapping a wrong card → **no buzzer**; the target sound is re-modeled stretched; the child can immediately retry; after 3 misses the correct card is gently auto-assisted so the child never dead-ends.
- [ ] Early rounds only ever contrast maximally-distinct sounds (verify: across 50 generated rounds, no round pairs near-minimal contrasts before the late tier).
- [ ] The difficulty governor keeps rolling success near ~75% (verify with a scripted sim of a fixed-accuracy "player").
- [ ] Per-phoneme accuracy persists across launches via `MasteryService`.
- [ ] Settings are unreachable without the hold-then-drag gate; a single stray tap never opens them.
- [ ] No network calls; airplane-mode launch is fully functional.
- [ ] Runs landscape on iPad with large, well-spaced touch targets (≥120pt cards).
- [ ] `ContentLibrary` / `AudioEngine` / `MasteryService` have no Sound Catcher-specific imports — ready to lift into `LearningKit`.
- [ ] Builds clean with `xcodebuild` and launches in an iPad simulator; the full core loop is demonstrably playable there (not just compiling).

---

## Status

**Idea spec'd + critiqued — not yet built.** A full SwiftUI build spec exists at `sound-catcher-spec.md` in this repo (numbered sections cover the data model, the two governing algorithms, audio production, screens, and acceptance). Next step is the run-to-completion build: scaffold the project, build the chassis, ship the Find-the-Sound MVP, verify it in an iPad simulator, and playtest with children.
