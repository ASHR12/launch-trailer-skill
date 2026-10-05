# Recipe: browser capture into lossless takes

A small capture "studio" for browser builds, condensed from the BlockHaven trailer tooling: Playwright drives Chromium at the output shape, the virtual clock ([virtual-clock.md](virtual-clock.md)) holds time still, and each frame is captured through the DevTools protocol at twice the output size and piped into ffmpeg as a lossless take. Adapt names to the product; keep the structure.

Plain ES modules for Node. Install `playwright` (or `@playwright/test`) and its Chromium in the project if they are not there.

## The studio

```js
import { chromium } from 'playwright';
import { spawn } from 'node:child_process';
import { mkdirSync, writeFileSync } from 'node:fs';
import { dirname } from 'node:path';

export const SHAPES = {
  '16x9': { id: '16x9', width: 1920, height: 1080 },
  '9x16': { id: '9x16', width: 1080, height: 1920 },
};

export class Studio {
  /** scale 2 and the Lanczos filter for 3D and vector rendering; scale 1 and 'neighbor' for pixel art. */
  static async open({ shape, profileDir, fps = 60, scale = 2, headed = false, format = 'png', initScripts = [], scaleFlags = 'lanczos+accurate_rnd+full_chroma_int' }) {
    mkdirSync(profileDir, { recursive: true });
    const ctx = await chromium.launchPersistentContext(profileDir, {
      headless: !headed,
      viewport: { width: shape.width, height: shape.height },
      deviceScaleFactor: scale,
      // GPU on, nothing throttled, no sound out (sound is rebuilt offline from the log).
      // On macOS also pass '--use-angle=metal'. Flags change between browser versions: drop any that are
      // rejected, and compare one still with a normal browser window, because a headless browser may
      // fall back to software rendering and look different.
      args: ['--enable-gpu', '--ignore-gpu-blocklist', '--mute-audio', '--autoplay-policy=no-user-gesture-required',
        '--disable-renderer-backgrounding', '--disable-background-timer-throttling'],
    });
    const page = ctx.pages()[0] ?? (await ctx.newPage());
    const cdp = await ctx.newCDPSession(page);
    const studio = Object.assign(new Studio(), { ctx, page, cdp, shape, fps, scale, format, scaleFlags, errors: [] });
    page.on('console', (m) => m.type() === 'error' && studio.errors.push(`[console] ${m.text()}`));
    page.on('pageerror', (e) => studio.errors.push(`[pageerror] ${e.message}`));
    for (const path of initScripts) await page.addInitScript({ path }); // clock.js first, then director.js
    return studio;
  }

  /**
   * One frame at output size x scale. The clip's scale is what makes it 2x; the device scale alone does
   * not. `optimizeForSpeed` is optional; drop it if the browser rejects it.
   */
  async grab() {
    const { width, height } = this.shape;
    const r = await this.cdp.send('Page.captureScreenshot', {
      format: this.format,
      ...(this.format === 'jpeg' ? { quality: 95 } : {}),
      optimizeForSpeed: true,
      clip: { x: 0, y: 0, width, height, scale: this.scale },
    });
    return Buffer.from(r.data, 'base64');
  }

  /**
   * Records `frames` frames into a lossless take. Randomness is reseeded first (`reseed` is extra page
   * code for the engine's own generator). Before each frame, `perFrame` (source of `(i, arg, d) => void`)
   * runs in the page, then the clock steps `stepMs`, then the page is captured. `warm` frames are stepped
   * first and not recorded. `before(i)` runs in Node (real key presses). `until` stops `after` frames past
   * a named mark; `preview` saves three stills instead of a take; `meta` (build id, script hash) goes
   * into the manifest.
   */
  async record(file, frames, perFrame, { arg = null, warm = 2, stepMs = 1000 / this.fps, before, until, preview = false, reseed = '', meta = {} } = {}) {
    const page = this.page;
    mkdirSync(dirname(file), { recursive: true });
    const ms = (i, marks) => (typeof stepMs === 'function' ? stepMs(i, marks) : stepMs);
    await page.evaluate((code) => { window.__vclock.reseed(); if (code) (0, eval)(code); }, reseed);
    await page.evaluate(({ code, arg }) => window.__dir.begin(code, arg), { code: perFrame, arg });
    for (let i = -warm; i < 0; i++) await page.evaluate(({ i, ms }) => window.__dir.frame(i, ms), { i, ms: ms(0, {}) });
    const enc = preview ? null : encoder(file, this.shape, this.fps, this.format, this.scaleFlags);
    const stills = preview ? [0, Math.floor(frames / 2), frames - 1] : [];
    let marks = {};
    let last = null;
    let recorded = 0;
    for (let i = 0; i < frames; i++) {
      if (before) await before(i);
      const step = await page.evaluate(({ i, ms }) => window.__dir.frame(i, ms), { i, ms: ms(i, marks) });
      marks = step.marks;
      recorded++;
      if (enc) {
        const img = step.hold && last ? last : await this.grab(); // a held frame repeats the last picture
        last = img;
        if (!enc.proc.stdin.write(img)) await new Promise((r) => enc.proc.stdin.once('drain', r));
      } else if (stills.includes(i)) {
        writeFileSync(file.replace(/\.mkv$/, `-p${stills.indexOf(i)}.${this.format === 'jpeg' ? 'jpg' : 'png'}`), await this.grab());
      }
      if (until && marks[until.mark] !== undefined && i >= marks[until.mark] + until.after - 1) break;
    }
    if (enc) { enc.proc.stdin.end(); await enc.done; }
    const { sounds } = await page.evaluate(() => window.__dir.end());
    const take = { file, frames: recorded, marks, sounds, ...meta, recorded: new Date().toISOString() };
    writeFileSync(file.replace(/\.mkv$/, '.json'), JSON.stringify(take, null, 1));
    return take;
  }

  async close() { await this.ctx.close(); }
}

/** ffmpeg reading images from stdin, resizing to the output size, writing lossless RGB. */
function encoder(file, shape, fps, format, scaleFlags) {
  const proc = spawn('ffmpeg', ['-nostdin', '-hide_banner', '-loglevel', 'error', '-y',
    '-f', 'image2pipe', '-framerate', String(fps), '-c:v', format === 'png' ? 'png' : 'mjpeg', '-i', '-',
    '-vf', `scale=${shape.width}:${shape.height}:flags=${scaleFlags},format=rgb24`,
    '-c:v', 'libx264rgb', '-preset', 'ultrafast', '-crf', '0', '-r', String(fps), file]);
  let err = '';
  proc.stderr.on('data', (d) => (err += d));
  const done = new Promise((ok, fail) => proc.on('close', (c) => (c === 0 ? ok() : fail(new Error(`ffmpeg: ${err}`)))));
  return { proc, done };
}
```

