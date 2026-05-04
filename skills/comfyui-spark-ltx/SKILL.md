---
name: comfyui-spark-ltx
description: Execute LTX video generation and ACE audio generation via ComfyUI on DGX Spark -- supports pure T2V, I2V, chained multi-segment video, and local music generation
version: 4.0.0
category: creative
tags: [ltx, ace, comfyui, video-generation, audio-generation, music-generation, dgx-spark, aeon-7]
trigger: "When given a prompt -- generates a video clip or music track via ComfyUI on the AEON-7 container"
---

# comfyui-spark-ltx (Cameraman) v4.0

Execute video generation using the AEON-7 ComfyUI container running on DGX Spark.
Receives expanded prompts from the *ltx-scenarist* and produces video clips.

**Role in the movie studio pipeline:**

```
[ltx-director] --> prompt --> [comfyui-spark-ltx]
                                  |
                                  v
                           ComfyUI / LTX
                                  |
                                  v
                           output .mp4 clip
```

---

## System Configuration

| Parameter | Value |
|-----------|-------|
| ComfyUI URL | http://localhost:8188 |
| Container name | comfyui-spark |
| Image | ghcr.io/aeon-7/comfyui-aeon-spark:bf16-flux2-ltx2.3 |
| Model | ltx-2.3-22b-dev.safetensors |
| LoRA | ltxv/ltx2/ltx-2.3-22b-distilled-lora-384-1.1.safetensors |
| GPU | NVIDIA GB10 (121.6GB VRAM total) |

---

## Critical: Seed Image & Bypass — The Key to T2V vs I2V

**The LTX workflow is fundamentally I2V.** Even with `bypass=True`, the LoadImage node is still referenced. The workflow works as pure T2V when bypass is set correctly AND the image reference chain is broken.

### How T2V Bypass Actually Works

In `LTXVImgToVideoConditionOnly`:
```python
use_i2v = image is not None and not bypass_i2v
```

For T2V, the workflow must:
1. Set `bypass=True` on both `LTXVImgToVideoConditionOnly` nodes (3159 and 4970)
2. When `bypass=True`, the image influence is zeroed but the image input still needs to be connected
3. The last-frame chaining works by: extract frame → use as image input → set `bypass=False` and `strength=1.0`

### The Seed Image Problem When Chaining

**Problem:** The `ImageScale` node (9990) references LoadImage (2004). Even when `bypass=True`, this creates a dependency. When you chain segments, the seed image from the FIRST segment pollutes subsequent segments.

**Symptoms:** "seed image showing" in output — means the I2V conditioning is leaking through bypass somehow, OR the LTXVPreprocess node is encoding a latent that still contains seed image influence.

**Root Cause:** The `LTXVPreprocess` node (3336) encodes the image into a latent representation. Even with `bypass=True` on `LTXVImgToVideoConditionOnly`, the preprocess pass may still be applied. When you chain (segment N → last frame → segment N+1), the last frame should be the ONLY image influence, but the workflow needs to be reset between generations.

**Solution for Chaining:**
- For segment 1: Use your actual seed image, or use T2V with bypass=True
- For segments 2+: Use ONLY the extracted last frame as image input, with bypass=False and strength=1.0
- The workflow MUST be re-loaded between segments to clear any cached state
- The `ImageScale` node (9990) always upscales to 1536×1536 — this is fine for chaining since it's just a preprocessing step

### Why Trimming 1.3s From Each Segment Doesn't Always Work

The "1.3s trim" was a workaround for the seed image leaking into the first frames. The real fix is:
1. Ensure bypass is properly set for T2V
2. For chained I2V, ensure the seed image is ONLY the extracted last frame
3. If seed still appears, set `bypass=True` on the relevant `LTXVImgToVideoConditionOnly` node that receives it

---

## Generation Modes

### Mode 1: Pure Text-to-Video (T2V)

No image input. Generate entirely from text prompt.
Use when: Starting a new scene with no reference image.

```
mode = "t2v"
bypass = True  # on both LTXVImgToVideoConditionOnly nodes
```

### Mode 2: Image-to-Video (I2V)

Use a seed image to guide the first frame and visual style.
Use when: You have a specific image/photograph to start from.

```
mode = "i2v"
image_file = "path/to/seed_image.png"  # must be in workspace/input/
strength_stage1 = 0.7   # first latent pass
strength_stage2 = 0.6    # second latent pass / upsampler
```

### Mode 3: Chained Segment Generation

