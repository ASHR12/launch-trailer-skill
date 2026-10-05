# Brief and story

Read before proposing a trailer, and again before fixing the timeline. The brief turns "make a crazy launch video" into a length, a shape, a hook and a storyboard the user has said yes to.

## Questions to ask

Ask only what the request leaves open; offer a default for each.

| Question | Why it matters | Default |
| :--- | :--- | :--- |
| Who should watch it, and where will it be posted? | Feeds reward a 1-second hook and vertical video; a store page or site suits 16:9 | social feeds and a site |
| How long, which shapes, which frame rate? | Sets the shot count and the grid | 60 s or less, 16:9 at 1920x1080, 60 fps |
| Everything, or the best few things? | "Cover everything" means about a second per feature | the best moments, every major feature glimpsed |
| Tone? | Music, colour and titles follow it | energetic and bright |
| Music and voice? | Rights, and whether someone records lines | music composed for it, no voice |
| End card and call to action? | The one thing viewers should do next | logo, "Play now" or "Try it free", the address |
| Which data or world to film in? | Real content looks best, but the user's own save must stay untouched | a copy loaded only into the capture profile |
| Anything never to show? | Personal names, accounts, unfinished features, other brands | none of those |
| Where do the files go, and may you install tools? | Videos are large and stay out of version control | a folder the user names; ask before installing |

When the user names a length, treat it as a maximum. Long cuts lose viewers: in the BlockHaven run the user cut the plan from two minutes to one ("2 min is too much").

## Formats and lengths

| Format | Length | Shots | Notes |
| :--- | :--- | :--- | :--- |
| Teaser | 6 to 15 s | 5 to 12 | one hook, the title, the end card |
| Launch trailer | 30 to 60 s | 20 to 55 | every major feature glimpsed, chapters with titles |
| Feature or gameplay movie | 60 to 120 s | 40 to 90 | slower, longer takes, can explain more |
| App demo | 30 to 90 s | 10 to 30 | the result first, then three to five payoffs |
| Vertical cut | 15 to 60 s | as the source | the same timeline reframed, or a tighter edit |
| Loop | 3 to 6 s | 1 to 3 | for a page header or a share card; seamless |

## Story shapes

1. **Montage** (the BlockHaven trailer): hook, title slam, chapters in energy order with a two-to-five-word title on each chapter's first beat, a finale, the end card.
2. **Transformation**: start from nothing (one block in the sky, an empty canvas, a blank document), the product builds it in seconds, the camera pulls back to reveal the whole, the logo.
3. **Small story**: a player or user's journey in five beats (start, gather, build, danger, triumph), with the title in the climax.
4. **App arc**: the result first (the finished thing on screen), the problem in two or three seconds, the product solving it in three to five payoffs, then the call to action.

## The hook

The first second decides whether anyone sees the second. Frame 1 is moving and already striking; the title or product name lands by about 3 seconds.

Good hooks:
- **Speed and scale**: a fast dive or flyover that passes close to something for parallax.
- **The action peak**: the biggest explosion, boss, crash or win, in slow motion, cut hard on the first big beat.
- **A transformation in two seconds**: empty to built, night to day, sketch to finished.
- **The result first** (apps): the impressive output before the steps that made it.

Never open on darkness, a slow fade in, a studio logo, a splash or loading screen, a menu, or the product's tools. In the BlockHaven run the user rejected an opening near the product's in-game guide: "you have to hook the people in the first few seconds. If you show the guide, why will people like it?" The fix: a sunrise dive over snowy peaks on frame 1, a hard cut to a slow-motion explosion on the first big beat, the title through the smoke at about 2 seconds.

## Energy order and pacing

- Bright spectacle and action first; contrast (night, danger, depth) in the middle; tools and conveniences as quick flashes near the end; the biggest moment and the logo in the finale.
- Interface moments get about a second in total, then cut to their payoff in the world: a search result pops up, cut to the arrow leading across the landscape; a command is typed, cut to the teleport landing.
- Slot lengths that worked at 60 fps and 144 BPM: lands and places 2 beats (0.83 s), interface flashes 1 to 1.5 beats, action set pieces 3 to 5 beats, the end card 6 beats (2.5 s). A 60-second cut held 54 shots.
- Vary scale and direction from shot to shot (wide, close, aerial, first person; left-moving after right-moving); never use a shot twice.

