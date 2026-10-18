# Adding and running other Hugging Face models (MacBook M1, 32 GB)

How to find a model on Hugging Face, work out whether it fits in 32 GB **before**
downloading it, and run it. It builds on `qwen-macbook-32gb-service.md` (llama.cpp
`llama-server` as a launchd service).

## 1. First, what kind of model is it?

The runtime depends on the model type, not on the `.gguf` extension:

| Model type | Examples | Run with |
|---|---|---|
| Text LLM (chat, code) | Qwen3-Coder, Qwen3, Llama, Gemma | `llama-server` (llama.cpp) |
| Vision-language LLM | Qwen2.5-VL, Gemma 3 | `llama-server` with `--mmproj` |
| Embeddings / rerankers | Qwen3-Embedding | `llama-server --embedding` |
| **Image generation** (diffusion) | **Qwen-Image**, FLUX, SDXL | `stable-diffusion.cpp` or ComfyUI + ComfyUI-GGUF |

> **About `qwen-image-2.1-UC-BF16.gguf`:** it is real: 14.23 GB, in
> `abenzerps/Qwen-Image-2.1-Uncensored-Quantized` ("UC" = an uncensored community
> fine-tune of Qwen-Image-2.1). Qwen-Image is an *image generation* model, so it does
> **not** run in `llama-server` and can't be used by OpenCode. Qwen-Image-2.1 has only
> **7B** parameters (the first Qwen-Image had ~20B), so even BF16 (14 GB) **fits in 32 GB**,
> though Q8_0 (7.6 GB) or Q4_K_M (4.2–4.6 GB) look almost the same and run faster. The
> full image-generation setup, with fast and detailed profiles, is in
> `image-generation.md`.

---

## 2. Will it fit? Estimating memory

### 2.1 The rule of thumb

```
RAM needed ≈ model file size  +  KV cache (LLMs only)  +  runtime overhead  +  macOS
```

- **Model file size**: use the actual size shown on Hugging Face. Rough formula:
  `parameters × bits per weight ÷ 8`.
- **Overhead**: ~1–2 GB for compute buffers.
- **macOS + your other apps**: ~5–6 GB.
- **GPU limit**: by default the GPU can wire only ~⅔ of RAM (~21 GB on 32 GB). Raise it
  with `sudo sysctl iogpu.wired_limit_mb=26624` (see the Qwen doc). **Practical budget
  for model + cache + overhead is therefore ≈ 22–25 GB.**

### 2.2 Bits per weight by quantization

| Format | ≈ bits/weight | 7B | 14B | 30B | 20B |
|---|---|---|---|---|---|
| BF16 / F16 | 16 | 14 GB | 28 GB | 60 GB | 40 GB |
| Q8_0 | 8.5 | 7.4 | 15 | 32 | 21 |
| Q6_K | 6.6 | 5.8 | 12 | 25 | 17 |
| Q5_K_M | 5.7 | 5.0 | 10 | 21 | 14 |
| Q4_K_M | 4.8 | 4.2 | 8.5 | 18 | 12 |
| Q3_K_M | 3.9 | 3.4 | 6.9 | 15 | 10 |
| Q2_K | 3.0 | 2.6 | 5.3 | 11 | 7.5 |

Rules: **Q4_K_M is the best size/quality default**; below Q3 quality drops fast; BF16/F16
only makes sense for models under ~10B.

Mixture-of-experts models (names like `30B-A3B`) need RAM for **all** parameters (30B),
but compute only the active ones (3B), so they are fast for their size. Dense models are
slower at the same file size.

### 2.3 Check the real file sizes without downloading

```bash
# list files and sizes in a repo (size in GB)
curl -s "https://huggingface.co/api/models/<ORG>/<REPO>/tree/main" \
  | python3 -I -c '
import sys, json
for f in json.load(sys.stdin):
    if f["type"] == "file":
        print(f"{f.get(\"size\", 0)/1e9:8.2f} GB  {f[\"path\"]}")'
```

