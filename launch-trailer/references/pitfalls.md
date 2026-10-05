# Pitfalls

Every row happened in a real trailer run (BlockHaven, Lightning Sortie, or a map app's trailer made the same day as Lightning Sortie). Read the section for the phase you are in; scan the whole file when something looks wrong and you do not know why.

## Story, process and the user

| Symptom | Cause | Fix |
| :--- | :--- | :--- |
| The user: "you have to hook the people in the first few seconds" | the plan opened on a slow dark cold open, and the tools appeared too early | spectacle on frame 1, the title by about 3 s, tools as late, short flashes |
| "2 min is too much" | "cover everything" planned at 2 to 3 s per feature | 60 s at most; about a second per feature; the best moments get the time |
| The user grew frustrated as estimates rose | a creative change added work and the estimate quietly grew | say what a change costs when it happens, and offer a faster path |
| "You said raw footage will be 14 GB?!" | scratch size was never mentioned up front | estimate disk in the proposal; separate the video's size from the scratch folder's |
| "Do we have any sample yet?" | hours of work with nothing to look at | a preview stills folder within the first hour, a sample before the full render |
| A status message to the user was wrong | a coordinator relayed a guess from a log tail | report only what you checked |
| "I just want to know the answer, don't make any edit" | a question was treated as a change request | answer questions; edit only when asked |
| "Remove version totally" | a version label on the end card and in a callout, stale within hours (a hotfix shipped mid-edit) | no version numbers, dates or "new in" labels on trailers |
| "...much sharper than what I have actually in the game" | 2x supersampled capture | say plainly: real content, rendered at the build's best quality |
| A map app's trailer could have shown where the user lives | real location data on screen | film somewhere else; check every frame for homes, street names and personal pins |

## Capture

| Symptom | Cause | Fix |
| :--- | :--- | :--- |
| A capture crashed partway ("target closed") | two high-quality captures ran at once on one machine | one capture at a time; shapes one after another |
| Scripts and takes changed under the agent | a second copy of the same agent was still running in the folder | one owner per folder; check running processes before resuming |
| A music helper's files were overwritten by another helper | a "stalled" helper was replaced without being stopped | confirm it stopped; separate output paths; tell helpers to write files early |
| A full recording run died at launch | a zsh cleanup line hit an empty glob | `find <dir> -type f -delete` |
| A watchdog reported a process that had ended | `pgrep -f harness` matched its own command line | `pgrep -f 'harnes[s]'` |
| 2x captures came back at 1x | the DevTools screenshot used the CSS size | pass `scale` in the screenshot clip |
| Takes at 2x were softer than expected | the product capped its pixel ratio at 1.5x | force the ratio; refuse to record unless the buffer is 3840x2160 |
| Headless frames differ from a normal browser | software rendering | GPU flags or a headed browser; compare one still |
| A menu's opening animation never appeared in stepped capture | finished fill-forwards animations kept restarting | seek animations to virtual time and remember finished ones ([recipes/virtual-clock.md](recipes/virtual-clock.md)) |
| A hole in the world on a take's first frame | geometry not yet meshed after the camera moved | warm-up frames before recording; a lead before the slot |
| Animals hidden in tall grass; a pillar blocking a landmark | framing guessed, not checked | preview stills; scouting searches; clear the stage |
| Night creatures almost black; a dungeon fully black | night and underground light | raise the brightness setting, film at dusk, add light sources |
| A storm with rain but no visible bolt | a dark night storm, the strike left to chance | a daytime storm, the strike triggered on a chosen frame |
| The hook's peaks lost in a washed-out orange haze | the camera sat in the fog and cloud layer, with the sun ahead | clouds off and view distance up for the shot, the sun behind the camera |
| A coloured band near the ground in a flight shot | the underside of a haze layer | fly higher |
| A blue fog over a first-person shot | the player was standing in water | check the stand point in the preview |
| Enemy aircraft flew out of frame | the game's AI steered them away | pin them in a formation for the take |
| Achievement pop-ups in unrelated shots, two at once in another | earned during staging, or as a side effect of granted items | hide toasts unless the shot is about them; a fresh data copy and a planned order for the shot that shows one |
| A sleep shot ended in black | the fade took longer than the slot | a speed ramp through the dark part |
| A furnace did nothing in its slot | the process takes longer than the slot | a faster step so it finishes on screen |
| An arrow filled the lens for two frames | the projectile spawns at the camera | hold the last good picture for those frames |
| A teleport showed its loading sky | the destination loaded during the slot | press the key on the landing shot's first frame, under the flash |
| The blast landed after the beat | the mark was the replay's cut to slow motion, not the explosion | mark from the explosion sound in the log |
| The whole view swapped two frames before a cut | a roof ridge crossed the camera's path | stop the move short of it; look at the last frames of every take |

## Edit, titles and encode

| Symptom | Cause | Fix |
| :--- | :--- | :--- |
| Titles one frame late in parts of the cut, a doubled frame at 0:51 | takes carried millisecond timestamps | renumber every stream on the frame grid (`settb=1/60,setpts=N`); run the titles probe |
| The master one frame short | overlay dropped the last frame when syncing inputs | spare frames on the titles layer; assert frame counts per segment |
| ffmpeg stalled at 0% CPU | one filter graph over about 21 full-HD inputs | cut in two stages: lossless segments, then concat and encode |
| A 58-second file at 326 to 572 MB, with garbage slices | an overlay with `enable=` switched pixel format mid-stream | `format=gbrp` after every overlay; check the bitrate per second |
| The world's logo ghosted behind the end card | a translucent backdrop | make it opaque |
| A title ran off the frame | wide letter spacing at a large size, a label placed from the block's width | check every title as a full-size still before rendering |
| A title covered the jet | layout ignored the shot | move it into empty sky |
| "10 places to find" where the game lists 9 | the count included a starter chest the game does not present as a place | read numbers from what the product shows its users |
| No `drawtext` filter | ffmpeg built without freetype | draw titles in a browser page captured with transparency |
| A contact sheet's concat list failed | relative paths resolved against the list file's folder | absolute paths and `-safe 0` |
| Colours shifted in some players | colour description missing from the stream | BT.709 tags plus the `h264_metadata` filter |

## Audio

| Symptom | Cause | Fix |
| :--- | :--- | :--- |
| Explosions distorted | the effects stem was written as integers and clipped at +1.5 dBFS before the limiter | 32-bit float stems |
| True peak over the target after encoding | AAC adds peak | measure the final file; lower the limiter ceiling |
| Loudness missed the target by several LU | two-pass `loudnorm` silently fell back to dynamic mode | measure, gain, oversampled limiter, measure again; or accept only `normalization_type: linear` |
| Music hits a few milliseconds late | a compressor's lookahead delayed the music by 6 ms | measure the delay and remove it |
| The music disappeared under the engine sound | a continuous bed mixed too loud | keep beds low; check a spectrogram |
| Chimes with nothing on screen | sounds logged from hidden toasts | keep only sounds whose source is on screen |
| Clutter of far-off sounds | distant creatures and blocks kept | a distance cut-off, except for explosions and thunder |
| ffmpeg in a background job stopped | it was waiting on standard input | `-nostdin` |
