# Project files

A trailer lives in plain files, so the user, another agent or a fresh session can re-render it, change one shot, or pick the job up after a break. The names below are defaults; if the project has conventions, follow them and note the mapping in `TRAILER.md`.

## Layout

```text
<product repo>/
  tools/trailer/              committed: everything needed to re-render
    README.md                 how to re-render, how to change one thing, how long it takes
    TRAILER.md                the brief, the storyboard, decisions in the user's words, the sources ledger
    BAR.md                    the trailer bar the critic grades (format in review.md)
    timeline.<ext>            the grid, chapters, hits and slots: the one source of truth
    shots/                    one script per shot, grouped by chapter
    lib/                      clock, capture loop, camera helpers, cut, mix
    overlay/                  the titles layer: a page that draws frame N on request
    music/                    the score and its renderer, or notes on the licensed track
    .work/                    gitignored: the scratch folder
      takes/<shape>/          <shot>.mkv lossless takes and <shot>.json manifests
      preview/<shape>/        three stills per shot
      audio/                  music, effects stem, premix, mix, measurement reports
      titles/                 <shape>.mov, the RGBA titles layer
      review/<shape>/v<NN>/   contact sheets, title frames, the checks file and the verdict for one cut
      profiles/               capture browser profiles (hold the copied user data)
<delivery folder>/            outside the repo, where the user wants it
  <Product>-Trailer-16x9-v03.mp4
  <Product>-Trailer-16x9.mp4  the approved master, once there is one
  <Product>-still-1-<what>.jpg ...
  CUTS.md
  previews/                   preview stills and samples shown to the user
```

Gitignore `tools/trailer/.work/` and any render output inside the repo. A gitignore entry with a trailing slash does not match a symbolic link, so if the scratch folder is ever moved and linked back, add an entry without the slash too.

## TRAILER.md

```markdown
# <Product>: trailer brief
Created: <date> · Status: <proposal | approved | filming | cut vNN | delivered>

## Goal
<one sentence: who should watch it, and what they should do after>

## Format
- Length: <seconds>, one edit rendered per shape
- Shapes: <16:9 1920x1080 | 9:16 1080x1920 | ...> at <fps>
- Where it will be posted: <platforms>, and where the files go: <delivery folder>

## Must show
1. <feature or moment; the critic's first criterion checks this list>

## Story
- Direction: <montage | transformation | story | app arc>
- Hook (frame 1 to about 3 s): <what the viewer sees>
- Storyboard: | Time | Chapter | What you see | Title on screen |
- End card: <logo, call to action, where to get it>

## Sound
- Music: <source>, <tempo> BPM; voice: <none | who, with consent>

## Decisions (the user's words)
- <date>: "<quote>" -> <what changed>

## Sources ledger
| Element | Source (filmed, composed, licensed, product asset) | License | Proof |

## Never show
- <personal names and data, real accounts, other brands, unfinished features>
```

## The shot list

One row per slot, in timeline order. The shot id is also the script name and the take file name.

```markdown
| Slot | Shot id | Beats | What we see | Camera | Action and timing | HUD | Stage (place, time, weather) |
| 1 | open-dive | 3 | snowy peaks at sunrise, fast | dive past a ridge, slight roll | none | off | peaks west of spawn, morning, clear |
```

## The timeline file

Code, not prose, so every tool reads the same numbers. Use the project's language.

