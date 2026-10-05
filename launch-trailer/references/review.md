# Review

Read before the first preview and before every cut. The loop: preview stills, a sample, then for every cut the mechanical checks, contact sheets and title frames, a fresh critic against the bar, fixes, and the user's watch. Builders never grade their own cut.

## Contents

- The trailer bar
- Mechanical checks
- Contact sheets and the watch-through
- The user's review
- When a shot keeps failing
- The final checklist

## The trailer bar

Write `BAR.md` from the brief before the first cut. Each criterion is observable in the review files, binary, and motivated by the brief.

```markdown
# Bar: <Product> trailer
BAR-VERSION: v1

## T1 Brief fit
- PASS when: every item under "Must show" in TRAILER.md appears and reads in at least one slot, cited by slot and frame; length, shapes and frame rate match the brief.
- FAIL signs: a must-show item missing, unreadable, or only named in a title.
```

**The core, in every bar:**

| Id | Criterion | PASS when | FAIL signs |
| :--- | :--- | :--- | :--- |
| T1 | Brief fit | every must-show item appears and reads; length, shapes, frame rate as briefed | a must-show missing or only named in text |
| T2 | Real footage | every slot comes from a take of the build named in the takes manifest, or is a declared graphic slot | mockups, stock, generated frames, retouched captures |
| T3 | Frame integrity | no broken, black, frozen, flickering or half-loaded frame outside planned fades and holds; no stray interface, debug text, cursor, toast, chat or notification the shot is not about | a missing chunk on a first frame, a loading sky, a projectile filling the lens, a toast from the previous shot |
| T4 | Opening | frame 1 moves and shows the opening TRAILER.md describes; the product name reads when the brief says it should | a static or empty first frame; a splash, loading, menu or tool screen the brief did not ask for; the name later than briefed |
| T5 | Rhythm and motion | every cut point and title slam is on the grid (mechanical report); every shot moves from its first frame; no shot repeats; energy builds to the finale | a cut between beats, a frozen slot, the same shot twice, a flat ending |
| T6 | Readable and true | every title and callout reads at phone size in its time on screen, inside the safe area, spelled right; every number matches the product's data | text over busy footage without contrast, clipped text, a wrong count |
| T7 | End card | opaque backdrop; exactly the content TRAILER.md lists, centred as one block; held as long as the brief says, and long enough to read twice | footage showing through, a missing or unplanned item, text that has gone stale, a card gone before it can be read |

**Add from this pack when the brief motivates it:**

