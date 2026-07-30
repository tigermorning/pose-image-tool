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
| 3. Extract pose | upload photos, extract skeletons, **inspect the preview** | ~10 s |
| 4. Generate image | `generate()` helper | instant |
| 5. Experiments | A and B, 4 images | ~4 min on a T4 |
| 6. Save results | writes `samples/` and downloads `samples.zip` | ~5 s |
| 7. Findings | write-up | - |

Section 3's side-by-side preview is not decoration. A missing arm in the skeleton is a missing arm in the output - check it before spending GPU time.

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
| Pose ignored | raise `conditioning_scale` toward 1.0 |
| Stiff, mannequin-like anatomy | lower `conditioning_scale` toward 0.6 |
| Prompt ignored | raise `guidance_scale`, or lower `conditioning_scale` |
| Doubled limbs | remove posture words that *contradict* the hint; words that agree with it ("seated" over a seated skeleton) are harmless |
| CUDA OOM on a T4 | replace `pipe.to('cuda')` with `pipe.enable_model_cpu_offload()` |
| `controlnet_aux` import error | use section 3b's MediaPipe fallback extractor |
| Black or NaN images | the fp16-fix VAE was skipped; the stock SDXL VAE overflows in fp16 |

## Test results

Two experiments, each isolating one variable. Full prompts and seeds in [`prompts.md`](prompts.md).
Measured on an 8 GB RTX 3060 Ti (`enable_model_cpu_offload()`, fp16, torch 2.6.0+cu124,
diffusers 0.39.0): 5.4 s per pose extraction, 435-553 s per generated image (mean 506 s).
A 16 GB T4 on the full-GPU path runs roughly 40-60 s per image.

| Experiment | Held fixed | Varied | Result |
|---|---|---|---|
| **A** same pose, different prompts | `pose_01`, seed 1234, 28 steps, guidance 6.0, conditioning 0.8 | African woman in a bohemian dress on neon-lit wet asphalt / Middle Eastern warrior in neon armour in a laundromat | Skeleton held completely. Raised hand at the head, elbow angle, braced left hand, extended leg, bent leg, head tilt, and the figure's position and scale in frame were all reproduced across two unrelated subjects. Subject, wardrobe, setting, palette and lighting changed freely. |
| **B** same prompt, different poses | neon-armour warrior prompt, seed 777, all sampler settings | `pose_01` (clean detection) / `pose_02` (tangled detection) | Identity carried across both. `B1` reproduced the seated leaning pose faithfully. `B2` lost the pose entirely: the crouch became a generic symmetric seated figure and the crop tightened from full-body to waist-up. |

**Headline finding: the skeleton owns where the body is, the prompt owns what the body is.**
A and B are the two halves of that claim.

**Second finding, from B2: a bad hint does not corrupt pose control, it silently switches it off.**
The tangled skeleton did not produce a tangled pose - it produced the blandest posture consistent
with the prompt, because ControlNet had nothing coherent to enforce and the model fell back on its
own prior. Nothing in the generation step reports that the hint was bad.

| Measure | Result |
|---|---|
| Pose-accurate images | 3 of 4 (`B2` failed) |
| Images with malformed hands | 4 of 4 |
| Pose extractions usable | 1 of 2 references (`pose_02` tangled) |

`samples/pose_02.png` was kept deliberately. It is the evidence behind limitation 1 rather than an
assumed failure mode.

## Limitations

Observed in this run, not assumed.

1. **Pose detection is the ceiling, and it fails silently.** `pose_02` produced a confident-looking
   but wrong skeleton; the only downstream symptom was a pose that ignored the reference. Always
   inspect the section 3 preview.
2. **Self-occlusion is the specific trigger.** Both references were seated. The one where the arms
   wrap across the shins is the one that failed. Crossed and overlapping limbs are the problem, not
   seated poses in general.
3. **The hint is 2D.** No depth is encoded, so limb crossings cannot be disambiguated - the
   mechanism behind failures 1 and 2. Facing direction also has to be stated in the prompt.
4. **Hands were malformed in all 4 images.** Fingers merged on every raised hand; in `B2` the hand
   fused with the knee armour. Hand keypoints were on and `deformed hands` was in the negative
   prompt - both reduced severity without fixing it.
5. **Framing is not independent of the pose.** Crop follows the skeleton's extent in frame:
   `pose_01` gave full-body compositions, the compact `pose_02` gave a waist-up crop.
6. **Setting and lighting clauses lose to subject clauses.** `neon armour` rendered emphatically
   every time while `futuristic laundromat` survived as vague panels, and `studio lighting` was
   overridden by neon spill from the armour.
7. **Speed on small VRAM is a real cost.** 8.4 min per image on 8 GB via CPU offload against about
   a minute on a 16 GB T4. Fitting the model is not the same as running it usefully.
8. **Single subject only.** Multi-person skeletons are detected, but SDXL blends identities between
   overlapping figures. Not exercised in this run.
9. **Reproducibility is per-environment.** Fixed seeds reproduce a comparison on the same GPU and
   library versions; exact pixels are not portable across GPUs or `diffusers` versions.
10. **`controlnet_aux` is lightly maintained.** Its imports break on `timm` upgrades - `0.0.9`
    cannot be resolved against a current `timm` at all, which is why `0.0.10` is pinned and a
    MediaPipe fallback ships in section 3b.

## Alternative: FLUX.1-dev

For better anatomy and prompt adherence, swap section 2 for `Shakker-Labs/FLUX.1-dev-ControlNet-Union-Pro-2.0` in `pose` mode on top of `black-forest-labs/FLUX.1-dev`. Costs:

- a paid L4 or A100 runtime, plus 8-bit quantisation
- a Hugging Face token, since FLUX.1-dev is a gated repo
- `guidance_scale` semantics differ from SDXL and need retuning

The notebook's structure and the `generate()` interface carry over unchanged.

## Reference images and licensing

No reference photos are committed. Section 3 uploads your own images, which keeps the repository free of third-party photo licensing questions. Model licences apply as published: SDXL under CreativeML Open RAIL++-M, the ControlNet weights under their own repository's terms.
