# Sound Catcher — SwiftUI build spec (MVP)

*App #1 of the slate. Build first: cheapest, letter-free (no reading skills required to play), and it validates the shared chassis every later app reuses.*

---

## 1. Context

**Who:** early learners (preschool / pre-K, roughly ages 3–5).
**Goal of this app:** Build the *oral* foundation of reading — phonemic awareness — with zero letters on screen, so it's a pure listening activity with no reading required. The child hears a sound, taps the matching picture.
**Why first:** No drawing, no physics, no text-rendering — just audio + picture-tap. It forces us to build the three shared-chassis pieces (`ContentLibrary`, `AudioEngine`, `MasteryService`) that Stretchy Sounds, Trace & Read, and Story Time will all reuse.

**Non-negotiable design rules (apply to every app in the slate):**
- **Audio-first.** Young children can't read menus. Every instruction is spoken; nothing requires reading.
- **Errorless.** A wrong tap is never a buzzer or a fail screen. It re-models the correct sound (stretched) and lets the child retry.
- **Calm.** Muted palette, soft transitions, one warm human voice. No confetti, no streaks, no timers, no scores shown to the child.
- **No ads, no IAP, no network, no accounts.** Fully on-device.
- **Parent gate.** Settings are behind a press-and-hold-then-drag gesture a young child can't trigger by accident.

---

## 2. Tech baseline & decisions

| Decision | Choice | Rationale |
|---|---|---|
| UI | SwiftUI | Modern, fast to iterate, fine for this UI complexity |
| Min OS | iPadOS 17+ | `@Observable`, SwiftData, modern animation APIs; targets current devices so no need to support old OS |
| Device | iPad only, landscape-locked | Tablet form factor; landscape gives big side-by-side cards |
| Audio | `AVAudioPlayer` w/ pre-recorded clips | The product *is* the voice — must be a real human, not synth (see §6) |
| Persistence | SwiftData (or Codable→JSON) for `MasteryStore` | Small data; SwiftData is clean and future-proofs the cross-app review layer |
| Content | Bundled JSON + asset catalogs | No backend; words/images/audio ship in the app |
| Architecture | MVVM-lite with `@Observable` | One `RoundViewModel` drives the loop; engine objects are injectable so they can be lifted into a shared Swift package later |
| Dependencies | None | Keep it buildable solo and offline |

**Chassis note:** put `ContentLibrary`, `AudioEngine`, and `MasteryService` in their own folder/group now. When you start Stretchy Sounds, lift that group into a local Swift package (`LearningKit`) shared across all apps. Design their public APIs as if that's already true.

---

## 3. The core loop (what the child does)

1. The child taps the mascot on the start screen → adaptive session begins.
2. The mascot speaks one prompt: *"Which one says sssss… like sun?"* (a small replay button lets the child hear it again, infinitely).
3. Two or three big picture cards slide in (e.g. sun / moon / fish), generously spaced.
4. The child taps a card. Tapping any card first says its word ("moon").
	- **Correct:** the word is said clearly, the card gives a small calm grow-and-settle animation, the mascot reacts happily, a soft chime. Advance.
	- **Wrong:** no buzzer. The target sound is re-modeled *stretched* ("listen… ssss… ssssun"), the card the child wanted gently wiggles back, and they try again. (Errorless hint ladder, §5.)
5. ~6–8 rounds = a ~90-second session. A quiet token/sticker accrues every few rounds; never a prize-wheel.

---

## 4. Game modes

MVP ships **only Find-the-Sound (initial)**. The others are designed in but built later — keep `GameMode` an enum from day one so the round generator and views branch cleanly.

| Mode | Prompt | Interaction | Status |
|---|---|---|---|
| `findInitialSound` | "Which says /sss/?" | tap the picture | **MVP** |
| `findFinalSound` | "Which *ends* with /t/?" | tap the picture | later |
| `rhymeMatch` | "Which rhymes with cat?" | tap the picture | later (cheap, high engagement — do next) |
| `oddOneOut` | "Which does NOT start with /m/?" | tap the picture | later |
| `segmenting` | "Tap the sounds in cat" | tap N bubble-boxes L→R, then it blends | later |
| (option) `voiceEcho` | after a win, "Say it!" | mic record + playback | later |

---

## 5. The two algorithms that make or break it

### 5.1 Round generator + distractor fairness
The single biggest correctness risk: if distractors are too close, the game degrades into guessing.

