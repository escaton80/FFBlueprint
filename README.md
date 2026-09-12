# FFBlueprint — User Manual #

FFBlueprint is a smart, zero-dependency command line builder for FFmpeg.
It comes as a single, portable HTML file that works 100% offline in any browser. 
Unlike generic builders, it features deep hardware encoder support 
(NVIDIA NVENC, Intel QSV, AMD AMF) using their true native flags, smart container rules, and precise two-pass workflows.
---

## Special features

What sets this builder apart from other FFmpeg command generators:

* **All three GPU encoder families, with their *real* flags.** AMD AMF, Intel QSV,
  and NVIDIA NVENC are covered — including the AV1 variant of each. The rate-control
  flags emitted are the ones each family actually accepts (`-rc constqp -qp` for NVENC,
  `-qp_i/-qp_p/-qp_b` for AMF, `-global_quality` for QSV), not generic flags that get
  silently ignored or rejected. QSV's implicit-CBR quirk (maxrate = bitrate, bufsize
  auto-doubled) is handled for you.
* **Quality boost checkbox.** One click adds the flags that close most of the
  quality gap between GPU and CPU encoders: adaptive quantization (AQ), lookahead,
  multipass, B-frames-as-references — gated per encoder, since the AV1 variants
  don't take every flag. On NVIDIA it also switches CQP to quality-mode VBR
  (`-cq`), which behaves like CRF. See [Hardware encoder options](#3-video).
* **Container selector with compatibility rules.** MKV is the default; switching to
  MP4 renames the output extension, locks subtitles to `mov_text`, switches on
  *Fast start*, and warns about Opus/FLAC audio. Invalid container/codec combos
  are prevented instead of produced.
* **Full subtitle handling.** Drop, stream copy, codec conversion (`mov_text` for
  MP4, `webvtt`, `srt`, `ass`), or permanent burn-in with automatic path escaping —
  plus a stream selector (`0:s:0`). Most generators ignore subtitles entirely.
* **Split by chapters.** Generates the ffprobe chapter probe, parses the pasted
  start times, and builds the `-f segment` command with `_001`, `_002`… output names.
* **Analyze Streams → stream pickers.** Builds the `ffprobe` command for you; paste
  the output and every stream is listed in a table (codec, resolution, fps, pixel
  format, language, bitrate, duration). Parses JSON, multi-object JSON, and plain
  terminal text — and the results feed straight into the **Audio Stream** and
  **Stream Selector** dropdowns, so multi-track files (several languages, several
  subtitles) can be picked by their actual content instead of guessed indices.
* **Correct two-pass workflow.** Pass 1 drops the subtitle filter and audio, writes
  to the null device, and warns about the classic `ffmpeg2pass-0.log` pitfall.
  A "Copy Both Passes" button grabs both commands at once.
* **Per-encoder everything.** CRF ranges and defaults match the codec (x264 0–51/23,
  x265 default 28, VP9/AV1 0–63, SVT-AV1 default 35), and preset lists adapt:
  p1–p7 for NVENC, speed/balanced/quality for AMF, deadline + cpu-used for VP9,
  profiles for ProRes. No invalid preset/CRF combinations.
* **Resolution done right.** Fill only one side and the other becomes `-2`
  (auto-calculated, kept even) — preventing the `width not divisible by 2` failure.
* **Single file, fully offline, zero dependencies.** One `FFBlueprint.html` — no Node
  build, no internet connection, nothing phoning home. Windows-aware (`NUL` device,
  backslash paths, automatic quoting). Just open it, build, copy, run.

---

## 1. Getting started

1. Open `FFBlueprint.html` in your browser (double-click it, or drag it into a browser window).
2. Fill in the sections top to bottom. The **Generated Command** box at the bottom
   updates live as you change anything.
3. Click **Copy** on the command box (or **Copy Both Passes** for two-pass encodes),
   paste into a terminal, and run.

> **Note about the input path:** browsers don't expose full file paths, so dropping a
> file or using the file picker only fills in the *filename*. The field flashes orange
> when that happens — type or paste the full path in front of it
> (e.g. `S:\Ai Videos\clip.mp4`). Paths are quoted for you automatically.

---

## 2. Input & Output

| Field | Meaning |
|---|---|
| **Input Path** | Full path to the source file. Surrounding quotes are OK; they get stripped. |
| **Output Filename** | Just a filename (`output.mkv`), or a full path if you want it elsewhere. |
| **Container** | **MKV (default)** or MP4. Switching it renames the output extension and applies compatibility rules: MP4 restricts subtitles to `mov_text`, enables *Fast start*, and warns about Opus/FLAC audio (poorly supported by MP4 players); MKV allows everything. |
| **Analyze Streams** | *Optional.* Opens a helper panel (see below). |
| **Drop zone** | Drag a video file here or click to browse (fills the filename only, see note above). |

### Analyze Streams (optional)

Since the app can't run FFmpeg itself, this panel helps you inspect the file in two
steps. You can skip it entirely — the command builds fine without it; the analysis
just fills the **Audio Stream** and **Subtitle Stream** dropdowns so you don't have
to guess track numbers:

1. **Copy the shown `ffprobe` command** and run it in a terminal against your file.
2. **Paste the output** into the text box. The app parses it (JSON or plain text)
   and lists every stream: index, type (video/audio/subtitle), codec, resolution,
   frame rate, pixel format, language, bitrate, and duration.

Use this to find e.g. which subtitle stream index to select, or whether your source
is 8- or 10-bit.

---

## 3. Video

### Encoder

* **Software** — runs on the CPU, best quality per bit, slowest:
  * `libx264` (H.264, most compatible), `libx265` (HEVC, smaller files)
  * `libvpx-vp9`, `libaom-av1`, `libsvtav1` (AV1)
  * `prores_ks` (editing/intermediate codec)
  * `copy` — remux without re-encoding (instant, no quality loss, no size change)
* **AMD AMF** — `h264_amf`, `hevc_amf`, `av1_amf`
* **Intel QSV** — `h264_qsv`, `hevc_qsv`, `av1_qsv`
* **NVIDIA NVENC** — `h264_nvenc`, `hevc_nvenc`, `av1_nvenc`

The options below the encoder change to match what that encoder actually supports —
the flag names shown in the generated command are the real per-encoder flags.

### Rate Mode (software encoders only)

| Mode | Behavior |
|---|---|
| **CRF** | Quality-targeted, one pass. You pick a quality level; the bitrate floats. Recommended default. |
| **1-Pass** | Fixed average bitrate (`-b:v`), fast, predictable size, lower quality per bit. |
| **2-Pass** | Bitrate-targeted, two runs. Best size/quality accuracy at a chosen bitrate. Slower — see [Two-pass workflow](#7-the-two-pass-workflow). |

* **CRF values** — lower = better quality and bigger files. Ranges/defaults differ per
  codec (x264 0–51 default 23, x265 default 28, VP9/AV1 0–63, SVT-AV1 default 35).
  Roughly ±6 ≈ half/double the bitrate. For x265, start around 26–30.
* **Preset** — slower presets compress better at the same quality. `medium` is a good
  default; `slow` for archiving. The label changes per encoder (Deadline / CPU-used /
  Profile for VP9 / AV1 / ProRes).
* **Tune** — optional per-content tweak (`film`, `animation`, `grain`, …). Leave at
  `none` unless you know you need it.

### Hardware encoder options

* **Rate Control** — `CQP` (constant quality, like CRF), `CBR` (constant bitrate,
  for streaming/recording), `VBR` (average bitrate with optional peak cap).
* **QP / Quality slider** — used in CQP mode. Lower = better. 20–26 is a sensible
  range for recording; 23 is the default.
* **Quality / Preset** — speed-vs-quality trade-off (`speed`…`quality` on AMF,
  `p1`…`p7` on NVENC). Middle is fine; higher costs GPU time for modest gains.
* **Bitrate / Max Rate / Buf Size** — used in CBR/VBR mode. Leave Max/Buf empty for a
  simple average-bitrate encode.

* **Quality boost** — one checkbox that adds the flags which most improve
  quality per bit for the selected encoder family: adaptive quantization
  (`-spatial-aq`/`-vbaq`), lookahead, multipass, and B-frames-as-references.
  On NVIDIA, CQP additionally switches to quality-mode VBR (`-rc vbr -cq … -b:v 0`),
  which behaves like CRF (the bitrate floats to hit the chosen quality).
  Pair it with pixel format `p010le` (10-bit) for the full effect.
  Needs a recent GPU and driver — see Troubleshooting if a flag is rejected.

### Common video options (hidden for `copy`)

* **Resolution** — width × height. Blank = keep original. You can fill only one side:
  the other becomes `-2`, which auto-calculates and keeps the size even
  (odd dimensions break many players/codecs). Examples: `1920 × ` or ` × 1080`.
* **Frame Rate** — common targets, or `custom...` for any value. Blank = keep original.
* **Pixel Format** — leave `auto` normally. Use `yuv420p10le` (software 10-bit) or
  `p010le` (hardware 10-bit) for HEVC/AV1 archival encodes; `yuv420p` fixes sources
  that some players can't decode (e.g. 4:4:4 or 4:2:2 x264).
* **GOP Size (`-g`)** — keyframe interval. Leave blank unless you have a specific
  requirement (e.g. `2×fps` for scrubbing/editing).

---

## 4. Audio

* **Codec** — `aac` (default, universal), `mp3`, `opus` (best quality at low
  bitrates, WebM/own container), `flac` (lossless, no bitrate setting),
  `ac3` (surround for home theater), `copy` (keep the original audio as-is),
  `none` (`-an`, strip audio).
* **Stream** — which audio track to keep, `auto (first)` by default (FFmpeg picks
  the track with the most channels). Run **Analyze Streams** and the dropdown
  lists every track with codec, channels, and language, e.g. `0:a:1 — aac · 6ch ·
  lang: ger`. Hidden when *Map all streams* is on (all tracks are kept anyway).
* **Bitrate** — 192k is a solid default for AAC; 128k is fine for speech.
* **Channels / Sample Rate** — usually *keep*. Set stereo/mono or 48 kHz when
  targeting a specific delivery spec.

---

## 5. Subtitles

| Mode | What it does |
|---|---|
| **Drop** (`-sn`) | No subtitles in the output. |
| **Copy stream** | Passes the subtitle through untouched. The text container must support the codec (e.g. SRT → MP4 is *not* copyable; use mov_text). |
| **Encode to codec** | Converts the selected stream — pick **mov_text** for MP4, webvtt for WebM/animated formats, srt/ass for MKV. |
| **Burn into video** | Permanently renders the subtitle file onto the picture (needs re-encode, styling is baked in). Enter the full path to the `.srt`/`.ass` file; special characters in the path are escaped for you. |

**Subtitle Stream** — which subtitle track to copy/encode, default `0:s:0` (the first).
Run **Analyze Streams** and the dropdown lists every subtitle track with its codec and
language; `custom...` restores free-text entry (e.g. `0:s:1`, or a full spec like
`0:s:m:language:ger`). Only used when *Map all streams* is off.

---

## 6. Extra

* **Overwrite (`-y`)** — don't ask before replacing the output file. On by default.
* **Fast start (`-movflags +faststart`)** — MP4 only: moves the index to the front so
  the file starts playing before it's fully downloaded. Managed by the Container
  selector: checked automatically for MP4, greyed out for MKV (where it does nothing).
* **Map all streams (`-map 0`)** — keep *every* stream (all audio tracks, all
  subtitles, chapters/data) instead of one of each. Hides the per-type **Audio
  Stream** and **Stream Selector** dropdowns, since individual picks don't apply.
* **Split by chapters** — see next section.
* **Custom Arguments** — appended verbatim at the end, e.g. `-ss 00:01:00 -t 60`
  to trim, `-metadata title="..."`, etc. Power-user escape hatch.

### Split by chapters

Cuts the output into one file per chapter:

1. Check **Split by chapters**. A third command box appears: the chapter probe.
2. Copy it, run it in a terminal — it prints one start time per line.
3. Paste **all** lines into the text box. The first line (`0.000`) is skipped
   automatically (the split points are the *ends* of each segment).
4. The main command gains `-f segment -segment_times ...` and the output name
   becomes `name_001.mp4`, `name_002.mp4`, …

---

## 7. The two-pass workflow

Choosing **2-Pass** shows two command boxes. **They must be run in order, from the
same folder:**

1. **Pass 1** analyzes the video and writes `ffmpeg2pass-0.log` into the current
   working directory. It produces no video file (`-f null NUL`) — that's normal,
   and it still takes a full encode's worth of time.
2. **Pass 2** uses that log for precise rate control and writes the actual output.

**If Pass 2 fails with `unable to open file ffmpeg2pass-0.log`:** Pass 1 wasn't run,
or it ran in a different folder. The log file is looked up relative to the directory
you launch the command from, so `cd` into one folder and run both passes there.
Use **Copy Both Passes** to grab them together. You can delete `ffmpeg2pass-0.log`
(plus `ffmpeg2pass-0.log.mbtree` for x265) afterwards.

---

## 8. Copying & the command box

* **Copy** (top-right of each box) copies that single command.
* **Copy Both Passes** appears only in two-pass mode and copies pass 1 and pass 2
  together, separated by a blank line.
* All paths and filter strings are quoted automatically; Windows-style backslash
  paths work as-is.

---

## 9. Quick recipes

* **Fast archive rip (GPU, NVIDIA):** encoder `hevc_nvenc`, RC `CQP`, QP 24,
  **Quality boost** on, pixel format `p010le`, audio `copy`, Fast start on.
* **Best-quality small file (CPU):** encoder `libx265`, CRF 27, preset `slow`,
  pixel format `yuv420p10le`, audio `opus` (MKV is the default container, which
  Opus needs), audio bitrate 160k.
* **Compatible web MP4:** container MP4, encoder `libx264`, CRF 21, preset `medium`
  (Fast start switches on automatically), audio `aac` 192k, subtitles `mov_text`.
* **Trim without re-encoding:** Custom Arguments `-ss 00:05:00 -to 00:15:00`,
  encoder `copy`, audio `copy`.
* **Fix a file that won't play on a TV:** encoder `copy` (or x264 re-encode if the
  codec itself is unsupported), pixel format `yuv420p`.

---

## 10. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `unable to open file ffmpeg2pass-0.log` | Pass 2 run before Pass 1, or in a different folder. Run Pass 1 first, same working directory. |
| `VBV maxrate specified, but no bufsize, ignored` | You set Max Rate but left Buf Size empty — the cap is silently dropped. Fill **Buf Size** (≈ 2× max rate is typical). |
| `Encoder not found` / `Unknown encoder 'h264_amf'` | Your FFmpeg build lacks that hardware encoder. Try a gyan.dev or BtbN "full" Windows build, and make sure the GPU drivers are current. |
| Output plays but looks blocky at high motion | CRF too high (raise quality = lower the number) or bitrate too low; check the Max Rate cap isn't biting. |
| Subtitle copy fails into MP4 | MP4 can't hold plain SRT — the Container selector switches subtitles to *mov_text* automatically; keep it on **Encode to codec**. |
| `Temporal AQ not supported`, `B frames as references are not supported`, or similar after enabling **Quality boost** | The GPU/driver is too old for one of the boost flags (temporal AQ & B-ref need NVIDIA Turing+, multipass needs driver 530+, AMF preanalysis needs recent AMD drivers). Uncheck **Quality boost**. |
| Odd-dimension error (`width/height not divisible by 2`) | Fill only one resolution field so the other auto-calculates with `-2`. |
