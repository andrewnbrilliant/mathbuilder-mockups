# Argument Builder mock-ups — context for the next iteration

This file is a handover. It describes a family of six HTML mock-ups in this repo, the
conventions they share, the design decisions that were argued out while building them, and the
verification discipline that caught the bugs. It is written to be handed to a Claudius instance
(or any coding agent) together with an instruction like:

> Read `ARGUMENT-BUILDERS.md` in https://github.com/andrewnbrilliant/mathbuilder-mockups,
> then iterate on `<file>` as follows: …

Everything here is a **mock-up**. None of it is Brilliant content and none of it is production
code. The point is to show learning designers and engineers what an interaction could feel like,
at a stable URL they can open on a phone.

---

## 1. What the family is

A learner is given a situation, and instead of answering one question, they **assemble an
argument**: a chain of short sentences with blanks, where the blanks they fill decide which
sentence they get next. The origin was a Brilliant WordProblem asking whether a table of
falling-stone data was linear and whether it was proportional. These extend that idea into
branching, multi-stage arguments with real solving in the middle.

| Mock-up | File | What it does |
|---|---|---|
| Relationship type | `relationship-argument-builder.html` | Three-column table (x, Δy, y). Two sentences, three blanks reach proportional / linear / quadratic / exponential. |
| Quadratic motion **(current)** | `quadratic-argument-builder-transformer.html` | Model → count solutions → solve in a QuadraticEquationTransformer-style workspace → interpret. |
| Quadratic motion *(frozen)* | `quadratic-argument-builder.html` | The earlier tile-only solve step. Kept deliberately as a before/after snapshot. **Do not "fix" it** to match the current one unless asked. |
| Sequence monotonicity | `sequence-monotonicity-builder.html` | Classify a sequence, then justify it — universally, or with counterexamples the learner constructs. |
| Linear system | `linear-system-argument-builder.html` | Set up two cost equations → compare rates → solve in a SystemOfEquationsTransformer-style workspace → interpret. Four endings. |
| Model choice | `model-choice-argument-builder.html` | Choose linear / quadratic / exponential *first*, get that template, solve with pills suited to it, interpret. |

Live at `https://andrewnbrilliant.github.io/mathbuilder-mockups/<file>`.

The repo also holds many older, unrelated mock-ups (`motion-*`, `units-*`, `cyclist-*`). They
are not part of this family.

---

## 2. How to work on them

- Each mock-up is **one self-contained HTML file**: no build step, no imports, no shared CSS.
  Duplication between files is intentional — each must open standalone from a URL.
- KaTeX 0.16.9 comes from the jsdelivr CDN. All maths goes through `katex.renderToString`.
- Editing: open the file, change it, commit **that one file by name** (never `git add -A`), push
  to `main`. A GitHub Actions workflow deploys Pages in roughly 60 seconds.
- End commit messages with the attribution trailer your harness specifies.
- If several agents are working at once, `git pull --rebase origin main` on rejection. Never
  force-push.

---

## 3. Shared architecture

Every file follows the same shape. Read `quadratic-argument-builder-transformer.html` first —
it is the most complete example.

**Layout.** A 375px white "phone" centred on grey, with a left rail card (scenario switcher) and
a right rail card (branch map + designer notes). The rails are for the author's benefit and are
not part of the interaction.

**State.** A single `state = { key, picks, checked, tf }` object. `key` is the scenario, `picks`
maps blank-id → chosen value, `tf` is the solve workspace where one exists. Rendering is
`renderAll()` → rebuild innerHTML → re-attach handlers. There is no framework.

**Blanks.** `blankOrder()` (or `ORDER`) returns the blank ids **for the current branch**, so the
list itself changes with the learner's answers. `activeBlank()` is the first unfilled one. The
bottom bank shows the options for that blank only. Tapping a filled blank clears it and
everything after it.

**Progressive reveal.** `frag(key, html)` wraps a fragment and adds a fade class the first time
that key is seen; a `seen` Set tracks which have appeared. Clearing a blank prunes `seen` so
later fragments animate again.

**Deep links.** Every file parses query parameters in `applyQuery()` — that function is the
definitive grammar, the notes below are a summary.

- Five files: `?s=<scenario>&picks=<id>:<value>,<id>:<value>&checked=1`
- `relationship-argument-builder.html` is the odd one out: `?d=<dataset>` and **positional**
  `picks=<s1>,<s2>,<s3>`.
