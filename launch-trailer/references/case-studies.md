# Case studies

Two real launch trailers, made by agents from two browser games. The first built the method from scratch; the second reused it and changed what it needed. Read them for scale, order and judgment, not as scripts to copy.

## BlockHaven: 60 seconds, every feature, built from scratch

**The product**: BlockHaven, a voxel sandbox game that runs in the browser (TypeScript, three.js on WebGL2, Vite), public at [blockhaven-blue.vercel.app](https://blockhaven-blue.vercel.app). Procedural worlds with 25 lands, places to find, building, crafting, creatures, weather, a guide and teleport commands.

**The brief**: a launch trailer that glimpses every feature, with music and the game's own sounds. The proposal offered three directions (an epic montage, the world building itself, a small player's story), each with its opening, plus extras (a day-to-night time-lapse, a slow-motion explosion chain, true numbers flying in) and an estimate. Length and shapes were settled before any capture code: 60 seconds in 16:9 at 1080p60; a planned vertical cut was not made.

**The edit**: 60.000 s at 60 fps and 144 BPM, so a beat is 25 frames and the cut is 36 bars, 3,600 frames, 55 slots (54 filmed shots and one graphic slot).

| Beat | Chapter | Title | Notes |
| :--- | :--- | :--- | :--- |
| 0 | Opening | | a dive over snowy peaks at sunrise; a hard cut on beat 3 to a slow-motion explosion chain; the logo through the smoke on beat 5 |
| 8 | World | A world that never ends | one block in the sky, the world streams in around it; "25 lands" over six lands |
| 24 | Build | Build anything | a house time-lapse, glass, a rainbow wool wall, beds, flying |
| 40 | Places | Places to find | eight places at 2 beats each; "9 places to find" |
| 56 | Craft | Craft and survive | a tree punched, a recipe shown, a furnace, armor, a bow shot, eating |
| 68 | Friends | Friends and foes | farm animals, sheep shorn, horses |
| 76 | Weather | Day, night, weather | sunset time-lapse, snow, a lightning strike on its hit |
| 84 | Foes (night) | | night creatures, a creature's explosion on its hit |
| 92 | Ember (night) | Ember power | a lever and door, lamps, a powered explosion line |
| 100 | Sleep | | a bed; morning light on its hit |
| 104 | Tools | Never get lost | the guide's search as a flash, its arrow across the world, a command, the teleport landing at a fortress |
| 116 | Extras | And lots more | third person, sun shadows, a screenshot, an achievement, the desktop app icon (graphic slot) |
| 128 | Finale | | pull back from the player on a peak at sunset; the logo built from blocks; the end card |

**The pipeline**:
- A separate worktree and branch with its own dev server, port and cache, so the live game, its desktop app and its server were never touched.
- Two debug-only hooks added to the game, each with an end-to-end test: a scripted camera (`camera.set(view)`, off unless set) and a HUD switch.
- The virtual clock in [recipes/virtual-clock.md](recipes/virtual-clock.md); Playwright driving Chromium at 2x device pixels; DevTools screenshots piped into lossless RGB takes; per-shot scripts using camera helpers (easing, look-at, orbit) and world searches (summits, flat ground, tree trunks, cave views).
- Staging: a copy of a real saved world in the capture profile only, top graphics, view bobbing and camera shake off, hints off, toasts and chat hidden by CSS unless shown on purpose, spawning off, brightness raised per shot.
- Sounds logged during capture, re-rendered offline with the game's own sound recipes, filtered and levelled ([audio.md](audio.md)).
- Music composed as data on the same grid with the game's synth kit, rendered offline, original themes.
- Titles drawn per frame in a page using the game's logo art and pixel font, with numbers read from the game's registries.
- Float stems, a limiter and a two-pass loudness step to -14 LUFS; a cut that trims, holds, joins, renumbers, shakes and overlays; an H.264 High master at CRF 15; eight stills.

**How the run went**:
1. First session, about 3 hours to the first full cut: tooling, 54 shot scripts, preview passes for framing, a "shots so far" folder of stills (with an honest list of rough shots), a 15-second sample with the music, the opening re-filmed after the sample showed a washed-out haze, the full render.
2. Between sessions the work was parked on its branch with a resume note and nothing running.
3. Second session, about 2 hours: the watch-through found and fixed the problems below, one change after review, then tests, a squashed commit series and the handover.

