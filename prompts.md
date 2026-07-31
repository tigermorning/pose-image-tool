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

`pose_02`'s closed fists were chosen to sidestep the finger problem - a closed fist is a shape SDXL
renders more reliably than a spread hand. **That did not work.** The hint's fists came back as open
hands with malformed fingers, so the mitigation failed and the fingers are wrong on both references.
The cause is size rather than shape: a hand spans roughly 54 px in this framing.

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
| A1 | `pose_01.png` | 1234 | a different person in the reference's pose - **step 1** | `output_01.png` |
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

Prompt, seed and all sampler settings fixed. Only the pose hint changes. The prompt is **A1, byte
for byte** - the assignment's step 2 asks for the same prompt over a different pose photo, and that
is what this sends.

| ID | Pose | Seed | Output file |
|---|---|---|---|
| B1 | `pose_01.png` - seated on a stool | 1234 | `output_01.png` - this *is* step 1's result, not regenerated |
| B2 | `pose_02.png` - standing, fists raised | 1234 | `output_02.png` |

A1 describes `pose_01` specifically (`perched on the edge of a metal stool`, and so on), so sending
it unchanged to a standing reference contradicts the hint. That was measured rather than assumed:
A1 unchanged over `pose_02` scored 0.008 against 0.010 for the same prompt with its posture clauses
stripped out, and no stool appeared in either. **A posture clause that fights the hint is discarded,
not blended**, so keeping it costs nothing and keeps the prompt byte-identical across both hints.

Diagnostic 1 in the notebook re-runs the stripped version every time, so the claim is checked in the
run rather than carried in from here.

## Posture words in a prompt

A1 contains posture clauses; the pose-free control does not. That difference is deliberate, and the rule behind
it is narrower than "never describe the pose":

- **A posture clause that agrees with the hint is useful.** A skeleton has no depth and no contact
  information. It cannot say which way the torso leans, what the hands are touching, or where the
  body's weight sits. A1's clauses supply exactly that for `pose_01`, and the depth ControlNet
  reinforces it.
- **A posture clause that contradicts the hint is dropped, not blended.** A1 sent unchanged to the
  standing `pose_02` produces a standing figure with no stool at all, and scores no worse than the
  pose-free control on the same hint. The skeleton wins the contradiction outright. The notebook re-measures
  this every run as **diagnostic 1**, so the figures quoted in section 7 come from the same run as
  everything else. An earlier draft that said "standing" over a seated hint did grow extra limbs, so
  this is not a guarantee at every scale, but at the settings above the failure mode is omission
  rather than duplication.
- **The demonstration still needs a pose-free prompt somewhere.** If every prompt described the
  posture, a reader could not tell whether the pose came from ControlNet or from the text.
  Diagnostic 1's prompt is that control: it names no posture and still reproduces the reference.

Posture is supplied by the skeleton and measured, not asserted. The prompt's remaining job is
subject, wardrobe, setting, light and medium.

## Prompt-writing notes

- **Concrete beats abstract.** `85mm lens at f/2` and `natural skin with visible pores` are rendered;
  bare abstract nouns tend to be ignored. Measured on an earlier prompt pair: `wet asphalt reflecting
  colourful neon lights` (6 tokens, a material plus an optical effect) was rendered faithfully, while
  `futuristic laundromat` (3 tokens, an abstract noun) was ignored almost entirely. Concreteness
  decided it, not length.
- **Not every concrete clause survives either.** `cropped jacket` appears in both A1 and the pose-free control
  and was not honoured in any generated image - the jacket comes out hip length every time, across
  different hints and different prompts. Garment cut is not reliably controllable here. *This is an
  eye judgement, not a measurement:* `pose_fidelity()` compares twelve limb joint coordinates and
  has no view of hem length, fingers or faces.
- **The negative prompt suppresses, it does not guarantee.** `text` and `watermark` are both in it,
  and one background still came back carrying garbled lettering on a wall. Also an eye judgement.
- **Two conditioning scales, not one.** Pose is held at 0.8 rather than 1.0 because the depth map
  already pins the limbs; giving pose full weight on top of depth stiffens the body. The notebook
  re-measures this every run as **diagnostic 2**: the same prompt, hint and seed at `pose_scale=1.0,
  depth_scale=0.0` against the defaults. On a single-ControlNet pipeline the opposite held - there,
  0.8 let a folded knee land 0.45 of the frame from the hint and 1.0 was required. That figure comes
  from a configuration this notebook no longer runs and is not comparable with the current numbers.
- Keep the negative prompt identical across a comparison, otherwise it stops being a
  single-variable experiment.

## The 77-token ceiling is real

Each SDXL text encoder truncates at 77 tokens and `diffusers` does it silently, from the tail. A
prompt that quietly loses its last clauses is indistinguishable from a prompt that was ignored.

Counts scanned with `CLIPTokenizer` against the text currently in the notebook:

| Prompt | Tokens | Posture clauses | Scope |
|---|---|---|---|
| A1 (steps 1 and 2b) | 66/77 | 3, agreeing with `pose_01`, ignored on `pose_02` | person, posture, wardrobe, setting, lens |
| A2 (step 2a) | 65/77 | 3, same clauses in the drawing's register | subject, posture, wardrobe, medium, paper |
| pose-free control (diagnostic 1) | 40/77 | none | person, wardrobe, setting, lens |
| negative | 44/77 | none | defects to suppress |

Re-run the scan after editing any prompt.
