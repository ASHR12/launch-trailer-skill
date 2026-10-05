---
name: launch-trailer
description: >-
  Plans, films and cuts launch trailers, teasers, promos, store-page videos, social clips and other short
  movies from the user's own game or app, using only footage captured from the real running build. A lead
  agent agrees a brief and storyboard with the user (length, pacing, voice, music and end card as the user
  asks, or proposed to suit the use case and platform), scripts every shot on a music beat grid, captures
  takes deterministically (stepped time, scripted camera paths, clean interface, sharp lossless frames),
  assembles titles, an end card and mixed audio with ffmpeg, and loops preview stills, a sample cut,
  mechanical checks and a fresh critic until the cut passes a PASS/FAIL bar and the user approves.
  Instruction-only. Use when the user asks for a trailer, launch video, teaser, promo, store-page video,
  gameplay movie, cinematic, sizzle reel, app demo video or a movie made from their game. Not for
  live-action shoots.
license: MIT
compatibility: >-
  Any agent environment with a shell, ffmpeg and ffprobe; the recipes also use Node.js and Python 3.
  Browser builds need browser automation (for example Playwright driving Chromium); engine and native
  builds need the engine's own fixed-rate render route. Works best with subagents and an image-capable
  critic; without one, the user reviews the stills.
metadata:
  version: "1.0.0"
---

# Launch Trailer: short films from the real build

Turn the user's own game or app into a short film that makes people want it. You are the **Director**: you agree the brief with the user, script every shot on a beat grid, film takes from the real running build with time under your control, cut them to music with titles and an end card, and loop review until the cut passes the bar and the user approves. Every frame is the product itself: no mockups, stock footage or generated video.

This skill is instructions only. Use the tools your environment already has, and write the small tools a trailer needs (a clock, a capture loop, a cut script) inside the user's project. The recipes in [references/recipes/](references/recipes/) are tested starting points, not a library to install.

**Terms.** A **trailer** is any short film made from the product: launch trailer, teaser, promo, store-page video, social clip, sizzle reel, gameplay or app demo movie. A **shot** is one scripted moment; a **take** is one recorded clip of a shot; a **slot** is a shot's place and length in the **timeline**; a **cut** is one rendered version of the whole trailer (`v01`, `v02`, ...). The **scratch folder** holds the takes (the raw footage), stems, the titles layer and capture profiles; the **delivery folder** holds what the user keeps.

## When to use

- The user wants a trailer, teaser, promo, launch video, store-page video, social clip, sizzle reel, cinematic or demo movie of a game or app they built.
- Any build you can drive and capture: a browser game or web app, an engine project, a desktop or mobile app.
- Re-cuts: a vertical version, a shorter teaser, an updated trailer after the product changed.

Skip it for live-action shoots and for editing footage the user filmed themselves. Generated video is out of scope: this skill films the product.

## The pipeline

| Phase | You produce | Gate |
| :--- | :--- | :--- |
| 1. Brief | `TRAILER.md`: goal, audience, use case, length and pace, shapes, must-show list, opening, storyboard, voice, music, end card, decisions | the user says go (a `plan` critic first, if you can spawn one) |
| 2. Timeline | slots on a beat grid, the shot list, the hits where picture, titles and music land together | a check that the slots add up |
| 3. Tooling | access to the product, a virtual clock, a capture loop, proven on one test shot | stills from the test take look right |
| 4. Preview | three stills per shot (first, middle, last) in a folder the user can open | framing checked (by a `preview` critic or by you); the user has seen them |
| 5. Takes | one sharp, lossless take per shot, with its logged sounds and marks | every slot has a take |
| 6. Music and sound | music on the same grid; the product's own sounds placed from the takes | measured loudness and peaks |
| 7. Titles | a titles layer drawn as a pure function of the frame number | title stills checked at full size |
| 8. Sample | the opening (the first 10 to 15 seconds) with music | the user has seen the opening |
| 9. Cut | `v01`, `v02`, ...: assembled, encoded, stills taken | mechanical checks pass; a fresh critic returns WIN |
| 10. Delivery | the approved master, platform copies, stills, a re-render README | the user approves; raw footage kept |

Once the timeline is fixed, the music and the titles can be written while the takes are filmed, because they read the same timeline file. Rendering the titles layer is another browser capture: run it after the takes, not alongside them.

## Rule 0: talk before filming

Before writing capture code, reply with a proposal and wait for the user's go:

