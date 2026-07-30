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
| `samples/` | Pose hints (`pose_NN.png`) and generated images (`output_NN.png`). |

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
| 1. Install dependencies | pinned `diffusers` / `controlnet_aux` / `timm` | ~2 min |
| 2. Load models | OpenPose annotator, then SDXL + ControlNet + fp16-fix VAE | ~3 min |
| 3. Extract pose | upload photos, extract skeletons, **inspect the preview** | ~10 s |
| 4. Generate image | `generate()` helper + one smoke test | ~50 s |
| 5. Experiments | A, B and C | ~6 min |
| 6. Save results | writes `samples/` and downloads `samples.zip` | ~5 s |
| 7. Findings | write-up | - |

Section 3's side-by-side preview is not decoration. A missing arm in the skeleton is a missing arm in the output - check it before spending GPU time.

### 4. Commit the results

Section 6 downloads `samples.zip`. Unzip it into `samples/` and commit. Canonical filenames:

| File | Content |
|---|---|
| `samples/pose_01.png` | skeleton from the first reference |
| `samples/output_01.png` | `pose_01` + prompt A1 (astronaut) |
| `samples/pose_02.png` | skeleton from the second reference |
| `samples/output_02.png` | `pose_02` + prompt B (red-dress dancer) |

The zip also contains the remaining experiment images (`exp_a_*`, `exp_b_*`, `exp_c_*`).

### Tuning

| Symptom | Fix |
|---|---|
| Pose ignored | raise `conditioning_scale` toward 1.0 |
| Stiff, mannequin-like anatomy | lower `conditioning_scale` toward 0.6 |
| Prompt ignored | raise `guidance_scale`, or lower `conditioning_scale` |
| Doubled limbs | remove pose words from the prompt - the skeleton already supplies the pose |
| CUDA OOM on a T4 | replace `pipe.to('cuda')` with `pipe.enable_model_cpu_offload()` |
| `controlnet_aux` import error | use section 3b's MediaPipe fallback extractor |
| Black or NaN images | the fp16-fix VAE was skipped; the stock SDXL VAE overflows in fp16 |

## Test results

Three experiments, each isolating one variable. Full prompts and seeds in [`prompts.md`](prompts.md).

| Experiment | Held fixed | Varied | Result |
|---|---|---|---|
| **A** same pose, different prompts | `pose_01`, seed 1234, 28 steps, guidance 6.0, conditioning 0.8 | astronaut / knight / watercolour ballet dancer | Skeleton holds across all three. Shoulder line, limb angles and hip position are stable while identity, wardrobe, background and lighting change freely. Pose adherence is strongest on the two photorealistic prompts and weakest on the watercolour prompt. |
| **B** same prompt, different poses | red-dress dancer prompt, seed 777, all sampler settings | `pose_01`, `pose_02`, (`pose_03`) | Subject and wardrobe stay recognisably the same while posture follows whichever skeleton is supplied. Framing shifts with the skeleton's position in frame. Foreshortened and self-occluding poses degrade first. |
| **C** conditioning scale sweep | `pose_01`, rock climber prompt, seed 42 | conditioning scale 0.4 / 0.7 / 1.0 | 0.4 looks best but is only loosely posed. 0.7-0.8 is the useful range. 1.0 tracks the skeleton almost exactly at the cost of stiff anatomy and more hand artifacts. |

The headline finding: **the skeleton owns where the body is, the prompt owns what the body is.** A and B are the two halves of that claim.

The `TODO` slots in notebook section 7 are for run-specific numbers (seconds per image on your runtime, pose-accuracy count) - fill them in after your own run.

## Limitations

1. **Pose detection is the ceiling.** A keypoint OpenPose misses is gone; the generator then invents a limb there. Occluded, cropped or foreshortened limbs are the common cause.
2. **The hint is 2D.** No depth is encoded, so "arm toward camera" and "arm raised sideways" can project to nearly identical lines. Facing direction has to be stated in the prompt.
3. **Hands and faces stay weak.** Hand keypoints help; malformed fingers still appear in a meaningful share of samples. The negative prompt reduces this rather than solving it. Face keypoints are off by default - at 832x1216 they add more noise than control.
4. **Single subject only.** Multi-person skeletons are detected, but SDXL blends identities between overlapping figures.
5. **Style trades against pose.** The further a prompt moves from photorealism, the weaker pose adherence gets at the same conditioning scale. Non-photoreal prompts need roughly +0.1-0.2.
6. **Aspect ratio must match.** Hint and output share one resolution; the notebook enforces this by cropping every reference to 832x1216.
7. **Reproducibility is per-environment.** Fixed seeds reproduce a comparison on the same GPU and library versions. Exact pixels are not portable across GPUs or `diffusers` versions - cuDNN kernel selection and fp16 accumulation order differ.
8. **`controlnet_aux` is lightly maintained.** Its imports break on `timm` and `huggingface_hub` upgrades, which is why versions are pinned and a MediaPipe fallback ships in section 3b.

## Alternative: FLUX.1-dev

For better anatomy and prompt adherence, swap section 2 for `Shakker-Labs/FLUX.1-dev-ControlNet-Union-Pro-2.0` in `pose` mode on top of `black-forest-labs/FLUX.1-dev`. Costs:

- a paid L4 or A100 runtime, plus 8-bit quantisation
- a Hugging Face token, since FLUX.1-dev is a gated repo
- `guidance_scale` semantics differ from SDXL and need retuning

The notebook's structure and the `generate()` interface carry over unchanged.

## Reference images and licensing

No reference photos are committed. Section 3 uploads your own images, which keeps the repository free of third-party photo licensing questions. Model licences apply as published: SDXL under CreativeML Open RAIL++-M, the ControlNet weights under their own repository's terms.
