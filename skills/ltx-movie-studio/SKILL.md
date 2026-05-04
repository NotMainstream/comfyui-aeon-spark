---
name: ltx-movie-studio
description: End-to-end LTX video movie production using chained I2V segments — scenarist, director, cameraman in one skill. Run multi-shot productions on DGX Spark.
triggers:
  - "movie production"
  - "film production"
  - "ltx video chain"
  - "film studio"
  - "director's cut"
---

# LTX Movie Studio

A complete end-to-end movie production pipeline for AEON-7 ComfyUI + LTX Video Generation on DGX Spark. Produces multi-segment, temporally-chained videos from a story concept to finished movie.

## ⚠️ Critical: Batch Workflow — Videos FIRST, Music LAST

**ACE (music) and LTX (video) CANNOT run simultaneously.** Both models require GPU VRAM. Loading one evicts the other.

### Optimal Production Order

```
┌─────────────────────────────────────────────────────────┐
│  PHASE 1: VIDEO SEGMENTS (all segments, no music)       │
│  1. Generate ALL video segments in chain order           │
│  2. Segment 1: T2V (bypass=True, no seed image)         │
│  3. Segments 2+: I2V (seed = last frame of prev)        │
│  4. No model switching — LTX stays loaded               │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  PHASE 2: MUSIC (after all video done)                  │
│  5. ACE models auto-load when music workflow submits     │
│  6. Generate ACE music track(s)                         │
│  7. LTX model evicted from VRAM — that's OK             │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  PHASE 3: POST-PRODUCTION                                │
│  8. Concatenate video segments                           │
│  9. Mix ACE music + LTX ambient audio + ambient layers   │
│  10. Mux video + mixed audio                             │
│  11. Compress for delivery                               │
└─────────────────────────────────────────────────────────┘
```

### Why This Order Matters

- LTX video generation: ~5 min per segment, sequential
- ACE music generation: ~1-5 min per track
- If you switch mid-video, models反复 reload (minutes lost each time)
- Generate all video first, then all music, then mix

## System

- **ComfyUI**: `http://localhost:8188` (AEON-7 container `comfyui-spark`)
- **GPU**: NVIDIA GB10, 121.6GB VRAM
- **Container**: `comfyui-spark` (Docker)
- **Workflows**: `/home/a/comfyui-spark/workflows/ltx_i2v_api.json` (I2V), `ltx_t2v_pure.json` (pure T2V)
- **Output**: `/home/a/comfyui-spark/outputs/`
- **Container input**: `/workspace/ComfyUI/input/` (for chain seed images)

## Workflow Files

### `ltx_i2v_api.json` — I2V Chain Workflow (use for all chained segments)
API-format workflow with 3 required API fixes:
1. `LoadImage` node 2004: `"image"` must be a real filename in `/workspace/ComfyUI/input/`
2. `COMFY_DYNAMICCOMBO_V3` (ResizeImageMaskNode): **broken via API** — pre-replaced with `ImageScale` node 9990 (1536×1536, lanczos)
3. `LTXVPreprocess` node 3336: must include `img_compression=3`

Key nodes:
- `2483`: Positive prompt (CLIPTextEncode)
- `2612`: Negative prompt (CLIPTextEncode)
- `3159`: Stage 1 I2V conditioning — `strength` controls image influence (0.7 = moderate)
- `4970`: Stage 2 I2V conditioning — `strength=1.0` for chained segments
- `3059`: Empty latent video (121 frames × 24fps ≈ 5 seconds per segment)

### `ltx_t2v_pure.json` — Pure T2V Workflow (use for opening shot only)
Both I2V nodes (3159, 4970) have `bypass=True`. No seed image needed. Use for the first/establishing shot.

## Chaining Protocol

Each segment uses the last frame of the previous segment as its I2V seed image. This ensures temporal continuity:

```
Segment 01 (T2V, no seed)
  → extract last frame → segment_01.png
  → upload to container input
  → Segment 02 (I2V, seed=segment_01.png)
    → extract last frame → segment_02.png
    → upload to container input
    → Segment 03 (I2V, seed=segment_02.png)
      → ... continue chain ...
```

### Step-by-step chain

**⚠️ CRITICAL: Crop the last frame before using as chain seed.** LTX outputs 1920×1088 (not 1920×1080). The 8 extra pixels at the bottom cause problems if not removed.

