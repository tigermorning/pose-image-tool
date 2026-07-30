# pose-image-tool

Generate images that follow **both** a reference pose and a text prompt, in a single Google Colab notebook.

```
reference photo -> OpenPose keypoints -> skeleton image -> ControlNet -> SDXL -> output image
```

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/tigermorning/pose-image-tool/blob/main/pose_tool.ipynb)

## Contents

| Path | What it is |
|---|---|
| `pose_tool.ipynb` | The tool. Install, load models, extract pose, generate, run experiments, save results, findings. |
| `prompts.md` | Every prompt, seed and setting used, so each result is reproducible. |
| `samples/` | Pose hints (`pose_NN.png`), the canonical outputs (`output_NN.png`), and every experiment image (`exp_a_*`, `exp_b_*`). |

## Description

The notebook separates *where the body is* from *what the body is*.

- **OpenPose** (`controlnet_aux.OpenposeDetector`, weights from `lllyasviel/Annotators`) reads a reference photo and returns an 18-keypoint skeleton drawn as a colour-coded image. It generates nothing; it only detects.
- **ControlNet** (`thibaud/controlnet-openpose-sdxl-1.0`) injects that skeleton into every denoising step of the base model, constraining limb placement.
- **SDXL** (`stabilityai/stable-diffusion-xl-base-1.0`) supplies everything the skeleton does not encode: identity, wardrobe, setting, lighting, medium.

All generation goes through one `generate(pose_image, prompt, seed=..., conditioning_scale=...)` helper, so each experiment changes exactly one variable.

### Why SDXL and not FLUX

The assignment suggested FLUX.2-klein. That path was checked and rejected for this deliverable:

- FLUX.2-klein is **9B**, not 4B, and its text encoder is Mistral-Small-3.1 **24B** - roughly 50 GB in bf16.
- It has **no official ControlNet**. The only OpenPose route is a third-party community implementation.
- It cannot run on a free Colab T4, which would make the notebook unreproducible for anyone opening it.