If a quant is split into parts (`-00001-of-00003.gguf`), add up all parts; llama.cpp
loads the series by pointing at part 1.

### 2.4 KV cache (LLMs): grows with context

```
KV bytes = 2 × layers × KV_heads × head_dim × context_tokens × bytes_per_element
```

(`bytes_per_element` is 2 for f16 cache, ~1 for `q8_0`.) Get `layers`, KV heads
(`num_key_value_heads`) and `head_dim` from the model's `config.json` on Hugging Face.

Worked example, Qwen3-Coder-30B-A3B (48 layers, 4 KV heads, head_dim 128):

```
2 × 48 × 4 × 128 × 2 bytes ≈ 98 KB per token
32k context ≈ 3.2 GB     64k ≈ 6.4 GB     (halve with -ctk q8_0 -ctv q8_0)
```

Total for that setup: 18.6 GB model + 3.2 GB cache + ~1.5 GB overhead ≈ **23 GB**, which
fits under the 26 GB GPU limit.

Models with many KV heads (older, non-GQA) use far more cache: check `config.json`
before choosing a large context.

### 2.5 Verify against the real thing

Once it runs, look at the actual numbers:

```bash
# llama.cpp prints buffer sizes at startup
tail -n 80 ~/Library/Logs/llama/server.err.log | grep -Ei "buffer|KV|offload"

# live memory, GPU memory and swap
sudo powermetrics --samplers gpu_power -n 1 | head -30
memory_pressure | tail -3
vm_stat | head -8
```

Signs you're over budget: Activity Monitor memory pressure goes yellow/red, swap used
grows by GBs, or tokens/s collapses. Fix by using a smaller quant, a smaller context, or
a quantized KV cache.

### 2.6 Context length: native limit vs. what your RAM allows

Three different numbers are easy to mix up:

| Term | Meaning |
|---|---|
| **Native / trained context** | What the model was trained for (e.g. 262,144 tokens for Qwen3-Coder and Qwen3.8). Shown on the model card. |
| **`-c` / `--ctx-size`** | What `llama-server` actually allocates. **This is what costs RAM.** Default is small, so always set it. |
| **`limit.context` in OpenCode** | What the client believes it may send. Keep it equal to (or below) `-c`. |

Rules:

- A bigger `-c` costs RAM even when you don't fill it (the KV cache is allocated up front).
- The KV cache per token depends on the architecture. Compute it with §2.4 or measure
  it from the `llama_kv_cache` line in the server log.
- With `--parallel N`, `-c` is **divided** over N slots.
- Longer prompts are slower: prompt processing on Apple Silicon is a few hundred
  tokens/s, so a 60k-token prompt can take minutes the first time. Later turns reuse the
  prompt cache, so keep sessions long instead of restarting them.
- Quality drops when the context is nearly full, and for most models long before the
  native limit. Treat 32k–64k as the sweet spot for agentic coding and use bigger values
  only when you need them.
- Going beyond the native limit (e.g. 1M with YaRN, `--rope-scaling yaml --rope-scale 4
  --yarn-orig-ctx 262144`) hurts short-prompt quality and is pointless on 32 GB.
- **Quantizing the KV cache** halves its size: `-ctk q8_0 -ctv q8_0` (needs `-fa on`).
  `q4_0` quarters it but visibly hurts quality, so avoid for coding.
- OpenCode's system prompt plus tool definitions already use roughly 10k tokens, so
  anything below `-c 16384` leaves little room. OpenCode compacts the conversation as it
  approaches `limit.context`; a lower limit means more frequent compaction.

Context budget examples (f16 cache; halve it for `q8_0`). The file sizes are the Q4
quants; `+1.5 GB` overhead included, and the total must stay under ≈ 24–25 GB.

