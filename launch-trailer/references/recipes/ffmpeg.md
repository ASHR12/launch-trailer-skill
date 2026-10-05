# Recipe: ffmpeg for trailers

Commands for every step after capture. Each one was run end to end on short generated test clips; adapt sizes, frame counts and paths. They are written for bash; long filter graphs go in a file passed with `-filter_complex_script`, which avoids quoting trouble. In filter files and expressions, escape commas inside functions as `\,`.

The commands assume 60 fps (for another rate, change every `60` that is a frame rate: `-r`, `settb=1/60`, `r=60`, and `-g` to twice the rate). `FRAMES` is the timeline's `TOTAL_FRAMES` and `DUR` its length in seconds, rounded up to a whole number; set both from the timeline, for example `FRAMES=2760 DUR=46` for 23 bars at 120 BPM.

## Contents

- Lossless takes
- Assembling the cut
- Screen shake on hits
- The master encode
- Platform copies
- Stills, contact sheets and side by side
- Mechanical checks
- Audio: stems, mix, loudness, offset
- Things that bite

## Lossless takes

The capture loop pipes images into this ([capture-harness.md](capture-harness.md)):

```bash
ffmpeg -f image2pipe -framerate 60 -c:v png -i - \
  -vf "scale=1920:1080:flags=lanczos+accurate_rnd+full_chroma_int,format=rgb24" \
  -c:v libx264rgb -preset ultrafast -crf 0 -r 60 takes/16x9/open-dive.mkv
```

`-c:v ffv1 -level 3` is an alternative lossless codec. Matroska stores these with a 1/1000 time base, so a 60 fps frame lasts 17 or 16 ms, not 16.67: the cut must renumber frames (next section).

For pixel art, capture at the output size with the game's own whole-number scaling and keep edges hard: `scale=1920:1080:flags=neighbor` instead of the Lanczos flags, and only when a resize is still needed.

## Assembling the cut

Generate the graph from the timeline, one chain per slot. Each chain trims its take from the in-point, holds the last frame if the take is short (`tpad`), and then every stream is renumbered on the 60 fps grid before the titles go on top:

```text
[0:v]trim=start_frame=57:end_frame=182,setpts=PTS-STARTPTS,tpad=stop_mode=clone:stop=125,trim=end_frame=125,setsar=1,format=gbrp[s0];
[1:v]trim=start_frame=0:end_frame=100,setpts=PTS-STARTPTS,tpad=stop_mode=clone:stop=100,trim=end_frame=100,setsar=1,format=gbrp[s1];
[s0][s1]concat=n=2:v=1:a=0,settb=1/60,setpts=N[base];
[2:v]settb=1/60,setpts=N,format=rgba[tt];
[base][tt]overlay=0:0:format=gbrp:shortest=1[vout]
```

```bash
ffmpeg -i takes/16x9/open-tnt.mkv -i takes/16x9/world-builds.mkv -i titles/16x9.mov \
  -filter_complex_script cut.filter -map "[vout]" -frames:v 225 -r 60 \
  -c:v libx264rgb -preset ultrafast -crf 0 cut-picture.mkv
```

- `settb=1/60,setpts=N` on the joined picture and on the titles layer is what keeps every title frame on its own picture frame; without it, frames are dropped or doubled at joins and titles slip by a frame. The frame-counter probe below proves it.
- A slot drawn entirely by the titles layer takes a black source: `-f lavfi -i color=c=black:s=1920x1080:r=60:d=<seconds + 0.1>`.
- The titles layer is an RGBA clip, for example PNG in MOV: `ffmpeg -f image2pipe -framerate 60 -c:v png -i - -c:v png -pix_fmt rgba -r 60 titles/16x9.mov`.
- Warn in the log whenever `tpad` actually pads a slot: a held last frame is a frozen frame unless it was meant.
- **Large cuts in two stages.** One graph over about 21 full-HD inputs stalled at 0% CPU in the Lightning Sortie run. Render each slot (or chapter) to a lossless segment first (trimmed, shaken, titled, `format=gbrp` pinned, its frame count checked with `ffprobe -count_frames`), then join the segments with the concat demuxer and encode the master in a second pass.
- **Overlay can drop the last frame** when it syncs two inputs (that run came out 3,479 frames instead of 3,480). Give the titles layer and any padded stream a couple of spare frames, cut with `-frames:v`, and assert the frame count of every segment and of the master.

## Screen shake on hits