SDXL + OpenPose ControlNet is first-class in `diffusers`, runs in fp16 on a free T4 at ~10 GB peak, and demonstrates the identical pipeline concept. The upgrade path is documented under [Alternative: FLUX.1-dev](#alternative-flux1-dev).

## Usage

### 1. Prepare references

Have **2-3 photos of people in clearly different poses** ready. Requirements that actually matter:

- one person per photo
- full body, or at least head-to-knee
- limbs not overlapping the torso, not cropped by the frame

Crowds, heavy occlusion and extreme foreshortening are where pose detection fails, and nothing downstream can repair it.

### 2. Open the notebook

Open `pose_tool.ipynb` in Colab, then `Runtime > Change runtime type > T4 GPU`. A GPU assertion in section 1 fails fast if you forget.

### 3. Run the sections in order

| Section | What happens | Time |
|---|---|---|
| 1. Install dependencies | `diffusers`, `transformers`, `accelerate`, pinned `controlnet_aux` | ~2 min |
| 2. Load models | OpenPose annotator, then SDXL + ControlNet + fp16-fix VAE | ~3 min |
| 3. Extract pose | upload photos, extract skeletons, **inspect the preview**, validate numerically | ~15 s |
| 4. Generate image | `generate()` helper | instant |
| 5. Experiments | A and B, 4 images | ~4 min on a T4 |
| 6. Save results | writes `samples/` and downloads `samples.zip` | ~5 s |
| 7. Findings | write-up | - |

Section 3's preview and its numeric validation are not decoration. A hint with an unclosed limb chain silently disables pose control for that limb, and nothing later in the run will tell you. Replace any reference that fails before spending GPU time.

### 4. Commit the results

Section 6 downloads `samples.zip`. Unzip it into `samples/` and commit. Canonical filenames:

| File | Content |
|---|---|
| `samples/pose_01.png` | skeleton from the first reference |
| `samples/output_01.png` | `pose_01` + prompt A1 (bohemian dress, neon street) |
| `samples/pose_02.png` | skeleton from the second reference |
| `samples/output_02.png` | `pose_02` + prompt B (neon-armour warrior) - the failure case |

The zip also contains the experiment images under their own names (`exp_a_*`, `exp_b_*`).

### Tuning

| Symptom | Fix |
|---|---|
| Pose ignored, or a folded limb in the wrong place | `conditioning_scale` 1.0, and score it with `pose_fidelity()` rather than trusting your eye |
| Stiff, mannequin-like anatomy | lower `conditioning_scale` toward 0.8, but re-check pose fidelity - that is what 0.8 costs |
| Prompt ignored | raise `guidance_scale`, or lower `conditioning_scale` |
| Doubled limbs | remove posture words that *contradict* the hint; words that agree with it ("seated" over a seated skeleton) are harmless |
| CUDA OOM on a T4 | replace `pipe.to('cuda')` with `pipe.enable_model_cpu_offload()` |
| `AttributeError: module 'mediapipe' has no attribute 'solutions'` | `!pip uninstall -y mediapipe`, restart the runtime. controlnet_aux imports mediapipe at package level and breaks against Colab's build; it does not need it |
| Black or NaN images | the fp16-fix VAE was skipped; the stock SDXL VAE overflows in fp16 |

## Test results

> The numbers below were measured on an earlier pair of references (a crouch and a profile shot) that failed the section 3a keypoint check. The reference pair has since been replaced with two that pass it cleanly, and this section is regenerated from that run. The methodology and the conclusions about hint construction, framing and conditioning scale carry over unchanged.

Two experiments, each isolating one variable. Full prompts and seeds in [`prompts.md`](prompts.md).
Measured on an 8 GB RTX 3060 Ti (`enable_model_cpu_offload()`, fp16, torch 2.6.0+cu124,
diffusers 0.39.0): 5.4 s per pose extraction, 435-553 s per generated image (mean 506 s).
A 16 GB T4 on the full-GPU path runs roughly 40-60 s per image.

| Experiment | Held fixed | Varied | Result |
|---|---|---|---|
| **A** same pose, different prompts | `pose_01`, seed 1234, 28 steps, guidance 6.0, conditioning 0.8 | African woman in a bohemian dress on neon-lit wet asphalt / Middle Eastern warrior in neon armour in a laundromat | The skeleton held across both: raised hand at the head, elbow angle, braced second hand, raised knee, extended leg, head tilt, and the figure's position and scale in frame. Subject, wardrobe, setting, palette and lighting changed freely. |
| **B** same prompt, different poses | neon-armour warrior prompt, seed 777, all sampler settings | `pose_01` / `pose_02`, two clearly different seated poses | Identity carried across both. `B1` reproduced the seated leaning pose. `B2` lost it: the crouch became a generic symmetric seated figure and the crop tightened from full-body to waist-up. |

**The skeleton owns where the body is, the prompt owns what the body is.** A and B are the two
halves of that claim.

**A bad hint does not corrupt pose control, it silently switches it off.** `B2`'s broken skeleton
did not produce a broken pose - it produced the blandest posture consistent with the prompt,
because ControlNet had nothing coherent to enforce. Nothing in the run reports that the hint was
bad, which is why the notebook validates hints numerically in section 3a before generating.

### The hint has to be sharp, and the right shape

An earlier version of this notebook produced **three legs** from `pose_01`, whose detection is
perfect (18/18 keypoints, score 1.00). The cause was hint rendering, not detection.
`OpenposeDetector.__call__` draws at its own internal resolution and rescales: 832x1216 in,
832x**1280** out. Rescaling back to 832x1216 blurred the stick figure - 2210 distinct colours
against the ~43 a clean skeleton has - and squashed the body vertically by 5%.

Given smeared, mis-proportioned limb lines, ControlNet cannot tell which shin belongs to which
thigh, and rendered several interpretations of the same limb at once. Taking keypoints from
`detect_poses()` and drawing them once at exactly the generation resolution removed the extra limb
with prompt, seed and conditioning scale unchanged. **Hint sharpness is part of the conditioning
signal, not cosmetics.**

### Conditioning scale 1.0, measured not guessed

Judging pose fidelity by eye failed twice here, so it is measured: the pose is re-detected in the
generated image and compared with the hint joint by joint (`pose_fidelity()` in section 4).

| conditioning_scale | mean joint error | worst joint | folded knee |
|---|---|---|---|
| 0.8 | 0.099 | 0.448 | lands 0.45 of the frame from the hint |
| 1.0 | **0.031** | **0.066** | within 0.024 |

At 0.8 the seated reference's raised knee and mid-height ankle were simply not reproduced - both
feet ended up on the floor - while the torso and arms matched well enough that the image looked
correct. **Folded and foreshortened limbs are where a low conditioning scale fails first, and it
fails without looking wrong.** 1.0 is now the default; the cost is slightly stiffer material
rendering and a marginally worse raised hand.

| Measure | Result |
|---|---|
| Mean joint error against the hint, conditioning 1.0 | 0.031 (worst joint 0.066) |
| Pose hints passing numeric validation | 2 of 2 once letterboxed; 1 of 2 when centre-cropped |
| Images reproducing the reference pose | `A1`, `A2`, `B1` yes; `B2` no |
| Images with malformed raised hands | all of them, at both conditioning scales |

`samples/pose_02.png` is kept deliberately - it is the evidence behind limitations 2 and 3 rather
than an assumed failure mode.

### Hands: three hint variants, no improvement

Hands were the one defect that survived every other fix, so the hint itself was varied while the
prompt, seed, steps, guidance and conditioning scale were held constant:

| Hint | mean joint error | worst joint | l_wrist | r_wrist | fingers |
|---|---|---|---|---|---|
| **A** body + hand keypoints, drawn at 832x1216 | 0.085 | r_knee 0.266 | **0.028** | 0.161 | merged |
| B body only | 0.108 | l_wrist 0.278 | 0.278 | 0.077 | merged |
| C body only, drawn at 512 and upscaled NEAREST (thicker limb lines) | 0.084 | r_knee 0.245 | 0.204 | 0.054 | merged |

Three conclusions, all measured:

1. **Keep the hand keypoints.** Community guidance for this ControlNet says to disable hand
   detection. That is wrong for this combination: B, the body-only hint, was the worst of the three
   and put the braced wrist 0.278 away from the hint - visibly, the supporting hand slid from the
   machine onto the knee. Hand keypoints anchor the wrist even when they do not fix the fingers.
2. **Thicker limb lines are not the answer either.** C matched A on mean error (0.084 vs 0.085) but
   lost the same wrist accuracy B did, because C also dropped the hand keypoints. Line thickness and
   hand keypoints were not separated in this test; what is clear is that thickness alone bought
   nothing.
3. **Finger quality is not reachable through hint construction.** All three variants produced merged
   fingers on the raised hand. The hint controls where a hand *is*, not how many fingers it has.

The remaining lever is a second pass: crop the hand region, regenerate it at a higher effective
resolution with inpainting, and composite it back. That addresses the real cause - hands occupy too
few pixels at 832x1216 - but it is a separate stage beyond the scope of this tool, and it is not
implemented here.

## Limitations

Observed in this run, not assumed.

1. **A perfect skeleton is not sufficient.** `pose_01` scored 18/18 and still produced three legs
   until the hint was rendered at the right size and sharpness. Detection quality and hint quality
   are separate problems.
2. **Detection failure is silent.** `pose_02` produced a confident-looking but wrong skeleton; the
   only downstream symptom was a pose that ignored the reference. Section 3a turns that into a
   number - run it every time.
3. **How the reference is framed is part of the pipeline.** `pose_02` lost its left ankle, and
   with it all pose control over that leg, because `ImageOps.fit` centre-cropped the reference
   to the target aspect ratio and cut the ankle out of frame. Letterboxing recovers all 18
   keypoints. The crouch was never the problem.
4. **The hint is 2D.** No depth is encoded, so overlapping limbs cannot be disambiguated - the
   mechanism behind both failures above. Facing direction has to be stated in the prompt.
5. **Hands are the one defect nothing here fixed.** Fingers merged on every raised hand in every
   image - before and after the hint fix, at both conditioning scales, and across all three hint
   variants tested above. Hand keypoints were enabled and `deformed hands` was in the negative
   prompt. Braced or resting hands come out clearly better than raised ones. Hands are small at
   832x1216, and no amount of conditioning changes that; an inpainting pass is the real fix and is
   not part of this tool.
6. **Framing cannot be set independently of the pose.** Crop follows the skeleton's extent in
   frame.
7. **Setting and lighting clauses lose to subject clauses.** `neon armour` rendered emphatically
   every time while `futuristic laundromat` survived as vague panels, and `studio lighting` was
   overridden by neon spill from the armour.
8. **Speed on small VRAM is a real cost.** 8.4 min per image on 8 GB via CPU offload against about
   a minute on a 16 GB T4.
9. **Single subject only.** Multi-person skeletons are detected but SDXL blends identities between
   overlapping figures. Not exercised here.
10. **Reproducibility is per-environment.** Fixed seeds reproduce a comparison on the same GPU and
    library versions; exact pixels are not portable.
11. **`controlnet_aux` is lightly maintained.** Its imports break on `timm` upgrades - `0.0.9`
    cannot be resolved against a current `timm` at all, which is why `0.0.10` is pinned and a
    MediaPipe fallback ships in section 3c.

## Alternative: FLUX.1-dev

For better anatomy and prompt adherence, swap section 2 for `Shakker-Labs/FLUX.1-dev-ControlNet-Union-Pro-2.0` in `pose` mode on top of `black-forest-labs/FLUX.1-dev`. Costs:

- a paid L4 or A100 runtime, plus 8-bit quantisation
- a Hugging Face token, since FLUX.1-dev is a gated repo
- `guidance_scale` semantics differ from SDXL and need retuning

The notebook's structure and the `generate()` interface carry over unchanged.

## Reference images and licensing

No reference photos are committed. Section 3 uploads your own images, which keeps the repository free of third-party photo licensing questions. Model licences apply as published: SDXL under CreativeML Open RAIL++-M, the ControlNet weights under their own repository's terms.
