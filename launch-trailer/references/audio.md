# Audio

Read when choosing music, building the effects stem and mixing. Commands are in [recipes/ffmpeg.md](recipes/ffmpeg.md), Audio.

## Contents

- What you may use
- Music on the grid
- Cutting to a licensed track
- The product's own sounds
- Mixing and loudness
- Sync
- Voice
- The sources ledger

## What you may use

| Source | Use it when | Record |
| :--- | :--- | :--- |
| Composed in code with the product's own sound engine or synths | always: it is the product's own material | the ledger row |
| Synthesized from scratch (oscillators, noise, your own samples) | always | the ledger row |
| Written or performed by the user, or by someone they hired | they hold the rights for promotional use | the agreement |
| A licensed library track or effects pack | its license covers promotional video on every platform planned, and paid ads if any | the license file, track id and proof of purchase; automated claims can still appear, so keep the proof |
| Output of a music-generation service | its terms grant commercial rights to the output | the terms as they stood when used |
| Commercial songs, another game's or film's music, sound-alike melodies, unlicensed samples | never | |

The same rules cover fonts (the product's own, with their licenses, for example the SIL Open Font License) and recorded voice.

**What the case studies did.** BlockHaven: music composed in code from the game's own synth kit, instruments and mix graph (three original themes, D major with a minor turn for the night monsters and a lift to E major for the finale, 144 BPM), effects rendered with the game's own sound recipes, no voice. Lightning Sortie: original music synthesized from oscillators and noise in an offline audio context (120 BPM, D minor lifting to D major on the end card), effects synthesized and driven by logged gameplay, no voice.

## Music on the grid

- **Score as data.** Notes, drums and effects are events in beats, built from the timeline's chapters and hits: every hit gets a hit in the music, sections follow the chapters, and a re-timed edit re-times the music on the next render.
- **A trailer arc**:
  - energy from the first frame, or a short riser straight into the first big hit on the hook's hard cut;
  - the title on a hit;
  - a fill or a crash into each chapter's first beat;
  - a darker colour for the contrast section (a minor or modal turn, a half-time feel), pitched to the audience (BlockHaven's young players got cheeky rather than scary);
  - a lift for the finale (up a step, the full arrangement);
  - the final hit under the logo or the call to action, with a tail that rings out to the last frame.
- **Original melodies only**; never quote or imitate a known tune.
- **Render offline and deterministically**: an `OfflineAudioContext` in the browser or any offline synth, a seeded random generator, 48 kHz stereo, written as WAV.
- **Measure processing latency.** The BlockHaven game's master compressor delayed the music by 288 samples (6 ms at 48 kHz); the renderer measured it and removed it so hits land exactly on their beat times.
- **Report what cannot be heard**: integrated loudness, peaks, loudness per bar or per section, and a waveform and spectrogram image. An agent cannot listen; these numbers and pictures stand in, and the user listens to the sample.
- **Music helpers**: if a subagent composes, give it the timeline file, its own output folder, and tell it to write a complete first version early, then improve it. The first BlockHaven music helper stalled without saving anything; its replacement then collided with it, because the first was still running.

## Cutting to a licensed track

When the user wants a track they bought or licensed, the music leads and the picture follows.

1. **Read the license first.** Ask for the license text or the library account page. It must cover promotional or advertising video, online distribution on the platforms planned, paid ads if any are planned, and the trailer's lifetime; note any credit line it requires, and whether the track is registered with a content-identification system (libraries can usually clear a channel or a video). If the user cannot find a license that says so, do not use the track: compose instead and tell them why.
2. **Find the tempo and the first downbeat.** Take the tempo from the library's listing, and the first onset from the audio (`silencedetect`, [recipes/ffmpeg.md](recipes/ffmpeg.md)). Make a check file with a click on every beat from that downbeat mixed under the track, and have the user listen: drift or an off-beat click shows at once. A track with a drifting tempo (a live recording) needs a beat map, a list of measured beat times, instead of one tempo.
3. **Let the track set the grid.** The timeline takes the track's tempo, beat 0 at its first downbeat, and chapters on its phrase boundaries (every 4 or 8 bars). If the tempo does not give whole frames per beat, slot starts round on the cumulative beat.
4. **Cut the track to length on bar lines**: keep the sections that fit the arc (the strongest opening, a build, the peak, the track's own ending under the end card), and join them exactly on bar lines with crossfades of a few tens of milliseconds. Use the track's real ending; a fade-out is the last resort.
5. **Hits come from the music**: put the picture's big moments (the hook's hard cut, the title, the finale) on the track's accents, not the other way round.
6. **Record it** in the sources ledger with the license file, the track id and the proof of purchase.

## The product's own sounds

Real product sounds, placed exactly where they happen, make the footage feel alive.

1. **Log them during capture.** Wrap every way the product plays a sound (a manager's play, sound objects' play and stop, loops) to record each sound's id, frame, and its position and distance from the camera when it has one ([recipes/capture-harness.md](recipes/capture-harness.md)). Mute the browser rather than the product, and check the log fills on a test take: some products skip sound calls at zero volume.
2. **Choose and level them** with a keep-list by sound family. BlockHaven's, as the gain in dB applied to each family before the mix:

   | Family | Level |
   | :--- | :--- |
   | explosion, thunder | -1, 0 |
   | lightning, fuse hiss | -2, -3 |
   | bow, arrow hit, shears, levers and doors | -3 to -5 |
   | block break, block place, block hit | -5, -7, -9 |
   | interface clicks, toasts, level-up, eating | -4 to -7 |
   | creature calls | -10 |
   | footsteps, ambience | left out |

3. **Filter**:
   - only sounds whose source is on screen (a toast chime only where the toast shows); for products whose sounds carry no positions, decide from the shot what is on screen;
   - drop distant sounds, except explosions and thunder, and attenuate the rest by distance (BlockHaven dropped sounds beyond 32 blocks);
   - the same sound at most once within three frames, and a chain of booms thinned to a handful;
   - a sound that started just before its slot (a fuse) joins in progress;
   - in slow-motion shots, pitch the later booms down;
   - actions the capture makes silently (blocks placed by a debug call) get placement taps on eighth notes.
4. **Render offline** with the product's own sound recipes or files, each at `(slot start frame + frame in take - in-point) / fps` seconds, into a 32-bit float stem.

Synthesized effects work the same way when the product's sounds are not usable: the Lightning Sortie stem had an engine bed driven by camera distance and throttle, gun fire at the real fire rate, whooshes on pass-bys and explosions sized by distance. Keep continuous beds low: its engine bed masked the music until it was turned down.

## Mixing and loudness

- Stems as 32-bit float WAV; music plus effects summed without automatic gain changes; an optional dip in the music under the biggest hits.
- Gain to about **-14 LUFS integrated**, a common target for web and social video, then a limiter, with **true peak at or under -1 dBTP measured on the final encoded file**. Most feed and streaming players normalize loudness, so a louder mix gains nothing and adds distortion.
- Loudness range around 3 to 6 LU suits a trailer; much less sounds squashed (the Lightning Sortie test mix at 3.9 LU with a loud bed was over-compressed).
- Check by measurement: integrated loudness, true peak, loudness range, a per-second loudness contour (sections should step, hits should stand out), a spectrogram (a continuous bed covering the music shows as a band), and no clipped samples.
- 48 kHz stereo throughout; AAC at 256 kbit/s in the master.

Both case studies landed at -14.0 LUFS. Lightning Sortie's final file peaked at -2.2 dBTP; BlockHaven's at -0.9 dBTP, a tenth over the target after AAC encoding, accepted at the time. A limiter ceiling of -1.5 dBFS leaves room for that overshoot.

## Sync

- Place every sound at its frame time on the same grid as the picture, from the take logs and the timeline.
- Remove the measured delay of any processor with lookahead (compressors, limiters) from the music and the effects.
- Check the final file against the mix by cross-correlation at a few sharp transients (expect 0 samples), and spot-check that flashes, impacts and door clicks land on their frames.

## Voice

Only when the user asks for it, and only with the speaker's consent; the BlockHaven plan offered optional recorded lines, and the user chose music and game sounds only. Record clean, dip the music under speech, and caption it. A synthetic voice needs a license that covers promotional use and any disclosure the platforms require.

## The sources ledger

In `TRAILER.md`, one row per element that is not filmed footage:

```markdown
| Element | Source | License | Proof |
| Music | composed in code with the game's synth kit (music/score.ts) | the product's own | repo commit |
| Effects | the game's sound recipes, placed from take logs | the product's own | repo commit |
| Title font | the game's pixel font; reading font (SIL Open Font License 1.1) | OFL | font package |
```
