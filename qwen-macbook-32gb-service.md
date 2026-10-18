# Running Qwen3-Coder as a LAN service on a MacBook M1 (32 GB) for OpenCode

Goal: turn a MacBook M1 with 32 GB RAM into a headless, always-on, OpenAI-compatible
coding model server, and use it from OpenCode on another MacBook on the same network.

- **Server** (the "model Mac"): runs `llama-server` (llama.cpp) under `launchd`.
- **Client** (the "workstation"): runs the OpenCode CLI pointed at the server.

> **Model choice.** Qwen's current coding line is **Qwen3-Coder**. The variant that fits
> 32 GB is **Qwen3-Coder-30B-A3B-Instruct** (Mixture-of-Experts, 30B total / ~3B active
> per token). At 4-bit it is ~18 GB on disk and generates fast on Apple Silicon because
> only ~3B parameters are active per token. Larger Qwen coder models (80B+, 480B) do not
> fit. Qwen releases often, so before you start, check https://huggingface.co/Qwen and
> https://huggingface.co/unsloth for a newer coder GGUF that is ≤ ~20 GB, and swap the
> model name in the commands below if there is one.
>
> **Update (checked 2026-10-18):** a newer, general-purpose Qwen exists that also fits:
> **Qwen3.8-27B** (dense, Aug 2026; `unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M`, 16.5 GB). On its
> model card it beats the older coder model on coding benchmarks, and its hybrid
> architecture needs much less memory per token of context, so it can run 128k. The
> trade-off is speed: being dense, it is several times slower per token than the 3B-active
> Qwen3-Coder used below. This guide keeps Qwen3-Coder-30B-A3B as the fast default; see
> `hugging-models.md` §4c for how to swap in Qwen3.8-27B or lighter models.

---

## 0. Placeholders used below

| Placeholder | Meaning | Example |
|---|---|---|
| `<SERVER_HOST>` | Hostname or IP of the model Mac | `modelmac.local` or `192.168.1.50` |
| `<API_KEY>` | Shared secret you generate | output of `openssl rand -hex 24` |
| Port | `8080` | change if taken |

---

## 1. Server setup (model Mac)

### 1.1 Install llama.cpp

```bash
# Xcode CLI tools + Homebrew (skip if already installed)
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install llama.cpp
llama-server --version
```

Keep it current; llama.cpp moves fast and new Qwen releases often need a recent build:

```bash
brew upgrade llama.cpp
```

### 1.2 Generate an API key

Anyone on your network can otherwise use the server, so always set a key.

```bash
openssl rand -hex 24
# copy the output; it is <API_KEY>
```

### 1.3 Test run in the foreground

`-hf` downloads the model from Hugging Face on first run (~18 GB) and caches it in
`~/Library/Caches/llama.cpp`.

```bash
llama-server \
  -hf unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF:Q4_K_M \
  --alias qwen3-coder \
  --host 0.0.0.0 --port 8080 \
  --api-key <API_KEY> \
  --jinja \
  -ngl 99 \
  -c 32768 \
  -fa on \
  --temp 0.7 --top-p 0.8 --top-k 20 --repeat-penalty 1.05
```

What the flags do:

| Flag | Why |
|---|---|
| `-hf repo:quant` | download + use a GGUF straight from Hugging Face |
| `--alias qwen3-coder` | the model name clients will request |
| `--host 0.0.0.0` | listen on the LAN, not only localhost |
| `--api-key` | require `Authorization: Bearer <API_KEY>` |
| `--jinja` | use the model's chat template; **required for tool calling**, which OpenCode relies on |
| `-ngl 99` | offload all layers to the Metal GPU |
| `-c 32768` | context window (32k). Safe default on 32 GB; see tuning for more |
| `-fa on` | flash attention (faster, less memory; needed for quantized KV cache) |
| `--temp … --repeat-penalty` | sampling values Qwen recommends for Qwen3-Coder |

If your `llama-server` rejects `-fa on`, use plain `-fa` (older builds).

Wait for `server is listening on http://0.0.0.0:8080`, then test from the same Mac:

```bash
curl http://localhost:8080/v1/models -H "Authorization: Bearer <API_KEY>"

curl http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-coder",
    "messages": [{"role":"user","content":"Write a Python function that reverses a linked list."}]
  }'
```

Stop it with `Ctrl+C` once it works.

### 1.4 Give the GPU enough memory (important on 32 GB)

By default macOS lets the GPU wire only about two thirds of RAM (~21 GB on a 32 GB
machine). The ~18 GB model plus KV cache gets tight. Raise the limit to 26 GB:

```bash
sudo sysctl iogpu.wired_limit_mb=26624
```

This resets on reboot. To persist it, create a launch daemon:

```bash
sudo tee /Library/LaunchDaemons/com.local.iogpu-wired-limit.plist >/dev/null <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.local.iogpu-wired-limit</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/sbin/sysctl</string>
    <string>iogpu.wired_limit_mb=26624</string>
  </array>
  <key>RunAtLoad</key><true/>
</dict>
</plist>
EOF
sudo chown root:wheel /Library/LaunchDaemons/com.local.iogpu-wired-limit.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/com.local.iogpu-wired-limit.plist
sysctl iogpu.wired_limit_mb
```

Leave at least ~6 GB for macOS. Don't go above ~27000 on a 32 GB machine.

### 1.5 Run it as a service (launchd)

A LaunchAgent starts the server at login and restarts it if it crashes.

```bash
mkdir -p ~/Library/LaunchAgents ~/Library/Logs/llama
LLAMA_BIN="$(which llama-server)"
echo "$LLAMA_BIN"   # normally /opt/homebrew/bin/llama-server
```

Create the plist (replace `<API_KEY>` and, if needed, the binary path):

```bash
cat > ~/Library/LaunchAgents/com.local.llama-server.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.local.llama-server</string>
  <key>ProgramArguments</key>
  <array>
    <string>${LLAMA_BIN}</string>
    <string>-hf</string><string>unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF:Q4_K_M</string>
    <string>--alias</string><string>qwen3-coder</string>
    <string>--host</string><string>0.0.0.0</string>
    <string>--port</string><string>8080</string>
    <string>--api-key</string><string><API_KEY></string>
    <string>--jinja</string>
    <string>-ngl</string><string>99</string>
    <string>-c</string><string>32768</string>
    <string>-fa</string><string>on</string>
    <string>--temp</string><string>0.7</string>
    <string>--top-p</string><string>0.8</string>
    <string>--top-k</string><string>20</string>
    <string>--repeat-penalty</string><string>1.05</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>${HOME}/Library/Logs/llama/server.log</string>
  <key>StandardErrorPath</key><string>${HOME}/Library/Logs/llama/server.err.log</string>
</dict>
</plist>
EOF
chmod 600 ~/Library/LaunchAgents/com.local.llama-server.plist   # it contains the API key
```

Load and manage it:

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.local.llama-server.plist   # start
launchctl kickstart -k gui/$(id -u)/com.local.llama-server                              # restart
launchctl bootout gui/$(id -u)/com.local.llama-server                                   # stop/unload
tail -f ~/Library/Logs/llama/server.err.log                                             # logs
```

(Re-run `bootout` then `bootstrap` after editing the plist.)

### 1.6 Keep the Mac awake and reachable

A sleeping Mac drops the service. For a plugged-in, lid-closed or headless server:

```bash
sudo pmset -a sleep 0 disksleep 0 displaysleep 10 powernap 0
sudo pmset -a disablesleep 1      # stay awake even with the lid closed
pmset -g | grep -E "sleep|disablesleep"
```

Also:

- Keep it **plugged into power**.
- System Settings → Users & Groups → enable **automatic login** if you want the
  LaunchAgent to start after a reboot with nobody at the keyboard (FileVault must be off
  for this; otherwise log in once after each reboot).
- Prefer Ethernet (USB-C adapter) over Wi-Fi for stability.
- To undo `disablesleep`: `sudo pmset -a disablesleep 0`.

### 1.7 Network access

Find the address clients should use:

```bash
scutil --get LocalHostName           # → modelmac, reachable as modelmac.local
ipconfig getifaddr en0               # Wi-Fi IP (en1 or another interface on some Macs)
```

Reserve a fixed IP for this Mac in your router (DHCP reservation) so `<SERVER_HOST>`
doesn't change.

If the macOS firewall is on (System Settings → Network → Firewall), allow incoming
connections for `llama-server` when prompted, or:

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add "$(which llama-server)"
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --unblockapp "$(which llama-server)"
```

Verify from the **workstation**:

```bash
curl http://<SERVER_HOST>:8080/v1/models -H "Authorization: Bearer <API_KEY>"
```

> **Security note.** This is plain HTTP with a shared key, fine on a trusted home or office
> LAN, not on public Wi-Fi. Never port-forward 8080 to the internet. For access
> outside the LAN, use Tailscale or WireGuard and point OpenCode at the tailnet name.

---

## 2. Workstation setup (OpenCode client)

