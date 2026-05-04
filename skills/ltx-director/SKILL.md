---
name: ltx-director
description: Orchestrate multiple LTX video scenes into coherent sequences -- manages shotlists, continuity, transitions, chaining, and post-production ordering
version: 1.1.0
category: creative
tags: [ltx, movie-production, orchestration, sequencing, continuity, chaining]
trigger: "When the user wants to produce a multi-scene video production -- coordinates scenarist and cameraman subagents"
---

# ltx-director

The *Director* agent in an AI movie production studio. Receives a story or scene list from a producer,
breaks it into individual shots/scenes, and orchestrates the *Scenarist* (prompt expansion)
and *Cameraman* (LTX execution) subagents to produce each scene. Then sequences the final clips.

**Role in the pipeline:**

```
Producer (human/AI)
    |
    v
[ltx-director] <-- orchestrates everything
    |
    +-- [ltx-scenarist] --> expanded prompts
    |
    +-- [comfyui-spark-ltx] --> video clips
    |
    v
Sequenced movie
```

---

## Core Concepts

### Scene vs. Shot
- **Scene**: A unit of story (e.g., "the hero enters the tavern")
- **Shot**: One continuous camera recording (one LTX generation = one shot)
- One scene may require multiple shots from different angles

### Shot Types
- **Establishing shot (EWS)**: Opens a location, wide, usually first
- **Primary shot**: Main coverage of the scene's action
- **Insert shot (CU)**: Close detail (key, letter, face reaction)
- **Cutaway**: Related but parallel action
- **OTS (Over-the-shoulder)**: Standard for dialogue coverage

### Generation Modes
- **T2V (Text-to-Video)**: Pure prompt generation. Use for opening shots of scenes.
- **I2V (Image-to-Video)**: Use a seed image to guide style/first frame.
- **Chained**: For long sequences, extract last frame from segment N and use as seed for segment N+1.

### Coverage
In traditional film, directors shoot multiple angles for the same action (wide + close-up + OTS).
This gives editors flexibility. For AI video, pre-plan coverage by generating multiple
variations of key shots.

### Continuity
Elements that must remain consistent across shots in a scene:
- Character appearance (clothing, hair)
- Location and time of day
- Lighting direction and color temperature
- Props and set dressing
- Camera angle logic (180-degree rule)

### Transitions
- **Cut**: Direct join (default, use this most)
- **Fade in/out**: For scene beginnings/endings
- **Dissolve**: For time passage or dream sequences
- **Match cut**: Action connects across cuts

---

## The Production Workflow

### Phase 1: Pre-Production -- Shotlist

Given a story description, create a shotlist:

```
SCENE 01: "The hero enters the tavern"
  SHOT 01.1: EWS -- Wide establishing shot of the tavern exterior at night
  SHOT 01.2: MS -- Hero pushes open the wooden door, looks around
  SHOT 01.3: CU -- Close-up of the hero's weathered face, eyes adjusting to dim light
  SHOT 01.4: MS -- Hero walks to the bar, places a coin on the counter

SCENE 02: "The negotiation"
  SHOT 02.1: OTS -- Hero and the tavern keeper over the bar, mid-conversation
  SHOT 02.2: CU -- The coin on the bar, a hand reaches for it
  SHOT 02.3: MS -- Hero leans in, voice low, the keeper's expression hardens
```

### Phase 2: Expansion -- Scenarist

Each shot is sent to the *ltx-scenarist* skill to expand into a full LTX prompt.
The director ensures prompts maintain character/location continuity by passing shared context.

### Phase 3: Production -- Cameraman

Each expanded prompt is sent to the *comfyui-spark-ltx* skill to execute on ComfyUI/LTX.
The director manages:
- Generation order (establishing shots first)
- Failed generations (retry with prompt refinement)
- Quality check (does clip match the intended shot?)
- Chaining for long segments (extract last frame, use as next seed)

### Phase 4: Post-Production -- Sequencing

Order the clips into a timeline and define transitions.

---

## Chaining for Long Videos

LTX generates approximately 5-6 seconds of video per generation. For longer sequences,
use the chaining technique:

```
Step 1: Generate Segment 1 (T2V or I2V with seed image)
  --> output: segment_001.mp4

Step 2: Extract last frame from segment_001.mp4
  --> output: chain_seed_001.png
  Command: ffmpeg -y -sseof -1 -i segment_001.mp4 -frames:v 1 -q:v 2 chain_seed_001.png

Step 3: Generate Segment 2 using chain_seed_001.png as seed image
  --> output: segment_002.mp4
  Important: Set strength=1.0 on the I2V stage for maximum continuity

Step 4: Repeat: Extract last frame from segment_002.mp4
  --> chain_seed_002.png

Step 5: Generate Segment 3 using chain_seed_002.png as seed
  --> segment_003.mp4

... continue until the full sequence is complete
```

### Chaining Parameters
- **Mode**: Always "i2v" for chained segments (the seed image is the anchor)
- **Image file**: The extracted last frame PNG
- **Strength**: Use 1.0 for maximum continuity (the seed image must fully determine the first frame)
- **Prompt**: Continue the action from where the previous segment ended

### Continuity Across Chained Segments
Each chained segment must:
1. Start with a prompt that continues naturally from where the previous ended
2. Reference the visual state established at the end of the previous segment
3. Maintain consistent lighting, camera angle, and character positioning

Example of chained prompts:
- Segment 1: "A woman walks into the tavern. She pauses at the entrance, eyes adjusting to the dim light."
- Segment 2 (chain from last frame): "The woman takes three steps toward the bar, her boots echoing on the wooden floor, her gaze scanning the room."
- Segment 3 (chain from last frame): "She reaches the bar and places a silver coin on the worn wood surface."

---

## Shot Type Reference

| Shot Type | Code | Use When |
|-----------|------|---------|
| Extreme Wide / Establishing | EWS | Opening a location, establishing geography |
| Wide Shot | WS | Full figure, environment matters |
| Medium Shot | MS | Dialogue, action, natural framing |
| Close-Up | CU | Emotion, reaction, key detail |
| Extreme Close-Up | ECU | Intense focus (eyes, hands, object) |
| Over-the-Shoulder | OTS | Dialogue between two characters |
| Point of View | POV | Subjective camera |
| Insert | INS | Cut to object (phone, letter, clock) |
| Cutaway | CUT | Parallel action intercut |

---

## Transition Reference

| Transition | Use When |
|-----------|---------|
| Cut (default) | Most shots -- immediate join |
| Fade in | Scene opening, waking up |
| Fade out | Scene ending, fade to black |
| Dissolve / Crossfade | Time passage, dream, memory |
| Match cut | Action continues across scenes |
| Jump cut | Time compression, jarring |
| Wipe | Editorial transition, flashbacks |

---

## Production Context Structure

```python
@dataclass
class Character:
    name: str
    age: str
    appearance: str  # hair, build, skin
    clothing: str    # detailed outfit description
    distinctive_features: str

@dataclass
class Shot:
    id: str           # "SHOT_01.1"
    scene_id: str
    shot_type: str   # EWS, MS, CU, OTS, etc.
    mode: str        # "t2v" or "i2v"
    simple_description: str
    expanded_prompt: str
    seed_image: str  # None for T2V, filename for I2V
    strength_s1: float  # stage 1 strength (for I2V)
    strength_s2: float  # stage 2 strength (for I2V)
    status: str      # pending, generating, done, failed
    output_file: str
    chain_seed: str  # previous segment's last frame (for chaining)
    attempts: int

@dataclass
class Scene:
    id: str
    description: str
    location_override: str
    shots: List[Shot]

@dataclass
class ProductionContext:
    title: str
    scenes: List[Scene]
    shared_context: dict  # location, time_of_day, color_palette, lighting, atmosphere
    characters: Dict[str, Character]
    timeline: List[dict]  # ordered shots with transitions
    chain_state: dict    # current chain seed for continuous segments
```

---

## Practical Execution

### Step-by-step production:

1. **Receive story description** from producer
2. **Break into scenes** (logical story units)
3. **Create shotlist** per scene (EWS first, then coverage)
4. **Build shared context** (characters, location, palette)
5. **For each shot:**
   - Determine mode: T2V (first shot of scene or new location) or I2V (continuity with previous)
   - If chained segment: extract last frame from previous output
   - Send simple description + shared context to *ltx-scenarist*
   - Send expanded prompt to *comfyui-spark-ltx* with correct mode/seed
   - Verify output quality
   - On failure: refine prompt, retry (max 3 attempts)