1. **Two or three directions**, each with what frame 1 shows: a spectacle montage, a transformation ("the world builds itself"), a short story, an app's problem-to-magic arc.
2. **A storyboard table** sized to the length: time, chapter, what the viewer sees, the title on screen.
3. **What the brief decides**: length and pace, voice-over and captions, music style and source (and the license, if the user wants a track they bought), and the end card's content come from the user. Where the request does not say, propose what suits the use case and platform from the ranges in [references/brief-and-story.md](references/brief-and-story.md), with a reason, for the user to confirm.
4. **Defaults to confirm**: shapes (one edit rendered per shape), resolution and frame rate, the data or world to film in (always a copy: ask the user for an export of their save, never read their browser or account), where the files go (outside version control), anything to install (a new dev dependency changes the project's lockfile), permission for any debug-only change to the product, the branch the trailer scripts go on, and where the product's own sounds, fonts and art came from if the user did not make them (their licenses go in the ledger).
5. **Estimates**: time to the preview stills, the sample and the first full cut, and scratch disk space (budget 10 to 15 GB per minute of 1080p60 trailer: takes, segments, titles and kept versions; more for a second shape).

Record every decision in `TRAILER.md`. If the user pre-approves, start, and still show the preview stills and the sample before the full render. Details: [references/brief-and-story.md](references/brief-and-story.md).

## Story defaults

Defaults for the common case, a fast montage. The brief's length, pace, opening and end card override any of them.

1. **Open strong.** Frame 1 already moves and shows the product at its most striking. When the name appears depends on the use case: early in feeds and ads, where viewers decide within seconds; as the payoff in a story trailer or a teaser; optional in a store-page video, whose page already shows it. Avoid opening on a splash or loading screen, a menu or a tool's interface unless the brief asks for it.
2. **Energy order.** Bright spectacle and action first, contrast (night, danger, depth) in the middle, tools and conveniences as quick punches near the end, the biggest moment in the finale.
3. **Show the payoff.** A menu, search box, command or settings screen is a short beat, then the cut goes to its result in the world. In an app demo the interface is the subject, and each step still ends on its result.
4. **Motion from the first frame.** Every shot moves when it starts, holds only on purpose, and appears once.
5. **One idea per shot**, readable on a phone in the time it is on screen.
6. **Cut on the music.** Cut points, title slams and big actions land on beats; chapters start on bar lines.
7. **True claims only.** Every number and name on screen is read from the product's own data and checked against what a user can see.
8. **The end card the brief asks for**, laid out as one block and held long enough to read twice.

## The timeline and beat grid

The timeline is one data file that the takes, the music, the titles and the cut all read: frame rate, tempo, length, chapters, slots and hits. The frame rate and length come from the brief, the tempo from the music. Frames per beat = fps × 60 / BPM; pick a tempo that gives whole frames (at 60 fps, 120 BPM is 30 frames a beat and 144 BPM is 25), then make the length a whole number of bars (a 45-second brief is exactly 27 bars at 144 BPM; at 120 BPM it is 22.5, so 22 bars or 23, rounding down when the length is a cap). Slots are whole or half beats; where a half beat is not a whole number of frames (12.5 at 144 BPM), slot starts are rounded on the cumulative beat, so no cut point is more than half a frame off. A licensed track sets the tempo instead. Chapters start on bar lines, and a check stops every render if the slots do not add up, a chapter does not start on a cut point, or a shot is used twice. Grid table, pacing and shot lengths: [references/brief-and-story.md](references/brief-and-story.md); file formats: [references/project-files.md](references/project-files.md).

## Capture

Film the real build with time under your control, so every frame is exactly one frame of product time however long it takes to capture:

- **A frozen build**: the commit to film (the release, plus any debug-only trailer hooks on the trailer branch) checked out in its own working folder and served on its own port, so other work cannot change the footage mid-run.
- **Control without changing play**: use what the product already exposes (a debug or test API, live objects an init script can wrap, source modules a dev server can import); if the game instance is private, expose it behind a URL flag in one debug-only line. Add hooks only where needed: a ready flag, a scripted camera, an interface switch, setters for time of day, weather and spawning. Off unless set, tested, and never changing normal play.
- **A virtual clock** injected before the page's own scripts: time stands still until you step exactly one frame, and timers, animation frames, CSS and Web Animations all follow it. Reseed every random generator (the page's and the engine's own) and turn off frame-time smoothing at the start of each take, so a take repeats. Engines use their fixed capture frame rate.
- **Scripted camera paths** between keyframes with easing: flyovers and orbits in 3D, pans, tracking and zoom steps in 2D, with speed ramps done by changing the time step.
- **Staging per shot**: a copy of the user's data loaded only into the capture profile, the area loaded and settled, time and weather set, stray creatures, toasts and hints hidden. Keep the interface where it is the gameplay (a tension bar, a score) and hide it where it is clutter.
- **Sharp, lossless takes**: for 3D and vector rendering, render at twice the output size (check the product really does: many cap their pixel ratio) and downscale with a sharp filter; for pixel art, render at an integer multiple of the game's base resolution and scale with nearest neighbour. Pipe the frames into a lossless intermediate, one file per take, after a few unrecorded warm-up frames.
- **Logged events**: wrap every way the product plays a sound to record each sound with its frame, and mark moments (an explosion, a landing) so a take can start or stop relative to them. Mute the browser, not the product: some products skip sound calls at zero volume.
- **Preview first**: three quick stills per shot at normal size before any full take; one capture process at a time on one machine.

Capture routes per stack, staging, camera language and scouting: [references/capture.md](references/capture.md). Code: [references/recipes/virtual-clock.md](references/recipes/virtual-clock.md), [references/recipes/capture-harness.md](references/recipes/capture-harness.md).

## Edit, titles and end card

- **Assemble from the timeline**: trim each take from its in-point, hold its last frame if it runs short (and say so), join the slots, then renumber every frame on the output frame grid before laying anything on top. Lossless intermediates often carry millisecond timestamps, and without renumbering, frames get doubled or dropped at joins and titles slip a frame.
- **Titles layer**: one page or canvas draws every title, callout, caption, flash and the end card as a pure function of the frame number, in the product's own fonts and logo art, captured with a transparent background into a lossless clip and laid over the footage.
- **Readability**: large type with contrast against busy footage, inside the safe area of every shape, held long enough to read twice, spelled right.
- **Transitions**: hard cuts on beats by default; a white flash or a short screen shake only on big hits; fades only where time passes or at the end. For pixel art nothing after capture resamples: shake by whole art pixels, slam titles through whole-number scales.
- **End card**: an opaque backdrop (footage showing through reads as a mistake), the brief's content (typically the logo, a call to action and where to get it) centred as one block, held long enough to read twice under the music's last hit.

Details: [references/edit-and-titles.md](references/edit-and-titles.md). Commands: [references/recipes/ffmpeg.md](references/recipes/ffmpeg.md).

## Audio

- **Music** in the style the brief's tone calls for, from a source you may use: composed for this trailer (in code with the product's own sound engine, or by the user), licensed for this use with the license read and kept, or generated by a service whose terms grant commercial rights to the output. If the user cannot show a license that covers promotional video, compose instead. A licensed track sets the grid: its tempo, its first downbeat, cuts on its bar lines. Never commercial songs, another product's music or a sound-alike melody. Record every source and license in `TRAILER.md`.
- **The product's own sounds**: render the logged sound calls offline from the product's own recipes or files, place them on their frames, and keep only what is on screen and close enough to hear.
- **Mix**: 32-bit float stems; music plus effects (and voice, if any); gain to about -14 LUFS integrated (or the target the platform or brief sets), then a limiter, with true peak at or under -1 dBTP measured on the final encoded file, because lossy audio adds peak.
- **Sync**: place sounds at frame times on the same grid, and measure and remove the delay of any processor in the chain.
- **Voice and captions**: as the brief says. Propose voice-over where it helps (app demos, story trailers) and captions where viewers watch muted (feeds); record a voice only with the speaker's consent.