### 2.1 Install OpenCode

```bash
# any one of these
curl -fsSL https://opencode.ai/install | bash
brew install sst/tap/opencode
npm install -g opencode-ai

opencode --version
```

### 2.2 Configure the remote provider

Create `~/.config/opencode/opencode.json` (global), or `opencode.json` in a project root
(per project):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "macserver": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Qwen3-Coder (MacBook server)",
      "options": {
        "baseURL": "http://<SERVER_HOST>:8080/v1",
        "apiKey": "{env:LLAMA_API_KEY}"
      },
      "models": {
        "qwen3-coder": {
          "name": "Qwen3-Coder 30B-A3B",
          "limit": {
            "context": 32768,
            "output": 8192
          }
        }
      }
    }
  },
  "model": "macserver/qwen3-coder",
  "small_model": "macserver/qwen3-coder"
}
```

Notes:

- The key under `models` (`qwen3-coder`) must equal the server's `--alias`.
- `limit.context` must not exceed the server's `-c`, otherwise OpenCode will send
  prompts the server truncates or rejects.
- `small_model` is used for titles and summaries. Without it OpenCode may try a hosted
  model you haven't configured.

Put the key in your shell profile rather than in the file:

```bash
echo 'export LLAMA_API_KEY="<API_KEY>"' >> ~/.zshrc
source ~/.zshrc
```

### 2.3 Use it

```bash
cd ~/some/project
opencode
```

Inside the TUI run `/models` and confirm **Qwen3-Coder (MacBook server)** is selected,
then try `/init` or ask a question about the repo. One-shot, non-interactive:

```bash
opencode run "explain what this repository does"
```

### 2.4 Troubleshooting

| Symptom | Check |
|---|---|
| `Connection refused` / timeout | `curl http://<SERVER_HOST>:8080/v1/models …`; server awake? firewall? same subnet? |
| `401 Unauthorized` | `LLAMA_API_KEY` doesn't match `--api-key` in the plist |
| Model doesn't call tools / edits fail | server started without `--jinja`; restart with it |
| Very slow first reply | the first request loads/warms the model; later ones are faster |
| OpenCode says context exceeded | lower `limit.context` or raise `-c` on the server |
| Server killed / output garbage | out of memory; reduce `-c`, see tuning below |
| `.local` name not resolving | use the IP address instead |

---

## 3. Tuning (after it works)

Change one thing at a time and restart the service with
`launchctl kickstart -k gui/$(id -u)/com.local.llama-server`.

### 3.1 Context window

Agentic coding eats context quickly: OpenCode's system prompt and tool definitions use
roughly 10k tokens before you type anything, and every file read and command output
adds to it. Three settings must agree:

| Setting | Where | Meaning |
|---|---|---|
| Model's native context | model card | 262,144 tokens for Qwen3-Coder-30B-A3B |
| `-c` | server (`llama-server`) | memory actually allocated. This is the real limit |
| `limit.context` | `opencode.json` | what OpenCode thinks it may send; keep ≤ `-c` |

**What a larger context costs.** The KV cache grows linearly with `-c` and is allocated
up front. For Qwen3-Coder-30B-A3B it is about 96 KB per token:

| `-c` | f16 cache | `-ctk q8_0 -ctv q8_0` | Total with Q4_K_M (18.6 GB) + ~1.5 GB | Verdict on 32 GB |
|---|---|---|---|---|
| 16384 | 1.6 GB | 0.8 GB | ≈ 22 GB | fine, but little room for OpenCode |
| **32768** | 3.2 GB | 1.6 GB | ≈ 23 GB (f16) / 22 GB (q8) | **default** |
| 65536 | 6.4 GB | 3.2 GB | ≈ 26.5 GB (f16) / 23 GB (q8) | needs the q8 cache |
| 131072 | 12.9 GB | 6.4 GB | ≈ 33 GB / 27 GB | too much |
| 262144 | 25.8 GB | 12.9 GB | does not fit | no |

(`-ctk/-ctv` need `-fa on`; the totals must stay under the GPU limit of ~26 GB from §1.4,
and ideally under ~24 GB so macOS keeps some headroom.)

To go to 64k:

```
-c 65536 -ctk q8_0 -ctv q8_0
```

Then set `"limit": { "context": 65536, "output": 8192 }` in `opencode.json`, restart the
service and check memory (Activity Monitor → Memory pressure should stay green; swap
should stay near zero). If it turns yellow/red, step back to `-c 32768`.

Practical advice:

- **Longer prompts are slower.** Prompt processing is a few hundred tokens/s on this
  hardware, so the first 50k-token prompt can take minutes. Later turns reuse the cache,
  which is why long sessions are better than many short ones.
- **Quality degrades before the limit.** Treat 32k–64k as the sweet spot; do not set a
  larger value just because the model supports it.
- **When OpenCode compacts.** It summarizes old turns when the conversation approaches
  `limit.context`; a lower limit means more frequent compaction and lost detail.
- **Out of memory on startup or mid-session** (server killed, `failed to allocate`):
  lower `-c` first, then use the q8 cache, then a smaller quant.
- **`--parallel N` divides `-c`** across slots (see §3.4).
- **Beyond 256k (YaRN to 1M)** is not useful on 32 GB.
- The model decides the cost: Qwen3.8-27B, a hybrid model, uses ~64 KB per token and
  reaches 128k within the same budget. See `hugging-models.md` §2.6 for the formula,
  per-model numbers and how to measure it from the server log.

### 3.2 Quantization trade-offs

| Quant | Size | Notes |
|---|---|---|
| `Q4_K_M` | ~18.6 GB | recommended default |
| `UD-Q4_K_XL` (Unsloth dynamic) | ~17.7 GB | similar or better quality for less size; try this first if you want to save memory |
| `Q3_K_XL` / `IQ3_XXS` | ~12–13 GB | if you need lots of context or run other apps; some quality loss |
| `Q5_K_M` / `Q6_K` | 21–25 GB | too tight on 32 GB with usable context, so avoid |
| `Q8_0` | ~32 GB | does not fit |

Switch by changing the suffix after the colon in `-hf`, e.g.
`unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF:UD-Q4_K_XL`.

### 3.3 Speed

- `-t 4` (or `-t 6`) can help: on M1 only the performance cores matter, and
  oversubscribing threads hurts. Default is usually fine.
- `-b 2048 -ub 1024` raises the prompt-processing batch sizes, so large pasted files or
  repo context ingest faster, at slightly more memory.
- Prompt caching is on by default; keep OpenCode sessions long rather than starting new
  ones so the shared prefix is reused.
- Expect roughly 30–50 tokens/s generation and a few hundred tokens/s prompt processing on
  M1; M1 Pro/Max/Ultra are faster. Measure with the `timings` block in the API response
  or the server log.

### 3.4 Concurrency

Default is 1 slot at the full context. To serve two users or sub-agents, add
`--parallel 2`; note the context is **divided** between slots (so double `-c` to keep
the same per-slot window), which costs memory. On 32 GB, stay at 1 unless you
reduce the context.

### 3.5 Sampling

The values in this doc are Qwen's recommended ones for Qwen3-Coder. If edits look
sloppy, lower `--temp` to `0.3`–`0.5`. If output loops, raise `--repeat-penalty` to
`1.1`.

### 3.6 Reduce background load on the server

Close browsers and IDEs on the model Mac, and disable Spotlight indexing of the model
cache if you see I/O spikes:

```bash
sudo mdutil -i off ~/Library/Caches/llama.cpp
```

### 3.7 Alternatives if llama.cpp doesn't suit you

- **MLX** (`pip install mlx-lm`, `mlx_lm.server --model mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit --host 0.0.0.0 --port 8080`):
  often a bit faster on Apple Silicon, but its API-key and tool-calling support is weaker
  than llama-server's, so you'd need to add auth yourself.
- **Ollama** (`brew install ollama`, `OLLAMA_HOST=0.0.0.0 ollama serve`, `ollama pull qwen3-coder:30b`):
  simplest to operate, but defaults to a small context (set `OLLAMA_CONTEXT_LENGTH=32768`),
  and has no built-in API key. Point OpenCode at `http://<SERVER_HOST>:11434/v1`.

### 3.8 Updating the model

```bash
brew upgrade llama.cpp
# change the -hf repo/quant in the plist, then:
launchctl bootout gui/$(id -u)/com.local.llama-server
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.local.llama-server.plist
# remove old models to reclaim disk
ls -lh ~/Library/Caches/llama.cpp
```

---

## 4. Quick reference

```text
Server:   launchctl kickstart -k gui/$(id -u)/com.local.llama-server
Logs:     tail -f ~/Library/Logs/llama/server.err.log
Health:   curl http://<SERVER_HOST>:8080/health
Models:   curl http://<SERVER_HOST>:8080/v1/models -H "Authorization: Bearer $LLAMA_API_KEY"
Client:   opencode   (model: macserver/qwen3-coder)
```