1. **Extract last frame** from completed video:
   ```bash
   # Extract 1 second before end, 1 frame — gets true last frame
   ffmpeg -y -sseof -1 -i video.mp4 -frames:v 1 -q:v 2 /tmp/seg_N_frame.png

   # CRITICAL: Crop to 1920x1080 (removes 8px bottom padding from LTX output)
   ffmpeg -y -i /tmp/seg_N_frame.png -vf "crop=1920:1080:0:0" /tmp/seg_N_cropped.png
   ```

2. **Upload to container**:
   ```bash
   docker cp /tmp/seg_N_cropped.png comfyui-spark:/workspace/ComfyUI/input/segment_N.png
   ```

3. **Load fresh workflow** for the next segment (clear state between segments):
   ```python
   with open("/home/a/comfyui-spark/workflows/ltx_i2v_api.json") as f:
       wf = json.load(f)
   ```

4. **Configure I2V seed** for the next segment:
   ```python
   wf["2004"]["inputs"]["image"] = f"segment_N.png"  # the chain seed
   wf["3159"]["inputs"]["strength"] = 1.0    # full image influence
   wf["3159"]["inputs"]["bypass"] = False    # I2V mode
   wf["4970"]["inputs"]["strength"] = 1.0
   wf["4970"]["inputs"]["bypass"] = False
   ```

### Seed Image Leaking — Troubleshooting

**Problem:** The seed image (or a ghost of it) appears in the output video.

**Causes and fixes:**
| Symptom | Cause | Fix |
|---------|-------|-----|
| First few frames show original seed | `bypass=False` on stage 1 | Set `bypass=True` on stage 1 for T2V |
| Ghost of first frame in chained segment | Last frame extraction bad | Use `ffmpeg -sseof -1` not `-sseof -0.1` |
| Seed appearing mid-segment | I2V strength too high | Lower stage strength (0.5-0.7) |
| Distorted seed in output | Frame not cropped to 1920×1080 | Always crop extracted frames |

### Segment Length

- **Default:** 121 frames × 24fps = ~5.04 seconds per segment
- **Longer segments:** Increase `EmptyLTXVLatentVideo` `length` parameter (test 200-300 frames on GB10)
- **For 37s video:** ~8 segments of 5s each (some trimming overlap)

3. **Submit next segment** with seed image:
   ```python
   wf["2004"]["inputs"]["image"] = "segment_N.png"  # LoadImage node
   wf["3159"]["inputs"]["strength"] = 0.7  # stage 1 moderate
   wf["4970"]["inputs"]["strength"] = 1.0  # stage 2 full for chain continuity
   ```

4. **Copy output** from container after completion:
   ```bash
   docker cp comfyui-spark:/workspace/ComfyUI/output/output_NNNN_.mp4 local_name.mp4
   ```

5. **Concatenate all segments** with ffmpeg:
   ```bash
   # Create concat list
   for f in beach_s01*.mp4 beach_s02*.mp4 ...; do echo "file '$PWD/$f'" >> concat.txt; done
   ffmpeg -y -f concat -safe 0 -i concat.txt -c copy final_movie.mp4
   ```

## Python Production Script

