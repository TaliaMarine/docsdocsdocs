# KAT-Coder-V2.5-Dev on a MacBook M1 (32 GB) for OpenCode: smart, not fast

One model (`Kwaipilot/KAT-Coder-V2.5-Dev`), served by `llama-server` from the local disk,
used from OpenCode. Two profiles: **256k context** (the model's native limit, use this)
and **1M context** (YaRN-extended, with compromises).

Checked 2026-10-24 against the model card, `config.json` and the Hugging Face file listing.
I have not run these commands on your machine. Sizes come from the repo listing, the
memory numbers are my calculation, so confirm them with the server log (§6).

The launchd service, the API key and the keep-awake/network setup are the same as in
`qwen-macbook-32gb-service.md` (§1.2, 1.5–1.7); this doc repeats only what changes.

## 1. The model

| Item | Value |
|---|---|
| Base | post-trained from Qwen3.6-35B-A3B (SFT + RL), text only, Apache-2.0 |
| Architecture | MoE, 35B total / ~3B active (256 experts, 8 per token), 40 layers |
| Attention | hybrid: 10 of 40 layers use full attention, the rest linear attention |
| Native context | 262,144 tokens; YaRN factor 4 gives ~1M |
| Mode | thinks by default; tool calls use the `qwen3_coder` format |
| Original size | 69 GB (BF16), so you run a quantized GGUF |

KV cache is small because only 10 layers keep one: 10 layers × 2 KV heads × head dim 256
× 2 (K and V) × 2 bytes ≈ **20 KB per token** at f16. The linear layers add a small fixed
state (~0.1 GB).

## 2. Pick the file

From `bartowski/Kwaipilot_KAT-Coder-V2.5-Dev-GGUF`:

| Quant | Size | Use |
|---|---|---|
| `IQ4_XS` | 18.8 GB | **256k default**: ~23 GB total, safe headroom |
| `Q4_K_M` | 21.4 GB | the smartest that can fit at 256k: ~25.5 GB total, very tight (see §4) |
| `IQ3_M` | 16.9 GB | **1M profile**: needs the space for the cache |

Q6/Q8 (29 / 37 GB) do not fit in 32 GB. Disk is not a limit: your data volume has
354 GB free.

## 3. One-time setup

```bash
brew install llama.cpp huggingface-cli
brew upgrade llama.cpp                 # the hybrid architecture needs a recent build
llama-server --version

mkdir -p ~/models/kat-coder
hf download bartowski/Kwaipilot_KAT-Coder-V2.5-Dev-GGUF \
  --include "*IQ4_XS*" --local-dir ~/models/kat-coder
# for the 1M profile also: --include "*IQ3_M*"

openssl rand -hex 24                   # this is <API_KEY>
```

Raise the GPU memory limit (resets on reboot; the persistent launch daemon is in
`qwen-macbook-32gb-service.md` §1.4):

```bash
sudo sysctl iogpu.wired_limit_mb=26624
```

Close other heavy apps on this Mac. The model, the cache and macOS share 32 GB.

## 4. Profile A: 256k context (recommended)

Run it in the foreground first:

```bash
llama-server \
  -m ~/models/kat-coder/Kwaipilot_KAT-Coder-V2.5-Dev-IQ4_XS.gguf \
  --alias kat-coder \
  --host 0.0.0.0 --port 8080 --api-key <API_KEY> \
  --jinja -ngl 99 \
  -c 262144 -fa on -ctk q8_0 -ctv q8_0 \
  -b 2048 -ub 1024 \
  --temp 1.0 --top-p 0.95 --top-k 20 --presence-penalty 1.5
```

Memory: 18.8 GB model + 2.6 GB cache (262,144 × 20 KB, halved by q8) + ~1.5 GB buffers
≈ **23 GB**, under the 26 GB GPU limit.

For more intelligence, swap in `Q4_K_M` (change the file name): ≈ 25.5 GB, which leaves
almost nothing. Only keep it if Activity Monitor's memory pressure stays green and swap
stays near 0 (`sysctl vm.swapusage`) during a long session. If it turns yellow, go back
to `IQ4_XS`. The quality gap between the two is small.

`q8_0` cache is close to lossless; don't go lower for coding.

## 5. Profile B: 1M context (YaRN)

```bash
llama-server \
  -m ~/models/kat-coder/Kwaipilot_KAT-Coder-V2.5-Dev-IQ3_M.gguf \
  --alias kat-coder-1m \
  --host 0.0.0.0 --port 8080 --api-key <API_KEY> \
  --jinja -ngl 99 \
  -c 1048576 --rope-scaling yarn --rope-scale 4 --yarn-orig-ctx 262144 \
  -fa on -ctk q4_0 -ctv q4_0 \
  -b 2048 -ub 512 \
  --temp 1.0 --top-p 0.95 --top-k 20 --presence-penalty 1.5
```

Memory: 1,048,576 × 20 KB = 21 GB at f16, 10.5 GB at q8 (does not fit), **5.2 GB at q4_0**.
So: 16.9 GB model + 5.2 GB cache + ~1.5 GB ≈ **24 GB**. That is the only combination I
found that fits, and it costs you in two ways:

- A 3-bit model and a 4-bit cache are both visibly less accurate than Profile A.
- YaRN scaling is static: it is applied to every prompt, so short-prompt quality drops a
  little as well (the model card says the same).

Be realistic about using it. Prompt processing on M1 is a few hundred tokens/s, so
filling even 500k tokens takes on the order of an hour on the first pass, and attention
cost grows with length. I haven't tested that it holds up at that size. Use Profile B
only for a task that really needs a whole huge codebase in one prompt, and run Profile A
the rest of the time.

If the server dies at start with `failed to allocate`, lower `-c` (for example 786432),
keeping the same flags.

## 6. Check it

```bash
curl http://localhost:8080/v1/models -H "Authorization: Bearer <API_KEY>"
curl http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer <API_KEY>" -H "Content-Type: application/json" \
  -d '{"model":"kat-coder","messages":[{"role":"user","content":"Reverse a linked list in Python."}]}'
```

The answer contains `reasoning_content` (the thinking) plus the final `content`. That is
normal. The cache size appears in the startup log on the `llama_kv_cache` line; compare
it with the numbers above. While working, watch Activity Monitor → Memory pressure.

## 7. Run it as a service

Same plist as `qwen-macbook-32gb-service.md` §1.5. Put these `ProgramArguments`
(Profile A; for Profile B use the flags of §5):

```xml
<string>/opt/homebrew/bin/llama-server</string>
<string>-m</string><string>/Users/YOU/models/kat-coder/Kwaipilot_KAT-Coder-V2.5-Dev-IQ4_XS.gguf</string>
<string>--alias</string><string>kat-coder</string>
<string>--host</string><string>0.0.0.0</string>
<string>--port</string><string>8080</string>
<string>--api-key</string><string><API_KEY></string>
<string>--jinja</string>
<string>-ngl</string><string>99</string>
<string>-c</string><string>262144</string>
<string>-fa</string><string>on</string>
<string>-ctk</string><string>q8_0</string>
<string>-ctv</string><string>q8_0</string>
<string>-b</string><string>2048</string>
<string>-ub</string><string>1024</string>
<string>--temp</string><string>1.0</string>
<string>--top-p</string><string>0.95</string>
<string>--top-k</string><string>20</string>
<string>--presence-penalty</string><string>1.5</string>
```

Use the absolute path (no `~`) for the model. Switching profiles means editing these
arguments and running `launchctl bootout …` then `bootstrap …`, since one server holds
one model.

## 8. OpenCode (workstation)

`~/.config/opencode/opencode.json`; the model key must equal `--alias`, and
`limit.context` must not exceed `-c`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "macserver": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "KAT-Coder (MacBook server)",
      "options": {
        "baseURL": "http://<SERVER_HOST>:8080/v1",
        "apiKey": "{env:LLAMA_API_KEY}"
      },
      "models": {
        "kat-coder": {
          "name": "KAT-Coder V2.5 Dev",
          "limit": { "context": 262144, "output": 32768 }
        }
      }
    }
  },
  "model": "macserver/kat-coder",
  "small_model": "macserver/kat-coder"
}
```

For Profile B use the key `kat-coder-1m` and `"context": 1048576`. Set
`export LLAMA_API_KEY="<API_KEY>"` in `~/.zshrc`. Titles and summaries also go to the
same model, which is slow with thinking on but acceptable.

## 9. Settings that matter for "smart"

- **Keep thinking on** (the default). It is how the benchmarks were run. The card's
  sampling (`temp 1.0, top-p 0.95, top-k 20, presence-penalty 1.5`) is what the commands
  above use.
- If code output drops repeated identifiers or looks odd, lower `--presence-penalty` to
  ~0.5–1.0. This is my suggestion, not the card's.
- Tool calls require `--jinja`. If edits fail or tool calls show up as plain text, update
  llama.cpp first (`brew upgrade llama.cpp`).
- Long sessions beat many short ones: the prompt cache reuses the shared prefix, and
  at 256k the first big prompt is slow (minutes), the following turns are not.
- Expect roughly 30–50 tokens/s once warm, less with a full context, and long thinking
  before answers. That is the price of "smart".

## 10. Troubleshooting

| Symptom | Fix |
|---|---|
| `failed to allocate` / killed at start | lower `-c`; check `sysctl iogpu.wired_limit_mb`; use a smaller quant |
| Memory pressure yellow/red, swap grows | back to `IQ4_XS`, or lower `-c` to 131072 |
| Unknown architecture error | `brew upgrade llama.cpp` |
| 401 | `LLAMA_API_KEY` ≠ `--api-key` |
| OpenCode "context exceeded" | `limit.context` larger than server `-c` |
| Connection refused | server asleep or not running; see the Qwen doc §1.6–1.7 |