For a frame rate other than 60, pass `fps` here and set `window.__VCLOCK_FPS` to the same value in an init script that runs before the clock. `ffv1` (`-c:v ffv1 -level 3`) is an equally good lossless intermediate. Either way the takes carry millisecond timestamps; the cut renumbers frames ([ffmpeg.md](ffmpeg.md)).

## In-page director helpers (`director.js`, injected after the clock)

```js
(() => {
  if (window.__dir) return;
  const lerp = (a, b, t) => a + (b - a) * t;
  const angleLerp = (a, b, t) => { // the short way round
    let d = (b - a) % (Math.PI * 2);
    if (d > Math.PI) d -= Math.PI * 2;
    if (d < -Math.PI) d += Math.PI * 2;
    return a + d * t;
  };
  const ease = {
    linear: (t) => t,
    inOut: (t) => (t < 0.5 ? 2 * t * t : 1 - 2 * (1 - t) * (1 - t)),
    smooth: (t) => t * t * (3 - 2 * t),
    in: (t) => t * t, out: (t) => 1 - (1 - t) * (1 - t),
    inCubic: (t) => t ** 3, outCubic: (t) => 1 - (1 - t) ** 3,
    outExpo: (t) => (t >= 1 ? 1 : 1 - 2 ** (-10 * t)),
  };
  const state = { fn: null, arg: null, frame: -1, sounds: [], marks: {}, recording: false, hold: false };
  const d = {
    lerp, angleLerp, ease,
    /** Yaw and pitch looking from a to b. Match the product's convention (here yaw 0 looks along -Z). */
    look(a, b) {
      const [dx, dy, dz] = [b[0] - a[0], b[1] - a[1], b[2] - a[2]];
      return { yaw: Math.atan2(-dx, -dz), pitch: Math.atan2(dy, Math.hypot(dx, dz)) };
    },
    viewAt(pos, target, fov = 70, roll = 0) {
      const l = d.look(pos, target);
      return { x: pos[0], y: pos[1], z: pos[2], yaw: l.yaw, pitch: l.pitch, fov, roll };
    },
    lerpView(a, b, t) {
      return { x: lerp(a.x, b.x, t), y: lerp(a.y, b.y, t), z: lerp(a.z, b.z, t), yaw: angleLerp(a.yaw, b.yaw, t),
        pitch: lerp(a.pitch, b.pitch, t), fov: lerp(a.fov, b.fov, t), roll: lerp(a.roll ?? 0, b.roll ?? 0, t) };
    },
    orbit(center, radius, angle, height, fov = 70) {
      return d.viewAt([center[0] - Math.sin(angle) * radius, center[1] + height, center[2] - Math.cos(angle) * radius], center, fov);
    },
    cam(view) { window.__game.camera.set(view); },          // the product's debug camera hook
    mark(name) { if (!(name in state.marks)) state.marks[name] = state.frame; },
    hold() { state.hold = true; },                         // this frame repeats the last picture
    /**
     * Log every sound the product plays, with its frame. Call once per entry point: `target` is an object
     * or a prototype (wrap the prototype when sound objects are created later), `methods` maps method
     * names to kinds ('play', 'loop', 'stop'), and `describe(args, self)` returns { id, position?, ...extra }
     * where extra holds whatever the offline render needs (volume, rate, detune, loop).
     */
    logSounds(target, methods, describe, cameraPos) {
      for (const [name, kind] of Object.entries(methods)) {
        const original = target[name];
        if (typeof original !== 'function') continue;
        target[name] = function (...args) {
          if (state.recording) {
            const { id, position, ...extra } = describe(args, this) ?? {};
            const c = cameraPos?.();
            const distance = position && c ? Math.hypot(...position.map((v, k) => v - c[k])) : 0;
            state.sounds.push({ kind, id, position, ...extra, frame: state.frame, distance });
            d.mark(`${kind === 'stop' ? 'stop' : 'sound'}:${id}`);
          }
          return original.apply(this, args);
        };
      }
    },
    begin(code, arg) {
      state.fn = code ? (0, eval)(`(${code})`) : null;
      Object.assign(state, { arg, sounds: [], marks: {}, frame: -1, recording: true });
    },
    async frame(i, ms) {
      state.frame = i;
      state.hold = false;
      if (state.fn) await state.fn(i, state.arg, d);
      window.__vclock.step(ms);
      return { marks: state.marks, hold: state.hold };
    },
    end() { state.recording = false; state.fn = null; return { sounds: state.sounds, marks: state.marks }; },
  };
  window.__dir = d;
})();
```