```python
import json, requests, uuid, subprocess, time, os

HOST = "http://localhost:8188"
WORKFLOW_I2V = "/home/a/comfyui-spark/workflows/ltx_i2v_api.json"
WORKFLOW_T2V = "/home/a/comfyui-spark/workflows/ltx_t2v_pure.json"
OUTPUT_DIR = "/home/a/comfyui-spark/outputs/"
CONTAINER = "comfyui-spark"
NEGATIVE = "blurry, low quality, distorted, deformed, ugly, bad anatomy, watermark, text, cartoon, animated, illustration, painting, drawing, video game, childish, deformed face, bad hands"

def load_workflow(path):
    with open(path) as f:
        return json.load(f)

def submit(wf):
    r = requests.post(f"{HOST}/prompt", json={"prompt": wf, "client_id": str(uuid.uuid4())}, timeout=30)
    if r.status_code != 200:
        raise RuntimeError(f"Submit failed: {r.text}")
    return r.json()["prompt_id"]

def wait_for_completion(prompt_id, max_wait=900):
    start = time.time()
    while time.time() - start < max_wait:
        r = requests.get(f"{HOST}/history/{prompt_id}", timeout=10)
        if r.status_code == 200 and prompt_id in r.json():
            data = r.json()[prompt_id]
            status = data["status"]["status_str"]
            if status == "success":
                for node_id, out_data in data.get("outputs", {}).items():
                    for key in ["images", "video"]:
                        if key in out_data:
                            return out_data[key][0]["filename"]
            elif status == "error":
                raise RuntimeError("Generation failed")
        time.sleep(10)
    raise TimeoutError("Timed out")

def gen_seg(seg_id, prompt, mode, seed=None, s1=0.7, s2=1.0, out_name="segment.mp4"):
    wf = load_workflow(WORKFLOW_T2V if mode == "t2v" else WORKFLOW_I2V)
    wf["2483"]["inputs"]["text"] = prompt
    wf["2612"]["inputs"]["text"] = NEGATIVE
    if mode == "i2v" and seed:
        wf["2004"]["inputs"]["image"] = seed
        wf["3159"]["inputs"]["strength"] = s1
        wf["4970"]["inputs"]["strength"] = s2
    pid = submit(wf)
    cfname = wait_for_completion(pid)
    dst = os.path.join(OUTPUT_DIR, out_name)
    subprocess.run(["docker", "cp", f"{CONTAINER}:/workspace/ComfyUI/output/{cfname}", dst])
    # Extract and upload frame
    frame = f"/tmp/seg_{seg_id}_frame.png"
    subprocess.run(["ffmpeg", "-y", "-sseof", "-0.1", "-i", dst,
                    "-frames:v", "1", "-q:v", "2", frame])
    subprocess.run(["docker", "cp", frame, f"{CONTAINER}:/workspace/ComfyUI/input/segment_{seg_id}.png"])
    return dst
```

**IMPORTANT**: Use `terminal(background=true)` for long chains (10+ segments). The execute_code sandbox has a 5-minute hard timeout. Background processes continue running even if the shell that started them terminates.

## Segment Planning

A good movie structure:
1. **Opening shot** (T2V): Wide establishing shot, sets the scene
2. **Shot/reverse**: Move closer, introduce subjects
3. **Close-up detail**: Emotional or physical detail
4. **Action**: Motion, movement
5. **Reaction**: Emotional response
6. **Transition**: Scene change or time shift (sunset, night)
7. **New location**: Beach bar, dance floor, etc.
8. **Social scene**: Group activity
9. **Intimate moment**: Couple or individual
10. **Wide finale**: Pull back, reveal scope, fade out

## Timing

- Each segment: ~121 frames @ 24fps = ~5 seconds (default)
- **Longer segments possible:** Increase `EmptyLTXVLatentVideo` `length` to 200-300 (test on GB10)
- 10 segments ≈ ~50 seconds of raw content (no trim needed with proper chaining)
- Generation time: ~3-5 minutes per segment on GB10
- Total production time: ~45-60 minutes for a 10-segment movie (video only)

**Segment length strategy:**
| Latent frames | Output (~24fps) | Notes |
|---------------|-----------------|-------|
| 121 (default) | ~5.0s | Standard, fast |
| 200 | ~8.3s | Test for GB10 VRAM |
| 300 | ~12.5s | May exceed GB10 VRAM |

To increase latent frames, edit workflow node `3059` ("EmptyLTXVLatentVideo"): change `"length"` from 121 to desired number.

## Common Errors

| Error | Fix |
|-------|-----|
| `Invalid image file: segment_N.png` | Frame not uploaded to container. Run: `docker cp /tmp/seg_N_cropped.png comfyui-spark:/workspace/ComfyUI/input/segment_N.png` |
| VRAM exhausted | Wait for running job to finish. GB10 has ~11GB free during generation |
| Workflow validation error | Check LoadImage node path, img_compression=3 on LTXVPreprocess |
| `sseof` PNG write fails | Use `-frames:v 1 -q:v 2` (NOT `-update`) |
| Seed image showing in output | Set `bypass=True` on LTXVImgToVideoConditionOnly nodes for T2V; use cropped 1920x1080 frames for chaining |

## Post-Production Pipeline

After all segments are generated with proper chaining (no seed image leak):

### 1. Verify Segment Quality

```bash
# Check all segments exist and have proper duration
for f in outputs/beach_s{01..10}.mp4; do
  dur=$(ffprobe -v quiet -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "$f" 2>/dev/null)
  echo "$f: ${dur}s"
done
```

### 2. Concatenate Segments

