# Recipe: a virtual clock for browser builds

Inject this before any page script (with Playwright, `page.addInitScript({ path })`; with raw DevTools protocol, `Page.addScriptToEvaluateOnNewDocument`). While a take records, time stands still until the recorder calls `step()`, which advances exactly one frame: due timers fire, queued animation frames run with the new timestamp, and CSS and Web Animations are seeked to it. The product's frame loop, its timeouts and its CSS all see the same, perfectly even time, however long a frame takes to capture.

This is the clock both case-study trailers used (the second reused the first one's file unchanged), with three additions: the frame rate is configurable, `reseed()` restarts the random sequence for each take, and the date can be pinned. Set them from an earlier init script: `window.__VCLOCK_FPS = 30` changes the frame rate from 60; `window.__VCLOCK_EPOCH = Date.UTC(2026, 0, 1)` makes `Date.now()` start from a fixed moment instead of the real time.

```js
(() => {
  if (window.__vclock) return;
  const real = {
    now: performance.now.bind(performance),
    dateNow: Date.now.bind(Date),
    raf: window.requestAnimationFrame.bind(window),
    caf: window.cancelAnimationFrame.bind(window),
    setTimeout: window.setTimeout.bind(window),
    clearTimeout: window.clearTimeout.bind(window),
  };
  const FRAME_MS = 1000 / (window.__VCLOCK_FPS || 60);
  let stepped = false;
  let offset = 0;
  let vnow = 0;
  const now = () => (stepped ? vnow : real.now() - offset);
  const epoch = window.__VCLOCK_EPOCH ?? real.dateNow() - real.now();
  performance.now = now;
  Date.now = () => Math.floor(epoch + now());

  // Seeded randomness, so a re-render repeats. reseed() restarts it at the start of each take.
  const SEED = 0x9e3779b9;
  let seed = SEED;
  Math.random = () => {
    seed = (seed + 0x6d2b79f5) | 0;
    let t = seed;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };

  // Animation frames.
  let nextFrameId = 1;
  const frames = new Map(); // id -> { cb, rid }
  const runFrame = (id, t) => {
    const f = frames.get(id);
    if (!f) return;
    frames.delete(id);
    f.cb(t);
  };
  const requestLive = (id) => {
    frames.get(id).rid = real.raf(() => runFrame(id, now()));
  };
  window.requestAnimationFrame = (cb) => {
    const id = nextFrameId++;
    frames.set(id, { cb, rid: 0 });
    if (!stepped) requestLive(id);
    return id;
  };
  window.cancelAnimationFrame = (id) => {
    const f = frames.get(id);
    if (!f) return;
    if (f.rid) real.caf(f.rid);
    frames.delete(id);
  };

  // Timers. Zero-delay timeouts stay real (they only yield to the event loop). Virtual ids start far
  // above the browser's own, so clearing one can never clear the other.
  const timers = new Map(); // id -> { due, cb, args, every, rid }
  let nextTimer = 1_000_000_000;
  const fire = (id) => {
    const t = timers.get(id);
    if (!t) return;
    if (t.every !== null) {
      t.due += Math.max(1, t.every);
      if (!stepped) t.rid = real.setTimeout(() => fire(id), Math.max(0, t.due - now()));
    } else timers.delete(id);
    try {
      if (typeof t.cb === 'function') t.cb(...t.args);
      else (0, eval)(String(t.cb));
    } catch (e) {
      real.setTimeout(() => { throw e; }, 0);
    }
  };
  const schedule = (cb, ms, args, every) => {
    const delay = Math.max(0, Number(ms) || 0);
    if (every === null && delay === 0) return real.setTimeout(cb, 0, ...args);
    const id = nextTimer++;
    const t = { due: now() + delay, cb, args, every, rid: 0 };
    timers.set(id, t);
    if (!stepped) t.rid = real.setTimeout(() => fire(id), delay);
    return id;
  };
  const clear = (id) => {
    const t = timers.get(id);
    if (!t) return real.clearTimeout(id);
    if (t.rid) real.clearTimeout(t.rid);
    timers.delete(id);
  };
  window.setTimeout = (cb, ms, ...args) => schedule(cb, ms, args, null);
  window.setInterval = (cb, ms, ...args) => schedule(cb, ms, args, Math.max(1, Number(ms) || 0));
  window.clearTimeout = clear;
  window.clearInterval = clear;

  // CSS and Web Animations: paused, then seeked to virtual time each step. Finished animations that
  // fill forwards stay in getAnimations(); remember them, or they restart in a loop.
  const tracked = new Map(); // animation -> virtual start time
  const done = new WeakSet();
  const seekAnimations = () => {
    for (const a of document.getAnimations()) {
      if (done.has(a)) continue;
      let start = tracked.get(a);
      if (start === undefined) {
        if (a.playState === 'finished') { done.add(a); continue; }
        start = vnow - Math.max(0, a.currentTime ?? 0);
        tracked.set(a, start);
        try { a.pause(); } catch {}
      }
      const t = vnow - start;
      const end = a.effect?.getComputedTiming().endTime;
      try {
        if (typeof end === 'number' && Number.isFinite(end) && t >= end) {
          a.finish();
          tracked.delete(a);
          done.add(a);
        } else a.currentTime = t;
      } catch {}
    }
  };

  window.__vclock = {
    FRAME_MS,
    get stepped() { return stepped; },
    now,
    /** Restart Math.random's sequence (call at the start of every take; reseed engine generators too). */
    reseed(n = SEED) { seed = n | 0; },
    /** Stop time: animation frames and timers wait for step(). */
    pause() {
      if (stepped) return;
      vnow = now();
      stepped = true;
      for (const f of frames.values()) { if (f.rid) real.caf(f.rid); f.rid = 0; }
      for (const t of timers.values()) { if (t.rid) real.clearTimeout(t.rid); t.rid = 0; }
      tracked.clear();
      seekAnimations();
    },
    /** Let time run again from where it stood (loading, streaming, staging). */
    resume() {
      if (!stepped) return;
      stepped = false;
      offset = real.now() - vnow;
      for (const [id] of frames) requestLive(id);
      for (const [id, t] of timers) t.rid = real.setTimeout(() => fire(id), Math.max(0, t.due - now()));
      for (const a of tracked.keys()) { try { a.play(); } catch {} }
      tracked.clear();
    },
    /** Advance one frame of `ms` (default one frame): timers, then animation frames, then animations. */
    step(ms = FRAME_MS) {
      if (!stepped) this.pause();
      const target = vnow + ms;
      for (;;) {
        let next = null;
        let pick = 0;
        for (const [id, t] of timers) if (t.due <= target && (next === null || t.due < next.due)) { next = t; pick = id; }
        if (!next) break;
        vnow = Math.max(vnow, next.due);
        fire(pick);
      }
      vnow = target;
      for (const id of [...frames.keys()]) runFrame(id, vnow);
      seekAnimations();
      return vnow;
    },
  };
})();
```

## Using it

- Load and stream in live time (`resume()`), wait for the product's ready signals, call `reseed()` (and the engine's own reseed, if it has a random generator), then record: before each frame run the shot's per-frame code (camera, input), call `step(ms)`, then capture.
- `step(ms)` with a smaller `ms` is slow motion; larger is a speed ramp. Vary it per frame for ramps inside one take.
- The first `step()` pauses automatically. Call `resume()` between takes so loading and streaming run at full speed.
- Rendering may finish asynchronously on the GPU. Capture through the DevTools protocol after `step()` returns, and verify (below) that each image shows the frame just stepped, not the one before.

