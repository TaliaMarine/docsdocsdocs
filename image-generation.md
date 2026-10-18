# Image generation on the MacBook (32 GB) from a remote computer

Goal: generate images on the model Mac (M1 Pro/Max, 32 GB) and drive it from the
workstation, using **Qwen-Image-2.1** with two profiles: **fast** (Turbo, 8 steps) and
**detailed** (base model, 20–40 steps).

*Checked against Hugging Face and the stable-diffusion.cpp docs on 2026-10-18. I have not
run any of these commands on a Mac, and the docs have no Apple Metal performance numbers,
so every timing below is a guess to be measured.*

Prerequisites: SSH access to the Mac (`macos-openssh.md`), the GPU memory limit from
`qwen-macbook-32gb-service.md` §1.4, and the memory rules from `hugging-models.md` §2.

---

## 1. The model

**Qwen-Image-2.1** (Qwen, Sep 2026): a **7B** image generator (not 20B like the first
Qwen-Image), so it **does fit in 32 GB, even at full precision**.

| | Qwen-Image-2.1 (base) | Qwen-Image-2.1-Turbo |
|---|---|---|
| Use for | **detail**, best quality, typography | **speed**, drafts, iterations |
| Steps | 20 (sd.cpp example) – 40 (reference) | **8** (fixed schedule) |
| CFG scale | 6.0 | 1.0 |
| Also does | image editing (up to 10 reference images), transparent PNG (RGBA) | same |
| License | Qwen Research License (check terms for commercial use) | same |

An image model is **three files**, all required:

| Part | File | Size |
|---|---|---|
| Diffusion model ("denoiser", the quantized one) | GGUF, see §1.1 | 2.5–14.2 GB |
| Text encoder (Qwen3-VL-8B) | `Qwen3-VL-8B-Instruct-UD-Q4_K_XL.gguf` | ≈ 5.2 GB |
| VAE | `qwen_image_2.1_vae_bf16.safetensors` | small (≈ 0.3 GB) |

Older Qwen-Image / Wan VAEs and Qwen2.5-VL encoders do **not** work with 2.1.

### 1.1 Which quant of the diffusion model

Sizes from the `unsloth/Qwen-Image-2.1-Turbo-GGUF` and `unsloth/Qwen-Image-2.1-GGUF`
listings (same sizes for both).