- The three with a workspace also take `steps=` to replay working-out:
  - quadratic: `steps=divide:-5,factor,split,subtract:1,side:right,add:4`
  - linear: `steps=row:B,substitute,subtract:2n,subtract:3,row:A,substitute,combine`
  - model choice: `steps=divide:50,log:2`

Scenario keys: relationship `proportional | linear | quadratic | exponential`; quadratic
`lands | grazes | misses`; sequence `linear-up | linear-down | alternating | constant`; linear
system `taxis | gyms | scooters | posters`; model choice `bacteria | tank | ball`.

**Solve workspaces.** Three files embed a small equation transformer imitating a real EPU
package: action pills, one step row per transformation, undo, and refusal messages. They run
actual algebra over a small equation representation — they are not scripted paths, and an
illegal move is refused **with a reason**, never silently ignored.

---

## 4. Design principles

These were argued out during review. Several are counter-intuitive and were the result of a
specific correction, so please do not quietly undo them.

**Evidence before conclusion.** Sentences pair an observation blank with a conclusion blank:
"the ratio ⟨varies⟩, so the relationship is ⟨linear, not proportional⟩". The learner commits to
the evidence before naming the thing.

**The learner's answer reshapes the question.** Claim two solutions and you get two root blanks
plus a rejection step; claim none and the solve stage is never generated and the later stages
renumber. This is the core mechanic — not decoration.

**A wrong answer must still produce a complete, coherent argument that fails on Check.** Never
a dead end, never a block. Getting to the end of a false argument and seeing it go red is the
lesson.

**Never tell the learner why a solution might not count.** The quadratic originally read "Time
cannot be negative, so we reject ⟨t = −1⟩" — that hands over the entire insight. Now both
solutions are offered as neutral toggles to keep or drop, and the bank caption says only "Tap
the solutions you want to keep". The same toggle appears on *every* branch that produces a
solution; if it only appeared where a solution was bogus, its presence would be the answer. See
`acceptTogglesHTML()`.

**The workspace is scaffolding; the final-answer blanks are the commitment.** Nothing is
auto-filled from the working-out. This keeps the argument chain intact, lets a learner answer
without using the workspace, and means the workspace can never strand anyone.

**Solve the equation the learner built, not the correct one.** Workspaces seed from the learner's
own model blanks (`builtEq()`, `startRows()`) and re-seed when the model changes, keyed by a
`modelSig()` signature. Wrong models then fail *in the mathematics*: a parallel pair of cost
lines reduces to a visible `5 = 9`, and an unfactorable quadratic gets "No real factors — the
discriminant is negative".

**Don't put arithmetic in the way of the argument.** The discriminant step asks for
`b² − 4ac ⟨>⟩ 0`, not the value 625. Only the sign decides anything.

**Reveal trailing words with their blank, not after it.** "For longer distances, ⟨_⟩ are
cheaper" is unanswerable until "are cheaper" is on screen. Progressive reveal must not withhold
the words a blank depends on.

**Cut blanks that cost nothing to answer.** "EasyTaxis and ⟨SpeedyCars⟩ both cost…" was busywork
with only two providers; it is static text now.

**Grade by rule, not by answer key.** Counterexample pairs are accepted if each statement is true
and non-vacuous and between them they rule out both increasing and decreasing — so non-adjacent
indices and either operator orientation work, and nobody has to guess the "expected" pair.
Kept-solution sets are graded as "exactly the ones that are ≥ 0". Rules survive edits to the
numbers; answer keys do not.

**Reveal the graph only at the end.** Shown earlier it gives away the solution count or the
crossing. Shown last it turns the algebra back into the situation. Before Check it mirrors what
the learner chose; after Check it shows the truth. Never draw a marker the learner has not
earned, and if an answer falls outside the plotted range, pin it to the edge as a hollow marker
rather than letting it vanish — an absent marker reads as agreement.

**Prefer typed (spanny) input where the operand space is open.** The linear workspace originally
picked operands from a fixed list, which meant some equations had no sensible option. It now has
a small keyboard. Dynamically-tuned clicky options were rejected as the alternative: options
chosen for "what makes sense now" telegraph the next move.

### Brilliant conventions worth respecting

- Spanny keyboard digits run **1 2 3 4 5** then **6 7 8 9 0** — zero after nine.
- At most one tall key.
- MathBuilder-style tile banks are ordered smallest → largest.
- In JSON text fields Brilliant uses `$$…$$` and `\dfrac`; in Elm assets, single `$…$`. Not
  directly relevant to these HTML mock-ups, but it is what the real content does.

---

## 5. Verification — read this before claiming anything works