Details: [references/audio.md](references/audio.md).

## Encoding and delivery

- **Master**: H.264 High, `yuv420p`, BT.709 colour tags written into the stream, a high-quality CRF, AAC stereo at 48 kHz, `+faststart`, the exact frame count and duration the timeline says.
- **Platform copies** come from the master or from their own capture: a smaller upload copy, a vertical version filmed and titled for 9:16 rather than a centre crop, stills, a share image.
- **Versions**: write every cut to a new numbered file, log it in `CUTS.md`, and never overwrite a cut the user has seen.
- **Storage**: renders and scratch stay out of version control; the scripts that make the trailer go into the project on their own branch (merged only when the user agrees), so it can be re-rendered when the product changes.

Details: [references/delivery.md](references/delivery.md).

## The review loop

Builders never grade their own cut.

1. **Preview folder**: as soon as shots exist, put their stills, named by chapter and shot, in a folder the user can open, and say which ones are stale. A `preview` critic can check framing before the full takes.
2. **Sample**: cut the opening (the first 10 to 15 seconds, or all of a short teaser) with the music before the full render. The opening decides whether anyone watches the rest.
3. **Mechanical checks** on every cut: the spec, decode errors, black and frozen frames (only the planned ones), detected cut points against slot starts, a frame-counter probe for the titles layer, loudness and true peak on the final file, and the audio offset against the mix.
4. **Contact sheets and title frames**: the first, middle and last frame of every slot, plus every title and the end card at full size.
5. **A fresh critic** grades each criterion of the trailer bar PASS or FAIL from those files only, citing slots and frame numbers, and returns a punch list ([references/critic-prompt.md](references/critic-prompt.md)). Each shape is its own cut with its own review files and verdict. Check the verdict; if it fails the check, run a new critic rather than editing it.
6. **Fix and re-cut**: re-take only the shots the punch list names; titles and music changes need no re-takes. When approved frames must stay unchanged, prove it with a frame comparison.
7. **The user watches**: give the path, say what may still change, record their notes, apply them as a new version.
8. **Final checklist** before handing over.