```bash
# Create concat list (all files must have same codec/dimensions)
for f in outputs/beach_s{01..10}.mp4; do echo "file '$f'"; done > /tmp/concat.txt

# Stream copy (fast, but may have DTS issues)
ffmpeg -y -f concat -safe 0 -i /tmp/concat.txt -c copy video_raw.mp4

# If DTS issues: re-encode
ffmpeg -y -f concat -safe 0 -i /tmp/concat.txt \
  -c:v libx264 -preset fast -crf 18 -r 30 -pix_fmt yuv420p \
  video_raw.mp4
```

### 3. Crop to Proper 16:9

LTX outputs 1920×1088. Crop to 1920×1080:
```bash
ffmpeg -y -i video_raw.mp4 \
  -vf "crop=1920:1080:0:4" \
  -c:v libx264 -preset fast -crf 18 -r 30 \
  video_cropped.mp4
```

### 4. Audio Composition

#### ACE 1.5 Music + LTX Ambient Mix

**ACE generates the theme music. LTX generates ambient sounds (waves, crowd, etc.). Mix them together:**

```bash
# 4a. Extract ACE music (already generated separately)
# beach_v8_reggae.mp3 — ACE reggae theme track

# 4b. Extract audio from LTX concatenated video
ffmpeg -y -i video_cropped.mp4 -vn -acodec pcm_s16le -ar 44100 ltx_audio.wav

# 4c. Mix ACE music + LTX ambient + fade out
ffmpeg -y \
  -i beach_v8_reggae.mp3 \
  -i ltx_audio.wav \
  -filter_complex "
    [0:a]volume=0.38,afade=t=out:st=33:d=4[music];
    [1:a]volume=0.15[ambient];
    [music][ambient]amix=inputs=2:duration=longest[mixed];
    [mixed]loudnorm=I=-16:TP=-1.5:LRA=11[out]
  " \
  -map "[out]" -ar 44100 final_audio.wav

# 4d. Combine video + final audio
ffmpeg -y \
  -i video_cropped.mp4 \
  -i final_audio.wav \
  -map 0:v -map 1:a \
  -c:v libx264 -preset fast -crf 18 -r 30 \
  -c:a aac -b:a 192k \
  final_video.mp4

# 4e. Compress for Telegram delivery
ffmpeg -y -i final_video.mp4 \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  final_compressed.mp4
```

#### Layered Ambient (Optional)

If you want more ambient layers (ocean, birds, crowd) on top of LTX audio:
```bash
ffmpeg -y \
  -i ace_theme.mp3 \
  -i ltx_audio.wav \
  -i ocean_synthesized.wav \
  -i birds_synthesized.wav \
  -filter_complex "
    [0:a]volume=0.35,afade=t=out:st=33:d=4[music];
    [1:a]volume=0.12[ambient];
    [2:a]volume=0.10[ocean];
    [3:a]volume=0.08[birds];
    [music][ambient][ocean][birds]amix=inputs=4:duration=longest[mixed];
    [mixed]loudnorm=I=-16:TP=-1.5:LRA=11[out]
  " \
  -map "[out]" -ar 44100 final_audio.wav
```

### 5. Deliver

Final files:
- `final_video.mp4` — Full quality, 1920×1080, ACE music + ambient mix
- `final_compressed.mp4` — Compressed for Telegram/social media (smaller file size)

## Audio Generation — ACE 1.5 (No API Plan Needed)

The AEON-7 container includes **ACE 1.5** (`acestep_v1.5_xl_turbo_bf16.safetensors`) for on-device music generation. No external API needed.

### ACE 1.5 Music Workflow

**API endpoint**: `POST http://localhost:8188/prompt`

Workflow: `ace_1.5_music.json` in `workflows/api/`

Pipeline:
```
UNETLoader (acestep_v1.5_xl_turbo_bf16)
     ↓
DualCLIPLoader (qwen_0.6b_ace15, type=ace)
     ↓
TextEncodeAceStepAudio1.5 (tags, lyrics, bpm, key)
     ↓
EmptyAceStep1.5LatentAudio (seconds)
     ↓
RandomNoise → ConditioningZeroOut → CFGGuider → BasicScheduler → KSamplerSelect → SamplerCustomAdvanced
     ↓
VAEDecodeAudio (ace_1.5_vae) → SaveAudioMP3
```

⚠️ **Critical**: Use `VAELoader` + `VAEDecodeAudio` (NOT `LTXVAudioVAELoader` + `LTXVAudioVAEDecode`). ACE VAE and LTX VAE are different formats.

### Key Parameters