## The beat grid

Frames per beat = fps × 60 / BPM. Pick a tempo that gives whole frames, so every cut point lands exactly on a frame:

| fps | Tempos with whole frames per beat (frames) |
| :--- | :--- |
| 24 | 80 (18), 90 (16), 96 (15), 120 (12), 144 (10), 160 (9) |
| 30 | 90 (20), 100 (18), 120 (15), 150 (12), 180 (10) |
| 60 | 90 (40), 100 (36), 120 (30), 144 (25), 150 (24), 180 (20) |

- Half-beat slots are fine even when a half beat is not a whole number of frames (12.5 at 144 BPM and 60 fps): slot starts are rounded on the cumulative beat (as `placedCut()` in [project-files.md](project-files.md) does), never slot by slot, so no cut point is more than half a frame off. Prefer a tempo where half beats are whole too (120 BPM at 60 fps gives 15) when slots will be that short.
- If the trailer must also exist at another frame rate, choose a tempo that works for both.
- Make the length a whole number of bars: 60 s is 36 bars at 144 BPM or 30 bars at 120 BPM; 30 s is 18 bars at 144 BPM or 15 at 120.
- **A licensed or existing track sets the grid**: its tempo and its first downbeat decide the timeline, and chapters follow its phrases. See [audio.md](audio.md), Cutting to a licensed track.
- **Hits** are named beats where the picture, the titles and the music all land together: the first big cut, the title, a lightning strike, the logo, the call to action, the final hit. The music and the shake read them from the timeline.

## Chapter titles and callouts

- Two to five words, a verb or a promise: "Build anything", "Places to find", "Never get lost". Avoid internal feature names that newcomers would not know.
- Callouts carry one true number and a noun: "25 lands", "9 places to find". Read the number from the product's own registry and check it against what a user can reach: the BlockHaven plan said 10 places, but one of the ten was a starter chest that the game's own tools do not list, so the trailer says 9.

## The shot list

For each shot, write the subject, the camera move, the action, the start state and the **payoff frame** (the frame that makes the point). Then ask:
- Does it read in one second with no explanation?
- Does it move from its first frame?
- Is its payoff inside its slot, or does the action need a speed ramp or an earlier start?
- For risky shots (darkness, crowds, physics), is there a fallback angle or a neighbour that can grow to fill the slot?

## The storyboard for approval

Show a table the user can read in a minute, with your estimate below it:

```markdown
| Time | Chapter | What you see | Title |
| 0:00 | Hook | fast dive over sunlit peaks, hard cut to a slow-motion explosion | |
| 0:02 | Title | the logo slams in through the smoke | BLOCKHAVEN |
| 0:03 | World | one block in the sky, the world builds itself, six lands | A world that never ends · 25 lands |
| ... | | | |
| 0:53 | Finale | pull back from the player on a peak at sunset; the logo built from blocks | |
| 0:57 | End card | logo and call to action | PLAY NOW |
```

## Vertical and other shapes

Plan one timeline rendered per shape; each shape is then its own cut with its own versions, review files and verdict. Neither case study rendered its vertical version, so treat this as the plan to prove with previews, not a tested route:
- **Reframe for the shape**: in 3D, widen the vertical field of view so the subject stays in frame (the BlockHaven scripts add about 22 degrees, capped at 100); in 2D, switch the game to a portrait size or scale mode, or point the camera at a tall area; in an app, use a phone-width viewport so its mobile layout shows.
- **Re-lay the titles** (stacked logo, narrower lines) inside the vertical safe area.
- **Preview every shot in the new shape** before the long render, and give a shot that does not survive its beats to its neighbours. Make a separate, tighter edit when many shots fail.
- Render the shapes one after another, never at the same time on one machine.