| Model | KV per token | 32k | 64k | 128k | 256k | Fits at (q8 cache) |
|---|---|---|---|---|---|---|
| Qwen3-Coder-30B-A3B (18.6 GB) | ~96 KB | 3.1 GB | 6.3 GB | 12.6 GB | 25 GB | 64k (≈ 23 GB) |
| Qwen3.8-27B UD-Q4_K_M (16.5 GB) | ~64 KB* | 2.1 GB | 4.2 GB | 8.4 GB | 16.8 GB | 128k (≈ 22 GB) |
| Qwen3.5-9B Q8_0 (9.5 GB) | small* | ~1 GB | ~2 GB | ~4 GB | ~8 GB | 256k (≈ 15 GB) |

\* Qwen3.8 and Qwen3.5 are hybrid models: only one layer in four uses full attention
(Qwen3.8-27B: 16 of 64 layers, 4 KV heads, head dim 256 on the model card); the rest
keep a small fixed-size state. So their cache grows much more slowly than classic
models. My Qwen3.8 numbers come from the model card's config; the Qwen3.5-9B row is an
estimate. Confirm with the server log.

Pick the context like this:

1. Start with `-c 32768`.
2. If OpenCode compacts too often or you need to feed whole files, double it and
   switch the cache to `q8_0`.
3. Check §2.5 under a long prompt. Stop increasing when memory pressure leaves green
   or swap grows.

---

## 3. Running another LLM (GGUF) in `llama-server`

### 3.1 Straight from Hugging Face

Syntax: `-hf <ORG>/<REPO>:<QUANT>`. The quant tag selects the file.

```bash
llama-server \
  -hf bartowski/Qwen2.5-Coder-14B-Instruct-GGUF:Q6_K \
  --alias qwen-coder-14b \
  --host 0.0.0.0 --port 8080 --api-key <API_KEY> \
  --jinja -ngl 99 -c 32768 -fa on
```

Files are cached in `~/Library/Caches/llama.cpp`.

For gated models (licence acceptance required) log in first:

```bash
brew install huggingface-cli          # or: pipx install "huggingface_hub[cli]"
hf auth login                         # paste a read token from huggingface.co/settings/tokens
export HF_TOKEN=<token>               # also needed in the launchd plist EnvironmentVariables
```

### 3.2 Download manually, then point at the file

Useful for fixed versions or offline use:

```bash
mkdir -p ~/models
hf download unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF \
  --include "*Q4_K_M*" --local-dir ~/models/qwen3-coder-30b

llama-server -m ~/models/qwen3-coder-30b/<file>.gguf --alias qwen3-coder ...
```

### 3.3 Picking a good repo and file

- Prefer well-known quantizers: `unsloth`, `bartowski`, `lmstudio-community`, `ggml-org`,
  or the original publisher (`Qwen/...-GGUF`).
- Read the model card for the **recommended sampling parameters** and set `--temp`,
  `--top-p`, `--top-k`, `--min-p` accordingly.
- `UD-` quants (Unsloth dynamic) are often better than the same-size plain quant.
- Skip `imatrix` vs. not debates for now; both work.
- Models with a chat template that supports tools need `--jinja` for OpenCode.
- Reasoning ("thinking") models emit long thoughts, costing context and time. Fine for
  chat, often poor for agent loops.
- Vision models need a second file: add `--mmproj <mmproj-file>.gguf` (or let `-hf`
  fetch it automatically on recent builds).

### 3.4 Make it available to OpenCode

Edit the launchd plist (`-hf`, `--alias`, `-c`), restart, then add the model under the
provider in `~/.config/opencode/opencode.json` on the workstation:

```json
"models": {
  "qwen3-coder":     { "name": "Qwen3-Coder 30B-A3B", "limit": { "context": 32768, "output": 8192 } },
  "qwen-coder-14b":  { "name": "Qwen2.5-Coder 14B",   "limit": { "context": 32768, "output": 8192 } }
}
```