For videos longer than one segment. LTX generates ~5s per segment on DGX Spark (longer on low-end GPU).
Strategy: Extract the LAST FRAME from the previous segment's output.
Use that frame as the I2V seed for the next segment with strength=1.0.

```
Segment 1: T2V or I2V (seed image) --> video_001.mp4
    |
    v
Extract last frame from video_001.mp4
    |
    v
Segment 2: I2V with extracted last frame (bypass=False, strength=1.0) --> video_002.mp4
    |
    v
Segment 3: I2V with last frame from video_002.mp4 (bypass=False, strength=1.0) --> video_003.mp4
... and so on
```

⚠️ **Critical**: Load a FRESH workflow JSON between each chain segment. Do not reuse the same workflow object — clear state between generations.

---

## Segment Lengths

| Model Variant | Resolution | FPS | Max Duration |
|--------------|------------|-----|-------------|
| ltx-2.3-fast (distilled) | 960×544 → 1920×1080 | 24/25 | 20s |
| ltx-2.3-dev (full) | 960×544 → 1920×1080 | 24/25 | ~5-6s on GB10 |
| ltx-2.3-pro (API) | up to 4K | 24-50 | 6-20s |

**On DGX Spark (GB10):** The full `ltx-2.3-22b-dev` model produces ~5s per segment comfortably. The distilled LoRA variant could produce longer segments but the AEON-7 container uses the dev model.

**Practical recommendation:** Generate 5s segments, then chain. For a 37s video: 8 segments of ~5s each (accept slight overlap trimming).

**To get longer segments:** Use the `EmptyLTXVLatentVideo` node to request more frames. At 24fps, 121 frames = ~5s. At 241 frames = ~10s. Test incrementally — GB10 VRAM (121GB) can likely handle 200+ frames.

---

## Workflow Files

| File | Purpose |
|------|---------|
| /home/a/comfyui-spark/workflows/ltx_t2v_pure.json | Pure T2V (bypass=True on both I2V nodes) |
| /home/a/comfyui-spark/workflows/ltx_i2v_api.json | Full I2V with image conditioning |

**IMPORTANT:** The `ltx_t2v_pure.json` has `bypass=False` on the I2V nodes. For true T2V, verify `bypass=True` is set on both nodes 3159 and 4970.

---

## API Submission Protocol

### Step 1: Verify Server Ready

```python
import requests
r = requests.get("http://localhost:8188/system_stats", timeout=5)
assert r.status_code == 200
info = r.json()
print(f"ComfyUI {info['version']}, GPU: {info['devices'][0]['name']}")
```

### Step 2: Load Workflow

```python
import json

MODE = "t2v"  # "t2v" or "i2v"
if MODE == "t2v":
    with open("/home/a/comfyui-spark/workflows/ltx_t2v_pure.json") as f:
        wf = json.load(f)
else:
    with open("/home/a/comfyui-spark/workflows/ltx_i2v_api.json") as f:
        wf = json.load(f)
```

### Step 3: Apply Prompt and Parameters

```python
main_prompt = "A breathtaking tropical beach at golden hour..."
negative_prompt = "blurry, low quality, distorted, deformed, ugly, bad anatomy, watermark, text, cartoon, animated, illustration, painting, drawing, video game, childish"

wf["2483"]["inputs"]["text"] = main_prompt
wf["2612"]["inputs"]["text"] = negative_prompt

# For I2V mode:
if MODE == "i2v":
    wf["2004"]["inputs"]["image"] = "your_seed_image.png"  # must exist in workspace/input/
    wf["3159"]["inputs"]["strength"] = 0.7   # stage 1 strength
    wf["4970"]["inputs"]["strength"] = 0.6   # stage 2 strength

# For CHAINED segments (segments 2+):
# Use the last frame from previous segment as image input
# Set bypass=False and strength=1.0 on the relevant LTXVImgToVideoConditionOnly
wf["2004"]["inputs"]["image"] = "chain_seed_002.png"  # extracted last frame
wf["3159"]["inputs"]["bypass"] = False
wf["3159"]["inputs"]["strength"] = 1.0
wf["4970"]["inputs"]["bypass"] = False
wf["4970"]["inputs"]["strength"] = 1.0
```

### Step 4: Submit

```python
import uuid, requests

client_id = str(uuid.uuid4())
payload = {"prompt": wf, "client_id": client_id}
r = requests.post("http://localhost:8188/prompt", json=payload, timeout=30)
assert r.status_code == 200, f"Submit failed: {r.text}"
prompt_id = r.json()["prompt_id"]
print(f"Submitted: {prompt_id}")
```

