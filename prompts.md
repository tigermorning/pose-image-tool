# Prompts & Settings

Every generation in `pose_tool.ipynb` is recorded here so any result can be reproduced exactly.

## Shared settings

| Setting | Value |
|---|---|
| Base model | `stabilityai/stable-diffusion-xl-base-1.0` |
| ControlNet | `thibaud/controlnet-openpose-sdxl-1.0` |
| VAE | `madebyollin/sdxl-vae-fp16-fix` |
| Pose annotator | `controlnet_aux.OpenposeDetector` (`lllyasviel/Annotators`), body + hands, no face |
| Resolution | 832 x 1216, references letterboxed (not cropped) to that ratio |
| Steps | 28 |
| Guidance scale | 5.0 |
| Conditioning scale | 1.0 (see the note below) |
| Precision | fp16 |

Negative prompt, used for every image:

```
lowres, blurry, deformed hands, extra limbs, extra fingers, watermark, text, jpeg artifacts
```

## Pose hints

Both references are single-subject full-body photographs, letterboxed to 832x1216. Each was checked
numerically before use (section 3a of the notebook) rather than judged by eye.

| Hint | Source pose | Validation |
|---|---|---|
| `pose_01.png` | seated on a stool, front on, one leg extended to the floor, the other knee raised, both arms hanging clear of the torso | 18/18 keypoints, all four limb chains closed, no limb crossings |
| `pose_02.png` | standing, arms folded across the chest, feet wide apart | 18/18 keypoints, all chains closed; the two forearms cross in 2D |

The pair was chosen deliberately. `pose_01` is the clean case: nothing overlaps, so the hint is
unambiguous. `pose_02` keeps one known difficulty - crossed forearms - because a 2D skeleton carries
no depth and cannot say which forearm is in front. The legs, however, are wide apart and completely
unambiguous, so the difficulty is confined to the arms.

Three candidate references were rejected by the same numeric check and are kept out of the run:

| Rejected | Why |
|---|---|
| profile shot, seated on a block | 15/18 - the left arm was hidden inside a coat, so its chain never closed |
| upper-body crop | 12/18 - no leg keypoints at all |
| crouch, arms around the shins | left ankle lost, and both legs crossed in 2D |

Rejecting references on a keypoint count rather than on appearance is the point of section 3a: an
unclosed limb chain silently disables pose control for that limb, and the generated image gives no
warning that it happened.

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
- **Conditioning scale 1.0, not 0.8.** Measured by re-detecting the pose in the generated image
  and comparing it joint by joint with the hint: at 0.8 the folded knee landed 0.45 of the frame
  away from the hint (mean error 0.099), at 1.0 every joint is within 0.07 (mean 0.031). A folded
  or foreshortened limb is where a low scale fails first, and it fails without looking obviously
  wrong. The cost of 1.0 is slightly stiffer material rendering.
- Keep the negative prompt identical across a comparison, otherwise it stops being a
  single-variable experiment.