- Phonemes are grouped into an ordered **teaching sequence** matching Stretchy Sounds: continuous consonants first → `m, s, f, n, l, r` + short `a`, then stop consonants `t, p, b, d, k, g`, then remaining.
- Each `WordCard` is tagged with its initial/medial/final phoneme and a rhyme family.
- For a round on target phoneme `P`:
	- **Correct card:** a word whose initial phoneme == `P`, weighted toward words on phonemes `MasteryService` says the child is wobbly on.
	- **Distractors:** words whose initial phoneme is **maximally distant** from `P` early on (different manner *and* place of articulation, e.g. `/s/` vs `/m/` vs `/f/` are all fine and distinct). Never a near-minimal contrast (`/s/` vs `/f/`, `/m/` vs `/n/`) until late levels.
	- Define a small hand-authored **confusability table** (or a tier list: "safe distractors for /s/ = [m, d, b, g, …]"). Don't try to compute articulatory distance generically for MVP — hand-author it; it's ~20 sounds.
- Card count ramps 2 → 3 (never more than 3 for this age group).

### 5.2 Difficulty governor
- Target **~75% success**. Keep a rolling window (last ~8 rounds).
- Above ~85% → introduce 3rd card / advance to next phoneme in the sequence / tighten distractor distance one notch.
- Below ~60% → drop to 2 cards, widen distractor distance, repeat the current phoneme, surface the hint sooner.
- Persist per-phoneme accuracy to `MasteryService` (this is the seed of the cross-app review service).

### 5.3 Hint ladder (errorless)
1. 1st wrong tap → re-model the target sound stretched, wiggle the wrong card back.
2. 2nd wrong tap → re-model + the correct card does a subtle "breathe"/pulse to draw the eye.
3. 3rd → gentle auto-assist: correct card pulses brighter and a soft "here it is" plays; any tap on it counts and moves on. The child never dead-ends.

---

## 6. Audio (the thing to over-invest in)

- **Production:** record a real, warm, slightly-slowed human voice. Per phoneme you need: (a) a **clean isolated** sound, (b) a **stretched** version ("ssssss"), (c) the **whole word** said normally. Per word: the spoken word + ideally a stretched lead-in for re-modeling.
- **Dev placeholder:** wire everything to `AVSpeechSynthesizer` first so the loop is testable today; swap in recorded clips file-by-file. Keep `AudioEngine`'s API identical for both backends.
- `AudioEngine` API (sketch):
	```swift
	protocol AudioEngine {
		func play(_ clip: AudioClip) async   // phoneme / stretchedPhoneme / word / cue
		func replayLastPrompt() async
		func stopAll()
	}
	```
- Duck/queue so prompt → card-word → feedback never overlap muddily. Pair correct/feedback with light `CoreHaptics`.

---

## 7. Screens

1. **StartView** — full-bleed calm scene, centered mascot (tappable, gently idle-animating), spoken "Tap Ollie to play!", a dim parent-gate cog top-trailing. One tap starts the session.
2. **RoundView** — top: small mascot + large circular **replay** button + a spoken prompt (rendered visually only as a speech-bubble *waveform*, never as readable text the child must parse). Center: 2–3 `PictureCardView`s in an `HStack`, big hit targets, generous spacing. Bottom: a quiet progress strip (dots).
3. **FeedbackOverlay** — non-modal; correct = grow+chime+mascot smile; wrong = re-model + wiggle (no overlay blocking retry).
4. **ParentGateView** — press-and-hold the cog 1.5s, then drag a slider to unlock. Opens:
5. **ParentDashboardView** — mode toggles, active phoneme sets, per-sound mastery (green/amber/red), session length, voice volume, "reset progress". Text here is for the adult.

`PictureCardView`: illustration only — **no word text ever on the child's screen**. Large rounded tile, soft fill, the illustration centered, springy tap scale.

---

## 8. Data model