### Step 5: Poll for Completion

```python
import time

max_wait = 600  # 10 minutes max
interval = 10
start = time.time()

while time.time() - start < max_wait:
    r = requests.get(f"http://localhost:8188/history/{prompt_id}", timeout=10)
    if r.status_code == 200 and prompt_id in r.json():
        data = r.json()[prompt_id]
        if data["status"]["status_str"] == "success":
            outputs = data.get("outputs", {})
            for node_id, out_data in outputs.items():
                if "images" in out_data or "video" in out_data:
                    filename = out_data.get("images", out_data.get("video"))[0]["filename"]
                    return filename
        elif data["status"]["status_str"] == "error":
            raise RuntimeError(f"Generation failed: {data['status']}")
    time.sleep(interval)
raise TimeoutError(f"Generation timed out after {max_wait}s")
```

### Step 6: Copy Output from Container

```python
import subprocess

filename = "output_00001_.mp4"
src = f"comfyui-spark:/workspace/ComfyUI/output/{filename}"
dst = f"/home/a/comfyui-spark/outputs/{scene_id}_{shot_id}.mp4"
subprocess.run(["docker", "cp", src, dst], check=True)
```

---

## Extracting Last Frame for Chaining

```bash
# Extract the last frame (1 second before end, 1 frame)
ffmpeg -y -sseof -1 -i video_001.mp4 -frames:v 1 -q:v 2 last_frame.png
```

The `-sseof -1` seeks to 1 second before the end, ensuring we get the last actual frame.

**IMPORTANT:** LTX outputs at 1920×1088 (not 1920×1080). The last frame will have 8 extra pixels at the bottom. Crop before using as chain seed:

```bash
ffmpeg -y -i last_frame.png -vf "crop=1920:1080:0:0" chain_seed_002.png
```

---

## Batch Workflow Strategy (IMPORTANT)

**ACE (music) and LTX (video) cannot run simultaneously.** The ACE UNet and LTX model both require GPU VRAM. Loading one evicts the other.

### Optimal Order: Videos FIRST, Then Music

```
1. Generate ALL video segments (no music interruptions)
   - Segment 1 (T2V or I2V)
   - Segment 2 (I2V with last frame from seg 1)
   - Segment 3 (I2V with last frame from seg 2)
   - ... continue until all segments done
   
2. Unload LTX / Load ACE
   - ACE models auto-load when you submit an ACE workflow
   - No manual intervention needed
   
3. Generate all music tracks (ACE 1.5)
   - Theme track (full duration)
   - Ambient tracks if needed
   
4. Concatenate + Mix
   - ffmpeg concat for video
   - ffmpeg amix/crossfade for audio
```

This avoids反复 reloading models.

---

## Audio Mixing with ffmpeg

LTX generates video WITH synchronized audio (ambient sounds, dialogue, music hints). After concatenation, mix the ACE music track with the LTX ambient audio.

### Extracting Audio from LTX Output

```bash
# Extract audio from a single LTX segment
ffmpeg -y -i segment_001.mp4 -vn -acodec pcm_s16le -ar 44100 segment_001_audio.wav

# Extract audio from concatenated video
ffmpeg -y -i final_video_raw.mp4 -vn -acodec pcm_s16le -ar 44100 final_audio.wav
```

### Mixing Layers with amix

```bash
# Basic mix: ACE music + LTX ambient
ffmpeg -y \
  -i ace_music.mp3 \
  -i ltx_ambient.wav \
  -filter_complex "[0:a]volume=0.4[music];[1:a]volume=0.15[ambient];[music][ambient]amix=inputs=2:duration=longest[mixed]" \
  -map "[mixed]" -ar 44100 mixed_audio.wav
```

### Crossfade Between Music Sections

```bash
# Crossfade: fade out old music, fade in new music over 3 seconds
ffmpeg -y \
  -i music_v1.mp3 \
  -i music_v2.mp3 \
  -filter_complex "[0:a]afade=t=out:st=27:d=3[out0];[1:a]afade=t=in:st=0:d=3[out1];[out0][out1]acrossfade=d=3[cross]" \
  -map "[cross]" music_crossfaded.mp3
```

### Ducking (Lower Music When People Speak)

```bash
# Duck: lower music to 20% when speech is present
ffmpeg -y \
  -i music_with_speech.mp3 \
  -i speech_track.wav \
  -filter_complex "[1:a]volume=3[speech];[0:a][speech]sidechfade=level_in=1:level_out=0.2:attack=0.1:release=0.3[ducked]" \
  -map "[ducked]" ducked_music.mp3
```