## A shot script

```js
/** A flyover between two views over the slot, eased; the stage step is product-specific. */
export const landDesert = {
  id: 'land-desert',
  async run({ studio, frames, file }) {
    await stage(studio, { stand: [-435, 289], time: 'morning', weather: 'clear', hud: false });
    const from = { x: -428, y: 92, z: 289, yaw: 1.37, pitch: -0.34, fov: 72, roll: 0 };
    const to = { x: -442, y: 90, z: 290, yaw: 1.62, pitch: -0.3, fov: 72, roll: 0 };
    const perFrame = ((i, a, d) => {
      const t = d.ease.inOut(i / (a.frames - 1));
      d.cam(d.lerpView(a.from, a.to, t));
    }).toString();
    const take = await studio.record(file, frames, perFrame, { arg: { frames, from, to } });
    return { ...take, inPoint: 0 };
  },
};
```

`stage()` resumes live time, closes open screens, sets mode, time and weather, teleports the player near the shot, waits for `ready()` and `areaReady()`, clears stray creatures and hides overlays ([capture.md](../capture.md), Staging a shot). Event-timed takes pass `until: { mark: 'sound:world.explosion', after: 125 }` and set `inPoint` to the mark minus the frames the slot wants before the event. A 2D shot is the same with a different view: interpolate the camera's scroll position, and change zoom only by whole steps for pixel art.

## Installing the sound log

Once per page, after the product is reachable and before the first take, wrap every way it plays sounds, at the lowest level they all pass through:

```js
// An audio engine with play(id, { position, volume }) and loop(id, { position }):
d.logSounds(engine, { play: 'play', loop: 'loop' },
  (a) => ({ id: a[0], position: a[1]?.position, volume: a[1]?.volume, loop: false }), cameraPos);
// Sound objects with play(config) and stop(): wrap their class's prototype, reached from a sound the
// product already made (or the framework's exported class), so sounds created later are logged too.
d.logSounds(Object.getPrototypeOf(existingSound), { play: 'play', stop: 'stop' },
  (a, self) => ({ id: self.key, volume: a[0]?.volume ?? self.volume, rate: a[0]?.rate ?? self.rate, loop: a[0]?.loop ?? self.loop }));
```

- **Count each sound once.** If a manager's own `play(key)` creates a sound object and calls its `play()`, wrapping both logs every sound twice: wrap only the lower level, or drop a second entry with the same id on the same frame.
- **Do not create sounds just to reach a prototype**: a throwaway sound may need a loaded asset and stays registered with the manager.
- **Record what the offline render needs**: loop flags, volume, playback rate, detune.
- Then record a short test take and check that the log fills and matches what the take shows. A product that skips sound calls when its own volume is zero logs nothing, so mute the browser instead.

## A runner

One command per step, and existing takes are kept, so a change redoes only what it touches:

```text
node tools/trailer/run.js preview --shape 16x9 [--only id,id]   three stills per shot
node tools/trailer/run.js takes   --shape 16x9 [--only id,id] [--force]
node tools/trailer/run.js music
node tools/trailer/run.js sfx     --shape 16x9                  effects stem and the mix
node tools/trailer/run.js titles  --shape 16x9
node tools/trailer/run.js cut     --shape 16x9                  the next numbered cut, then stills
```

The runner calls `checkCut()` first, starts its own server on its own port (or takes `--base URL`), films in timeline order with all shots that need a fresh data copy grouped, passes each take's build id and a hash of its shot script as `meta`, adds the shot's `inPoint` to the manifest, logs each take's frame count against its slot, and prints the page errors at the end. Inputs that are not in the repo, such as the data copy to film in, come from an option or an environment variable.
