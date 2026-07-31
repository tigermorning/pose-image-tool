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
| A1 | `pose_01.png` | 1234 | a different person in the reference's pose | `output_01.png` |
| A2 | `pose_01.png` | 1234 | a different subject **and** a different medium | `output_01_alt_prompt.png` |

A1 - the assignment's step 1:

```
A beautiful young African woman, short natural hair, small gold hoop earrings, wearing a flowing bohemian maxi dress in deep teal with gold embroidery, in a sunlit concrete gallery with tall arched windows and a polished terrazzo floor, warm late-afternoon light raking from the left, 85mm lens at f/2, editorial fashion photograph, fine skin texture
```

A2 - the harder variation:

```
An ink and watercolour drawing of a travelling puppeteer in a patched wool coat and worn leather boots, a canvas satchel at the hip, on cream paper with visible fibres, loose confident brush strokes, muted earth pigments of ochre and burnt umber, ink bleeding at the outlines, wide untouched margins
```

The two prompts probe different axes. A1 stays photographic and changes who the person is. A2 leaves
photography altogether, and moving away from photorealism is where pose adherence weakens first at a
given conditioning scale - so A2 is the informative case for "which change does the tool follow less
well".

### Why no prompt describes the pose

Neither prompt contains a posture word, and that is a requirement rather than a style choice.

1. **It is what makes the demonstration valid.** If A1 said "seated on a stool with one leg extended",
   a reader could not tell whether the pose in `output_01.png` came from ControlNet or from the text.
   Leaving posture out of the prompt is precisely what proves the skeleton did the work.
2. **A contradicting posture word doubles limbs.** An early draft of A1 said "standing" over a seated
   hint and the output grew extra limbs. A word that merely *agrees* with the hint turned out to be
   harmless, but relying on that is fragile - the hint can change later, as it does in step 2b - so
   neither prompt here contains one at all.
3. **A1 is reused verbatim over a standing hint in step 2b.** Any posture in it would fight `pose_02`.

Posture is supplied by the skeleton, and measured: at conditioning scale 1.0 the generated pose sits
within 0.031 mean joint error of the hint. The prompt's job is subject, wardrobe, setting and light.

Both prompts spend their tokens on things the model can draw. `85mm lens at f/2`,
`warm late-afternoon light raking from the left` and `polished terrazzo floor` are concrete;
`professional photography` and `ultra-realistic` are not, and were dropped. This follows the one
thing measured on the earlier prompt pair: `wet asphalt reflecting colourful neon lights` (6 tokens,
a material plus an optical effect) was rendered faithfully, while `futuristic laundromat` (3 tokens,
an abstract noun) was ignored almost entirely. Concreteness decided it, not length.

### The 77-token ceiling is real

Each SDXL text encoder truncates at 77 tokens and `diffusers` does it silently. A first draft of A1
ran to 84 tokens, which would have dropped `85mm lens at f/2`, `editorial fashion photograph` and
`fine skin texture` - the last clauses, and three of the most useful. It was trimmed by removing `in her late twenties`
(redundant next to "young") and a second garment layer; A1 now sits at 75.

Current counts: **A1 75/77, A2 65/77, negative prompt 35/77.** Check any edit against the tokenizer
before running it; a prompt that silently loses its tail is indistinguishable from a prompt that was
simply ignored.


### Verified, not assumed

Scanned straight out of `pose_tool.ipynb` rather than from memory:

| Prompt | Tokens | Posture words | Scope |
|---|---|---|---|
| A1 (step 1, and step 2b) | 75/77 | none | person, wardrobe, setting, light, lens |
| A2 (step 2a) | 65/77 | none | subject, wardrobe, medium, pigments, paper |
| negative | 35/77 | none | defects to suppress |

`PROMPT_B` is byte-identical to A1, so step 2b really does hold the prompt constant. Re-run the scan
after editing any prompt: a posture word that slips in is invisible until the limbs double.

## Experiment B - same prompt, different poses

Prompt, seed and all sampler settings fixed. Only the pose hint changes. The prompt is A1 verbatim,
so this is the assignment's step 2: the same words, a different pose photo.

```
A beautiful young African woman, short natural hair, small gold hoop earrings, wearing a flowing bohemian maxi dress in deep teal with gold embroidery, in a sunlit concrete gallery with tall arched windows and a polished terrazzo floor, warm late-afternoon light raking from the left, 85mm lens at f/2, editorial fashion photograph, fine skin texture
```

| ID | Pose | Seed | Output file |
|---|---|---|---|
| B1 | `pose_01.png` - seated on a stool | 1234 | `output_01.png` - identical to A1, not regenerated |
| B2 | `pose_02.png` - standing, fists raised | 1234 | `output_02.png` |

Both experiments share one seed, 1234. That makes B1 byte-identical to A1, so it is not generated
twice - `output_01.png` serves as both step 1's result and the `pose_01` arm of step 2b. The whole
run is therefore three generated images, and `output_01.png` against `output_02.png` is a genuine
single-variable comparison: same prompt, same seed, same sampler settings, and the only difference
is which photograph the skeleton came from.

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