### Layered Ambient Sounds

```bash
# Mix multiple ambient tracks (ocean, birds, crowd)
ffmpeg -y \
  -i ocean_waves.wav \
  -i birds.wav \
  -i crowd.wav \
  -filter_complex "[0:a]volume=0.25,aecho=0.8:0.88:60:0.4[ocean];[1:a]volume=0.12[birds];[2:a]volume=0.08[crowd];[ocean][birds][crowd]amix=inputs=3:duration=longest[ambient]" \
  -map "[ambient]" ambient_layered.wav
```

### Complete Final Mix Pipeline

```bash
# 1. Concatenate video segments
ffmpeg -y -f concat -safe 0 -i segment_list.txt -c copy video_raw.mp4

# 2. Extract audio from raw video
ffmpeg -y -i video_raw.mp4 -vn -acodec pcm_s16le -ar 44100 video_audio.wav

# 3. Mix ACE music + video ambient + layered ambients
ffmpeg -y \
  -i ace_theme.mp3 \
  -i video_audio.wav \
  -i ocean_layered.wav \
  -filter_complex "
    [0:a]volume=0.38,afade=t=out:st=33:d=4[music];
    [1:a]volume=0.12[ambient];
    [2:a]volume=0.15[ocean];
    [music][ambient][ocean]amix=inputs=3:duration=longest[mixed];
    [mixed]loudnorm=I=-16:TP=-1.5:LRA=11[out]
  " \
  -map "[out]" -ar 44100 final_audio_mixed.mp3

# 4. Combine video + mixed audio
ffmpeg -y \
  -i video_raw.mp4 \
  -i final_audio_mixed.mp3 \
  -map 0:v -map 1:a \
  -c:v libx264 -preset fast -crf 18 -r 30 \
  -c:a aac -b:a 192k \
  final_output.mp4

# 5. Compress for delivery
ffmpeg -y -i final_output.mp4 \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  final_compressed.mp4
```

---

## Complete generate_clip() Function

```python
import uuid, json, requests, subprocess, time, os

HOST = "http://localhost:8188"
WORKFLOW_T2V = "/home/a/comfyui-spark/workflows/ltx_t2v_pure.json"
WORKFLOW_I2V = "/home/a/comfyui-spark/workflows/ltx_i2v_api.json"
OUTPUT_DIR = "/home/a/comfyui-spark/outputs/"

NEGATIVE = "blurry, low quality, distorted, deformed, ugly, bad anatomy, watermark, text, cartoon, animated, illustration, painting, drawing, video game, childish"


def generate_clip(
    prompt: str,
    mode: str = "t2v",
    image_file: str = None,
    strength_s1: float = 0.7,
    strength_s2: float = 0.6,
    negative: str = NEGATIVE,
    output_name: str = "clip.mp4",
    max_wait: int = 600
) -> str:
    # Load FRESH workflow each time (clear state between segments)
    wf_path = WORKFLOW_T2V if mode == "t2v" else WORKFLOW_I2V
    with open(wf_path) as f:
        wf = json.load(f)

    # Set prompts
    wf["2483"]["inputs"]["text"] = prompt
    wf["2612"]["inputs"]["text"] = negative

    if mode == "t2v":
        # True T2V: bypass image conditioning
        wf["3159"]["inputs"]["bypass"] = True
        wf["4970"]["inputs"]["bypass"] = True
    elif mode == "i2v" and image_file:
        wf["2004"]["inputs"]["image"] = image_file
        wf["3159"]["inputs"]["strength"] = strength_s1
        wf["3159"]["inputs"]["bypass"] = False
        wf["4970"]["inputs"]["strength"] = strength_s2
        wf["4970"]["inputs"]["bypass"] = False

    # Submit
    client_id = str(uuid.uuid4())
    r = requests.post(f"{HOST}/prompt", json={"prompt": wf, "client_id": client_id}, timeout=30)
    r.raise_for_status()
    prompt_id = r.json()["prompt_id"]

    # Poll
    for _ in range(max_wait // 10):
        time.sleep(10)
        r = requests.get(f"{HOST}/history/{prompt_id}", timeout=10)
        if r.status_code == 200 and prompt_id in r.json():
            data = r.json()[prompt_id]
            if data["status"]["status_str"] == "success":
                outputs = data.get("outputs", {})
                for node_id, out_data in outputs.items():
                    if "images" in out_data or "video" in out_data:
                        filename = out_data.get("images", out_data.get("video"))[0]["filename"]
                        break
                break

    # Copy from container
    src = f"comfyui-spark:/workspace/ComfyUI/output/{filename}"
    dst = os.path.join(OUTPUT_DIR, output_name)
    subprocess.run(["docker", "cp", src, dst], check=True)
    return dst


def extract_last_frame(video_path: str, output_path: str) -> str:
    """Extract last frame and crop to 1920x1080 (removes 8px bottom padding)."""
    tmp = output_path + ".tmp.png"
    subprocess.run([
        "ffmpeg", "-y", "-sseof", "-1", "-i", video_path,
        "-frames:v", "1", "-q:v", "2", tmp
    ], check=True)
    subprocess.run([
        "ffmpeg", "-y", "-i", tmp,
        "-vf", "crop=1920:1080:0:0", output_path
    ], check=True)
    os.remove(tmp)
    print(f"Extracted and cropped last frame: {output_path}")
    return output_path
```

