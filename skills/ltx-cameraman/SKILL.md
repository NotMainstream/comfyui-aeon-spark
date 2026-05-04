---
name: ltx-cameraman
description: Execute LTX video generation via ComfyUI -- this skill is the Cameraman subagent that runs on the DGX Spark AEON-7 ComfyUI container. Use comfyui-spark-ltx for the actual generation functions.
version: 1.1.0
category: creative
tags: [ltx, comfyui, video-generation, execution, subagent]
trigger: "When given an expanded LTX prompt and mode -- generates the video clip via ComfyUI/LTX"
---

# ltx-cameraman

The *Cameraman* subagent. Executes video generation using the AEON-7 ComfyUI container on DGX Spark.

**This skill delegates to comfyui-spark-ltx for the actual generation logic.**

In the movie studio pipeline, the Cameraman does not make creative decisions -- it executes the
generation as instructed by the Director. It receives:
- An expanded LTX prompt from the Scenarist
- A generation mode (T2V, I2V, or chained)
- A seed image path (for I2V or chaining)
- Output filename

And produces a video clip file.

---

## Role in the Pipeline

```
[ltx-director]
    |
    | "Generate SHOT 01.1 with this prompt, T2V mode"
    v
[ltx-cameraman]
    |
    | delegates to
    v
[comfyui-spark-ltx]
    |
    v
[video clip file]
```

---

## Cameraman Responsibilities

1. **Receive generation parameters** from Director
   - Prompt (expanded by Scenarist)
   - Mode: T2V, I2V, or chained
   - Seed image path (if I2V or chained)
   - Output filename

2. **Validate inputs** before generation
   - If I2V: verify seed image exists in workspace/input/
   - If chained: verify chain seed exists
   - Verify ComfyUI is reachable

3. **Execute generation** via comfyui-spark-ltx generate_clip()

4. **Verify output**
   - File exists and size > 100KB
   - Duration approximately 5-6 seconds
   - Copy to final output location

5. **Report results** to Director
   - Success: output path
   - Failure: error reason, prompt used (for debugging)

---

## Usage

The Cameraman is typically called via delegate_task from the Director:

```python
delegate_task(
    goal="Generate a video clip for the beach scene.",
    context="Mode: t2v. Prompt: Wide shot, cinematic -- a breathtaking tropical beach at golden hour...",
    toolsets=["terminal", "file", "web"]
)
```

Or the Director can call comfyui-spark-ltx directly. The Cameraman skill
exists as a conceptual role, not a separate tool.

---

## Quick Reference

```
ComfyUI URL:     http://localhost:8188
T2V workflow:    ../../workflows/ltx_t2v_pure.json
I2V workflow:    ../../workflows/ltx_i2v_api.json
Seed images:     ../../workspace/input/
Output dir:      ../../outputs/
```

For complete generation code, see the **comfyui-spark-ltx** skill.
