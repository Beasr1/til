# How I like to study

Applies to every course folder in here — `fingerprint/`, `git/`, `docker/`, `face/`, and
whatever comes next. Read this before adding or editing a single file.

This is a **learning** repo. It is not documentation for a system I'm building, and it is
not a place for project notes. Everything here should still make sense to a stranger with
no access to my work.

---

## The contract

**Teach me, don't brief me.** Write as a teacher, not as a peer reporting findings. That
means explaining things I might already know and letting me skim, rather than assuming and
leaving me stuck.

**Why before how. Analogy before math.** If a concept has an intuition, the intuition comes
first and the formalism second. A reader who understands *why* the field does something can
rederive the *how*; the reverse isn't true.

**Motivate before you define.** Never open with a definition. Open with the problem that
made the definition necessary, then name it. A term introduced this way sticks; a glossary
entry dropped cold does not.

**Earn every claim.** Cite the standard, the paper, or `file:line` in a named repo. If a
number is uncertain, say so in the sentence rather than a footnote. Never invent a
benchmark figure, a line number, or a citation — a plausible-looking wrong number is worse
than an admitted gap, because I'll repeat it in a meeting.

**Say when something isn't solved.** The honest state of a field is part of the material.
Where results are soft, or where a method fails, that goes in the main text, not a caveat
at the bottom.

**Verify before you write, not after I object.** Anything factual — a standard's contents,
a metric definition, a model's input contract, a dataset's size, whether a method is
deprecated — gets checked against a primary source before it goes in. Search for it. Read
the spec, the model card, the repo. Writing it from memory and presenting it in a
confident table is the failure mode this whole folder is vulnerable to, because I will
quote these files at people.

When sources disagree, **that disagreement is the lesson** — put both in, say which the
evidence supports and why. Don't silently pick one.

**Question my questions.** Don't treat what I ask for as settled just because I asked.
If I've assumed something wrong, say so before building on it. If a decision I've made
conflicts with how the field actually does it, tell me and cite the standard — I'd rather
be corrected than accommodated. If my question is ambiguous, ask me to sharpen it rather
than guessing; asking a good question is my job and I want to be held to it.

**Don't rewrite this file or the material on a passing remark.** A change to conventions
or to a factual claim needs a reason that survives checking. If I say something offhand
that contradicts what's here, tell me what the conflict is and ask, rather than quietly
editing. Corrections to *facts* need a source; corrections to *my own instructions* need
me to have actually meant them.

---

## Structure

One folder per subject. Inside it:

```
README.md                  the index — read first
01-<topic>.md              numbered chapters, dependency-ordered
...
NN-glossary.md             reference, not reading
NN-exercises.md            answers to every Check yourself, plus things to try
```

Rules that have earned their place:

- **Chapter numbers are permanent.** A chapter added later gets the next free chapter number
  even if it belongs conceptually in the middle. Renumbering chapters breaks cross-references
  in every other file. Say where it belongs in the README's reading order instead.
- **The reference files always sit last** — glossary, then exercises, in that order, holding
  the final numbers in the folder. They are not chapters and they are not read in sequence, so
  a new chapter takes the slot below them and **they get renumbered upward**. That is the one
  renumbering that is allowed, and it is required: an exercises file stranded in the middle
  makes the reading order unreadable.

  When you renumber them, fix every reference in the same pass — the README's reading order,
  the "answers are in …" line, and any `file NN` mention in a chapter. Grep for the old number
  before you finish.
- **Cross-link by file and section** — "file 05 §5.2", as a real markdown link where it's
  a different file.
- **Every course opens with a foundations chapter, `01`.** It teaches what the subject is,
  its vocabulary and its working model end to end, for a reader who has never used it, so
  that no later chapter is the first place a core term appears. It says near the top who can
  skip it. The README marks it as the prerequisite for everything else. A course that is
  missing one gets it under the permanence rule above: the next free number, listed first in
  the README's reading order.

### Folder depth — split late, and only for big chunks

Subject folders are good and I want more of them as this grows. But a folder must earn
itself by **classifying a large chunk**, not by tidying a few files.

| Depth | Example | When |
|---|---|---|
| `subject/` | `git/`, `docker/` | Default. A subject that fits one reading order |
| `subject/area/` | `face/spoof/`, `face/match/` | The subject splits into areas that are genuinely separate courses — different metrics, different literature, read independently |
| `subject/area/sub/` | — | Almost never. Needs a real argument |

The test for splitting: **would someone read one of the two areas and never open its
sibling?** Spoof detection and face matching pass — different failure modes, different
papers, different metrics, and plenty of people only ever need one. Chapters within a
course fail — you read them in order.

It is a test about *siblings*, not about the parent. Once the parent has shared chapters
everyone reads those, and that's the design working, not a failed split.

### No sibling dependencies; shared material lives in the parent

The corollary, and the rule that actually keeps this clean as it grows:

> **A child folder must have no sibling dependencies.** If two children need the same
> material, that material belongs in the parent — not duplicated in both, and not
> cross-referenced sideways.

Note what that does and doesn't say. A child may absolutely depend on its **parent** — that
is a prerequisite, and prerequisites are normal. What it may never depend on is a
**sibling**, because that makes the reading order depend on which door you came in
through.