Shake in the edit, never in the capture. A scaled-up copy is cropped with a decaying wobble on hit frames and laid over the picture only there. Generate one term per hit (`at`, strength `k`, length `len`) in the script that writes the graph:

```bash
# x term for a hit at frame 60, strength 1, 16 frames, amplitude 21 px (1.1% of 1920):
#   if(between(n\,60\,76)\,21*pow(1-(n-60)/16\,2)*sin((n-60)*2.3+0)\,0)
# y uses cos((n-at)*2.9+...) and about 1.5% of the height; join the terms of all hits with +.
cat > shake.filter <<EOF
[0:v]settb=1/60,setpts=N,split=2[b1][b2];
[b2]scale=1997:1123:flags=bicubic,crop=1920:1080:x='(iw-ow)/2+$SX':y='(ih-oh)/2+$SY'[shk];
[b1][shk]overlay=0:0:format=gbrp:enable='between(n\,60\,76)+between(n\,110\,124)'[v]
EOF
```

Keep `format=gbrp` on the overlay: without it the overlay converts every frame to YUV and back, frames away from the hits change too, and a format switch partway through can corrupt the segment (see Things that bite). In the full cut this chain sits between `[base]` and the titles overlay; BlockHaven used a strength of 1 for the biggest explosions, 0.3 for title slams, 8 to 18 frames each.

**Pixel art** must not be resampled. Pad the frame with smeared edges instead of scaling it up, and round every offset to the art scale (4 output pixels for art drawn at 4x), so art pixels move whole and stay sharp:

```bash
S=4; A=24   # art scale, and the padding (at least the largest offset)
X="if(between(n\,10\,26)\,$S*round(10*pow(1-(n-10)/16\,2)*sin((n-10)*2.3)/$S)\,0)"
Y="if(between(n\,10\,26)\,$S*round(8*pow(1-(n-10)/16\,2)*cos((n-10)*2.9)/$S)\,0)"
cat > shake.filter <<EOF
[0:v]settb=1/60,setpts=N,pad=iw+$((2*A)):ih+$((2*A)):$A:$A,\
fillborders=left=$A:right=$A:top=$A:bottom=$A:mode=smear,crop=1920:1080:x='$A+$X':y='$A+$Y',format=gbrp[v]
EOF
```

On a test clip this changed only the hit's frames, and every 4x4 art cell of a shaken frame stayed one solid colour.

## The master encode

```bash
ffmpeg -i cut-picture.mkv -i audio/mix.wav -map 0:v -map 1:a -frames:v "$FRAMES" -r 60 \
  -vf "scale=out_color_matrix=bt709:out_range=tv,format=yuv420p" \
  -c:v libx264 -preset slow -crf 15 -profile:v high -pix_fmt yuv420p \
  -colorspace bt709 -color_primaries bt709 -color_trc bt709 -color_range tv \
  -bsf:v h264_metadata=colour_primaries=1:transfer_characteristics=1:matrix_coefficients=1:video_full_range_flag=0 \
  -g 120 -c:a aac -b:a 256k -ar 48000 -ac 2 -shortest -movflags +faststart \
  "<Product>-Trailer-16x9-v01.mp4"
```

- The `h264_metadata` filter writes the colour description into the stream itself; x264 otherwise leaves primaries and transfer unset and players guess.
- `-frames:v` equal to the timeline's total frames makes the duration exact. CRF 15 at `slow` gave about 30 Mbit/s for a busy 1080p60 minute (227 MB); CRF 18 to 20 is plenty for an upload copy.
- `-g 120` is a keyframe every 2 seconds at 60 fps, which players and editors seek well.
- `+faststart` moves the index to the front so the file plays while it downloads.
- Pixel art: `yuv420p` stores colour at half resolution in 2x2 blocks. Keep each art pixel at least 2x2 output pixels and the art's grid on even output coordinates, so colour edges stay clean.

## Platform copies

```bash
# Smaller upload copy, same picture and sound
ffmpeg -i master.mp4 -c:v libx264 -preset slow -crf 20 -maxrate 12M -bufsize 24M -profile:v high \
  -pix_fmt yuv420p -c:a copy -movflags +faststart upload.mp4

# Vertical fallback from a 16:9 master: blurred fill behind the centred frame.
# Prefer filming the 9:16 shape and re-laying its titles; use this only when that is not possible.
ffmpeg -i master.mp4 -filter_complex "[0:v]split=2[bg][fg];\
[bg]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,gblur=sigma=30[bgb];\
[fg]scale=1080:-2[fgs];[bgb][fgs]overlay=(W-w)/2:(H-h)/2,format=yuv420p[v]" \
  -map "[v]" -map 0:a -c:v libx264 -preset slow -crf 18 -c:a copy -movflags +faststart vertical.mp4

# Share image (1200x630) from a chosen frame, without titles
ffmpeg -i cut-picture-no-titles.mkv -vf "select=eq(n\,1210),scale=1200:-2,crop=1200:630" \
  -fps_mode passthrough -frames:v 1 -q:v 2 share.jpg
```

