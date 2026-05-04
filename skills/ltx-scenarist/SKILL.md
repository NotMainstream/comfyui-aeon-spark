---
name: ltx-scenarist
description: Transform simple scene concepts into detailed, production-ready LTX video generation prompts with cinematic language
version: 1.1.0
category: creative
tags: [ltx, prompting, video-generation, screenplay, movie-studio]
trigger: "When the user wants to create a video scene -- takes a simple concept and expands it into a full cinematic LTX prompt"
---

# ltx-scenarist

Transform a simple scene concept into a detailed, production-ready LTX video generation prompt.
Used in a **movie production studio** workflow where a director/scene-writer describes a shot,
and this skill expands it into the full cinematic language LTX understands.

**Role in the pipeline:**

```
Producer (human) --> ltx-director (orchestrates)
                         |
                         v
                   ltx-scenarist <-- YOU ARE HERE
                         |
                         v
                   ltx-cameraman
```

---

## The 6 Elements of an LTX Prompt

Every LTX prompt must include these 6 elements:

### 1. Establish the Shot
- **Shot scale**: wide establishing (EWS), wide shot (WS), medium shot (MS), close-up (CU), extreme close-up (ECU)
- **Shot type**: tracking shot, static frame, handheld, crane shot, aerial
- **Category**: cinematic, animation, stylized, documentary, film noir, fantasy, thriller
- **Perspective**: over-the-shoulder, POV, low angle, high angle, bird's eye, Dutch angle

### 2. Set the Scene
- **Location**: specific and grounded ("rain-soaked Tokyo street" not just "street")
- **Time of day**: golden hour, blue hour, noon, night, dawn, dusk
- **Lighting**: see Lighting Reference below
- **Color palette**: warm tones, cool tones, desaturated, high contrast, vibrant
- **Textures and surfaces**: worn brick, polished marble, rough stone, glossy metal
- **Atmosphere**: fog, rain, dust motes, smoke, particles, haze

### 3. Describe the Action
- **What happens**: specific physical action in present tense
- **Sequence**: flows naturally from beginning to end of the shot
- **Pacing**: slow and deliberate, rapid and energetic, methodical
- **Interaction**: how characters interact with objects and environment
- **Physical causality**: A leads to B leads to C (be aware LTX struggles with complex physics)

### 4. Define the Character(s)
- **Appearance**: age, hair color/style, clothing, build, distinguishing features
- **Physical actions**: HOW emotion is expressed (NOT "she is sad" but "her shoulders slump, eyes cast down")
- **Movement quality**: graceful, abrupt, hesitant, fluid, clumsy
- **Physical cues for emotion**: see Emotion Cues Reference below

### 5. Identify Camera Movement(s)
- **Movement type**: tracking, dolly, pan, tilt, crane up/down, handheld, steadicam
- **Relative to subject**: "camera follows her as she...", "dolly in toward his face"
- **Speed and feel**: slow and smooth, jarring and handheld, gliding
- **Describe subjects AFTER movement**: LTX renders the result of movement more accurately

### 6. Describe the Audio
- **Ambient sound**: rain, wind, crowd murmur, traffic, birds, machinery
- **Music**: jazz quartet, orchestral swell, electronic ambient, silence
- **Dialogue**: in quotation marks with acting directions between segments
- **Voice qualities**: warm and resonant, gravelly, childlike, robotic monotone
- **Volume**: whisper, quiet murmur, shout, screaming

---

## LTX Prompt Formula

[SHOOT SCALE] -- [SHOT TYPE] -- [CATEGORY/STYLE]

[CHARACTER NAME if applicable], [AGE], [PHYSICAL DESCRIPTION INCLUDING CLOTHING], [DISTINGUISHING FEATURES]. [PHYSICAL EMOTION CUE], [CURRENT ACTION SEQUENCE IN PRESENT TENSE]. [HOW CHARACTER INTERACTS WITH SCENE OBJECTS].

The camera [MOVEMENT TYPE] [SUBJECT] [SPECIFIC SUBJECT DESCRIPTION AFTER MOVEMENT]. [ADDITIONAL CAMERA NOTES].

[ENVIRONMENT DETAIL], [LIGHTING CONDITIONS], [COLOR PALETTE], [TEXTURES/ATMOSPHERE]. [TIME OF DAY], [WEATHER/ATMOSPHERIC CONDITIONS].

The audio: [AMBIENT SOUND], [MUSIC DESCRIPTION], [DIALOGUE WITH ACTING DIRECTIONS].

---

## LTX-Specific Rules