6. **Sequence clips** with transitions
7. **Deliver final production** with shotlist and timeline

### Managing Failed Shots

If a shot fails after 3 attempts:
1. Note the failure with the failed prompt
2. Skip to next shot
3. At end, report failed shots for manual review
4. Can try: different camera angle, simplified prompt, remove complex element

### Continuity Enforcement

Always pass consistent context to each scenarist call:
- Include current character appearance in every prompt
- Keep lighting/time consistent within a scene
- Reference previous shot's detail in new prompt for visual consistency

---

## Example: Full Production Run

**Input:** "A mysterious stranger walks into a rain-soaked tavern at night, sits at the bar, and orders a drink."

**Director -- Shotlist:**

```
CONTEXT:
  location: "The Rusty Anchor, a weathered tavern on a rain-soaked cobblestone street"
  time_of_day: "night"
  lighting: "warm amber candlelight and oil lamps, cold blue moonlight through wet windows"
  color_palette: "warm ambers contrasting with cold blues and grays"
  atmosphere: "rain hammering the wooden shutters, a fire crackles in the hearth"

SCENE 01: Arrival
  SHOT 01.1 (EWS, T2V): [TO SCENARIST] "A tavern exterior on a rain-soaked street at night"
  SHOT 01.2 (WS, I2V, chain): [TO SCENARIST] "A cloaked figure pushes open the tavern door"
  SHOT 01.3 (MS, I2V, chain): [TO SCENARIST] "The stranger walks to the bar, water dripping from cloak"
  SHOT 01.4 (CU, I2V, chain): [TO SCENARIST] "The stranger's gloved hand places a silver coin on the bar"

SCENE 02: The Order
  SHOT 02.1 (OTS, I2V, chain): [TO SCENARIST] "Over the shoulder, the keeper reaches for the coin"
  SHOT 02.2 (MS, I2V, chain): [TO SCENARIST] "The keeper pours amber liquid into a wooden tankard"
  SHOT 02.3 (CU, I2V, chain): [TO SCENARIST] "The stranger lifts the tankard, steam rising"
```

**Director -- Chaining Logic:**
```
Segment 01.1: T2V (no seed) --> tavern_exterior.mp4
  Extract last frame --> chain_01.1.png
Segment 01.2: I2V (chain_01.1.png, strength=1.0) --> door_opens.mp4
  Extract last frame --> chain_01.2.png
Segment 01.3: I2V (chain_01.2.png, strength=1.0) --> walks_to_bar.mp4
  ... and so on
```

**Director -- Final Timeline:**
```
01.1 EWS exterior -> fade_in
01.2 WS door opening -> cut
01.3 MS walking to bar -> cut
01.4 CU coin on bar -> cut
02.1 OTS keeper -> dissolve
02.2 MS pouring -> cut
02.3 CU drinking -> fade_out
```

---

## Important Limitations

1. **LTX duration**: Each clip is approximately 5-6 seconds. Plan accordingly.
2. **No multi-character complex interaction**: Stick to 1-3 characters per shot.
3. **Physics hallucinations**: Characters may pass through thin objects. Avoid thin barriers.
4. **Consistency across scenes**: LTX has no built-in memory. Each shot is independent.
   The director MUST maintain character descriptions and enforce them manually.
5. **Same-location trick**: Reference the previous shot's detail in the new prompt
   to encourage visual consistency.
6. **Chaining is essential for long videos**: Without it, each segment will have
   inconsistent lighting, camera angle, and character positioning.

---

## Tools Available to Director

When running as a subagent, the director can use:
- **delegate_task**: Call ltx-scenarist and comfyui-spark-ltx subagents in parallel
- **write_file**: Save shotlist, production context, timeline to files
- **read_file**: Load previous production state for resumption
- **send_message**: Report progress to producer
- **cronjob**: For long productions, schedule generation batches
- **execute_code**: Run Python for chaining logic and file operations