## Stills, contact sheets and side by side

```bash
# A still at an exact frame
ffmpeg -i master.mp4 -vf "select=eq(n\,100)" -fps_mode passthrough -frames:v 1 -q:v 2 still-1-opening.jpg

# One contact sheet: first, middle and last frame of three slots, three per row
ffmpeg -i master.mp4 -vf "select='eq(n\,0)+eq(n\,30)+eq(n\,59)+eq(n\,60)+eq(n\,85)+eq(n\,109)+eq(n\,110)+eq(n\,130)+eq(n\,149)',\
scale=384:-2,tile=3x3:padding=4:color=white" -fps_mode passthrough -frames:v 1 -q:v 3 sheet-01.jpg

# Two versions side by side
ffmpeg -i v01.mp4 -i v02.mp4 -filter_complex "[0:v]scale=960:-2[a];[1:v]scale=960:-2[b];[a][b]hstack=inputs=2,format=yuv420p[v]" \
  -map "[v]" -c:v libx264 -crf 20 v01-v02.mp4
```

Many ffmpeg builds lack `drawtext` (it needs freetype; check with `ffmpeg -filters | grep drawtext`), so keep a separate text file mapping each sheet row to its slot, or draw labels in the titles page. For sheets made from many single frames, write them out first, then tile them with the concat demuxer: it resolves relative paths against the list file's folder, so write absolute paths and pass `-safe 0`.

## Mechanical checks

```bash
V=master.mp4
# Spec: codec, profile, pix_fmt, colour tags, frame rate, frame count, duration, audio
ffprobe -v error -show_entries stream=codec_name,profile,pix_fmt,color_space,color_primaries,color_transfer,color_range,r_frame_rate,nb_frames,sample_rate,channels:format=duration -of default=nw=1 "$V"

# Decode errors (the output must be empty)
ffmpeg -v error -i "$V" -f null - 2> decode-errors.txt

# Black stretches and frozen stretches (each must be a planned fade or hold)
ffmpeg -hide_banner -nostats -i "$V" -vf "blackdetect=d=0.1:pix_th=0.10,freezedetect=n=0.003:d=0.25" -an -f null - 2>&1 \
  | grep -E "black_start|freeze_start|freeze_duration"

# Detected cut points as frame numbers, to compare with the slot starts
ffmpeg -hide_banner -nostats -i "$V" -vf "select='gt(scene,0.3)',metadata=print:file=cuts.txt" -an -f null -
python3 -c "import re;print([round(float(x)*60) for x in re.findall(r'pts_time:([\d.]+)',open('cuts.txt').read())])"

# Brightness (YAVG) and motion (YDIF) per frame: dark shots and still shots per slot
ffmpeg -hide_banner -nostats -i "$V" -vf "signalstats,metadata=print:file=stats.txt" -an -f null -

# Which frames changed between two versions (decoded-frame hashes)
ffmpeg -v error -i v01.mp4 -map 0:v -f framemd5 v01.md5
ffmpeg -v error -i v02.mp4 -map 0:v -f framemd5 v02.md5

# How much they changed, per frame (inf means identical)
ffmpeg -hide_banner -nostats -i v01.mp4 -i v02.mp4 -lavfi "[0:v][1:v]psnr=stats_file=psnr.log" -f null -

# Bitrate per second: a sudden spike far above its neighbours means corrupted or noisy frames there
ffprobe -v error -select_streams v:0 -show_entries packet=pts_time,size -of csv=p=0 "$V" | python3 -c "
import sys,collections; b=collections.Counter()
for l in sys.stdin: t,s=l.strip().split(',')[:2]; b[int(float(t))]+=int(s)*8
[print(f'{k}s {b[k]/1e6:.1f} Mbit/s') for k in sorted(b)]"
```

**Frame-counter probe** for the titles overlay: render a stand-in titles layer whose top-left 16x16 pixels hold the frame number as brightness, run the real cut graph with it, and read the corner back. Every output frame must show its own number (modulo 256):

