# KAT-Coder-V2.5-Dev on the MacBook M1 (32 GB) for OpenCode, with an external NVMe SSD

Adds `Kwaipilot/KAT-Coder-V2.5-Dev` to the service from `qwen-macbook-32gb-service.md`.
Read that doc first: the launchd service, the API key, the GPU wired limit and the
OpenCode client setup are unchanged. This doc only covers what is different.

Checked 2026-10-24 against the model card and the Hugging Face API.

## 1. What the model is

| Item | Value |
|---|---|
| Base | post-trained from Qwen3.6-35B-A3B (SFT + RL), text only, Apache-2.0 |
| Architecture | MoE, 35B total / ~3B active, 256 experts (8 per token), 40 layers |
| Attention | hybrid: 1 full-attention layer in 4 (10 of 40), the rest linear attention |
| Native context | 262,144 tokens (YaRN to ~1M possible, useless on 32 GB) |
| Mode | **thinks by default** (`enable_thinking`); tool-call format `qwen3_coder` |
| Reported (by Kwaipilot) | SWE-bench Verified 69.4, Terminal-Bench 2.1 41.0 |

Consequences for your Mac:

- Same class as the Qwen3-Coder-30B-A3B you run now: all 35B weights must sit in RAM,
  ~3B are computed per token, so speed should be similar (roughly 30–50 tok/s on M1).
- It is a bit **bigger** (35B vs 30B), so the quant matters more (§2).
- The KV cache is tiny: 10 full-attention layers × 2 KV heads × head dim 256 × 2 (K and V)
  × 2 bytes = **~20 KB per token**, versus ~96 KB for Qwen3-Coder. 128k context costs
  ~2.6 GB. The linear-attention layers add a small fixed state (about 0.1 GB).
  Verify with the `llama_kv_cache` line in the server log, as I derived this from
  `config.json`, not from a measurement.
- Thinking makes agent turns slower and uses context. See §5.

## 2. Which GGUF to pick

The original is BF16 (~70 GB), so you need a quant. There are 70+ community GGUFs; these
are the useful ones. Sizes from the Hugging Face file listing.

| Repo : tag | Size | Verdict on 32 GB |
|---|---|---|
| `bartowski/Kwaipilot_KAT-Coder-V2.5-Dev-GGUF:Q4_K_M` | 21.4 GB | tight: ~24.5 GB with 64k at q8 cache. Only with the §3 wired limit |
| `bartowski/...:IQ4_XS` | 18.8 GB | **recommended**: near-Q4 quality, room for 128k |
| `bartowski/...:Q4_K_S` | 20.6 GB | OK up to ~64k |
| `bartowski/...:IQ3_M` / `Q3_K_M` | 16.9 / 16.2 GB | pick if you also run other apps |
| `mudler/KAT-Coder-V2.5-Dev-APEX-GGUF` `APEX-I-Compact` | 16.5 GB | mixed-precision, designed to be small; popular (most downloaded) |
| `mudler/...` `APEX-I-Mini` | 13.5 GB | lowest quality that I'd use for coding |
| `bartowski/...:Q2_K` and IQ2 | 10–13 GB | avoid for code |
| `*-MTP-*` repos | | MTP (speculative decoding) variants; need a very recent llama.cpp, so try later |
| `*ABLITERATED*` | | safety-removed fine-tunes, not what you want for an agent |

Budget: file + KV + ~1.5 GB overhead should stay under ~24 GB (GPU limit 26 GB from the
Qwen doc §1.4). `IQ4_XS` at 128k with a q8 cache: 18.8 + 1.3 + 1.5 ≈ **21.6 GB**.

Caveats: I haven't run these quants on your hardware. Quality ranking between
`IQ4_XS`, `APEX-I-Compact` and `Q4_K_M` is not something I can tell you; try two on your
own repo. `IQ` quants decode a little slower than `Q4_K` on Apple GPUs.

Needs a recent llama.cpp (Qwen3.5/3.6 hybrid architecture): `brew upgrade llama.cpp`
and check `llama-server --version` reports a build from the last few months.

## 3. External NVMe SSD

### 3.1 Format: APFS

Use **APFS** (the default, case-insensitive variant).

APFS only shows up in Disk Utility's Format list if the drive uses the **GUID Partition
Map** scheme. A new or previously used drive often has a different scheme, so the APFS
option is missing. To initialize it:

1. Disk Utility → View → **Show All Devices**.
2. Select the top-level *drive* (the manufacturer/enclosure name), not the volume below it.
3. Erase → Name: `Models`, Format: **APFS**, Scheme: **GUID Partition Map**.

Do not enable encryption: an encrypted volume needs a password to mount and won't mount
by itself after a headless reboot.

Hardware notes:

- M1 has no internal NVMe slot, so the enclosure is **USB 3.2 Gen 2 (10 Gbit/s) or
  Thunderbolt/USB4**. Real reads: ~900 MB/s (USB 3.2 Gen 2) up to ~2.5 GB/s (TB3 enclosure
  with a fast NVMe). Prefer a Thunderbolt/USB4 enclosure; a 20 GB model loads in ~10 s
  rather than ~25 s.
- The SSD speed matters only at **load time**: once the weights are mmapped into RAM,
  token generation does not touch the disk (unless you run out of RAM and it starts to
  page, which is the sign to use a smaller quant).
- Plug it in directly, not through a hub, and keep the cable short.

### 3.2 Prepare the volume

Name the volume `Models` (mounts at `/Volumes/Models`):

```bash
diskutil list external            # find the disk id, e.g. disk4; check it's the SSD!
# GUI (§3.1) is safer. CLI equivalent, which DESTROYS the disk:
# diskutil eraseDisk APFS Models GPT /dev/disk4

mkdir -p /Volumes/Models/llama.cpp /Volumes/Models/gguf
sudo mdutil -i off /Volumes/Models           # no Spotlight indexing of model files
sudo mdutil -E /Volumes/Models >/dev/null    # drop any existing index
touch /Volumes/Models/.metadata_never_index
```

Make sure the volume is not slept: `sudo pmset -a disksleep 0` (already in the Qwen doc §1.6).

### 3.3 Put the models on it

Either keep using `-hf` and move the cache, or download explicitly.

**A. Move llama.cpp's cache to the SSD** (every `-hf` download goes there):

```bash
# already-downloaded files can be moved: mv ~/Library/Caches/llama.cpp/* /Volumes/Models/llama.cpp/
export LLAMA_CACHE=/Volumes/Models/llama.cpp   # for the test run in the foreground
```

In the launchd plist add:

```xml
<key>EnvironmentVariables</key>
<dict><key>LLAMA_CACHE</key><string>/Volumes/Models/llama.cpp</string></dict>
```

**B. Download a specific file (recommended, versions stay fixed):**

```bash
hf download bartowski/Kwaipilot_KAT-Coder-V2.5-Dev-GGUF \
  --include "*IQ4_XS*" --local-dir /Volumes/Models/gguf/kat-coder
# → /Volumes/Models/gguf/kat-coder/Kwaipilot_KAT-Coder-V2.5-Dev-IQ4_XS.gguf
```

### 3.4 Gotchas when the model lives on an external drive

1. **Mount timing.** A LaunchAgent starts at login; the drive may not be mounted yet and
   `KeepAlive` will restart-loop. Use a wrapper script that waits (§4.1). (Normally a drive
   attached at boot mounts before login, but not after a USB hiccup.)
2. **Privacy permission.** The first time a background process reads `/Volumes/...`, macOS
   may silently deny it ("Removable Volumes"). If the log shows `Operation not permitted`,
   add `/opt/homebrew/bin/llama-server` (and `/bin/bash` if you use the wrapper, and
   `llama-swap`) under System Settings → Privacy & Security → **Full Disk Access**.
   Test by rebooting without opening Terminal.
3. **Always eject** (`diskutil unmount /Volumes/Models`) after stopping the service, or you
   risk corruption. Stop it first: `launchctl bootout gui/$(id -u)/com.local.llama-server`.
4. If the SSD disconnects mid-session the mmapped model can crash the server; launchd
   restarts it once the wrapper sees the volume again.

## 4. Ways to run it

You asked for options for OpenCode. All of them expose an OpenAI-compatible `/v1` API.