```swift
enum Phoneme: String, Codable, CaseIterable {   // IPA-ish keys
	case m, s, f, n, l, r, a, t, p, b, d, k, g    // …extend later
}

struct WordCard: Identifiable, Codable {
	let id: String          // "sun"
	let word: String        // "sun" (adult/debug only)
	let imageName: String   // asset-catalog name
	let wordAudio: String   // clip filename
	let initial: Phoneme
	let medial: Phoneme?
	let final: Phoneme?
	let rhymeFamily: String?    // "-un"
}

enum GameMode: String, Codable { case findInitialSound, findFinalSound, rhymeMatch, oddOneOut, segmenting }

struct Round: Identifiable {
	let id = UUID()
	let mode: GameMode
	let target: Phoneme
	let correct: WordCard
	let cards: [WordCard]   // includes correct, shuffled
}

@Model final class MasteryRecord {   // SwiftData — the cross-app review seed
	var phoneme: String
	var attempts: Int
	var correct: Int
	var lastSeen: Date
	var accuracy: Double { attempts == 0 ? 0 : Double(correct)/Double(attempts) }
}
```

Bundled `words.json` = array of `WordCard`. MVP needs **~20 words** spanning the continuous-consonant set (sun, sock, sea / man, moon, mug / fan, fish, fox / net, nut / leg, lip / red, rug …) — each with a clear, unambiguous illustration.

---

## 9. File tree

```
SoundCatcher/
	App/SoundCatcherApp.swift
	Models/
		Phoneme.swift  WordCard.swift  GameMode.swift  Round.swift  MasteryRecord.swift
	Chassis/                ← lift into LearningKit package when building app #2
		ContentLibrary.swift  ← loads words.json + asset lookups
		AudioEngine.swift     ← protocol + AVAudioPlayer impl + TTS placeholder impl
		MasteryService.swift  ← read/update SwiftData, "what needs review" queries
		ParentGate.swift      ← hold-then-drag gesture + state
	Engine/
		RoundGenerator.swift     ← §5.1 incl. confusability table
		DifficultyGovernor.swift ← §5.2
		HintLadder.swift         ← §5.3
	ViewModels/RoundViewModel.swift
	Views/
		StartView.swift  RoundView.swift  PictureCardView.swift
		MascotView.swift  FeedbackOverlay.swift  ProgressStrip.swift
		ParentGateView.swift  ParentDashboardView.swift
	Resources/
		words.json
		Assets.xcassets/   ← one illustration per word + mascot states
		Audio/             ← phoneme (clean+stretched), word, cue clips
```

---

## 10. Build order (within the app)

1. Models + `ContentLibrary` loading `words.json` (5 words to start).
2. `AudioEngine` with the **TTS placeholder** — get sound out of the device immediately.
3. `RoundGenerator` (initial-sound only) + `RoundViewModel` + a bare `RoundView` with tappable cards. **Playable end-to-end here.**
4. Hint ladder + errorless feedback + `FeedbackOverlay`.
5. `MasteryService` + `DifficultyGovernor` wired to round selection.
6. `StartView` + `MascotView` + `ProgressStrip` polish.
7. `ParentGate` + `ParentDashboardView`.
8. Swap TTS → recorded human-voice clips, word by word. Expand to ~20 words.
9. **Playtest with children.** Tune distractor distance + governor thresholds from real behavior.

---

## 11. Acceptance criteria

- [ ] Launches to StartView; one tap on the mascot begins a session with spoken instruction. No reading required anywhere in the child flow.
- [ ] A round presents 2–3 illustration-only cards; the prompt is spoken and replayable on demand, infinitely.
- [ ] Tapping the correct card → spoken word + calm grow animation + chime + advance.
- [ ] Tapping a wrong card → **no buzzer**; target sound re-modeled stretched; child can immediately retry; after 3 misses the correct card is gently auto-assisted so the child never dead-ends.
- [ ] Early rounds only ever contrast maximally-distinct sounds (verify: across 50 generated rounds, no round pairs near-minimal contrasts before the late tier).
- [ ] Difficulty governor keeps rolling success near ~75% (verify with a scripted sim of a fixed-accuracy "player").
- [ ] Per-phoneme accuracy persists across launches via `MasteryService`.
- [ ] Settings are unreachable without the hold-then-drag gate; a single stray tap never opens them.
- [ ] No network calls; airplane-mode launch is fully functional.
- [ ] Runs landscape on iPad with large, well-spaced touch targets (≥120pt cards).
- [ ] `ContentLibrary` / `AudioEngine` / `MasteryService` have no SoundCatcher-specific imports — ready to lift into a shared package.

---

## 12. Deliberately deferred

Rhyme-match (build next — cheap, fun), final-sound, odd-one-out, segmenting (the embodied tap-per-sound), voice-echo capture, the richer sticker book. None block MVP; all reuse the same engine and DB.