```bash
ffmpeg -f lavfi -i "color=c=black@0:s=1920x1080:r=60:d=$((DUR + 1)),format=rgba,\
geq=r='if(lt(X\,16)*lt(Y\,16)\,mod(N\,256)\,0)':g='if(lt(X\,16)*lt(Y\,16)\,mod(N\,256)\,0)':\
b='if(lt(X\,16)*lt(Y\,16)\,mod(N\,256)\,0)':a='if(lt(X\,16)*lt(Y\,16)\,255\,0)'" \
  -frames:v "$FRAMES" -c:v png -pix_fmt rgba counter.mov
# ... run the cut graph with counter.mov as the titles input, writing probe.mkv, then:
ffmpeg -i probe.mkv -vf "crop=16:16:0:0,format=gray" -f rawvideo counter.gray
python3 -c "
d=open('counter.gray','rb').read(); n=len(d)//256
bad=[i for i in range(n) if abs(d[i*256+136]-i%256)>1]; print(n,'frames;',len(bad),'show the wrong titles frame',bad[:10])"
```

## Audio: stems, mix, loudness, offset

Write every stem as 32-bit float WAV (`-c:a pcm_f32le`): a float stem keeps a peak above full scale for the limiter to handle, while a 24-bit stem clips it (a +6 dBFS test tone came back at +6.0 from float and 0.0 from 24-bit).