`llama-server` serves **one model per process**. To switch models, restart the service,
or run a second instance on another port (`--port 8081`) **only if both fit in memory
together**, which they usually won't on 32 GB. Optionally use `llama-swap`
(`brew install llama-swap`) as a proxy that starts and stops models on demand.

---

## 4. Running an image model (Qwen-Image and similar)

Image models are not served through the OpenAI chat API. Two common ways to run them:

### 4.1 stable-diffusion.cpp (command line, Metal)

```bash
git clone https://github.com/leejet/stable-diffusion.cpp ~/src/stable-diffusion.cpp
cd ~/src/stable-diffusion.cpp
git submodule update --init --recursive
cmake -B build -DSD_METAL=ON && cmake --build build --config Release -j
```

Qwen-Image needs **three** files: the diffusion model (the big GGUF), a VAE, and a
text encoder. **Qwen-Image 2.1 uses Qwen3-VL-8B and its own VAE**
(`qwen_image_2.1_vae_bf16.safetensors`); the older Qwen-Image used Qwen2.5-VL-7B, as in
the sketch below, and the two are not interchangeable. The exact flags change between
releases; run `build/bin/sd-cli --help` and the project's `docs/qwen_image_2.1.md`.
`image-generation.md` has the verified 2.1 commands. The older-model shape is:

```bash
build/bin/sd-cli \
  --diffusion-model ~/models/qwen-image/<qwen-image-quant>.gguf \
  --vae             ~/models/qwen-image/<vae>.safetensors \
  --llm             ~/models/qwen-image/<qwen2.5-vl-7b-quant>.gguf \
  --diffusion-fa --offload-to-cpu \
  -p "a lighthouse at dawn, photorealistic" \
  --cfg-scale 2.5 --steps 30 -H 1024 -W 1024 -o ~/out.png
```

### 4.2 ComfyUI + ComfyUI-GGUF (browser UI)

Install ComfyUI, add the `ComfyUI-GGUF` custom node, and place files in
`ComfyUI/models/unet/` (diffusion GGUF), `models/vae/`, and `models/text_encoders/`.
Run with `python main.py --listen 0.0.0.0` to use it from the workstation's browser
at `http://<SERVER_HOST>:8188` (it has no authentication; use only on a trusted LAN or via
an SSH tunnel: `ssh -L 8188:localhost:8188 modelmac`).

### 4.3 Memory for image models

Same file-size rule, but you must add the **text encoder** and **VAE**, and the
activations at high resolution:

```
total ≈ diffusion GGUF + text encoder (7B VL ≈ 5–8 GB at Q4–Q8) + VAE (~0.3 GB) + activations (1–4 GB)
```

For Qwen-Image-2.1 (7B; the older Qwen-Image was ~20B and would need ~2.5× these
numbers), with a ≈ 5.2 GB Q4 text encoder:

| Diffusion quant | File | + encoder + VAE + activations | Fits 32 GB? |
|---|---|---|---|
| BF16 / F16 | 14.2 GB | ~21–24 GB | Yes, with `iogpu.wired_limit_mb` raised; little gain over Q8 |
| Q8_0 | 7.6 GB | ~15–17 GB | Yes, comfortable |
| Q4_K_M | 4.2 GB | ~11–13 GB | Yes, publisher's recommended balance |

Image models are more sensitive to quantization than text models; use Q8_0 for quality
and Q4_K_M for speed, and lower the resolution (768²) before dropping the quant if memory
is tight. An M1 is slow at this: expect minutes per image, not seconds. Commands,
profiles and remote access are in `image-generation.md`.

---

## 4b. DeepSeek "Flash" on 32 GB

*Checked against Hugging Face on 2026-10-18. DeepSeek ships often, so re-check
https://huggingface.co/deepseek-ai.*

### The official Flash models do not fit

| Model | Size | Fits 32 GB? |
|---|---|---|
| `deepseek-ai/DeepSeek-V4.1-Flash` (latest; also `-1017` update, `-Vision-Exp`) | 552B backbone, 8B active (prefill) / 16B (decode); ~755B including the Engram tables | **No** |
| `deepseek-ai/DeepSeek-V4-Flash` / `-0731` | 284B total, 13B active | **No** |