- **Be detailed**: The more specific you are about subject, action, lighting, camera movement, and audio, the closer the output matches your vision.
- **Match prompt length to video length**: Short prompts for long videos leave the model without enough direction to fill the duration.
- **Break dialogue into segments**: Use short phrases with acting directions between each line; use physical cues rather than emotional labels.
- **Write as single flowing paragraph**: Present tense verbs throughout.
- **Describe camera relative to subject**: "camera follows her as she walks" not "camera pans 30 degrees".
- **Describe the result of movement**: LTX renders the result of movement more accurately than the movement itself.

---

## Lighting Reference

| Type | Terms |
|------|-------|
| Natural | golden hour, blue hour, harsh noon sun, overcast diffused, soft window light, candlelight |
| Artificial | neon glow, fluorescent flicker, tungsten warm, LED bounce, streetlamp, fireplace |
| Cinematic | volumetric light beams, god rays, lens flares, anamorphic bokeh, light leaks |
| Shadow | high contrast, deep shadows, chiaroscuro, rim light, backlit silhouette, underlighting |
| Atmosphere | volumetric haze, fog, smoke, dust motes, mist, rain streaks, heat shimmer |

---

## Camera Language Reference

### Shot Scales
- EWS / Establishing Wide Shot: Orient viewer in space/location
- WS / Wide Shot: Full figure, environment visible
- MS / Medium Shot: Waist up (dialogue), knee up (action)
- CU / Close-Up: Face fills frame, emotion readable
- ECU / Extreme Close-Up: Detail (eyes, hands, object)

### Movements
- Tracking shot: camera moves alongside subject (on rails, dolly, steadicam)
- Pan: camera rotates horizontally on fixed position
- Tilt: camera rotates vertically on fixed position
- Dolly in/out: camera moves toward/away from subject
- Crane up/down: camera rises/falls on arm
- Handheld: organic shake, used for urgency, intimacy, documentary feel
- Steadicam: smooth tracking without mechanical rig feel
- POV: from character's perspective
- Over-the-shoulder (OTS): behind one character looking at another
- Dutch angle / canted frame: tilted horizon for unease/disorientation

### Temporal
- Slow motion: dramatic emphasis
- Time-lapse: compressed time
- Rapid cuts: high energy, editing-driven rhythm
- Linger: held shot, tension building
- Continuous take: single uncut shot

---

## Emotion Physical Cues Reference

Always use physical cues, never abstract emotion labels.

| Emotion | Physical Cue |
|---------|-------------|
| Happy | eyes crinkle at corners, broad smile, shoulders lift, spring in step, laugh bubbles up |
| Sad | eyes cast down, shoulders slump, moves slowly, pauses mid-step, breath catches |
| Angry | jaw tightens, fists clench, voice drops low, stride lengthens, leans in |
| Fearful | eyes widen, body freezes mid-step, breath shallow, steps backward |
| Surprised | eyes widen, head pulls back, hand to mouth, breath inhales sharply |
| Curious | head tilts to one side, leans in, eyes narrow, steps closer |
| Nervous | fidgets with hands, shifts weight side to side, voice wavers, swallows |
| Confident | stands tall, strides purposefully, eye contact steady, chin lifts |
| Tired | eyelids heavy, moves in slow motion, yawns, sits heavily |
| Confused | frowns, head scratches, looks around, mutters to self, freezes |

---

## Audio Description Guide

### Ambient Sound Format
Describe what you hear as background texture:
- "The low hum of an air conditioner, distant traffic noise, occasional footsteps on pavement"
- "Rain pattering on windowpanes, a dog barking in the distance, the rustle of newspaper pages"

### Music Format
Describe the mood and instrumentation:
- "Soft ambient jazz piano, barely audible, building toward emotional swell"
- "Tense orchestral strings, low drone, sudden sharp brass hit on cut"

### Dialogue Format (CRITICAL)
Use quotation marks for spoken words, with physical acting directions between segments:

"[SPEAKER NAME], [AGE], with [VOICE QUALITY], speaks in a [PACE] voice: '[FIRST LINE]'" [PHYSICAL ACTION/PAUSE]. "[CONTINUATION]" [PHYSICAL BEAT]. "[FINAL LINE]". [FINAL CAMERA MOVE].

Example:
A middle-aged man, voice gravelly and tired: "I told you not to go there." He pauses, eyes cast down. "But you never listen, do you?" His hand reaches for the glass. The camera slowly pushes in toward his face.

---

## What to Avoid in LTX Prompts

| Avoid | Use Instead |
|-------|------------|
| Abstract emotions ("she is sad") | Physical cues ("her shoulders slump, eyes cast down") |
| Readable text or logos | Describe the surface without text |
| Complex multi-object physics | Simple single-threaded action |
| Conflicting lighting directions | Single consistent light source |
| Exact numerical camera specs | Natural language ("slowly drifts right") |
| Overloaded scenes (>3 characters) | Focus on primary subject |
| "The camera pans at 2 deg/sec" | "Camera slowly tracks right following her" |
| Short prompts for long videos | Detailed prompts that fill the duration |