The bar, the checks' pass conditions, the stuck-shot ladder and the checklist: [references/review.md](references/review.md).

## Working with the user

- **Status**: what is done, what is next, and an honest estimate. When a change adds time, say so at once and offer a faster path (drop weak shots instead of re-filming them, one shape first).
- **Show real things early**: the preview folder within the first hour, then the sample. Separate "the video is watchable" from "housekeeping is left".
- **Disk**: report the scratch size before it surprises anyone. Keep raw footage, earlier cuts and previews until the final cut is approved, and ask before deleting or moving any of them.
- **Their data**: ask the user to export a copy of their save (the product's own backup, or a one-line console snippet they run), film in that copy loaded only into the capture profile, and never touch the user's own browser, app install, save files or accounts.
- **Helpers**: give each subagent its own output paths and tell it to write files early. Make sure a helper that looks stalled has really stopped before starting a replacement, and keep one owner per working folder.

## Non-negotiables

1. Every frame comes from the real build. Only declared graphic slots, such as the end card or a plain device mock, are drawn.
2. Every claim on screen is true and checked against the product.
3. Music, sounds, fonts and voice are original or licensed for this use, and each is recorded with its license.
4. No personal data, real user accounts, other products' names or logos, and no person's name, face or voice without their consent.
5. Every grade comes from a fresh critic or from the user, never from whoever built the cut.
6. Nothing the user may still need is deleted or moved without their go-ahead.
7. Renders stay out of version control; secrets stay out of logs, files and commits.
8. Publishing and uploading stay with the user unless they explicitly hand them over.
9. No AI vendor or model names in the trailer, its files or its commits unless the user asks for them.

## When a tool is missing

- **No browser automation**: install it with the user's consent, or use the product's own capture route; for a real-time screen recorder, re-take until each take is clean and capture at the highest frame rate available.
- **No ffmpeg**: ask to install it with the system's package manager; nothing else here replaces it.
- **Headless rendering looks different** (software rendering, missing effects): run with GPU flags or headed, compare one still with a real browser, and keep one route for every take.
- **No scripted camera and no way to add one**: film the product's own view with scripted input, and lean on the edit.
- **No image-capable critic**: the user grades the visual criteria from the contact sheets; the mechanical checks still run.

## Resuming

Read `TRAILER.md`, the timeline, `CUTS.md`, the latest verdict and the takes manifest. Re-take any shot whose script changed after its take was recorded, then continue from the last gate passed. Never restart from the brief.

## References

- [references/project-files.md](references/project-files.md): folder layout, `TRAILER.md`, the shot list, the timeline file, the takes manifest, `CUTS.md`, the re-render README.
- [references/brief-and-story.md](references/brief-and-story.md): questions to ask, use cases with their lengths and pacing, story shapes, openings, the beat grid, the shot list.
- [references/capture.md](references/capture.md): capture routes, frozen builds, reaching the product, hooks and wrappers, deterministic time, staging, the user's data, 2D and 3D camera language, pixel art, scouting, apps.
- [references/edit-and-titles.md](references/edit-and-titles.md): assembly, the titles layer, readability, transitions, shake, colour, the end card, vertical layouts.
- [references/audio.md](references/audio.md): music sources and rights, composing in code, cutting to a licensed track, the product's own sounds, mixing, loudness, sync, voice and captions.
- [references/delivery.md](references/delivery.md): the master spec, platform copies, stills, naming, versions, storage, budgets, the handover report.
- [references/review.md](references/review.md): the trailer bar, mechanical checks, the watch-through, user review, the stuck-shot ladder, the final checklist.
- [references/critic-prompt.md](references/critic-prompt.md): the drop-in critic prompt, verdict formats, the verdict check, grading by the user.
- [references/pitfalls.md](references/pitfalls.md): symptoms, causes and fixes from real trailer runs.
- [references/case-studies.md](references/case-studies.md): how two real launch trailers were made, as worked examples.
- [references/recipes/virtual-clock.md](references/recipes/virtual-clock.md): a stepped clock for browser builds, with its tests.
- [references/recipes/capture-harness.md](references/recipes/capture-harness.md): browser automation capture into lossless takes, camera helpers, a shot script, the sound log, a runner.
- [references/recipes/ffmpeg.md](references/recipes/ffmpeg.md): tested commands for assembly, shake, encoding, platform copies, checks, the effects stem, licensed-track edits and the audio mix.
