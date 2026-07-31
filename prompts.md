# Prompts & Settings

Every generation in `pose_tool.ipynb` is recorded here so any result can be reproduced exactly.
Prompt bodies below are copied out of the notebook, not from memory.

## Shared settings

| Setting | Value |
|---|---|
| Base model | `SG161222/RealVisXL_V5.0` (photoreal SDXL fine-tune) |
| ControlNet 1 - pose | `thibaud/controlnet-openpose-sdxl-1.0` |
| ControlNet 2 - depth | `diffusers/controlnet-depth-sdxl-1.0` |
| VAE | `madebyollin/sdxl-vae-fp16-fix` |
| Pose annotator | `controlnet_aux.OpenposeDetector` (`lllyasviel/Annotators`), body + hands, no face |
| Depth annotator | `controlnet_aux.MidasDetector` (`lllyasviel/Annotators`) |
| Resolution | 832 x 1216, references letterboxed (not cropped) to that ratio |
| Steps | 28 |
| Guidance scale | 5.0 |
| Conditioning scale | **pose 0.8, depth 0.6** (see the note below) |
| Seed | 1234 for every image |
| Precision | fp16 |

Negative prompt, used for every image:

```
doll, mannequin, plastic skin, waxy, airbrushed, lowres, blurry, deformed hands, extra fingers, extra limbs, three legs, duplicated limbs, fused limbs, watermark, text
```

The base is a photoreal fine-tune rather than SDXL base 1.0. On this reference, base 1.0 rendered
faces and skin that read as a mannequin, and no prompt or sampler setting fixed it - it is the
checkpoint's own character. Swapping the checkpoint did fix it. The first five negative terms
(`doll, mannequin, plastic skin, waxy, airbrushed`) are what remains of that fight.

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
A beautiful young African woman perched on the edge of a metal stool, leaning forward with her weight on her arm, one hand gripping the seat between her knees, wide dark trousers and a cropped jacket, sunlit concrete gallery, 85mm lens at f/2, candid editorial photograph, natural skin with visible pores
```

A2 - the harder variation:

```
An ink and watercolour drawing of a travelling puppeteer perched on the edge of a wooden stool, leaning forward with his weight on his arm, one hand gripping the seat between his knees, patched wool coat and worn boots, cream paper with visible fibres, loose brush strokes, ochre and burnt umber
```

The two prompts probe different axes. A1 stays photographic and changes who the person is. A2 leaves
photography altogether, and moving away from photorealism is where pose adherence weakens first at a
given conditioning scale - so A2 is the informative case for "which change does the tool follow less
well".

## Experiment B - same prompt, different poses

Prompt, seed and all sampler settings fixed. Only the pose hint changes.

```
A beautiful young African woman, wide dark trousers and a cropped jacket, sunlit concrete gallery, 85mm lens at f/2, candid editorial photograph, natural skin with visible pores
```

| ID | Pose | Seed | Output file |
|---|---|---|---|
| B1 | `pose_01.png` - seated on a stool | 1234 | `output_02_pose01.png` |
| B2 | `pose_02.png` - standing, fists raised | 1234 | `output_02.png` |

`PROMPT_B` is **A1 with its three posture clauses deleted** and nothing else changed - same subject,
wardrobe, setting, lens and photographic register, byte-identical from `wide dark trousers` onward.
It is not A1 verbatim, and the reason is that A1 describes `pose_01` specifically: `perched on the
edge of a metal stool`, `leaning forward with her weight on her arm`, `one hand gripping the seat
between her knees`. Those clauses agree with the seated hint and contradict the standing one, so
sending A1 unchanged to both references would have varied two things at once.

Both B images share `PROMPT_B`, so B1 against B2 is a genuine single-variable comparison: the only
difference between the two files is which photograph the skeleton came from.

## Posture words in a prompt

A1 contains posture clauses; `PROMPT_B` does not. That difference is deliberate, and the rule behind
it is narrower than "never describe the pose":

- **A posture clause that agrees with the hint is useful.** A skeleton has no depth and no contact
  information. It cannot say which way the torso leans, what the hands are touching, or where the
  body's weight sits. A1's clauses supply exactly that for `pose_01`, and the depth ControlNet
  reinforces it.
- **A posture clause that contradicts the hint is dropped, not blended.** Measured: A1 sent
  unchanged to the standing `pose_02` produced a standing figure with no stool at all, at 0.0098
  mean joint error - no worse than `PROMPT_B` on the same hint (0.0113). The skeleton wins the
  contradiction outright. An earlier draft that said "standing" over a seated hint did grow extra
  limbs, so this is not a guarantee at every scale, but at the settings above the failure mode is
  omission rather than duplication.
- **The demonstration still needs a pose-free prompt somewhere.** If every prompt described the
  posture, a reader could not tell whether the pose came from ControlNet or from the text.
  `PROMPT_B` is that control: it names no posture and still reproduces both references.

Posture is supplied by the skeleton and measured, not asserted. The prompt's remaining job is
subject, wardrobe, setting, light and medium.

## Prompt-writing notes

- **Concrete beats abstract.** `85mm lens at f/2` and `natural skin with visible pores` are rendered;
  bare abstract nouns tend to be ignored. Measured on an earlier prompt pair: `wet asphalt reflecting
  colourful neon lights` (6 tokens, a material plus an optical effect) was rendered faithfully, while
  `futuristic laundromat` (3 tokens, an abstract noun) was ignored almost entirely. Concreteness
  decided it, not length.
- **Not every concrete clause survives either.** `cropped jacket` appears in both A1 and `PROMPT_B`
  and was not honoured in any generated image - the jacket comes out hip length every time, across
  different hints and different prompts. Garment cut is not reliably controllable here.
- **The negative prompt suppresses, it does not guarantee.** `text` and `watermark` are both in it,
  and one background still came back carrying garbled lettering on a wall.
- **Two conditioning scales, not one.** Pose is held at 0.8 rather than 1.0 because the depth map
  already pins the limbs; giving pose full weight on top of depth stiffens the body. Measured on
  `pose_01`: pose 0.8 with depth 0.6 gave 0.013 mean joint error, against 0.030 for pose alone at
  1.0. On a single-ControlNet pipeline the earlier finding still holds - there, 0.8 let a folded knee
  land 0.45 of the frame from the hint and 1.0 was required.
- Keep the negative prompt identical across a comparison, otherwise it stops being a
  single-variable experiment.

## The 77-token ceiling is real

Each SDXL text encoder truncates at 77 tokens and `diffusers` does it silently, from the tail. A
prompt that quietly loses its last clauses is indistinguishable from a prompt that was ignored.

Counts scanned with `CLIPTokenizer` against the text currently in the notebook:

| Prompt | Tokens | Posture clauses | Scope |
|---|---|---|---|
| A1 (step 1) | 66/77 | 3, all agreeing with `pose_01` | person, posture, wardrobe, setting, lens |
| A2 (step 2a) | 65/77 | 3, same clauses in the drawing's register | subject, posture, wardrobe, medium, paper |
| `PROMPT_B` (step 2b) | 40/77 | none | person, wardrobe, setting, lens |
| negative | 44/77 | none | defects to suppress |

Re-run the scan after editing any prompt.