---

## Example Transformations

### Example 1: Simple to Cinematic

Simple concept: "A woman walks down a street."

Expanded LTX prompt:
Wide tracking shot -- cinematic, overcast afternoon. A young woman in her late twenties, dark hair pulled back in a practical ponytail, wearing a worn leather jacket and dark jeans, walks briskly down a narrow European cobblestone street. Her breath makes small clouds in the cold air. She adjusts the strap of her bag on her shoulder and glances over her shoulder with eyes that narrow with caution. The camera tracks alongside her at medium distance, slight handheld sway, as storefront windows cast pale light across her path. Late afternoon, cool blue-gray atmosphere, the street lined with shuttered market stalls and a single cafe with warm yellow light spilling from its windows. The audio: distant chatter from the cafe, the sound of her boots on cobblestones, a bicycle bell somewhere behind.

### Example 2: Dialogue Scene

Simple concept: "A man tells someone bad news."

Expanded LTX prompt:
Medium close-up, static frame -- dramatic cinematic. An older man, late sixties, deep lines around tired eyes, wearing a rumpled wool sweater, sits at a worn wooden table, hands clasped in front of him. His voice, low and measured with the weight of exhaustion: "There's things I should've told you before." He pauses, jaw tightening. "Before you went there." The camera holds perfectly still on his face as the silence stretches. Warm amber lamplight cuts across his features in sharp relief against the dark background, deep shadows pooling in the hollows of his eyes. The audio: the creak of the old chair, the man's measured breathing, the muffled sound of rain on glass somewhere off-screen.

### Example 3: Action Sequence

Simple concept: "Someone gets chased."

Expanded LTX prompt:
Rapid tracking shots alternating with wide shots -- thriller, handheld. A young man in a torn hoodie sprints down a narrow alley, his sneakers slapping wet concrete. He vaults a discarded crate and spins past a stack of crates, never breaking stride. Behind him, two figures in dark jackets maintain the chase, their footfalls echoing off brick walls. The camera cuts between: a wide shot showing all three figures as they round a corner, then a shaky handheld close-up of the young man's face, sweat and rain streaming down, teeth gritted. A low angle shot picks up his feet as he leaps over a chain-link fence. Golden-red neon from a convenience store reflects off the wet ground. The audio: heavy breathing and footsteps, a distant car horn, the young man's ragged gasps, the echo of pursuit behind him.

### Example 4: Atmospheric Scene

Simple concept: "A library at night."

Expanded LTX prompt:
Extreme wide establishing shot, static -- atmospheric, mysterious. A vast circular library stretches upward into darkness, its spiral staircase disappearing into shadow. Towering mahogany shelves hold leather-bound books that catch the warm glow of scattered desk lamps. Dust motes drift lazily through the light beams. The silence is so complete it feels physical. A single figure stands at a mahogany reading desk, the warm cognac glow of an antique banker's lamp illuminating an open book before her. A brass mechanical owl sits beside the lamp, its eyes glinting in the light. The camera holds on this tableau, refusing to move, as the massive scale of the space slowly sinks in. The audio: the soft creak of old wood, the faint tick of a brass clock, the whisper of pages turning.

### Example 5: Beach Scene (tested and verified working)

Simple concept: "A sunny day at the beach with beautiful people."

Expanded LTX prompt:
Wide shot, cinematic -- tropical beach at golden hour. Crystal clear turquoise water gently laps against pristine white sand. Palm trees sway in a soft breeze, their long shadows stretching across the beach. In the distance, dramatic green volcanic rock formations emerge from the emerald water. A few happy people are visible walking along the shoreline, their laughter carrying on the wind. Warm golden sunlight bathes everything in a magical glow. Vibrant saturated colors, cinematic wide-angle composition, professional color grading, shallow depth of field with foreground in soft focus. The audio: the rhythmic sound of waves, distant laughter, seagulls calling, a gentle breeze rustling the palm fronds.

---

## Quick Reference Card

**Shot Scale to Detail Level:**
- EWS/WS: Less character detail, more environment
- MS/CU: More character detail, physical cues for emotion
- ECU: Single detail (eyes, hands), extreme focus

**6-Element Checklist:**
- [ ] Shot scale and type established
- [ ] Scene/location set with atmosphere
- [ ] Action sequence in present tense
- [ ] Character(s) defined with physical appearance and emotion cues
- [ ] Camera movement described relative to subject
- [ ] Audio (ambient/music/dialogue) specified

**LTX Golden Rules:**
1. Physical cues over emotion labels
2. Natural language over numerical specs
3. Single light source per scene
4. 1-3 characters per shot
5. Simple physics (no complex multi-object collision)
6. Single flowing paragraph, present tense
7. Be detailed -- more description = better output
8. Match prompt length to video duration