| Parameter | Values | Notes |
|-----------|--------|-------|
| `bpm` | 10-300 | 80-120 for reggae, 120-140 for upbeat |
| `keyscale` | C major, D major, ... B minor | Musical key |
| `timesignature` | "2", "3", "4", "6" | Beats per bar |
| `language` | "en", "ja", "zh", ... | Lyrics language |
| `generate_audio_codes` | `true` = slower/better, `false` = faster | LLM audio codes |
| `cfg_scale` | 1.0-10 | Higher = more prompt adherence |
| `duration` | 10-200 seconds | Longer needs more steps |

### Submit via API

```python
import json, requests, uuid

HOST = "http://localhost:8188"
with open("workflows/api/ace_1.5_music.json") as f:
    wf = json.load(f)

# Customize
wf["4"]["inputs"]["tags"] = "reggae tropical beach drums bass guitar relaxed happy"
wf["4"]["inputs"]["lyrics"] = ""  # instrumental
wf["4"]["inputs"]["bpm"] = 80
wf["4"]["inputs"]["duration"] = 45.0
wf["4"]["inputs"]["keyscale"] = "D major"
wf["5"]["inputs"]["seconds"] = 45.0
wf["4"]["inputs"]["seed"] = 12345
wf["6"]["inputs"]["noise_seed"] = 12345

r = requests.post(f"{HOST}/prompt", json={
    "prompt": wf,
    "prompt_id": str(uuid.uuid4()),
    "client_id": str(uuid.uuid4())
}, timeout=30)
prompt_id = r.json()["prompt_id"]
```

### Monitor & Retrieve

```python
import time
while True:
    r = requests.get(f"{HOST}/history/{prompt_id}", timeout=30)
    d = r.json()
    if str(prompt_id) in d:
        if d[str(prompt_id)]["status"]["status_str"] == "success":
            fname = d[str(prompt_id)]["outputs"]["13"]["audio"][0]["filename"]
            break
        elif d[str(prompt_id)]["status"]["status_str"] == "error":
            raise RuntimeError("ACE generation failed")
    time.sleep(15)

# Copy from container output dir
# docker cp comfyui-spark:/workspace/ComfyUI/output/ace_music_output_00001_.mp3 .
```

**Output location**: `/workspace/ComfyUI/output/ace_music_output_*.mp3`

### Timing

- `generate_audio_codes=True`: ~2-5 minutes for 30s audio (uses LLM for audio codes)
- `generate_audio_codes=False`: ~30-90 seconds (faster, slight quality tradeoff)
- With `steps=20` and `duration=20`: fastest test

### Common ACE Errors

| Error | Fix |
|-------|-----|
| `CFGGuider: 'NoneType' is not iterable` | `negative` cannot be `None`. Use `ConditioningZeroOut` to create a zeroed conditioning: `"negative": ["7", 0]` where node 7 is `ConditioningZeroOut` |
| `VAEDecodeAudio: tuple index out of range` | Wrong VAE type. Use `VAELoader` (ace_1.5_vae), NOT `LTXVAudioVAELoader` |
| `"timesignature": "4/4"` not in list | Use `"4"` (just the number) |
| `"keyscale": "major"` not in list | Use full name: `"D major"`, `"A minor"`, etc. |
| `clip` required | Link DualCLIPLoader to TextEncodeAceStepAudio1.5: `"clip": ["3", 0]` |
| `generate_audio_codes` default `true` | Set to `False` for faster generation |
| Output not in expected location | ACE saves to `/workspace/ComfyUI/output/ace_music_output_*.mp3` (same as video output) |

## Files

| File | Description |
|------|-------------|
| `workflows/api/ltx_i2v_api.json` | I2V chain workflow (all 3 API fixes applied) |
| `workflows/api/ltx_t2v_pure.json` | Pure T2V workflow (I2V nodes bypassed) |
| `workflows/api/ltx_t2v_api_fixed.json` | Reference workflow with fixes documented |
| `workflows/api/ace_1.5_music.json` | ACE 1.5 music generation workflow |
| `prompts/ltx_spiraling_library.txt` | Production-tested LTX prompts |
| `skills/ltx-movie-studio/SKILL.md` | This skill |
| `skills/ltx-scenarist/SKILL.md` | Prompt expansion |
| `skills/ltx-director/SKILL.md` | Chain orchestration |
| `skills/ltx-cameraman/SKILL.md` | Generation delegation |
| `skills/comfyui-spark-ltx/SKILL.md` | ComfyUI API operator |