**Deep links are not sufficient, and they have already hidden one real bug.** Query parameters
set state *before the first render*, which masks anything that depends on the order things are
created in. A workspace was being built on the first render, before any blank was filled, so it
always seeded from the scenario's correct equation — every deep-link test passed while the
actual UI ignored the learner entirely.

So: **drive the real DOM with clicks.** Copy the file to `/tmp` (never add a harness to the
repo), append a script that clicks real elements and reports what rendered, and read it back:

```bash
python3 - <<'PY'
src = open('/tmp/mathbuilder-mockups/<file>.html').read()
open('/tmp/test.html','w').write(src.replace('</body>', open('/tmp/harness.js').read() + '\n</body>'))
PY

"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu \
  --virtual-time-budget=4000 --dump-dom "file:///tmp/test.html" | grep -o "<title>[^<]*</title>"
```

A harness that works (report through `document.title`; avoid names the page already uses):

```html
<script>
const nz = s => s.replace(/[−]/g,'-').trim();          // normalise the maths minus
const out = [], say = x => { out.push(x); document.title = out.join(' || '); };
const bankTiles = () => Array.from(document.querySelectorAll('#bank .tile'));
function tapIt(txt){
  const t = bankTiles().find(e => nz(e.textContent) === txt);
  if (!t) { say('MISS(' + txt + ') bank=[' + bankTiles().map(e=>nz(e.textContent)).join(',') + ']'); return; }
  t.click();
}
try {
  ['-5','15','20','0','>','two real solutions'].forEach(tapIt);
  say('WS=' + nz(document.querySelector('.workspace').innerText).replace(/\s+/g,' '));
} catch(e) { say('THROWN ' + e.message); }
</script>
```

**And look at the pictures.** Screenshot and open the PNG — do not assume a render is fine:

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu \
  --screenshot=/tmp/shot.png --window-size=440,1500 --virtual-time-budget=3000 \
  "file:///tmp/mathbuilder-mockups/<file>.html?<deep link>"
```

Screenshots have caught things no assertion did: a substituted term rendering `c = −3(−5) + 8`
because a sign was applied twice; an off-range answer marker silently vanishing so a wrong graph
looked right; labels crossed by their own plot lines; KaTeX splitting "for n ≥ 1" across a line
break. Check at 375px width specifically — that is the phone frame the interaction lives in.

Worth testing every time you change a branch: forward play by clicking alone; back-editing an
early blank (it clears everything after, and must re-seed the workspace); each scenario reaching
its correct ending; at least one refusal message; and a wrong answer producing a coherent,
red-on-Check argument rather than a crash.

---

## 6. Open questions and known gaps

- **Grading against truth vs internal consistency.** Every step is graded against the scenario's
  correct answer. So a learner who mis-models and then solves *their own* equation perfectly
  still gets red root blanks. Grading later steps for internal consistency and flagging only the
  model is a genuine alternative and has not been decided.
- **The relationship and sequence mock-ups predate several principles above** — they have no
  workspace and no toggle step. That is fine, but if they grow a solve stage, the principles in
  §4 apply.
- **`quadratic-argument-builder.html` is a frozen earlier design.** Leave it alone unless asked.
- **The model-choice mock-up has no "no solution" branch** for its exponential or linear cases;
  the linear-system one shows how that would look.
- **Nothing here has been tried on a real phone**, only in a 375px headless frame, and no
  learner has used any of it.

---

## 7. Where the real interactives live

These mock-ups imitate Elm packages in the Brilliant `elm-package-universe` repo. Read them
before changing how a workspace behaves — the point is to imitate the real thing, not invent:

- `src/BrilliantUniverse/QuadraticEquationTransformer/V0/` — actions bank, step rows, split
  cases, undo, `x = [] or x = []` final answer. Its `Tests/__snapshots__/*.snap` files show the
  rendered scene graph, which is the quickest way to see what the UI actually contains.
- `src/BrilliantUniverse/SystemOfEquationsTransformer/V0/` — the same for a 2×2 system, with
  `Replace with A+B` and `Substitute`, and pills dragged onto a specific row.
- `src/BrilliantUniverse/LinearEquationTransformer/V0/` — the single-equation case.
- There is **no** ExponentialTransformer. The one in `model-choice-argument-builder.html` is
  invented (Take log base, Raise to power, alongside the usual moves).

In the real packages the operand is typed into a MathyKeyboard field, and action pills are
dragged onto a row. The mock-ups approximate dragging with "tap a row to make it active, then
tap a pill".
