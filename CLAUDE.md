# Games repo — project guide for Claude Code

Static games for Jacob (3rd grade), deployed with GitHub Pages from `main` (root).
Live: `https://achett2.github.io/games/` · Spell Slice: `https://achett2.github.io/games/spell-slice/`

- `index.html` (root) is a small hub that links the games.
- `.nojekyll` makes Pages serve the files exactly as they are.

## Spell Slice (`spell-slice/`)
A Fruit-Ninja-style English game: words or letters fly up, and the player swipes (or taps) the right ones.
**Word Chop** is the second mechanic: a word sits on a wooden log, and the player swipes down between
letters to split it (syllables, base | ending, vowel teams).

### Architecture
- **Everything is in `spell-slice/index.html`** — HTML, CSS, and JS inline. No build step, no framework, no JS deps (Google Fonts only).
- Sections in the script, in order: utils → CONTENT (word lists) → SKILLS/TOPICS → STATE → QUESTION GENERATORS → CHAINS → ROUND PLANNING → AUDIO/SPEECH → ENGINE (canvas) → HUD → SCREENS → SELF-TEST.
- **Curriculum = `TOPICS`**: Foundations (`vc` Vowels & Consonants, `nv` Nouns, Verbs & Adjectives, `sp` Subject & Predicate), then Unit 1 weeks `w1`–`w5`. Each topic has ordered `stages` (skill ids), `warm` (foundation skills recycled as 2 warm-up review questions at the start of Unit 1 rounds; Foundation topics have none) and `chains`.
- **Skills** (`SKILLS`) each have a generator in `GEN`. A question is either
  `{kind:"slice", items:[{label,target,why}], ...}` or `{kind:"chop", word, cuts:[gap indexes], ...}`. A chop with `units:[words]` chops a sentence between words (Subject | Predicate) instead of between letters.
  Every wrong option carries a `why` string, which is shown when it's sliced. `q.why` is the explanation shown on completion.
- **Chains** (`CHAINS`) are multi-step questions built on one word, e.g. RABBIT: slice the vowels → V/C code → chop rab|bit. They take `ok(skillId)` so later steps are only added once that skill is unlocked.
- **Mastery**: `S.sk[id].m` (0–1) goes up by 30% of the remaining gap on a first-try success and is multiplied by 0.55 on a miss; `rm` tracks recent misses. `need(id)` weights question selection toward weak skills. A topic's stage moves forward when the current skill reaches m ≥ 0.55 after 3+ questions, or after 2 rounds anyway, so progress is never blocked by perfection.
- **Fruit launch one at a time** (`launchOne` / `refillLaunchQueue`): the next fruit flies only after the current one is sliced or drops off screen, so fruits never overlap. Letters of a word come up in word order; other options are shuffled. Unsliced answers cycle back. `PACE` is the seconds one fruit spends in the air. Vowels & Consonants skills set `q.perWave = 2` (two fruit at once, left/right lanes) and `q.speed = 0.75` (faster) in `beginQuestion`.
- Nothing is locked: every topic and Mixed Review are playable from the start (`unlocked()` returns true). Mixed Review draws from each topic's reached stages, weighted by `need`.
- The "Let's lock it in" card (`showExplain`, shown after a question with mistakes) keeps its GOT IT button disabled for 3 seconds with a countdown.
- Spelling questions use the browser's `speechSynthesis` (🔊 button).
- **Sort** questions (`kind:"sort"`, skill `diph_sort`, Week 5): one word at a time; drag it into one of 4 bins (OW/OU/OI/OY) drawn along the bottom, or tap a bin while the word is in the air. `SORT_WORDS` includes the class worksheet words; `sortBins(w)` can return two bins (cowboy). Wrong drops fall away and come back.
- **Say it** questions (`kind:"say"`, skill `diph_say`): after each sort (preferring a word he missed) and in the `diph` chain. Uses `SpeechRecognition`; `judgeSound()` is deliberately generous (any alternative matching the sound family or the word counts) and treats long O ("oh") as wrong for OU/OW. There's a grown-up override button and Skip; without a microphone it becomes an ungraded self-check. Settings → "Say-it practice" turns it off.

### Working on it
- Validate the JS after editing: extract the `<script>` block and `node --check` it.
- Run `__selfTest()` in the browser console. It generates every skill/chain many times and checks invariants (a target exists, no duplicate options, every wrong option has a `why`, chop cuts are in range, VC/CV and word lists are consistent). It should return `[]`.
- Bump `BUILD` (renders bottom-right) once per change the user should verify. iOS caches hard, so reload with `?cb=N`.
- Content rules: Y is treated as a consonant in Foundation drills (letter drills and `F1_WORDS` exclude Y). Week 4 `ow` = long O (snow); week 5 `ow` = diphthong (cow), so use `groupOf(t, wk)` / `teamSound(t, wk)`.
- Sentences (`SENTS`) tag words `{w:n}` noun, `{w:v}` verb, `{w:a}` adjective. Conjunctions and interjections (`NO_FRUIT`: and, but, boo, wow…) stay in the sentence text but never become fruit.
- Subject & Predicate sentences (`SP_SENTS`) mark the split with ` | ` and use the same n/v/a tags; the self-test checks the predicate starts at the verb.
- State: `localStorage["ss_state_v1"]` (skills, topic progress, settings, best score). Settings has "Reset progress".