| Option | Pros | Cons | Use when |
|---|---|---|---|
| **A. `llama-server`, one model** (replace Qwen) | simplest, same as the working service, API key built in | one model at a time | you only want KAT |
| **B. `llama-swap` + `llama-server`** | **both models, same port**, loads on demand, OpenCode lists both | extra component, swap takes 10–30 s | you want to keep Qwen3-Coder **and** KAT ("same service") |
| C. MLX (`mlx_lm.server`) | often faster on Apple Silicon | weak auth/tool calling, less tested with OpenCode | experiments |
| D. LM Studio / Ollama | GUI / easy pulls | LM Studio needs a logged-in GUI app; Ollama has no API key and small default context | not for a headless server |
| E. vLLM / SGLang | model-card official | **CUDA/multi-GPU oriented; not for a 32 GB Mac** | no |

Can KAT run on **the same service** (one `llama-server` process)? Not simultaneously:
a `llama-server` process holds one model, and Qwen3-Coder (18.6 GB) + KAT (18.8 GB)
don't fit together in 32 GB. Option B gives you one endpoint that swaps between them.

### 4.1 Option A: launchd service for KAT only

Wrapper that waits for the SSD (`~/bin/run-llama.sh`, `chmod +x`):

```bash
#!/bin/bash
# Wait up to 5 minutes for the model drive, then exec llama-server.
MODEL=/Volumes/Models/gguf/kat-coder/Kwaipilot_KAT-Coder-V2.5-Dev-IQ4_XS.gguf
for i in $(seq 1 60); do [ -f "$MODEL" ] && break; sleep 5; done
[ -f "$MODEL" ] || { echo "model missing: $MODEL" >&2; exit 1; }

exec /opt/homebrew/bin/llama-server \
  -m "$MODEL" \
  --alias kat-coder \
  --host 0.0.0.0 --port 8080 \
  --api-key "$LLAMA_API_KEY" \
  --jinja \
  -ngl 99 \
  -c 131072 -fa on -ctk q8_0 -ctv q8_0 \
  -b 2048 -ub 1024 \
  --temp 1.0 --top-p 0.95 --top-k 20 --presence-penalty 1.5
```

In the existing plist, change `ProgramArguments` to the script and pass the key via the
environment (keeps it out of `ps`):

```xml
<key>ProgramArguments</key>
<array><string>/Users/YOU/bin/run-llama.sh</string></array>
<key>EnvironmentVariables</key>
<dict><key>LLAMA_API_KEY</key><string><API_KEY></string></dict>
```

Restart: `launchctl bootout …; launchctl bootstrap …` as in the Qwen doc §1.5.

First test in the foreground (before launchd) with the exact same flags and a smaller
`-c 32768` if you want to check memory first.

### 4.2 Option B: `llama-swap` in front of both models

```bash
brew install llama-swap      # if the formula isn't found: brew install mostlygeek/llama-swap/llama-swap
```

`~/llama-swap.yaml` (each model's `llama-server` also checks the API key, because
llama-swap forwards the `Authorization` header):

```yaml
healthCheckTimeout: 300        # the first load from the SSD can take a while
models:
  "qwen3-coder":
    ttl: 1800                  # unload after 30 min idle (frees RAM)
    cmd: >
      /opt/homebrew/bin/llama-server --port ${PORT}
      -m /Volumes/Models/gguf/qwen3-coder/Qwen3-Coder-30B-A3B-Instruct-Q4_K_M.gguf
      --alias qwen3-coder --api-key <API_KEY> --jinja -ngl 99
      -c 65536 -fa on -ctk q8_0 -ctv q8_0
      --temp 0.7 --top-p 0.8 --top-k 20 --repeat-penalty 1.05
  "kat-coder":
    ttl: 1800
    cmd: >
      /opt/homebrew/bin/llama-server --port ${PORT}
      -m /Volumes/Models/gguf/kat-coder/Kwaipilot_KAT-Coder-V2.5-Dev-IQ4_XS.gguf
      --alias kat-coder --api-key <API_KEY> --jinja -ngl 99
      -c 131072 -fa on -ctk q8_0 -ctv q8_0 -b 2048 -ub 1024
      --temp 1.0 --top-p 0.95 --top-k 20 --presence-penalty 1.5
```