```ts
export const FPS = 60;
export const BPM = 144;
export const FRAMES_PER_BEAT = (FPS * 60) / BPM;          // 25
export const TOTAL_BEATS = 144;                            // 36 bars of 4/4
export const TOTAL_FRAMES = Math.round(TOTAL_BEATS * FRAMES_PER_BEAT); // 3,600 = 60.000 s

export const CHAPTERS = [
  { id: 'open', startBeat: 0 },
  { id: 'world', startBeat: 8, title: 'A world that never ends' },
  // ...
];

/** Moments the picture, the titles and the music all hit, in beats. */
export const HITS = { boom: 3, title: 5, logo: 133, playNow: 138, final: 140 };

/** The cut, in order; a slot may be a half beat. */
export const CUT = [
  { shot: 'open-dive', beats: 3 },
  { shot: 'open-tnt', beats: 5 },
  // ...
];

/** Slot starts rounded on the cumulative beat, so rounding never drifts. */
export function placedCut() {
  let beat = 0;
  return CUT.map((s) => {
    const startFrame = Math.round(beat * FRAMES_PER_BEAT);
    const endFrame = Math.round((beat + s.beats) * FRAMES_PER_BEAT);
    const placed = { ...s, startBeat: beat, startFrame, frames: endFrame - startFrame };
    beat += s.beats;
    return placed;
  });
}

/** Run before every render step. */
export function checkCut() {
  const total = CUT.reduce((n, s) => n + s.beats, 0);
  if (total !== TOTAL_BEATS) throw new Error(`the cut is ${total} beats, not ${TOTAL_BEATS}`);
  const starts = new Set(placedCut().map((p) => p.startBeat));
  for (const c of CHAPTERS) if (!starts.has(c.startBeat)) throw new Error(`chapter ${c.id} does not start on a cut point`);
  if (new Set(CUT.map((s) => s.shot)).size !== CUT.length) throw new Error('a shot is used twice');
}
```

When a shape needs slots of its own (a shot that does not survive reframing, replaced by another), give each slot an optional `shapes` list, or keep one `CUT` per shape sharing the same chapters and hits, and run `checkCut()` for every shape.

## Takes manifest

Each take writes `<shot>.json` next to `<shot>.mkv`:

```json
{
  "file": ".work/takes/16x9/open-tnt.mkv",
  "frames": 182,
  "inPoint": 57,
  "marks": { "sound:world.explosion": 75 },
  "sounds": [{ "frame": 75, "id": "world.explosion", "kind": "play", "position": [12, 64, -3], "distance": 9.5 }],
  "build": "<commit id, plus a marker if the tree was dirty>",
  "script": "<hash of the shot script>",
  "recorded": "<ISO time>"
}
```

`inPoint` is the first frame the edit uses; frames before it are lead-in. Compare `script` with the current script to find stale takes when resuming.

## CUTS.md

Kept in the delivery folder, newest last. Every cut the user saw stays on disk. Each shape is its own cut, with its own version numbers and verdicts.

```markdown
| Shape | Version | File | When | What changed | Checks | Critic | User |
| 16x9 | v01 | <Product>-Trailer-16x9-v01.mp4 | <time> | first full cut | pass | FAIL: T3, T6 | not shown |
| 16x9 | v02 | <Product>-Trailer-16x9-v02.mp4 | <time> | re-took build-wool, craft-sleep; places count 9 | pass | WIN | "it's awesome"; drop the hours line |
| 9x16 | v01 | <Product>-Trailer-9x16-v01.mp4 | <time> | the 16x9 v02 timeline, reframed | pass | FAIL: T11 | not shown |
```

## Status line

After every phase or cut, one line for the user and the log:

```text
<shape> v<NN> <WIN|FAIL|SAMPLE|PREVIEW>: failing <criterion ids|none>; punch <count>; scratch <GB>; next: <action> (<estimate>)
```

## The re-render README

Write it before handover. It answers, in this order:

1. How to render everything, and how long it takes.
2. What each step makes and where (a table: step, command, output).
3. How to change one thing without redoing the rest: one shot, one title, the music, the order or a slot's length.
4. Inputs that are not in the repo (the data copy to film in, a licensed track) and how to pass them, by option or environment variable rather than a hard-coded personal path.
5. How to make the other shapes later, if they were planned but not rendered.
6. How much disk the scratch folder takes, and that it is safe to delete once the user is happy.