**What the watch-through fixed before the first review**: "10 places" corrected to 9; an opaque end-card backdrop; titles renumbered onto the frame grid (they had drifted a frame late, with a doubled frame at 0:51); the effects stem moved to float (a +1.5 dBFS peak had clipped); eight shots re-filmed (a half-meshed first frame, a furnace that did nothing, an arrow filling the lens, a sleep ending in black, a teleport's loading sky, a view swapping two frames before a cut).

**The change after review**: one line came off the end card. Only the end-card frames of the titles layer were re-rendered; frames 0 to 3401 and all of the sound stayed bit-identical to the approved version.

**Delivered**: 1920x1080, 60 fps, 3,600 frames, 227 MB, H.264 High with AAC stereo, -14.0 LUFS, true peak -0.9 dBTP (a tenth over the target after AAC encoding); eight stills; one trailer frame later became the website's share image. The scripts were committed with a re-render README; the 15 GB scratch folder was kept until sign-off, then moved to a synced folder with a link left at the old path.

**What it taught**:
- Agree the opening in the storyboard: changing it mid-run reshaped the whole order.
- Show progress early, with a preview stills folder and then a short sample.
- Give time and disk estimates up front, and say at once when a change adds time.
- Two copies of the trailer agent, and two music helpers, ran at once in one folder for a while and overwrote each other. One owner per folder.

## Lightning Sortie: 58 seconds, reusing the method

**The product**: Lightning Sortie, a browser flight-combat game (three.js, Vite) with dogfights, ground strikes, day and night, guided rockets and a kill replay.

**The brief**: a 16:9 launch trailer of 45 to 60 seconds showing how the game plays (flight, rockets, hits), mostly in wide shots. The agent started from the BlockHaven trailer's tooling.

**The run**: one trailer agent, about 2 hours 55 minutes in all, while a release agent shipped a new version in the same repo.
1. For the first 45 minutes no browser could run (the release agent was running timing-sensitive tests in the same repo), so the agent studied the game and both earlier trailer pipelines and wrote every script, the cut plan, the music and the titles without running anything.
2. When the release was tagged: a worktree at the tag, a production build, a preview server on a spare port.
3. Three preview passes at 1x for framing, then the full 2x recording in about 25 minutes, then the kill shot re-recorded to sync the blast to its beat.
4. The mix, a first cut, 50 minutes of encode fixes, delivery, then one change after review in about 12 minutes.

**The edit**: 58.0 s at 60 fps and 120 BPM (30 frames a beat), 116 beats, 19 slots: a slow push-in through a dark hangar with the title slamming in at 4 s over a night afterburner pass; the game's own aircraft choice and a take-off; "DAY OR NIGHT"; "DOGFIGHT"; "GROUND STRIKE"; a night city mode; a guided-rocket kill and the game's replay under "Every kill. Replayed."; keyboard and gamepad controls; an afterburner climb; the end card ("Play free in your browser", the address, "OUT NOW").

**What stayed the same**: the overall design (clock, director, capture studio, shot scripts, one timeline that the score, effects and titles read), the clock file itself unchanged, 2x lossless takes, the beat grid, shake on hits, -14 LUFS and -1 dBTP, nearly the same encode settings (CRF 16 instead of 15).

**What changed**:
- **No changes to the game**: init scripts wrapped the live game objects (the camera update ran first and the shot's rig overrode it), settings were written to storage before the page loaded, a stub gamepad made the HUD show controller glyphs, and the game's own test harness flew the autopilot.
- **Raw DevTools protocol** instead of Playwright, filming a tagged production build.
- **Forced sharpness**: the game capped its quality preset at 1.5x; the capture forced 2x and refused to record unless the buffer was 3840x2160.
- **Per-frame gameplay logs** (weapons fired, camera distance, throttle) drove synthesized effects: an engine bed, gunfire at the real rate, whooshes, explosions sized by distance.
- **Music synthesized from scratch** (120 BPM, D minor lifting to D major on the end card) instead of with the game's own instruments.
- **Titles as a transparent PNG sequence** styled after the game's HUD, because ffmpeg had no `drawtext`.
- **A two-stage cut** (lossless segments per slot, then concat and encode), after one big filter graph stalled, and after an overlay's pixel-format switch corrupted the shake segments.
- **OCR on every 15th frame**, proven first on a frame with text, to catch text that must not appear.

**The change after review**: two title texts were replaced. Only those titles differed: the audio stayed bit-identical, and one title frame that had shifted was restored.

**Delivered**: one 58-second MP4, 1920x1080 at 60 fps, 93 MB, H.264 High with AAC stereo, -14.0 LUFS, true peak -2.2 dBTP. No stills and no vertical cut. The 11 GB scratch folder and the worktree were kept.

**What it taught**:
- Reusing the tooling cut the time to a first full cut roughly in half, and blocked time is scripting time.
- Capture can need no changes to the product at all.
- It skipped the sample, the critic and a written checklist, so the first review was of the finished cut, and the one change was on-screen text that had gone stale: a version label, out of date once a hotfix shipped mid-edit. Checking every on-screen text against the brief just before delivery catches that.
- The two openings show the range: a cinematic push-in with the title at 4 seconds here, a spectacle dive with the title at 2 seconds in BlockHaven. Pick by where the video will be posted.