| Quant | Size | Total in RAM* | Recommendation |
|---|---|---|---|
| Q2_K / Q3_K | 2.5–3.6 GB | ~9–11 GB | visibly worse; only for tight memory |
| **Q4_K_M** | **4.2 GB** | ~11–13 GB | **fast profile default** (publisher's recommendation) |
| Q5_K_M | 5.4 GB | ~12–14 GB | |
| Q6_K | 6.3 GB | ~13–15 GB | |
| **Q8_0** | **7.6 GB** | ~15–17 GB | **detailed profile default**; closest to full precision |
| F16 / BF16 | 14.2 GB | ~21–24 GB | fits, but barely better than Q8_0 and needs the GPU memory limit raised |

\* denoiser + 5.2 GB text encoder + VAE + working memory at 1024², my estimate. Higher
resolutions need more: see §6.

About your file `qwen-image-2.1-UC-BF16.gguf`: it exists in `abenzerps/Qwen-Image-2.1-Uncensored-Quantized`
at **14.23 GB**, and "UC" there stands for an *uncensored community fine-tune*, not an
official Qwen release. It fits in 32 GB, but the Q8_0 (7.59 GB) or Q4_K_M (4.60 GB) from
the same repo give almost the same image for half the memory and time. The official
Qwen weights are the safer default (known behavior and license); use the community
variant only if you specifically need it and check its license.

*(This corrects `hugging-models.md`, which assumed the old 20B model.)*

---

## 2. Tools

| Tool | Role | Why / why not |
|---|---|---|
| **stable-diffusion.cpp** (`sd-cli`, `sd-server`) | **recommended**: runs GGUF models with Metal | single binary, no Python, supports Qwen-Image 2.1 officially, has a web UI and HTTP API |
| ComfyUI + `leejet/ComfyUI-GGUF` | graphical node editor in the browser | most flexible (workflows, LoRAs, editing); heavier install (Python); the GGUF page says to use leejet's node, not city96's |
| Draw Things / DiffusionBee | native macOS apps | listed on the model card as local apps; easy, but you drive them from the Mac's screen unless you enable their own API (not verified) |
| Diffusers (`QwenImage21Pipeline`) | reference implementation | needs diffusers from source, torch, transformers ≥ 5.17 and the unquantized 14 GB BF16 weights; the page's examples target CUDA, so skip it on Mac |

This guide uses stable-diffusion.cpp, with ComfyUI as an optional second route.

> **Performance warning from the project:** its build guide says Metal has issues with
> very large matrix operations and is "highly inefficient at the moment". Expect an M1
> to be **much slower than an NVIDIA GPU**: minutes per image, not seconds, especially for
> the base model. The Turbo model exists for exactly this reason.

---

## 3. Server Mac setup

### 3.1 Build stable-diffusion.cpp with Metal

```bash
xcode-select --install
brew install cmake

mkdir -p ~/src && cd ~/src
git clone --recursive https://github.com/leejet/stable-diffusion.cpp
cd stable-diffusion.cpp

mkdir build && cd build
cmake .. -DSD_METAL=ON
cmake --build . --config Release -j

ls bin/      # expect sd-cli and sd-server
./bin/sd-cli --help | head -50
```

Update later with `git pull origin master && git submodule update --init` and rebuild.
Qwen-Image 2.1 support is recent, so use a fresh checkout.

### 3.2 Download the model files

Keep them in their own directory.

```bash
brew install huggingface-cli          # or: pipx install "huggingface_hub[cli]"
mkdir -p ~/models/qwen-image-2.1 && cd ~/models/qwen-image-2.1

# diffusion models: Turbo (fast) and base (detail)
hf download unsloth/Qwen-Image-2.1-Turbo-GGUF qwen-image-2.1-turbo-Q4_K_M.gguf --local-dir .
hf download unsloth/Qwen-Image-2.1-GGUF qwen-image-2.1-Q8_0.gguf --local-dir .   # check exact filename in the repo

# text encoder
hf download unsloth/Qwen3-VL-8B-Instruct-GGUF Qwen3-VL-8B-Instruct-UD-Q4_K_XL.gguf --local-dir .

# VAE (from the Comfy-Org repo). hf keeps the repo's folder layout, so the file lands in a subfolder
hf download Comfy-Org/Qwen-Image-2.1 --include "*qwen_image_2.1_vae_bf16.safetensors" --local-dir .

# move it next to the other files and check it exists
mv "$(find . -name qwen_image_2.1_vae_bf16.safetensors | head -1)" . 2>/dev/null
ls -lh qwen_image_2.1_vae_bf16.safetensors
```

**Check every file exists before running `sd-cli`.** A missing VAE does not fail
cleanly: it prints "cannot inspect model source … No such file or directory", then
hundreds of `VAE tensor … not in model metadata` lines, then
`model metadata validation failed`. The fix is just the correct path.

```bash
ls -lh *.gguf qwen_image_2.1_vae_bf16.safetensors
```

The exact file names of the base GGUFs are not listed in the sources I read: the Turbo
file is `qwen-image-2.1-turbo-<QUANT>.gguf`; look in the base repo with the file listing
command from `hugging-models.md` §2.3 and adjust. `hf download` accepts the repo-relative
path if the VAE sits in a subfolder (`vae/qwen_image_2.1_vae_bf16.safetensors`).

Disk: Turbo Q4_K_M + base Q8_0 + encoder + VAE ≈ 17.5 GB.

### 3.3 Raise the GPU memory limit

```bash
sudo sysctl iogpu.wired_limit_mb=26624    # see qwen-macbook-32gb-service.md §1.4 to persist
```

---

## 4. Generate: the two profiles

Set shorthand variables once (dimensions must be divisible by 32):

```bash
cd ~/models/qwen-image-2.1
SD=~/src/stable-diffusion.cpp/build/bin
VAE=$PWD/qwen_image_2.1_vae_bf16.safetensors
LLM=$PWD/Qwen3-VL-8B-Instruct-UD-Q4_K_XL.gguf
ls -lh "$VAE" "$LLM"        # both must exist
```

### 4.1 Fast profile (Turbo, 8 steps, CFG 1)

```bash
$SD/sd-cli \
  --diffusion-model qwen-image-2.1-turbo-Q4_K_M.gguf \
  --vae "$VAE" --llm "$LLM" \
  -p "a lighthouse on a cliff at dawn, volumetric light, photorealistic" \
  --steps 8 --cfg-scale 1.0 --sampling-method euler \
  --sigmas qwen-image-2.1-turbo \
  -W 1024 -H 1024 --flash-attn off,diffusion=on --offload-to-cpu \
  -o ~/out/fast.png
```

`--sigmas qwen-image-2.1-turbo` is a preset for the model's fixed 8-step schedule
(equivalent to `--sigmas 1.0,0.978453,0.95418,0.926626,0.89508,0.845148,0.704534,0.414568,0.0`).
The unsloth page uses the older spelling `--diffusion-fa` (your build prints "deprecated;
use `--flash-attn off,diffusion=on`", which is what the commands here use). The sd.cpp
page adds `--offload-to-cpu`. If your build rejects the preset name, use the explicit
list.

### 4.2 Detailed profile (base model, 20–40 steps, CFG 6)

```bash
$SD/sd-cli \
  --diffusion-model qwen-image-2.1-Q8_0.gguf \
  --vae "$VAE" --llm "$LLM" \
  -p "a lighthouse on a cliff at dawn, volumetric light, photorealistic, fine detail" \
  --steps 30 --cfg-scale 6.0 --sampling-method euler \
  -W 1536 -H 1536 --flash-attn off,diffusion=on --offload-to-cpu \
  -o ~/out/detailed.png
```

20 steps is the sd.cpp example; the reference code uses 40 at 2048². Start at 20–30 and
raise only if you see flaws.

### 4.3 Practical settings

| Goal | Setting |
|---|---|
| Quick preview of many ideas | Turbo Q4_K_M, 768² or 1024², 8 steps |
| Final image | base Q8_0, 1024²–1536², 30 steps; or take the best Turbo seed and re-render |
| Reproduce / vary | `-s <seed>` (same seed = same image); `-b 4` for a batch |
| Out of memory | lower `-W/-H`, lower quant, keep `--offload-to-cpu`, close other apps |
| Text in the image | quote it in the prompt (`a sign that says "OPEN"`); Qwen-Image is strong at typography |
| Transparent PNG | prompt: *"This is an RGBA image with transparency. <description>. The image has alpha channel and the background is transparent."* and save as `.png`/`.webp` |
| Edit an existing image | add `-r input.png` (repeatable) and `--llm_vision mmproj-Qwen3VL-8B-Instruct-F16.gguf`, then describe the change in `-p`; use the base model (CFG 6) |

Supported presets of the model are 2048×2048 and the 4:3, 3:2, 16:9 and 9:16 variants
(for example 2752×1536, 1536×2752). On 32 GB, stay at ≈ 1024–1536 per side.

---

## 5. Using it from the remote workstation

### 5.1 Option A: SSH + `sd-cli` (simplest, no extra service)

```bash
ssh modelmac
# run the §4 command; then, from the workstation:
scp modelmac:~/out/fast.png ~/Downloads/
```

Or run the whole thing from the workstation in one line:

```bash
ssh modelmac '~/src/stable-diffusion.cpp/build/bin/sd-cli ... -o ~/out/a.png' \
  && scp modelmac:~/out/a.png .
```

Loading the models takes time on each run (~10 GB), which is why a server (Option B) is
better for many images.

### 5.2 Option B: `sd-server` (models stay loaded; web UI + HTTP API)

Start it on the model Mac (the 1234 port and web UI are its defaults; the server has
**no authentication**, so bind it to localhost and reach it through an SSH tunnel):

```bash
$SD/sd-server \
  --diffusion-model ~/models/qwen-image-2.1/qwen-image-2.1-turbo-Q4_K_M.gguf \
  --vae ~/models/qwen-image-2.1/qwen_image_2.1_vae_bf16.safetensors \
  --llm ~/models/qwen-image-2.1/Qwen3-VL-8B-Instruct-UD-Q4_K_XL.gguf \
  --cfg-scale 1.0 --offload-to-cpu \
  --listen-ip 127.0.0.1 --listen-port 1234
```

Tunnel from the workstation and open the web UI:

```bash
ssh -N -L 1234:localhost:1234 modelmac
open http://localhost:1234/
```

The API lives under `/sdcpp/v1/...` with compatibility routes under `/v1/...` and
`/sdapi/v1/...`. I could not confirm the exact routes or parameters, so check
`sd-server -h` and the project's server README before scripting against them. An
OpenAI-style call would look roughly like this (**unverified sketch**):

```bash
curl http://localhost:1234/v1/images/generations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "a cat in a spacesuit", "n": 1, "size": "1024x1024"}'
```

The server holds **one set of weights**. To switch between the fast and detailed model
you restart it with different `--diffusion-model` (and `--cfg-scale`/`--sigmas`), or run
two servers on different ports only if memory allows (§6).

Run it as a service the same way as `llama-server` (launchd plist from
`qwen-macbook-32gb-service.md` §1.5, replacing the program and arguments; keep
`--listen-ip 127.0.0.1`).

> Do **not** use `--listen-ip 0.0.0.0` on a shared network: there's no login, and anyone
> reaching the port could use your machine. Keep the SSH tunnel.

### 5.3 Option C: ComfyUI (node editor, more control)

```bash
git clone https://github.com/comfyanonymous/ComfyUI ~/src/ComfyUI && cd ~/src/ComfyUI
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
git clone https://github.com/leejet/ComfyUI-GGUF custom_nodes/ComfyUI-GGUF
pip install -r custom_nodes/ComfyUI-GGUF/requirements.txt

# models
ln -s ~/models/qwen-image-2.1/qwen-image-2.1-turbo-Q4_K_M.gguf models/diffusion_models/
# text encoder (safetensors) -> models/text_encoders/qwen3vl_8b_bf16.safetensors (Comfy-Org/Qwen-Image-2.1)
# VAE -> models/vae/qwen_image_2.1_vae_bf16.safetensors

python main.py --listen 127.0.0.1 --port 8188
```

In the Qwen-Image-2.1 text-to-image template swap *Load Diffusion Model* for *Unet Loader
(GGUF)*; for Turbo set KSampler to 8 steps, CFG 1, sampler `euler`, scheduler `simple`.
From the workstation: `ssh -N -L 8188:localhost:8188 modelmac`, then open
`http://localhost:8188`. Note the ComfyUI text encoder is the ~16 GB BF16 safetensors
(`qwen3vl_8b_bf16.safetensors`), so memory use is higher than with sd.cpp's 5 GB GGUF
encoder; ComfyUI may load and unload parts to fit. Its Mac (MPS) performance for this
model is also something I haven't been able to check.

---

## 6. Memory and what to run at the same time

The Mac also hosts the coding model, and 32 GB cannot hold both comfortably.

| Running | Approx. RAM |
|---|---|
| Qwen3-Coder-30B-A3B (32k) | ≈ 23 GB |
| Turbo Q4_K_M + encoder + VAE at 1024² | ≈ 11–13 GB |
| Base Q8_0 + encoder + VAE at 1536² | ≈ 16–19 GB |

→ **Don't run an image job while the 30B coding model is loaded.** Options: stop the
service first (`launchctl bootout gui/$(id -u)/com.local.llama-server`), or pair the image
server with a light coding model (Qwen3.5-9B, 9.5 GB, `hugging-models.md` §4c). The image
server loads weights at startup and keeps them; `--offload-to-cpu` keeps weights in RAM
and moves them to the GPU as needed (on Apple Silicon RAM is shared, so this mainly
helps avoid the GPU memory cap).

---

## 7. Tuning, quality and speed

- **Speed.** Turbo (8 steps) is ~3–5× fewer steps than the base model, and lower resolution
  is quadratic: 768² costs ≈ 0.56× of 1024², 1536² ≈ 2.25×. Expect minutes per image on
  M1-class hardware; measure with `time` and write down your numbers.
- **`--flash-attn off,diffusion=on`** (flash attention) saves memory and is used in all the
  reference commands.
- **Prefix cache.** Qwen-Image-2.1 caches text context in memory; the server docs
  expose it with `--model-args qwen_image_2_1_prefix_cache_type=<f32|f16|q8_0|q4_0>`
  (≈ 4 / 2 / 1.06 / 0.56 GiB at 4096 prefix tokens; `auto` by default). If you hit
  out-of-memory, try `q8_0`, or disable it with `--model-args qwen_image_2_1_prefix_cache=false`.
  It already retries once without the cache on memory failure.
- **Faster preview decoding.** The project mentions the TAEQI2.1 tiny decoder (see its
  `taesd.md`) for quick previews.
- **Quality ladder.** Q4_K_M is the publisher's recommended balance; Q8_0 ≈ original;
  below Q4 images "visibly drift". Try Q4 first and move up only if you see problems.
- **Black / white / NaN outputs.** See the project's troubleshooting doc; the options are
  `--linear-scale` and `--attn-scale`.
- **Upscale.** sd.cpp supports ESRGAN upscaling, which is often cheaper than generating at
  2048².
- **Prompting.** Qwen-Image takes long, natural-language descriptions; Chinese and English
  both work. Put wanted text in quotes.

### Other models if Qwen doesn't suit you

All of these are listed as supported by stable-diffusion.cpp (README) and work with the
same `sd-cli`/`sd-server` and the same remote setup; each needs its own text-encoder and
VAE files, per its docs in `docs/`:

| Model | Why |
|---|---|
| FLUX.1-schnell | 4-step, fast, strong quality (Apache-2.0), older (2024) |
| FLUX.3 Klein (GGUF repos appeared Oct 2026) | newest FLUX line; check size and license |
| SDXL-Turbo / SDXL | small, very fast on Mac, lower quality and weaker text |
| Krea-2-Turbo, Z-Image | newer fast models I haven't checked in detail |

---

## 8. Quick reference

```text
Build        cmake .. -DSD_METAL=ON && cmake --build . --config Release
Fast         sd-cli ... turbo-Q4_K_M.gguf --steps 8 --cfg-scale 1.0 --sigmas qwen-image-2.1-turbo -W 1024 -H 1024
Detailed     sd-cli ... Q8_0.gguf --steps 30 --cfg-scale 6.0 -W 1536 -H 1536
Server       sd-server ... --listen-ip 127.0.0.1 --listen-port 1234
Remote       ssh -N -L 1234:localhost:1234 modelmac   then   http://localhost:1234/
Copy result  scp modelmac:~/out/x.png .
Files        denoiser GGUF + Qwen3-VL-8B encoder GGUF + qwen_image_2.1_vae_bf16.safetensors
Memory       ≈ 11–13 GB (Turbo Q4) · 16–19 GB (base Q8) · stop the coding model first
```