Even though only 8–16B parameters are active per token, all of them have to be
available. The smallest community GGUF of V4.1-Flash (`antirez/deepseek-v4.1-flash-gguf`,
Q2) is ~340 GiB. It runs only on `ds4` (DwarfStar, a Metal-only runtime, not llama.cpp)
and only by streaming weights from the SSD, and its README targets a 128 GB Mac. On a
32 GB M1 with a typical internal SSD it would be neither practical nor fast, and it won't
load in `llama-server`. V4.1-Flash also has no Jinja chat template, so it is not a
drop-in for the setup in this repo.

### What does fit: the distilled 9B

`Jackrong/Qwen3.5-9B-DeepSeek-V4-Flash-GGUF` is Qwen3.5-9B fine-tuned on reasoning
traces from DeepSeek-V4 (it is **not** DeepSeek's own model, but a small Qwen model
trained to imitate it). Apache-2.0, standard GGUF, runs in `llama-server`:

| Quant | File | Notes on 32 GB |
|---|---|---|
| Q8_0 | 9.5 GB | best quality, plenty of room for context |
| Q6_K | 7.4 GB | good default |
| Q5_K_M | 6.5 GB | |
| Q4_K_M | 5.6 GB | smallest sensible |
| Q3_K_M | 4.6 GB | not needed on 32 GB |

On 32 GB use **Q8_0**: 9.5 GB + KV cache + overhead stays well under 16 GB, so you
could even keep it loaded next to the Qwen3-Coder service (≈23 GB) if you cap the
contexts; check §2.5 first.

```bash
llama-server \
  -hf Jackrong/Qwen3.5-9B-DeepSeek-V4-Flash-GGUF:Q8_0 \
  --alias deepseek-flash-9b \
  --host 0.0.0.0 --port 8081 --api-key <API_KEY> \
  --jinja -ngl 99 -c 32768 -fa on \
  --temp 0.7 --top-p 0.95
```

Notes from the model card: it thinks at length and may over-reason on simple prompts,
so it is slower per answer than a non-reasoning model; use temperature 0.7 for coding
(up to 1.0 for open-ended reasoning); its context length is not documented, so keep
`-c` at 32k and test. It needs a recent llama.cpp (Qwen3.5 architecture): `brew upgrade
llama.cpp`. The repo also has an `mmproj.gguf` (0.9 GB) for images, but the card doesn't
document vision support, so treat it as experimental.

To use it from OpenCode, add it as a model (it uses its own port here, 8081, so it is a
second provider entry, or change `-hf`/`--alias` in the existing service to swap):

```json
"deepseek-flash-9b": { "name": "DeepSeek-V4-Flash distilled (Qwen3.5-9B)", "limit": { "context": 32768, "output": 8192 } }
```

Not to be confused with: `deepseek-ai/dflash_*` and `eagle3_*` repos, which are
speculative-decoding drafter models for Qwen3/Gemma, not chat models.

---

## 4c. Smarter and faster options (Qwen and DeepSeek)

*Checked against Hugging Face on 2026-10-18; sizes marked ≈ come from my own estimates,
not the repo listing.*

### Speed is about the bytes read per token

Generation speed on Apple Silicon is limited by memory bandwidth:

```
tokens/s  ≲  memory bandwidth ÷ bytes of ACTIVE weights per token
```

A 32 GB MacBook from the M1 generation is an M1 Pro (200 GB/s) or M1 Max (400 GB/s).
Real speeds are about 50–70 % of this ceiling. So:

- **Dense 27B at Q4 (16.5 GB):** ≲ 12 tok/s on an M1 Pro. Smart, but slow.
- **MoE 3B active (≈ 2 GB read per token):** ≲ 100 tok/s ceiling; real 30–60.
- **Dense 9B at Q4 (5.7 GB):** ≲ 35 tok/s; at Q8 (9.5 GB) ≲ 20.

