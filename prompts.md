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
| `pose_02.png` | standing, both forearms raised with closed fists at chest height, feet wide apart | 18/18 keypoints, all four limb chains closed, no limb crossings |

The pair maximises pose contrast while keeping both hints unambiguous. They differ in the upper body
(arms hanging versus both forearms raised) *and* the lower body (seated with asymmetric legs versus
standing with the feet apart), so experiment B varies posture as completely as two references can
while every limb chain stays closed and no limb overlaps another in 2D.

`pose_02`'s closed fists are a deliberate choice. Fingers were the one defect no setting in this
notebook fixed, and a closed fist is a shape SDXL renders far more reliably than a spread hand, so
this reference sidesteps the known failure rather than walking into it.

Six candidates were checked; four were rejected by the same numeric test and kept out of the run:

| Rejected | Why |
|---|---|
| seated profile in a coat | 15/18 - the left arm was hidden behind the torso, so its chain never closed |
| upper-body crop | 12/18 - no leg keypoints at all |
| crouch, arms around the shins | left ankle lost, both legs crossed in 2D |
| three standing shots with folded arms | complete chains, but the forearms crossed in 2D - usable, simply worse than the pair above |

Rejecting a reference on a keypoint count rather than on appearance is the point of section 3a. An
unclosed limb chain silently disables pose control for that limb, and the generated image gives no
warning that it happened.

## Experiment A - same pose, different prompts

Pose hint, seed and all sampler settings fixed. Only the prompt changes.

| ID | Pose | Seed | Tests | Output file |
|---|---|---|---|---|
| A1 | `pose_01.png` | 1234 | a different person in the reference's pose | `exp_a_1.png` (also copied to `output_01.png`) |
| A2 | `pose_01.png` | 1234 | a different subject **and** a different medium | `exp_a_2.png` |

A1 - the assignment's step 1:

```
A young African woman, short natural hair, small gold hoop earrings, wearing a tailored ivory linen suit with rolled sleeves, in a sunlit concrete gallery with tall arched windows and a polished terrazzo floor, warm late-afternoon light raking from the left, 85mm lens at f/2, editorial fashion photograph, fine skin texture
```

A2 - the harder variation:

```
An ink and watercolour drawing of a travelling puppeteer in a patched wool coat and worn leather boots, a canvas satchel at the hip, on cream paper with visible fibres, loose confident brush strokes, muted earth pigments of ochre and burnt umber, ink bleeding at the outlines, wide untouched margins
```

The two prompts probe different axes. A1 stays photographic and changes who the person is. A2 leaves
photography altogether, and moving away from photorealism is where pose adherence weakens first at a
given conditioning scale - so A2 is the informative case for "which change does the tool follow less
well".

Neither prompt contains a posture word. The skeleton supplies posture; a prompt that contradicts it
produces doubled limbs. A1 in particular is reused verbatim over a standing hint in experiment B, so
it has to stay posture-neutral.

Both prompts spend their tokens on things the model can draw. `85mm lens at f/2`,
`warm late-afternoon light raking from the left` and `polished terrazzo floor` are concrete;
`professional photography` and `ultra-realistic` are not, and were dropped. This follows the one
thing measured on the earlier prompt pair: `wet asphalt reflecting colourful neon lights` (6 tokens,
a material plus an optical effect) was rendered faithfully, while `futuristic laundromat` (3 tokens,
an abstract noun) was ignored almost entirely. Concreteness decided it, not length.

### The 77-token ceiling is real

Each SDXL text encoder truncates at 77 tokens and `diffusers` does it silently. A first draft of A1
ran to 84 tokens, which would have dropped `85mm lens at f/2`, `editorial fashion photograph` and
`fine skin texture` - the last clauses, and three of the most useful. It was trimmed to 71 by
removing `in her late twenties` (redundant next to "young") and a second garment layer.

Current counts: **A1 71/77, A2 65/77, negative prompt 35/77.** Check any edit against the tokenizer
before running it; a prompt that silently loses its tail is indistinguishable from a prompt that was
simply ignored.

## Experiment B - same prompt, different poses

Prompt, seed and all sampler settings fixed. Only the pose hint changes. The prompt is A1 verbatim,
so this is the assignment's step 2: the same words, a different pose photo.

```
A young African woman, short natural hair, small gold hoop earrings, wearing a tailored ivory linen suit with rolled sleeves, in a sunlit concrete gallery with tall arched windows and a polished terrazzo floor, warm late-afternoon light raking from the left, 85mm lens at f/2, editorial fashion photograph, fine skin texture
```

| ID | Pose | Seed | Output file |
|---|---|---|---|
| B1 | `pose_01.png` - seated on a stool | 777 | `exp_b_01.png` |
| B2 | `pose_02.png` - standing, fists raised | 777 | `exp_b_02.png` (also copied to `output_02.png`) |

`output_01.png` and `output_02.png` are the two halves of the assignment. Same prompt, same seed, one
seated woman and one standing woman - and the only thing that differed was which photograph the
skeleton came from.

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
