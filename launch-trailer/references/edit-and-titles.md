# Edit and titles

Read when building the cut, the titles layer and the end card. Commands are in [recipes/ffmpeg.md](recipes/ffmpeg.md).

## Assembling a cut

1. `checkCut()` passes: the slots add up, chapters start on cut points, no shot is used twice.
2. For each slot, take its take from `inPoint` for exactly the slot's frames. If the take is short, hold its last frame and log a warning; a held frame must be a choice, not an accident.
3. Join the slots, renumber every frame on the output grid, add the shake on hits, lay the titles layer on top, convert to `yuv420p` with BT.709, mux the mix.
4. Write the next numbered cut, take the stills, log it in `CUTS.md`, run the mechanical checks ([review.md](review.md)).

Takes, music, effects and titles are separate inputs, so a change redoes only what it touches:

| Change | Redo |
| :--- | :--- |
| A title, a callout, the end card | the titles layer (or only its changed frames), the cut, the stills |
| One shot | its take (and the takes chained to it), the effects stem and mix (its sounds come from the take), the cut, the stills |
| The music | the music, the mix, the cut |
| The order or a slot's length | the timeline, the takes whose slots changed, the music, the effects, the titles, the cut |

Tell the user the cost of a change in these terms before making it.

## The titles layer

One page (or canvas program) draws every graphic: the title, chapter titles, callouts, captions, flashes, the end card and any graphic slot. It exposes three things:

```js
window.__titles = {
  setup(shape) { /* build the elements for '16x9' or '9x16' */ },
  frames: TOTAL_FRAMES,         // from the timeline file
  render(frame) { /* set every element's text, transform and opacity for this frame */ },
};
```

- **A pure function of the frame number**: no clock, no CSS animations, no unseeded randomness. `render(1234)` gives the same picture every time, in any order, so one frame can be re-rendered or checked alone.
- **Reads the timeline**: chapter starts, hits and slot positions come from the same file as the cut, so a re-timed edit re-times the titles.
- **The product's own art**: its logo, its fonts, its icon, drawn at whole-number scales for pixel art (`image-rendering: pixelated`). Wait for `document.fonts.ready` before capturing.
- **Numbers from the product**: import the registries that hold them (the list of lands, the places the product can locate) instead of typing numbers.
- **Captured with a transparent background**: set the page background to transparent through the DevTools protocol (`Emulation.setDefaultBackgroundColorOverride` with alpha 0), capture each frame at the output size and pipe it into an RGBA clip.
- **Checked as stills first**: render chosen frames (each title at its fullest, the end card) to images, and look at them at full size over mid-grey and over the busiest footage they will sit on, before rendering all frames.

## Title motion

| Element | Motion | Timing |
| :--- | :--- | :--- |
| Main title | slams in from large to exact with a small overshoot, a short decaying shake, a flash behind it | on its hit, where the storyboard puts the name; held about 1.5 s |
| Chapter title | the same slam, smaller | on the chapter's first beat, held about 1 to 1.2 s, gone before the next idea |
| Callout ("25 lands") | pops in with the number first | on a hit inside the chapter, about 1 s |
| Flash | full-frame white, fading over 6 to 12 frames | on big hits and hard transitions only |
| Logo reveal | builds or assembles, then holds | the finale, on its hit |
| End card | fades in over an opaque backdrop | the brief's end-card time, at the end |

A slam can be written as `scale = from + (1 - from) * outBack(t)` over about 9 frames, where `outBack` overshoots slightly before settling; the shake is a sine and cosine offset whose amplitude decays to zero over about 12 frames.

**Pixel-art titles** never scale smoothly: slam through whole-number scales on successive frames (for example 9x, 7x, 6x, then the final 5x), or slide and flash in at the final scale, and round every position to the art scale. The screen shake for pixel-art footage pads and crops by whole art pixels instead of scaling ([recipes/ffmpeg.md](recipes/ffmpeg.md), Screen shake on hits).

## Readability

- **Size**: on a 1080-pixel-tall 16:9 frame, BlockHaven's chapter titles were set at 150 px and callouts at 110 px; on the 1080-pixel-wide vertical layout, 104 px and 92 px. Anything a viewer must read should stay readable on a phone held upright.
- **Contrast**: a dark outline, a soft shadow or a dim band behind text over busy footage. Check the end card and every title over the actual footage, not a flat background.
- **Safe areas**: keep text and logos inside the central area of the frame. On vertical video, feed apps draw their own captions and buttons over the bottom fifth, the top and the right edge; keep important text out of those zones.
- **Reading time**: hold a title long enough to read twice: about a second for up to four words, longer for more, longest for the product name and the call to action.
- **Spelling**: check the product name's exact capitalization and every word at full size before the full render.
- **Text that ages**: version numbers, dates and "new in" labels date a trailer quickly. Use them only when the brief asks, and check them again just before delivery.

## Transitions

- **Hard cuts on beats** carry a fast montage. Dissolves blur the rhythm; save them for real time passing.
- **Flashes** sell hard transitions and big hits, and hide a moment that should not be seen (a teleport's loading frame). Too many flashes flatten everything; keep them for hits.
- **Screen shake** in the edit, on the biggest hits and lightly on title slams, decaying over 8 to 18 frames. Turn the product's own camera shake off during capture so the edit controls it.
- **Fades to black** only where the story sleeps or ends. A black stretch anywhere else reads as a broken file.

## Colour and exposure

- Stage for light rather than fixing it later: sun behind the camera, the brightness setting raised for caves and night, dusk instead of midnight for night creatures, a daytime storm so the lightning reads.
- Measure each slot's mean brightness ([recipes/ffmpeg.md](recipes/ffmpeg.md), Mechanical checks) and look hard at the darkest slots on a small screen. Night shots should still show their subject's silhouette.
- Keep one look across the cut. A global grade (`eq`, `curves`) is a last resort and must keep the product's own colours recognizable.
- Tag the master BT.709 so players do not shift its colours.

## The end card

- **An opaque backdrop**. In the BlockHaven cut the logo built from blocks in the world showed faintly through a partly transparent dim layer behind the card's own logo; it read as a mistake and was made opaque.
- **The brief's content, as one block**: usually the logo and a call to action, plus whatever else the brief lists (where to get it, platforms, store badges if the brand rules allow them, a date, credits), laid out and centred as a single group, not as separate pieces. When the brief is silent, propose content that suits the use case ([brief-and-story.md](brief-and-story.md)) and confirm it before building the titles. Every claim on the card is true and still current.
- **Long enough**: the brief's end-card time, and at least long enough to read twice once it is fully in (about 2.5 seconds for a logo and one short line, more for more text), under the music's last hit and its tail.

## Graphic slots

A slot with no footage under it (a plain mock of an app icon bouncing in a dock, a phone frame, a platform badge) is drawn by the titles layer over black. Declare it in the timeline as a graphic slot. Never film the user's real desktop, notifications or files for it.

## Vertical layouts

Lay the titles out again for 9:16 rather than shrinking the wide layout: stack the logo ("BLOCK" over "HAVEN"), break long titles onto two lines, raise the type slightly above centre, and keep the call to action above the bottom fifth.