Prompt processing (reading your repo) is compute-bound and scales with the active
parameters, which is also why small and MoE models feel much quicker in agent loops.

### Qwen: the ladder

| Tier | Model (GGUF repo) | Quant → file size | Notes |
|---|---|---|---|
| **Smartest that fits** | `unsloth/Qwen3.8-27B-GGUF` (also `ggml-org/Qwen3.8-27B-GGUF`) | `UD-Q4_K_M` 16.5 GB, `UD-Q4_K_XL` 17.6 GB, `UD-Q5_K_M` 19.8 GB | Dense, 27B, newest Qwen that fits (Aug 2026). Model card benchmarks: SWE-bench Pro 61.7, LiveCodeBench v6 90.3. 262k native context. **Slowest** of this list. |
| **Fast, strong (MoE)** | `Qwen3-Coder-30B-A3B-Instruct-GGUF` (the main doc) | `Q4_K_M` 18.6 GB | 3B active: best speed per quality here; older (Jul 2025). |
| Fast MoE, newer | `unsloth/Qwen3.5-35B-A3B-GGUF` | `UD-IQ4_XS` 17.5 GB, `UD-Q3_K_XL` 16.6 GB (`Q4_K_M` is 22 GB: too tight) | 3B active; general model (not coder-specific); the Q4 files only fit with a small context. |
| **Lighter and faster** | `unsloth/Qwen3.5-9B-GGUF` | `Q8_0` 9.5 GB, `Q6_K` 7.5 GB, `Q4_K_M` 5.7 GB | Dense 9B; 2–3× the speed of the 27B, but clearly less capable on multi-file edits. Good as the "small model". |
| Very fast, simple tasks | `unsloth/Qwen3.5-4B-GGUF` (≈ 2.7 GB Q4), `…-2B-GGUF`, `…-0.8B-GGUF` | ≈ | Titles, summaries, commit messages, autocomplete-style work; not for agentic coding. |