---

## Error Handling

| Error | Cause | Solution |
|-------|-------|----------|
| 400 Bad Request | Missing required widget | Check all required fields |
| Timeout | Generation took >10min | Retry; try shorter prompt |
| Empty outputs | Workflow crashed | Retry with simplified prompt |
| Seed image showing | I2V conditioning leaking | Set bypass=True on LTXVImgToVideoConditionOnly nodes |
| Non-monotonic DTS on concat | Timestamp discontinuity | Re-encode with ffmpeg instead of stream copy |

---

## Quick Reference

```
ComfyUI URL:     http://localhost:8188
T2V workflow:    /home/a/comfyui-spark/workflows/ltx_t2v_pure.json
I2V workflow:    /home/a/comfyui-spark/workflows/ltx_i2v_api.json
Prompt node:     2483 (text), 2612 (negative)
I2V image node:  2004 (LoadImage)
ImageScale:      9990 (1536x1536 lanczos)
Stage1 bypass:   3159 (bypass=True for T2V)
Stage2 bypass:   4970 (bypass=True for T2V)
Output node:     4852 (SaveVideo)
```

---

## Key Technical Findings

1. **T2V bypass:** Set `bypass=True` on BOTH `LTXVImgToVideoConditionOnly` nodes (3159 and 4970). This is already done in `ltx_t2v_pure.json`.

2. **The original workflow was fundamentally an I2V pipeline** — reducing strength alone does NOT remove image influence. Only `bypass=True` removes it completely.

3. **I2V strength tuning:** Stage 1=0.7, Stage 2=0.6 is a good starting point. For chain segments, use `bypass=False` and `strength=1.0`.

4. **Chaining is essential for long videos.** Extract the last frame from segment N and use it as the seed image for segment N+1. Crop the frame to 1920×1080 before use.

5. **Load a FRESH workflow JSON between each chain segment** — do not reuse the same workflow object.

6. **Batch order matters:** Generate ALL video segments FIRST, then generate music. This avoids反复 reloading LTX/ACE models.

7. **Segment length:** ~5s per segment on GB10 with full dev model. The distilled model could do longer but the container uses dev.

---

## Audio Generation — ACE 1.5 (No External API)

The AEON-7 container includes **ACE 1.5** (`acestep_v1.5_xl_turbo_bf16.safetensors`) for on-device music generation. No MiniMax API plan needed.

**Workflow file:** `/home/a/comfyui-spark/workflows/api/ace_1.5_music.json`

### Pipeline

```
UNETLoader (acestep_v1.5_xl_turbo_bf16)
     ↓
VAELoader (ace_1.5_vae.safetensors)         ← ACE VAE, NOT LTXVAudioVAELoader
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

⚠️ **Critical**: Use `VAELoader` + `VAEDecodeAudio` (NOT `LTXVAudioVAELoader` + `LTXVAudioVAEDecode`). ACE VAE and LTX VAE are different formats — using the wrong one causes `tuple index out of range` errors.

### Key Parameters

| Parameter | Values | Notes |
|-----------|--------|-------|
| `tags` | comma-separated descriptors | genre, instruments, mood, tempo/vibe |
| `lyrics` | text or `""` for instrumental | included in generation |
| `bpm` | 10-300 | 80-120 for reggae, 120-140 for upbeat pop |
| `keyscale` | full name: `"D major"`, `"A minor"`, etc. | NOT just `"major"` |
| `timesignature` | just the number: `"4"`, `"3"`, `"6"` | NOT `"4/4"` |
| `generate_audio_codes` | `true` = slower/better, `false` = faster | `false` for 30-90s generation |
| `cfg_scale` | 1.0-10.0 | Higher = more prompt adherence |
| `duration` | 10-200 seconds | Both `TextEncodeAceStepAudio1.5` AND `EmptyAceStep1.5LatentAudio` must match |

### Submit via API

```python
import json, requests, uuid, time

