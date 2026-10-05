# Delivery

Read before the first cut is encoded, and again at handover. Commands are in [recipes/ffmpeg.md](recipes/ffmpeg.md).

## The master

| Property | Value |
| :--- | :--- |
| Container | MP4 with `+faststart` (the index before the media, so it plays while downloading) |
| Video | H.264 High, `yuv420p`, limited range, BT.709 primaries, transfer and matrix tagged in the container and written into the stream |
| Quality | CRF 15 to 16 at preset `slow`: BlockHaven (CRF 15) came out at about 30 Mbit/s, 227 MB for 60 s; Lightning Sortie (CRF 16) at about 12.5 Mbit/s, 93 MB for 58 s |
| Frame rate | constant, as the timeline says: 60 for games and fast motion, 30 is fine for most apps |
| Keyframes | every 2 seconds |
| Audio | AAC-LC, 256 kbit/s, 48 kHz, stereo |
| Length | exactly the timeline's total frames |
| Loudness | about -14 LUFS integrated unless the brief or platform sets another target, true peak at or under -1 dBTP in this file |

## Platform copies

| Copy | How |
| :--- | :--- |
| Upload copy | CRF 18 to 20 with a bitrate cap, when the master is too large to send or upload comfortably |
| Vertical 9:16 | film the shape and lay its titles out for it; a blurred-fill version of the 16:9 master is only a fallback |
| Square or 4:5 | as vertical |
| Teaser | its own short timeline reusing the takes and the music's hook |
| Stills | six to eight JPEGs at chosen frames of the master: the opening, the title, three or four chapter highlights, the end card |
| Share image | 1200x630 from a clean frame without titles, with the product's logo laid on top; check it full size and small, as link previews show it |
| Thumbnail | a 1280x720 frame with one strong subject and no small text |
| Loop | 3 to 6 seconds, seamless, for a page header |

Platform limits and recommended specs change; check them when uploading rather than building them into the plan.

## Naming and versions

- Every cut gets a new file: `<Product>-Trailer-16x9-v03.mp4`. Never overwrite a cut the user has seen; earlier cuts are the only proof of what they approved.
- The approved cut is copied to a stable name (`<Product>-Launch-Trailer-16x9.mp4`), with stills named `<Product>-still-<n>-<what>.jpg` beside it.
- Log every cut in `CUTS.md` ([project-files.md](project-files.md)).
- When the user approves a cut but asks for a small change, show that nothing else moved: compare decoded-frame hashes and per-frame quality against the approved cut. After BlockHaven's end-card change, frames 0 to 3401 and all of the sound were bit-identical to the approved version, frames 3402 to 3449 differed only by encoder rounding (49.8 dB or better), and frames 3450 to 3599 were the new card. Lightning Sortie's late change to two titles kept the audio bit-identical and restored one title frame that had shifted, so only the two changed titles differed.

## Storage

- **Delivery folder**: outside the repo, where the user wants it (a movies or downloads folder). Videos never go into version control.
- **Scratch folder**: gitignored, inside the trailer tooling. Tell the user its size before it surprises them: BlockHaven's reached 15 GB (takes for one shape plus partial takes for a second, earlier cuts, capture profiles), Lightning Sortie's 11 GB (4.8 GB of takes, 4.1 GB of segments, 1.2 GB of title images).
- **Keep raw footage until the final cut is approved**, and ask before deleting or moving any of it; the takes are what make a small fix take minutes instead of a re-shoot. When asked to move it (for example to a folder that syncs to cloud storage), a move on the same disk is instant; leave a link at the old path if the scripts read it, and warn that files the sync service keeps online-only must download before a re-render.
- **Previews** shown along the way (stills folders, samples) are listed in the handover, so the user knows what can go.
- **The scripts** are committed with the re-render README; inputs that are not in the repo (the data copy to film in, a licensed track) are passed by option or environment variable, never a hard-coded personal path. Scan the diff for secrets and personal paths before committing.

## Budgets

| | BlockHaven | Lightning Sortie |
| :--- | :--- | :--- |
| Length and shots | 60 s, 54 filmed shots and one graphic slot | 58 s, 19 slots |
| Tooling | built from scratch | reused the first trailer's design and clock |
| To the first full cut | about 3 hours | about 1 hour 40 minutes, the first 45 spent scripting while the machine was busy |
| Polish and changes | about 2 hours the next morning | 50 minutes of encode fixes before delivery, then about 12 minutes for the one change |
| Full re-render with tooling ready | about 40 minutes | |
| Scratch disk | 15 GB | 11 GB |

Both were 3D browser games filmed at 2x; a 2D game filmed at 1x captures faster and needs less disk. Quote estimates from these, scaled by shot count, and say what is included (capture, music, titles, checks, handover).

## Handover report

```markdown
**The video:** <path>: <duration> s, <width>x<height>, <fps> fps, <size> MB, H.264 High with AAC stereo;
<LUFS> LUFS, true peak <dBTP>. Stills: <paths>.
**What it shows:** one line per chapter.
**Not filmed, and what was done instead:** <list or none>.
**Checks:** spec, decode errors, black and frozen frames (planned ones only), cut points on the grid, titles
on their frames, loudness and peak, audio offset 0, critic verdict <WIN, round n>, final checklist.
**Changes you asked for:** <each request, and what changed>.
**To re-render or change one thing:** <README path>.
**Scripts:** committed as <commit>; nothing rendered is in the repo.
**Scratch:** <size> at <path>, kept until you approve the final cut and agree it can go; earlier cuts and previews at <paths>.
```