Too big for 32 GB (don't bother): `Qwen3.8-Flash-Next` (125B, 6B active),
`Qwen3.8-2.4T-A95B`, `Qwen3-Coder-Next`, `Qwen3.5-122B-A10B`.

Sampling for Qwen3.8 (from its model card). Thinking mode (default): `--temp 1.0 --top-p
0.95 --top-k 20 --min-p 0`. Non-thinking: `--temp 0.7 --top-p 0.8 --top-k 20
--presence-penalty 1.5`. Thinking can be turned off per request with
`chat_template_kwargs: {"enable_thinking": false}`; or for the whole server:
`--chat-template-kwargs '{"enable_thinking":false}'`. Turning thinking off is the
cheapest speed-up you can make for coding agents.

Example, the smart one:

```bash
llama-server \
  -hf unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M \
  --alias qwen3.8-27b \
  --host 0.0.0.0 --port 8080 --api-key <API_KEY> \
  --jinja -ngl 99 -c 65536 -ctk q8_0 -ctv q8_0 -fa on \
  --temp 0.7 --top-p 0.8 --top-k 20 --presence-penalty 1.5 \
  --chat-template-kwargs '{"enable_thinking":false}'
```

Example, the light and fast one:

```bash
llama-server \
  -hf unsloth/Qwen3.5-9B-GGUF:Q8_0 \
  --alias qwen3.5-9b \
  --host 0.0.0.0 --port 8081 --api-key <API_KEY> \
  --jinja -ngl 99 -c 32768 -fa on \
  --temp 0.7 --top-p 0.8 --top-k 20
```

(The Qwen3.5 sampling advice is assumed to be the same as Qwen3.8's; confirm it on the
model card.) Qwen3.8 and Qwen3.5 need a **recent** llama.cpp: run `brew upgrade
llama.cpp` first. The `*-MTP-GGUF` repos (e.g. `unsloth/Qwen3.5-9B-MTP-GGUF`) carry
multi-token-prediction heads for faster decoding; I haven't verified `llama-server`
support for them, so use the plain GGUFs unless the repo card shows how to enable it.

### DeepSeek: smaller and faster

DeepSeek's own current models (V4.1-Flash, V4-Pro) are far too big (§4b). The small
DeepSeek-related models are older or fine-tunes:

| Model (GGUF repo) | Size at Q4 | What it is | Notes |
|---|---|---|---|
| `Jackrong/Qwen3.5-9B-DeepSeek-V4-Flash-GGUF` | 5.6 GB (`Q8_0` 9.5 GB) | Qwen3.5-9B fine-tuned on DeepSeek-V4 reasoning traces (Apr 2026) | See §4b. Reasons at length; the best current "DeepSeek-style" small model. |
| `unsloth/DeepSeek-R1-0528-Qwen3-8B-GGUF` | ≈ 5 GB | Qwen3-8B distilled from DeepSeek-R1-0528 (May 2025) | Strong reasoning for 8B, older, thinks at length. |
| `bartowski/DeepSeek-Coder-V2-Lite-Instruct-GGUF` | ≈ 10 GB (16B MoE, ≈ 2.4B active) | DeepSeek's own small coder (Jun 2024) | Very fast and 128k context, but clearly behind the 2026 models in quality; a lightweight fallback. |
| `unsloth/DeepSeek-R1-Distill-Qwen-14B-GGUF` | ≈ 9 GB | R1 distill (Jan 2025) | Superseded by the models above. |

Reasoning ("thinking") models spend many tokens before answering, which makes them
*feel* slower even at a good tokens/s figure. For OpenCode, prefer non-thinking or
low-effort modes, and keep thinking models for hard problems.

### Which one should I run? (quick guide)

| Situation | Use |
|---|---|
| Hard refactors, many files, you can wait | Qwen3.8-27B `UD-Q4_K_M`, thinking off |
| Everyday coding, fast feedback | Qwen3-Coder-30B-A3B `Q4_K_M` |
| Fast and light, plenty of RAM left for other apps | Qwen3.5-9B `Q8_0` |
| Titles/summaries (OpenCode `small_model`) | Qwen3.5-4B or 2B |
| Reasoning-heavy puzzles on a small budget | Jackrong 9B DeepSeek-V4-Flash |

You can point OpenCode's `model` at the strong one and `small_model` at a light model
served on a second port **if both fit in memory together** (e.g. 27B Q4 at 64k ≈ 22 GB
plus 4B Q4 ≈ 3 GB is too tight; the 30B-A3B plus the 4B is also tight; prefer
switching with `llama-swap`).

---

## 5. Choosing a variant that fits: a checklist

1. Identify the type (§1) and the runtime.
2. List the repo's files and sizes (§2.3).
3. Choose the largest quant with `file size + KV cache + ~2 GB ≤ ~24 GB` (§2.1, §2.4).
4. Raise `iogpu.wired_limit_mb` to `26624`.
5. Start with `-c 16384`/`32768`; increase later if memory pressure is green.
6. Check memory (§2.5) while generating a long response.
7. If it swaps, go down one quant or halve the context. Don't tune other flags first.

## 6. Quick reference

```text
Size of file        curl -s https://huggingface.co/api/models/<ORG>/<REPO>/tree/main
Run (LLM)           llama-server -hf <ORG>/<REPO>:<QUANT> --alias <name> --jinja -ngl 99 -c 32768 -fa on
Download            hf download <ORG>/<REPO> --include "*<QUANT>*" --local-dir ~/models/<name>
KV cache per token  2 × layers × kv_heads × head_dim × 2 bytes   (config.json)
GPU memory limit    sudo sysctl iogpu.wired_limit_mb=26624
Fit rule            file + KV cache + ~2 GB  ≤  ~24 GB
```