Only one runs at a time (llama-swap's default); requesting the other unloads the first.
Run it in launchd the same way as §4.1, with `ProgramArguments`:
`/opt/homebrew/bin/llama-swap --config /Users/YOU/llama-swap.yaml --listen 0.0.0.0:8080`.
The firewall/Full Disk Access notes apply to `llama-swap` and the `llama-server` it spawns.
Check the option names against `llama-swap --help` for your version.

Recent `llama-server` builds also have a built-in router mode (several models, loaded on
demand); look for `--models-dir` / `--models-preset` in `llama-server --help` if you'd
rather avoid llama-swap. I haven't verified its behaviour here.

### 4.3 Option C: MLX

```bash
pipx install mlx-lm
mlx_lm.server --model mlx-community/KAT-Coder-V2.5-Dev-OptiQ-4bit --host 0.0.0.0 --port 8080
HF_HOME=/Volumes/Models/hf   # put this in the environment to keep the download on the SSD
```

Check the repo's size first (§2.3 of `hugging-models.md`); a 4-bit 35B is ~19–20 GB.
No API key option, and tool calling depends on the mlx-lm version, so test `/init` in
OpenCode before relying on it. MLX also supports the qwen3_5_moe architecture only in
recent releases (`pipx upgrade mlx-lm`).

## 5. Thinking mode and sampling

The card's settings, with `llama-server` flags:

| Mode | Settings |
|---|---|
| Thinking (default) | `--temp 1.0 --top-p 0.95 --top-k 20 --presence-penalty 1.5` |
| Non-thinking | `--temp 0.7 --top-p 0.8 --top-k 20 --presence-penalty 1.5` plus `--chat-template-kwargs '{"enable_thinking":false}'` |

For OpenCode:

- Start with **thinking on**: it's what the benchmarks used and what is best for hard
  bugs. The reasoning goes into `reasoning_content`, and OpenCode shows it separately.
- If agent loops are too slow or the context fills quickly, run non-thinking
  (`enable_thinking:false`) for everyday edits. With llama-swap you can define two
  entries with the same file, `kat-coder` and `kat-coder-fast`, differing only in this flag.
- The card recommends `preserve_thinking: true` for agent use (keeps earlier reasoning in the
  prompt, at a context cost): `--chat-template-kwargs '{"preserve_thinking":true}'`.
  Whether the llama.cpp template honours it depends on the GGUF's embedded template, so
  I haven't confirmed it works; leave it off first.
- `presence-penalty 1.5` is unusually high, so if output gets odd (missing repeated
  identifiers in code), lower it to ~0.5–1.0.
- Tool calls use the `qwen3_coder` XML-ish format; `--jinja` is what makes llama-server
  parse it. If edits fail or tool calls appear as plain text, update llama.cpp first.

## 6. OpenCode client config

Add to `~/.config/opencode/opencode.json` on the workstation (the key must equal `--alias`,
or the llama-swap model name, and `limit.context` ≤ the server's `-c`):

```json
"models": {
  "qwen3-coder": { "name": "Qwen3-Coder 30B-A3B", "limit": { "context": 65536,  "output": 8192 } },
  "kat-coder":   { "name": "KAT-Coder V2.5 Dev",  "limit": { "context": 131072, "output": 16384 } }
}
```

Leave `small_model` on `macserver/qwen3-coder` if you use llama-swap with `ttl`; if
not, pointing it at KAT avoids a swap for titles. Switch models in the TUI with `/models`.
With llama-swap each switch costs a reload (10–30 s from NVMe), so use it deliberately.

## 7. Memory and speed check

```bash
# during a long prompt (watch the pressure graph in Activity Monitor too)
vm_stat 1 | head -5
sysctl vm.swapusage          # should stay near 0
grep -E "llama_kv_cache|CPU_Mapped|MTL.*buffer" ~/Library/Logs/llama/server.err.log
```

Stop increasing `-c` when memory pressure leaves green or swap grows. If it does, in order:
lower `-c` to 65536, drop to `IQ3_M` / `APEX-I-Compact`, close other apps.

## 8. Recommendation

1. Format the SSD **APFS** with the GUID Partition Map scheme (§3.1).
2. Download `IQ4_XS` from bartowski to `/Volumes/Models/gguf/kat-coder/`.
3. Run it in the foreground once (§4.1 flags, `-c 32768`) to confirm it loads and calls tools.
4. Then set up **llama-swap** (§4.2) if you want both Qwen3-Coder and KAT; otherwise Option A.
5. Compare the two on a real task in your repo before committing to a default.