**The effects stem from sound files**: when the product plays sound files (rather than synthesizing them in code, which is better rendered offline in the page with the product's own audio code), build the stem from the edit's sound list: each event's file, pitched by its playback rate, levelled, delayed to its time in the cut, all summed in float. Generate the graph from the list:

```bash
# events.json: [{"time": 0.5, "file": "sfx/splash.wav", "gainDb": -3, "rate": 1}, ...]  time is seconds in the cut
python3 - "$DUR" <<'EOF' > stem.args
import json, sys
ev = json.load(open('events.json')); seconds = float(sys.argv[1])
ins, parts = [], []
for i, e in enumerate(ev):
    ins += ['-i', e['file']]
    ms = round(e['time'] * 1000)
    rate = '' if e.get('rate', 1) == 1 else f"asetrate={round(48000 * e['rate'])},aresample=48000,"
    parts.append(f"[{i}:a]aresample=48000,{rate}volume={e.get('gainDb', 0)}dB,adelay={ms}|{ms}[e{i}]")
mix = ''.join(f'[e{i}]' for i in range(len(ev)))
print(' '.join(ins))
print(';'.join(parts) + f";{mix}amix=inputs={len(ev)}:normalize=0:duration=longest,apad=whole_dur={seconds}[out]")
EOF
ffmpeg -nostdin $(sed -n 1p stem.args) -filter_complex "$(sed -n 2p stem.args)" -map "[out]" -t "$DUR" -ac 2 -c:a pcm_f32le audio/sfx.wav
```

`asetrate` changes pitch and speed together, as a game's playback rate does. Paths with spaces need the argument list built in a script rather than through `$(...)`. On a test log the sounds started at exactly their times (0.5, 1.5 and 2.0 s).

```bash
# 1. Mix music and effects without automatic gain changes
ffmpeg -i audio/music.wav -i audio/sfx.wav \
  -filter_complex "[0:a][1:a]amix=inputs=2:normalize=0:duration=first[a]" -map "[a]" -c:a pcm_f32le audio/premix.wav

# 2. Measure integrated loudness and true peak
ffmpeg -hide_banner -nostats -i audio/premix.wav -af ebur128=peak=true -f null - 2>&1 | sed -n '/Summary:/,$p'

# 3. Gain to the target, then an oversampled limiter at -1.5 dBFS (gain = -14 - measured I)
ffmpeg -i audio/premix.wav -af "volume=<gain>dB,aresample=192000,alimiter=limit=0.8414:attack=1:release=60:level=false,aresample=48000" \
  -c:a pcm_s24le audio/mix.wav

# 4. Measure again; if the limiter pulled the level down, add the difference to the gain and repeat step 3 once
ffmpeg -hide_banner -nostats -i audio/mix.wav -af ebur128=peak=true -f null - 2>&1 | sed -n '/Summary:/,$p'
```

- `alimiter` takes a linear ceiling between 0.0625 and 1: 0.8414 is -1.5 dBFS, 0.7943 is -2 dBFS.
- Measure the **final encoded file** too: AAC adds peak. A mix at -1.6 dBTP came back at -0.5 dBTP after AAC at 256 kbit/s on a test with very sharp transients; BlockHaven's -1.0 came back at -0.9. If the final file is over -1 dBTP, lower the ceiling and encode again.
- ffmpeg's two-pass `loudnorm` with `linear=true` silently switches to dynamic mode when the gain would push peaks over its ceiling or the loudness range is wider than its target, and on short or spiky material it can then miss the target by several LU. If you use it, read `normalization_type` in the second pass's JSON (`print_format=json`) and accept only `linear`; otherwise use steps 2 to 4.

**Cutting a licensed track** ([audio.md](../audio.md), Cutting to a licensed track): find where the music starts, make a click-track check file, and join bar-aligned sections with a short crossfade.

```bash
# First onset: the first silence_end
ffmpeg -nostdin -hide_banner -nostats -i track.wav -af silencedetect=noise=-40dB:d=0.05 -f null - 2>&1 | grep -m1 'silence_end'

# A click on every beat from the downbeat (here 120 BPM from 0.35 s; d=60 makes 60 s of clicks, so set it to
# the track's length), mixed under the track for the user to hear
ffmpeg -f lavfi -i "aevalsrc='if(gte(t\,0.35)\,0.5*sin(2*PI*1500*t)*exp(-200*mod(t-0.35\,60/120))\,0)':s=48000:d=60" -ac 2 click.wav
ffmpeg -i track.wav -i click.wav -filter_complex "[0:a][1:a]amix=inputs=2:normalize=0" check-click.wav

# Bars 1-4 joined to bars 9-12 (a bar is 2 s at 120 BPM), each part 10 ms long past its bar line, 20 ms crossfade
ffmpeg -i track.wav -filter_complex "[0:a]asplit=2[x][y];\
[x]atrim=start=0.35:end=8.36,asetpts=PTS-STARTPTS[a];[y]atrim=start=16.34:end=24.35,asetpts=PTS-STARTPTS[b];\
[a][b]acrossfade=d=0.02:c1=tri:c2=tri[out]" -map "[out]" -c:a pcm_f32le music.wav
```

**Offset between the mix and the final file**: decode both to mono float and cross-correlate a short window around a few sharp, isolated transients (explosions, hits). On steady tones the answer is meaningless.

```bash
ffmpeg -i audio/mix.wav -ac 1 -f f32le mix.f32
ffmpeg -i master.mp4 -map 0:a -ac 1 -ar 48000 -f f32le final.f32
node offset.mjs mix.f32 final.f32 1.28 37.9 50.4   # seconds of chosen transients; expect 0 samples
```

```js
// offset.mjs
import { readFileSync } from 'node:fs';
const load = (p) => { const b = readFileSync(p); return new Float32Array(b.buffer, b.byteOffset, b.byteLength / 4); };
const [a, b] = [load(process.argv[2]), load(process.argv[3])];
for (const sec of process.argv.slice(4).map(Number)) {
  const s = Math.round(sec * 48000) - 4800, n = 9600;
  let best = -Infinity, lag = 0;
  for (let l = -2400; l <= 2400; l++) {
    let acc = 0;
    for (let i = 0; i < n; i += 2) acc += a[s + i] * (b[s + i + l] ?? 0);
    if (acc > best) { best = acc; lag = l; }
  }
  console.log(`t=${sec}s: final vs mix ${lag} samples (${(lag / 48).toFixed(2)} ms)`);
}
```

## Things that bite

- **ffmpeg reads stdin**: run it with `-nostdin` from scripts and background jobs, or it can stop and wait for a key.
- **Pixel format switching mid-stream**: an overlay with `enable=` changed format partway through a Lightning Sortie segment, which came out at 240 to 300 Mbit/s with garbage in a grid of slices. Pin `format=gbrp` after every overlay, and check the bitrate per second.
- **zsh does not split words in variables**: `F="ffmpeg -y"; $F ...` fails; use a function or bash. An unmatched glob also aborts a zsh command before it starts (`rm clips/*` with no files killed a full recording run at launch); use `find <dir> -type f -delete`. And zsh reads `:e`, `:h`, `:r` and `:t` right after a variable as modifiers, so `atrim=start=$A:end=$B` breaks; write `${A}:end=${B}`.
- **Concat demuxer paths** resolve against the list file's folder: write absolute paths and pass `-safe 0`.
- **`select=eq(n\,N)`** needs `-fps_mode passthrough` (or `-vsync 0` on old builds) to emit exactly the selected frames.
- **The order of `-map`** decides stream order; map video first.
- **Git and other tools page their output** in scripts and can hang waiting: use `git --no-pager` or set `GIT_PAGER=cat`.