So there are exactly two legitimate directions for a link:

| Direction | Verdict |
|---|---|
| child → parent | ✅ Fine. "Face detection is covered in `face/03-detection.md`" |
| child → sibling | 🚩 Smell. Fix it, don't link it |
| parent → child | ✅ Fine, in the index |

**A sibling cross-reference means one of two things**, and both are fixable:

1. The shared material belongs in the parent. Move it up, and have both children point at
   it. This is the common case.
2. The split was wrong and these are one area. Merge them.

What it must *never* mean is "go read a **sibling** first". Pointing up to the parent is
fine and expected; pointing sideways means the material is in the wrong place.

A worked example. `face/spoof/` and `face/match/` will both need face detection, bounding
boxes, crop conventions and capture quality. Those are shared, so they belong in `face/`
itself as numbered chapters, with each child assuming them. What stays in `spoof/` is
what only spoof needs — PAI taxonomy, APCER/BPCER, the cues, PAD models.

This means a parent folder can hold **both** an index README **and** its own chapters, once
it has shared material to hold. It starts as just an index and grows chapters when the
second child arrives and the overlap becomes visible. Don't pre-build the parent's chapters
for a child that doesn't exist yet — you'll guess the boundary wrong.

Three consequences:

- **Start flat.** A new subject is one folder with numbered chapters. Split it only when
  the second area actually arrives, not in anticipation.
- **Every folder that holds chapters holds a README.** A parent starts as a short index
  naming its areas and how they relate, and gains its own chapters only once it has shared
  material to hold.
- **Prefer a longer chapter over a new nesting level.** Two related pages in one file beat
  two files in a new folder.

Numbering restarts inside each area. `face/spoof/01-…` and `face/match/01-…` both exist and
that's correct — they're separate courses.

### Reading order across levels

Once a parent has chapters, a reader arriving at a child needs to know where to start. Two
requirements, and they're cheap:

- **The parent README says what its own chapters are for** — prerequisite (read first) or
  background (read when you need them). Pick one per chapter and say which.
- **Each child README opens by naming what it assumes.** One line: "Assumes `face/01–03`
  — detection, crops and capture quality." A reader who has them skips it; a reader who
  doesn't knows where to go.

Without this the structure is correct and the reader is still lost, which is the same as
being wrong.

### The README of every course must have

| Section | Why |
|---|---|
| What this is, and that it's reference material | Stops it drifting into project notes |
| The reference implementations, as a table with links | I want the primary sources |
| Reading order, in parts, with "after this you can…" | Tells me what I'm buying with my time |
| **If you're short on time** — 2–4 named paths | I usually am |
| **The one-paragraph summary of everything** | The thing I reread before a meeting |
| **How to use me** — example questions to ask | Reminds me the notes are a starting point |

---

## Inside a chapter

**Open with the problem.** One or two sentences on what breaks without this concept.

**Tables for anything comparative.** Three or more things being compared, or any
term/meaning pairing, is a table. Prose comparisons are unreadable.

**Diagrams where structure matters.** Mermaid for flow and hierarchy, fenced ASCII for
anything spatial or numeric. Colour the important node — don't make me hunt for the point.

**Callouts, used sparingly:**

- `> **Teacher's aside.**` — the mental model correction, the thing everyone gets wrong,
  the "people assume X; the truth is narrower" paragraph. These are the most valuable
  parts of the notes. Aim for one per chapter, not four.
- `> ⚠️` — the trap. Coordinate conventions, reversed metrics, the bug that costs a day.

**Every chapter ends with `## Check yourself`** — 4–6 questions in ascending difficulty.
Not recall prompts: questions that only work if the model actually formed. "What breaks,
mechanically, if you do X?" is a good one. "What is X?" is not.

Answers go in the exercises file, worked rather than stated, and it ends with a
**Questions worth asking me** list of prompts for going deeper.

---

## Register

Match the fingerprint course — that one's calibrated.

- Direct and plain. Contractions are fine.
- **Bold** for a term being defined or a genuine warning. Not for emphasis, and not three
  times a paragraph.
- No hype, no "it's important to note", no summarising what you just said.
- Short paragraphs. A wall of text is a wall.
- British spelling.
- Don't state something once and then restate it as a caution and again as a summary. Say
  it once, in the right place.

---

## What doesn't belong

- Project specifics — service names, our thresholds, our infrastructure. If a lesson came
  from something we built, extract the *general* principle and drop the context.
- Anything that rots: internal URLs, ticket numbers, scratch-repo paths. Cite the upstream
  source, not a local checkout of it.
- Marketing numbers presented as measurements.
- Sections defending a choice nobody questioned.

---

## When I ask you to add to a course

1. Read the existing README and at least two chapters before writing anything. Match what's
   there rather than what you'd naturally write.
2. Check whether it's a new chapter or an edit to an existing one. Prefer editing —
   fragments are worse than a slightly long chapter.
3. If it's new, add it to the README's reading order table in the same commit.
4. Add its **Check yourself** questions to the exercises file with worked answers. A
   chapter without them is unfinished.
5. Tell me plainly what you weren't sure about. I'd rather have a gap I know about.