## Tests before trusting it

Run these on the real product, once, and again after any change to the clock:

1. **Even motion**: move something a known distance per second (a scripted camera turn, or an element positioned from `performance.now()` in a frame loop), step a few frames capturing each, and check that the captured pictures advance by exactly the same amount every frame.
2. **CSS follows**: trigger a CSS transition of known duration (a toast sliding in, a menu opening), step through it, and check it is the right fraction of the way on each frame and finished at the end. It starts on the frame after the style change, as in a normal browser.
3. **No restarts**: after an animation with `fill: forwards` finishes, step on and confirm it stays finished.
4. **Timers**: schedule a `setTimeout` for 100 ms and confirm it fires on the sixth or seventh step at 60 fps, not before.
5. **Repeatability**: record the same shot twice, reseeding before each, and compare (a frame comparison should find them identical, or within encoder noise).

## Known limits

- Engines often keep their own random generator, seeded from the time at boot; reseed it through the engine at the start of each take. Engines that smooth or clamp frame times need that switched off, or one step will not equal one frame of motion.
- Web Workers, `AudioContext` time and anything off the main thread keep real time. Render audio offline from logged events, and give worker-driven simulation a step hook.
- Media elements (`<video>`) play in real time; seek them each step if a shot shows one.
- If the product schedules work with `requestIdleCallback` or `MessageChannel`, wrap those too or confirm they do not affect the picture.
- A product that measures frame time and drops quality when frames are slow (dynamic resolution) must have that switched off for capture.
