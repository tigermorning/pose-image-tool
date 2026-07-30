# Prompts & Settings

Every generation in `pose_tool.ipynb` is recorded here so any result can be reproduced exactly.

## Shared settings

| Setting | Value |
|---|---|
| Base model | `stabilityai/stable-diffusion-xl-base-1.0` |
| ControlNet | `thibaud/controlnet-openpose-sdxl-1.0` |
| VAE | `madebyollin/sdxl-vae-fp16-fix` |
| Pose annotator | `controlnet_aux.OpenposeDetector` (`lllyasviel/Annotators`), body + hands, no face |
| Resolution | 832 x 1216 (hint and output identical) |
| Steps | 28 |
| Guidance scale | 6.0 |
| Conditioning scale | 0.8 |
| Precision | fp16 |

Negative prompt, used for every image:

```
lowres, blurry, deformed hands, extra limbs, extra fingers, watermark, text, jpeg artifacts
```

## Pose hints

Both references are seated figures, which is why the prompts below may say "seated" or "sitting":
a posture word that *agrees* with the skeleton is harmless. A posture word that *contradicts* it
is what produces doubled limbs - see the prompt-writing notes at the bottom.

| Hint | Source pose | Detection quality |
|---|---|---|
| `pose_01.png` | seated on a low block, one hand to the head, one leg extended | clean - all four limbs traced to the extremities |
| `pose_02.png` | crouching, arms wrapped around the shins | poor - heavy self-occlusion, limb assignment tangles |

`pose_02` was kept deliberately. It is the concrete evidence behind limitation 1 (pose detection
is the ceiling) rather than an assumed failure mode.

## Experiment A - same pose, different prompts

Pose hint, seed and all sampler settings fixed. Only the prompt changes.

| ID | Pose | Seed | Output file |
|---|---|---|---|
| A1 | `pose_01.png` | 1234 | `exp_a_1.png` (also copied to `output_01.png`) |
| A2 | `pose_01.png` | 1234 | `exp_a_2.png` |

A1:

```
A beautiful funky African woman wearing a flowing bohemian dress, seated in a chic pose, under the night sky, wet asphalt reflecting colourful neon lights, cinematic lighting, professional photography, ultra-realistic, highly detailed face
```

A2:

```
A Middle Eastern warrior clad in glowing neon armour, sitting in a futuristic laundromat, studio lighting, ultra-realistic, highly detailed
```

The pair differs in subject, wardrobe, setting **and** lighting (night neon on wet asphalt versus
flat interior studio light), while both stay photorealistic. That isolates subject-and-scene
variation from style variation.

## Experiment B - same prompt, different poses

Prompt, seed and all sampler settings fixed. Only the pose hint changes.

Prompt (same text as A2):

```
A Middle Eastern warrior clad in glowing neon armour, sitting in a futuristic laundromat, studio lighting, ultra-realistic, highly detailed
```

| ID | Pose | Seed | Output file |
|---|---|---|---|
| B1 | `pose_01.png` | 777 | `exp_b_01.png` |
| B2 | `pose_02.png` | 777 | `exp_b_02.png` (also copied to `output_02.png`) |

B1 uses the clean hint and B2 the tangled one, so this experiment measures two things at once:
what the pose hint controls, and what happens when the hint itself is unreliable.

## Prompt-writing notes

- **Do not describe a posture that fights the hint.** The skeleton already supplies the pose.
  "arms raised" over a seated skeleton produces doubled limbs. A posture word that matches the
  skeleton (here, "seated" / "sitting") is fine and can even help.
- **Do describe subject, wardrobe, setting, lighting and medium.** Those are exactly the
  dimensions the skeleton leaves free.
- **Non-photorealistic styles need a higher conditioning scale** (roughly +0.1-0.2) to hold the
  same pose. Both prompts here are photorealistic, so the 0.8 default is used throughout.
- Keep the negative prompt identical across a comparison, otherwise it stops being a
  single-variable experiment.
