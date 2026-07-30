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
| Conditioning scale | 0.8 (except experiment C) |
| Precision | fp16 |

Negative prompt, used for every image:

```
lowres, blurry, deformed hands, extra limbs, extra fingers, watermark, text, jpeg artifacts
```

## Experiment A - same pose, different prompts

Pose hint, seed and all sampler settings fixed. Only the prompt changes.

| ID | Pose | Seed | Prompt | Output file |
|---|---|---|---|---|
| A1 | `pose_01.png` | 1234 | `a professional astronaut in a white spacesuit standing on red martian soil, cinematic lighting, photorealistic` | `exp_a_1.png` (also copied to `output_01.png`) |
| A2 | `pose_01.png` | 1234 | `a medieval knight in polished steel armour in a stone castle courtyard, overcast light, photorealistic` | `exp_a_2.png` |
| A3 | `pose_01.png` | 1234 | `a watercolour illustration of a ballet dancer, soft pastel palette, visible paper texture` | `exp_a_3.png` |

Prompt selection is deliberate: A1 and A2 change the **subject** while keeping photorealism, A3 changes the **medium**. That separates "does the pose survive a new subject" from "does the pose survive a new style".

## Experiment B - same prompt, different poses

Prompt, seed and all sampler settings fixed. Only the pose hint changes.

Prompt:

```
a professional dancer in a flowing red dress in an empty concrete studio, dramatic side light, photorealistic
```

| ID | Pose | Seed | Output file |
|---|---|---|---|
| B1 | `pose_01.png` | 777 | `exp_b_01.png` |
| B2 | `pose_02.png` | 777 | `exp_b_02.png` (also copied to `output_02.png`) |
| B3 | `pose_03.png` (if a third reference was uploaded) | 777 | `exp_b_03.png` |

The prompt is intentionally pose-neutral - it names no posture - so that any posture in the output comes from the hint alone.

## Experiment C - conditioning scale sweep

Pose hint, prompt and seed fixed. Only `controlnet_conditioning_scale` changes.

Prompt:

```
a rock climber in technical outdoor gear on a granite wall, midday sun, photorealistic
```

| ID | Pose | Seed | Conditioning scale | Output file |
|---|---|---|---|---|
| C1 | `pose_01.png` | 42 | 0.4 | `exp_c_scale_0.4.png` |
| C2 | `pose_01.png` | 42 | 0.7 | `exp_c_scale_0.7.png` |
| C3 | `pose_01.png` | 42 | 1.0 | `exp_c_scale_1.0.png` |

## Smoke test

Run once before the experiments to verify the pipeline end to end. Not part of the results.

```
a hiker in a red windbreaker on a mountain ridge, golden hour, photorealistic
```
Pose `pose_01.png`, seed 1234, defaults everywhere else.

## Prompt-writing notes

- **Do not describe the pose in the prompt.** The skeleton already supplies it. Words like "arms raised" fight the hint and produce doubled limbs.
- **Do describe subject, wardrobe, setting, lighting and medium.** Those are exactly the dimensions the skeleton leaves free.
- **Non-photorealistic styles need a higher conditioning scale** (roughly +0.1-0.2) to hold the same pose.
- Keep the negative prompt identical across a comparison, otherwise it stops being a single-variable experiment.