HOST = "http://localhost:8188"
with open("/home/a/comfyui-spark/workflows/api/ace_1.5_music.json") as f:
    wf = json.load(f)

# Customize
wf["4"]["inputs"]["tags"] = "upbeat reggae with warm bass guitar, laid back drum pattern, rhythmic guitar chords, organ melody, bob marley style, island vibes"
wf["4"]["inputs"]["lyrics"] = ""  # instrumental
wf["4"]["inputs"]["bpm"] = 80
wf["4"]["inputs"]["duration"] = 42.0
wf["4"]["inputs"]["keyscale"] = "D major"
wf["4"]["inputs"]["seed"] = 12345
wf["5"]["inputs"]["seconds"] = 42.0
wf["6"]["inputs"]["noise_seed"] = 12345

client_id = str(uuid.uuid4())
r = requests.post(f"{HOST}/prompt", json={"prompt": wf, "client_id": client_id}, timeout=30)
r.raise_for_status()
prompt_id = r.json()["prompt_id"]
```

### Monitor & Retrieve

```python
import time

while True:
    r = requests.get(f"{HOST}/history/{prompt_id}", timeout=30)
    d = r.json()
    if str(prompt_id) in d:
        status = d[str(prompt_id)]["status"]["status_str"]
        if status == "success":
            fname = d[str(prompt_id)]["outputs"]["13"]["audio"][0]["filename"]
            break
        elif status == "error":
            raise RuntimeError(f"ACE generation failed: {d[str(prompt_id)]['status']}")
    time.sleep(15)

# Output: /workspace/ComfyUI/output/ace_music_output_00001_.mp3
```

### Copy from Container

```python
import subprocess
src = f"comfyui-spark:/workspace/ComfyUI/output/{fname}"
dst = f"/home/a/comfyui-spark/outputs/{scene_id}_music.mp3"
subprocess.run(["docker", "cp", src, dst], check=True)
```

### Timing

| Setting | Duration | Approximate Time |
|---------|----------|-----------------|
| `generate_audio_codes=False`, `steps=30` | 30-45s audio | ~30-90 seconds |
| `generate_audio_codes=True`, `steps=50` | 30-45s audio | ~2-5 minutes |
| Any setting | 60s+ audio | ~3-5 minutes |

### Common ACE Errors

| Error | Fix |
|-------|-----|
| `CFGGuider: 'NoneType' is not iterable` | `negative` cannot be `None`. Use `ConditioningZeroOut`: `"negative": ["7", 0]` |
| `VAEDecodeAudio: tuple index out of range` | Wrong VAE. Use `VAELoader` (ace_1.5_vae), NOT `LTXVAudioVAELoader` |
| `timesignature: "4/4" not in list` | Use `"4"` (just the number) |
| `keyscale: "major" not in list` | Use full name: `"D major"`, `"A minor"`, etc. |
| `clip` required | Link DualCLIPLoader: `"clip": ["3", 0]` |
| `generate_audio_codes` default `true` | Set to `False` for faster generation |
| Output not found | ACE saves to `/workspace/ComfyUI/output/ace_music_output_*.mp3` |

### Audio Post-Processing

ACE outputs 44100Hz MP3 stereo. Trim/mix with ffmpeg:

```bash
# Trim to exact length
ffmpeg -y -i input.mp3 -t 37.4 -c copy trimmed.mp3

# Mix with ambient audio
ffmpeg -y -i ace_music.mp3 -i ambient_ocean.mp3 -i ambient_birds.mp3 \
  -filter_complex "[1:a]volume=-18dB[ocean];[2:a]volume=-24dB[birds];[ocean][birds]amix=inputs=2:duration=longest[amb];[0:a][amb]amix=inputs=2:duration=longest[mixed];[mixed]volume=1.4[out]" \
  -map "[out]" -ar 44100 final_mix.mp3

# Fade out over last 4 seconds
ffmpeg -y -i mixed.mp3 -af "afade=t=out:st=33.4:d=4" final_music.mp3
```
