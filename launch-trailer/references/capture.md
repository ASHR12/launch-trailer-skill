# Capture

Read before building the capture tooling and before scripting shots. The aim: every take comes from the real build, every frame is exactly one frame of product time, and the same script gives the same take twice.

## Contents

- Choosing a capture route
- Freezing the build
- Reaching the product
- Hooks and wrappers
- Deterministic time
- Staging a shot
- The user's data
- Camera language
- Timing events inside a take
- Scouting for views
- Gameplay and interface shots
- Sharp, lossless takes
- Apps, not games
- Resources and parallel work

## Choosing a capture route

| Product | Route | Time control |
| :--- | :--- | :--- |
| Browser game or web app | browser automation drives the page at the output size; screenshots through the browser's DevTools protocol pipe into ffmpeg | a virtual clock injected before the page's scripts ([recipes/virtual-clock.md](recipes/virtual-clock.md)) |
| Engine project (Unity, Unreal, Godot and others) | the engine's own recorder or movie-render route, writing frames at a fixed rate from a scripted camera or sequence | the engine's fixed capture frame rate; check the installed version's documentation for the exact setting |
| Native desktop or mobile app | an offscreen render or the app's own export if it has one; otherwise the simulator's or OS's screen recorder | usually real time: re-take until clean, at the highest frame rate available |

The browser route is the one the case studies used; [recipes/capture-harness.md](recipes/capture-harness.md) has its code. Whatever the route, prove it on one test shot before scripting the rest: capture a few frames, open them, and check the size, the sharpness, and that consecutive frames differ by exactly one step of motion.

## Freezing the build

Film one fixed version of the product: the release commit, plus any debug-only trailer hooks committed on the trailer branch above it. Check that commit out into its own working folder (a separate git worktree works well), install real dependencies there (a copy or clone, not a link into another checkout), serve it on its own port, and record the commit id in every take's manifest. Other work in the repo can then carry on without changing the footage halfway through.

- **Production build or dev server?** A production build with a preview server is closest to what players get; use it when everything the capture needs survives the build. Use the frozen commit's dev server (hot reload off, its own cache folder) when the capture imports source modules or needs dev-only hooks. Lightning Sortie filmed a production build through in-page wrappers; BlockHaven filmed a dev server and imported the game's own modules.
- In the Lightning Sortie run a hotfix shipped mid-edit; the frozen build kept the footage consistent, and the agent checked the affected shots and flagged the now-stale version label.
- Never film the user's running copy, and never touch their installed app, its server, or their own browser profile.

## Reaching the product

The capture needs a handle on the running product: its game or app instance, its stores, its settings. Try these in order:

1. **Objects already on `window`**: a debug or test API, a framework global, anything the product's own tests use.
2. **Source modules through a dev server**: a page script can `await import('/src/save/worldStore.ts')` and use the product's own code (the BlockHaven capture imported the game's backup parser and world store this way to load the copy of the user's world).
3. **One debug-only line** on the trailer branch that exposes the instance behind a URL flag, for example `if (new URLSearchParams(location.search).has('trailer')) window.__game = game;`, plus anything else the capture needs that is not reachable from it (the engine's random generator, a store). This is the smallest change to the product; it does nothing in normal play. Ask the user before changing their code, even this much.
4. **Wrapping from an init script** works for prototypes and globals loaded as plain scripts; a framework bundled into private modules cannot be reached this way, so fall back to 2 or 3.

## Hooks and wrappers

The more the capture can control, the less luck it needs. What a trailer usually wants:

| Control | What it does |
| :--- | :--- |
| ready | true once assets are loaded and the first frames drawn; for streaming worlds, also "the area around a point is generated and meshed" |
| scripted camera | draw from a set view (3D: position, yaw and pitch or a look-at target, vertical field of view, roll; 2D: scroll position and zoom), or give the normal view back exactly |
| interface switch | show or hide the HUD (and a first-person hand), like the product's own hide-HUD key |
| time, weather, cycles | stage the sky, and stop cycles from drifting during a take |
| spawning and entities | turn natural spawning off, list, remove, spawn and place things, teleport the player |

Get them by wrapping first and adding hooks second. The Lightning Sortie trailer changed no game code:
- It wrapped the camera update so the product's camera ran first and the shot's rig overrode it afterwards; level of detail, lights and the HUD then followed the rig.
- It wrote settings into storage before any page script ran (top quality, first-run screens marked as seen).
- It stubbed input devices where the HUD depends on them (a fake gamepad so console button glyphs show) and hid real ones.
- It toggled CSS classes per shot to hide interface layers (all of it for cinematic shots, only the corner chrome for HUD shots).
- It sent key presses the way the product's own tests do.

When the interface is drawn inside a canvas, hide it through the product (its scene or layer visibility) rather than CSS. Added hooks are debug-only: off unless set, changing nothing the product saves or generates, each with a test (the set view is what is drawn, the player does not move, clearing it restores the normal view exactly). BlockHaven added two, a scripted camera and a HUD switch.

## Deterministic time

A frame may take 200 ms to capture at twice the output size, while the product thinks 16.7 ms passed. Take time away from the product:

- **Virtualize every clock the page reads**: `performance.now`, `Date.now`, `requestAnimationFrame`, `setTimeout` and `setInterval` (leave zero-delay timeouts real; they only yield), and CSS and Web Animations (pause them and seek them to virtual time each step).
- **Step exactly one frame**: due timers fire in order, then the queued animation frames run with the new timestamp, then animations are seeked. A product with a fixed-step simulation and interpolated rendering keeps working unchanged.
- **Reseed at the start of every take**: loading runs a variable number of frames and draws random numbers, so seed `Math.random` again (the clock's `reseed()`), and reseed the engine's own random generator if it has one (many seed it from the time at boot). Fix `Date.now` to a constant start if the product shows or seeds from the date.
- **Turn off frame-time smoothing**: engines that average recent frame times, or clamp large ones, distort stepped time. Find the engine's option for it in its documentation, and check that one step moves the product exactly one frame's worth.
- **Know what stays real**: Web Workers, the audio clock and anything off the main thread. Render audio offline from logged events instead ([audio.md](audio.md)), and give worker-driven simulation a step hook.
- **Speed is the step size**: a smaller step is slow motion, a larger one a speed ramp or time-lapse; a step function per frame lets one take slow down for an explosion and speed up through a dull stretch. Physics stepped without interpolation can judder in slow motion; try a short slow take before relying on it.
- **Live while loading, stepped while recording**: resume real time to load and stream, pause it to record.

Verify the clock before filming with the tests in [recipes/virtual-clock.md](recipes/virtual-clock.md).

## Staging a shot

Each shot script starts with a stage step that puts the world in a known state:

1. Close any screen the last shot left open and release held keys.
2. Set the mode the shot needs (a free-flying or creative mode for cinematic shots, if the product has one), the time of day, the weather, the field of view or zoom, and turn off cycles that would drift (day-night, weather changes).
3. Move the player near the shot (streaming worlds load around the player even while the camera is elsewhere), then wait in live time until the area is ready.
4. Clear stray creatures near the subject and turn natural spawning off, so every creature in the shot was placed on purpose. Pin anything driven by AI that might wander out of frame (enemy aircraft held in a formation relative to the camera, a creature held on its mark).
5. Hide toasts, chat lines, tutorial hints, achievement pop-ups and debug overlays, unless the shot is about them. Keep the interface where it is the gameplay (a tension bar, a score, a lock-on box) and hide it where it is clutter.
6. Raise the product's brightness setting for caves, dungeons and night shots; dark footage reads as a broken frame on a phone.
7. Reseed randomness, then step a few warm-up frames so the first recorded frame is already settled.

Settings for the whole session: the highest graphics preset, view bobbing, camera shake and field-of-view effects off (add shake in the edit where you want it), placement previews and coordinates off. **Mute the browser, not the product**: some products skip sound calls entirely when their own volume is zero, and the effects stem is built from those calls. Turn the product's volume down only after checking that the sound log still fills.

## The user's data

Real progress looks best (an upgraded boat, a big world), but the user's own save stays untouched.

- **Ask for a copy**: the product's own export or backup if it has one; otherwise a file the save lives in, or a one-line console snippet the user runs in their own browser that copies the save for them to send. A save can span several keys; copy all of the product's keys, for example `copy(JSON.stringify(Object.fromEntries(Object.entries(localStorage).filter(([k]) => k.startsWith('<prefix>')))))`. Saves in IndexedDB need the product's own export, or a snippet written for its database.
- **Moments the save is past** (the first boat upgrade, when the boat is fully upgraded): ask for an earlier save as well, or stage that state through the product for the shot.
- **Load it only into the capture profile**: a new port is a new origin with empty storage, so write the copy into storage from an init script before the product's scripts run (guarded so it only writes once per take setup), or import it through the product's own import code, under a fixed id.
- **Fresh copies** for shots that need untouched state (an achievement not yet earned, a spot not yet built on).
- **Separate places** for shots that change the world (builds, explosions); shots that follow each other in story (type a command, then land) are filmed together, in order.

## Camera language

In 3D:

| Move | Use | Notes |
| :--- | :--- | :--- |
| Dive or flyover | hooks, scale, landscapes | fast, passing within a few units of a ridge or tree for parallax |
| Dolly or push-in | reveal a detail, add tension | ease in and out |
| Pull-back or crane up | reveal the whole, finales | start close on the subject, end wide and high |
| Orbit | hero objects, builds, creatures | a third of a turn is plenty in a 2-second slot |
| Tracking | follow movement | keep the subject in the same screen area |
| First person | doing things: building, crafting, combat | the product's own view, HUD on |
| Locked-off | almost never | a still camera reads as a still frame; add a slow drift |

In 2D:

| Move | Use | Notes |
| :--- | :--- | :--- |
| Pan | reveal a level, follow a path | eased; across, not just along, the subject's motion; for pixel art, a whole number of art pixels per frame (a slow eased pan steps one art pixel every few frames and stutters), easing only in the first and last few frames |
| Tracking | follow the player, a boat, a projectile | lead the subject: more space ahead than behind |
| Zoom step | punch in on an action, pull out to the whole map | for pixel art, change zoom by whole steps on a cut, never smoothly |
| Parallax | depth in side-on games | move layers at their own speeds if the product has them |
| Gameplay view | the product's normal camera | keep its interface when the interface is the game |

- **Keyframes and easing**: interpolate position, angles (yaw by the shortest way round), field of view or zoom, and roll between keyframes with an easing curve: ease-in-out for most moves, ease-out for arrivals, ease-in for departures, cubic or exponential for whips.
- **Look-at targets** keep the subject framed while a 3D camera moves; derive yaw and pitch from camera and target positions.
- **Lighting**: put the sun behind the camera for colourful, readable landscapes; backlight only for silhouettes on purpose.
- **Atmosphere**: high-altitude shots can sit inside fog or a cloud layer and come out washed out; turn clouds off and raise the view distance for that shot, then restore them. A very low camera can catch the underside of a haze layer as a coloured band; fly higher or change the angle. Keep the horizon level on climbs and dives unless the roll is the point.
- **Shapes**: frame each shape on its own. In 3D, a vertical frame needs a wider vertical field of view; in 2D, a portrait game size or scale mode, or a camera that frames a tall area; in an app, a phone-width viewport so the mobile layout shows. Neither case study rendered its vertical version, so preview every shot in 9:16 before the long render, and give a shot that does not survive its beats to its neighbours.

## Timing events inside a take

- **Script the trigger** at a chosen frame (prime the explosive, strike the lightning, press the key) rather than waiting for creatures or physics to decide.
- **Pre-roll and trim**: record from before the event and set the take's in-point so the event lands on its slot frame. With a mark logged at the event (the explosion sound's first frame), record until the mark plus N frames, then set the in-point to the mark minus the frames the slot wants before it.
- **Sync to the event, not the cut**: put the logged event on the beat; a camera cut on the beat with the blast a few frames late reads as out of sync. In the Lightning Sortie run the first "impact" mark was the replay's cut to slow motion, 1.5 s before the fireball; re-marking it from the explosion sound in the log fixed the sync.
- **Speed ramps** fit a long event into its slot: a sleep fade that took longer than the slot ended the take in black until the step size grew through the dark part, so morning landed on its beat. Ramp at record time with the step size; never drop or blend frames afterwards. The Lightning Sortie kill replay's lead-in was recorded about 1.5x fast so it fit its slot while the cut still used every recorded frame once.
- **Holds**: when a few frames look wrong while the product runs on correctly (a projectile filling the lens as it leaves the camera), repeat the last good picture for those frames and let the product keep running.
- **Hide transitions under flashes**: a teleport's loading sky disappeared once the key press moved to the first frame of the landing shot, under its white flash.

## Scouting for views

Do not guess positions for dozens of shots. Write small searches over the loaded world or level data: in a voxel world, the highest summit near a point, the flattest dry ground of a given size, a tree trunk with clear ground beside it, a cave spot with the most glowing blocks in sight; in a 2D map, the screen-sized areas with the most varied tiles, water and buildings; in an app, the seeded records that show each feature best. Capture a scout grid (a few small stills per candidate) and pick from that. Seed any randomness in a search so it finds the same spot in every shape, and record the chosen positions in the shot script.

## Gameplay and interface shots

- Drive them with real input from the automation side (key presses, mouse moves, clicks) in a "before each frame" hook, so the product's own input handling, sounds and animations run.
- For skill-based moments (a tension bar, aiming, timing), write a small autopilot that reads the product's state each frame and presses keys to keep it in range: the line taut near the red without snapping, the reticle on the target. If the state is reachable, setting it directly for the take is fine, as long as what the viewer sees is what a player would see.
- With the interface on, wait for setup messages and item names to fade before recording.
- Keep interface moments to about a second and cut to their result; if an open animation will not capture, start with the screen already open.

## Sharp, lossless takes

**3D and vector rendering**: render at twice the output size and downscale.
- Set the device scale factor to 2 and capture at that scale. With the DevTools protocol, pass the scale in the screenshot's clip; the capture otherwise comes back at the CSS size even with the device scale set. In the BlockHaven run, a 3840x2160 capture took about 185 ms through the protocol and about 1.2 s through the automation library's own screenshot call.
- Products often cap their own pixel ratio (the Lightning Sortie game capped its top preset at 1.5x). Force the full ratio for capture, and refuse to record unless the drawing buffer is the size you expect (3840x2160 for a 2x 1080p take).
- Downscale with Lanczos into the lossless take.

**Pixel art and fixed-resolution canvases**: a game that draws at a small base resolution and lets the browser scale it up gains nothing from a higher device scale, and Lanczos would blur its pixel edges.
- Choose an output size that is a whole multiple of the base resolution where you can (a 480x270 base, times 4, is 1920x1080); otherwise scale by the largest whole multiple and fill the rest with a border or the game's own background.
- Capture at device scale 1 with the game scaled up by that whole number (its own scale mode, or CSS `image-rendering: pixelated`), so each screenshot is already the output size. Anything that must still be resized uses nearest neighbour (`flags=neighbor` in ffmpeg), never Lanczos.
- The buffer check becomes "the canvas is the base resolution, drawn at the chosen whole multiple", and a full-size still should show square, equal-sized pixels.
- For the vertical version, switch the game to the portrait twin of its base size (270x480, times 4, is 1080x1920) if it can run that way, and check that its interface still lays out; otherwise frame a tall region of the world at the same multiple. A border around a small landscape picture is the last resort.
- Keep everything after capture free of resampling too: the edit's shake and the title slams have pixel-art variants ([edit-and-titles.md](edit-and-titles.md), [recipes/ffmpeg.md](recipes/ffmpeg.md)).

**For every take**:
- Pipe frames straight into a lossless RGB intermediate (H.264 RGB at CRF 0, or FFV1), one file per take; never write thousands of loose PNG files unless you must.
- Use a quick preview mode (three JPEG stills per shot at 1x) for framing, a draft mode (1x, JPEG) for trying a shot, and the full mode for final takes.
- Supersampled takes look sharper than live play. If the user asks whether the trailer matches the real game, say so plainly: the content is real, the image quality is the best the build can render.
- Budget: 0.2 to 0.4 s per recorded 2x frame including staging and pre-rolls, so 12 to 25 minutes per minute of 60 fps footage; 5 to 7 GB of lossless takes per minute at 1080p, and 10 to 15 GB of scratch in all for a one-minute trailer once segments, titles and kept versions are counted.

## Apps, not games

- Film with seeded, plausible demo data: no real names, emails, accounts, locations or customer content.
- Headless screenshots show no cursor; draw one in the page (a small overlay that moves along an eased path and clicks with a ripple), or show results without it.
- Type at a human pace with small variations; paste long text.
- Zoom into the region that matters, by a CSS transform during capture or a crop in the edit, so the interface reads on a phone.
- Hide notifications, tooltips that are not part of the story, and anything showing the date or a version number.
- Map and location features: film somewhere other than the user's home area, and check every frame for street names and personal pins.

## Resources and parallel work

- One capture process per machine at a time. In the BlockHaven run, two high-quality captures at once (one per shape) crashed a browser partway through; run the shapes one after another.
- Run side jobs (samples, contact sheets) at low priority (`nice -n 19`) so they do not slow the capture.
- Keep a log per capture run, save the page's console errors next to the takes, and look at them before the cut.
- One agent owns the capture folder. If a second copy of an agent may be running (a resumed or forked job), stop it before continuing; two writers in one folder overwrite each other's scripts and takes.
- When other agents run browser tests on the same machine, share one lock (a lock folder created with `mkdir`, which is atomic, with a time after which it counts as stale), and do the scripting while you wait for it. Write process checks that cannot match themselves: `pgrep -f 'harnes[s]'`, not `pgrep -f harness`.
