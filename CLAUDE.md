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
- **Curriculum = `TOPICS`**: Foundations (`vc` Vowels & Consonants, `nv` Nouns & Verbs), then Unit 1 weeks `w1`–`w5`. Each topic has ordered `stages` (skill ids), `warm` (foundation skills recycled as warm-ups) and `chains`.
- **Skills** (`SKILLS`) each have a generator in `GEN`. A question is either
  `{kind:"slice", items:[{label,target,why}], ...}` or `{kind:"chop", word, cuts:[gap indexes], ...}`.
  Every wrong option carries a `why` string, which is shown when it's sliced. `q.why` is the explanation shown on completion.
- **Chains** (`CHAINS`) are multi-step questions built on one word, e.g. RABBIT: slice the vowels → V/C code → chop rab|bit. They take `ok(skillId)` so later steps are only added once that skill is unlocked.
- **Mastery**: `S.sk[id].m` (0–1) goes up by 30% of the remaining gap on a first-try success and is multiplied by 0.55 on a miss; `rm` tracks recent misses. `need(id)` weights question selection toward weak skills. A topic's stage moves forward when the current skill reaches m ≥ 0.55 after 3+ questions, or after 2 rounds anyway, so progress is never blocked by perfection.
- **Fruit launch one at a time** (`launchOne` / `refillLaunchQueue`): the next fruit flies only after the current one is sliced or drops off screen, so fruits never overlap. Letters of a word come up in word order; other options are shuffled. Unsliced answers cycle back. `PACE` is the seconds one fruit spends in the air.
- Topics unlock after one round of the previous topic. Mixed Review unlocks after 2 topics and draws from every unlocked skill, weighted by `need`.
- Spelling questions use the browser's `speechSynthesis` (🔊 button).

### Working on it
- Validate the JS after editing: extract the `<script>` block and `node --check` it.
- Run `__selfTest()` in the browser console. It generates every skill/chain many times and checks invariants (a target exists, no duplicate options, every wrong option has a `why`, chop cuts are in range, VC/CV and word lists are consistent). It should return `[]`.
- Bump `BUILD` (renders bottom-right) once per change the user should verify. iOS caches hard, so reload with `?cb=N`.
- Content rules: Y is treated as a consonant in Foundation drills (letter drills and `F1_WORDS` exclude Y). Week 4 `ow` = long O (snow); week 5 `ow` = diphthong (cow), so use `groupOf(t, wk)` / `teamSound(t, wk)`.
- State: `localStorage["ss_state_v1"]` (skills, topic progress, settings, best score). Settings has a grown-ups "Unlock all" switch and "Reset progress".