| Id | Criterion | PASS when |
| :--- | :--- | :--- |
| T8 | Show the payoff | each interface moment cuts to its result on screen and lasts no longer than its slot in the storyboard |
| T9 | Light | every slot's subject reads on a small screen; night and cave slots show silhouettes; one consistent look |
| T10 | Sound | loudness on the brief's target (about -14 LUFS when it sets none) and true peak at or under -1 dBTP in the final file, audio offset 0, effects on their frames, music hits on the picture's hits (graded from measurements; the user grades the feel) |
| T11 | Vertical framing | in the 9:16 cut, subjects and titles sit inside the vertical safe area in every slot |
| T12 | Variety | no two neighbouring slots share both scale and direction of movement; no shot used twice (an opening may tease the finale's subject from another angle) |

Most bars land between 7 and 10 criteria. The bar is versioned like a ratchet: publish v1 before the first cut, raise or clarify criteria between cuts with a one-line reason in `CUTS.md`, and loosen one only with the user's explicit approval, logged with the reason.

## Mechanical checks

Run on every cut of every shape, before the critic. Commands: [recipes/ffmpeg.md](recipes/ffmpeg.md), Mechanical checks. Write the results to `.work/review/<shape>/v<NN>/checks.md`, next to that cut's sheets and frames; the critic reads that folder.

| Check | Passes when |
| :--- | :--- |
| Spec | H.264 High, `yuv420p`, BT.709 tags, constant frame rate as briefed, frame count and duration exactly the timeline's, AAC 48 kHz stereo |
| Faststart | the `moov` atom comes before `mdat` |
| Decode | no errors |
| Black | stretches only at planned fades, each listed with its slot |
| Frozen | stretches only at planned holds; a held last frame from a short take is a FAIL unless planned |
| Cut points | every slot start has a detected cut, or is a join between similar-looking shots you have looked at; every detection off a slot start is a flash, a title slam or an in-shot event you have looked at |
| Titles on frames | the frame-counter probe shows every output frame carrying its own titles frame |
| Bitrate per second | no second far above its neighbours (a sign of corrupted or noisy frames) |
| Motion per slot | no slot mostly static unless planned |
| Brightness per slot | the darkest slots are listed and looked at closely |
| Loudness | the brief's target (about -14 LUFS integrated when it sets none) and true peak at or under -1 dBTP, measured on the final file |
| Offset | 0 samples between the final file and the mix at a few sharp transients |
| Text that must not appear | optional: OCR on every 15th frame or so finds nothing on the never-show list (personal names, debug strings, and any version number or date the brief did not ask for); first prove the OCR on a frame that does contain text |

## Contact sheets and the watch-through

- **Sheets**: the first, 25%, 50%, 75% and last frame of every slot, in rows, with a separate file mapping rows to slots. Small enough to see a whole chapter at once.
- **Full size**: every cut point and the two frames either side; every title, callout and the end card at the frame where each is fullest; the darkest slots; anything the checks flagged.
- **The final file**: one frame per second of the encoded master, as a sheet, because the encode is what people see.
- **Watching**: if your environment can play video to you, watch the whole cut at normal speed; otherwise the sheets and frames stand in, and the user's watch is the real one.

What the BlockHaven watch-through caught before the first review: a callout claiming 10 places where the game's own tools list 9; the end card's backdrop letting the world's logo show through; titles drifting a frame late from 0:51 (and a doubled frame there); a +1.5 dBFS peak clipping in the effects stem before the limiter; and eight shots to film again (a missing chunk on a first frame, a furnace that did nothing in its slot, an arrow filling the lens for two frames, a sleep that ended in black, a teleport showing its loading sky, a view swapping two frames before a cut).

## The user's review

- Give the path and say plainly what is final and what may still change.
- Ask what works and what does not, record each note in `TRAILER.md`, and turn it into a change with its cost ([edit-and-titles.md](edit-and-titles.md), the table of what each change redoes).
- Apply changes as a new version; prove untouched frames are unchanged when the cut was already approved.
- Answer questions; edit only when asked.
- Keep raw footage, earlier cuts and previews until the final cut is approved.

## When a shot keeps failing

Repeating the same fix wastes takes. When a slot fails two cuts in a row, climb this ladder, skipping rungs when the cause is clear:

1. **Staging**: time of day, weather, brightness, clear the stage, hide overlays, warm-up frames.
2. **Framing**: camera position, field of view, path and speed, the side the light comes from.
3. **Timing**: marks, in-point, lead frames, a speed ramp, a hold, triggering on another frame.
4. **Set dressing**: build or place the subject (a house, a line of explosives, creatures on their marks), pin AI that wanders.
5. **Replace or drop**: another feature for the slot, or give the slot's beats to its neighbours. Tell the user what changed.

If a criterion stays stuck after that, spawn a fresh agent with the brief, the bar, the last two verdicts and the shot's script, and ask for the root cause, the rung to climb, and what must visibly change in the next take.

## The final checklist

Copy it into the handover and tick every line:

```text
- [ ] The user approved this cut, recorded in TRAILER.md
- [ ] Length, shapes, frame rate, voice, music and end card as the brief says
- [ ] Spec: H.264 High, yuv420p, BT.709 tags, constant frame rate, exact frames and duration, AAC 48 kHz stereo, faststart
- [ ] No decode errors; black and frozen stretches only where planned
- [ ] Every cut point on the grid; every titles frame on its picture frame
- [ ] Every title spelled right, readable on a phone, inside the safe area; every number checked against the product
- [ ] No stray interface, debug text, cursor, notifications, personal data or other brands; version numbers and dates only where the brief asks, and still current
- [ ] End card: opaque, the brief's content as one block, held long enough to read twice
- [ ] Loudness on target, true peak at or under -1 dBTP in the final file; audio offset 0
- [ ] Sources ledger complete: music, effects, fonts, voice, each with its license
- [ ] Stills and platform copies made from the approved master
- [ ] CUTS.md up to date; every cut the user saw still on disk
- [ ] Scripts committed with the re-render README; no renders, secrets or personal paths in the repo
- [ ] Scratch size reported; nothing deleted; previews listed
```
