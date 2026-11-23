---
created: 2026-11-23
updated: 2026-11-23
---

# Dual-Model Uncensored Setup Specification (Qwen 3.8 + Inkling 2 Mini)

## 1. Intent, Scope, and Hardware Target

- **Scope:** Bounded (two independent run configs and launcher scripts in CWD, model assets/packs in `..\Strata-data\`, separate `strata-inkling` shape component in `%LOCALAPPDATA%`; zero engine C++ or server Python code edits).
- **Primary Use Case:** Local uncensored script writing (including topics that trigger PRC government/political censorship on Chinese-aligned cloud or stock open models), with Vision support on both models. Coding/agent workloads are no longer a local requirement, so the censored `Swift 1.5 IQ2_XS` model is removed first to reclaim space.
- **Target Hardware:**
  - GPU: NVIDIA GeForce RTX 4090 (`24 GB` VRAM, Ada `sm_89`, PCIe Gen4 x16)
  - CPU: AMD Ryzen 7 3700X (`8C/16T`, Zen 2, AVX2)
  - RAM: `32 GB` DDR4 (`~25–27 GiB` free at idle boot)
  - OS: Windows 11 (`CUDA 13`)
  - Storage: `D:\` (`203.3 GB` free before cleanup), `C:\` (`215.3 GB` free)
- **Canonical Paths:** Always use `D:\workspace\github\devxplay-forks\Strata\` (CWD) and `D:\workspace\github\devxplay-forks\Strata-data\` in configs and batch scripts (`C:\Users\payne\github\` is only a symlink/junction to `D:\workspace\github\`).
- **Goals:**
  1. Keep all model weights, packed experts, tokenizers, MTP layers, and learned caches outside git in `D:\workspace\github\devxplay-forks\Strata-data\`.
  2. Maintain two independent gitignored config + launcher pairs in CWD (`strata-*.json` + `run-*.bat`, ignored via `.gitignore:41–46`) so each model runs, tunes, and calibrates without touching the other.
  3. Share port `8081` on `0.0.0.0` (Tailscale access, protected by `api_key: "sk-strata-local"`, one server active at a time) so writing clients keep a single endpoint.
- **Non-Goals / Anti-Goals:**
  - Do not run both models concurrently on one `24 GB` GPU.
  - Do not mix binaries into tracked `engine/` source or store large weights inside the repo.
  - Do not commit `.json` / `.bat` / `.log` files; document tuned invariants in `MACHINE_OPTIMIZATION_NOTE.md`.

---

## 2. Approaches Considered and Decisions

### 2.1 Uncensored Qwen 3.8 (`Model 1`, Chinese + Deep Knowledge)

| Approach | Fit on 32 GB RAM | Decision | Rationale |
| --- | --- | --- | --- |
| **`SC117/Qwen3.8-Flash-Next-GSQ-RCO-abliterated-GGUF` (`IQ2_XS`, `68.47 GB`)** | Yes (`~19.5 GiB` RAM + `~13 GiB` VRAM + `28.8 GB` SSD lookup) | **Selected** | Byte-level transplant of `144` `orcarouter` abliterated tensors onto `ISTA-DASLab` GSQ-RCO (`blake2b` verified, preserves GSQ scales and `104.4 tok/s` speed; [SC117 HF](https://huggingface.co/SC117/Qwen3.8-Flash-Next-GSQ-RCO-abliterated-GGUF), [lcz.me/2079](https://lcz.me/topic/2079)). Highest quality tier without SSD expert thrashing. |
| `Swift 1.5 IQ2_XS` + `alesha-pro` `480 KB` `cvec` | Yes | Rejected | Vector derived from base `Huihui` weights, not `Swift 1.5` RL/OPD weights; user explicitly prefers full baked uncensored replacement over patch workaround. |
| `SC117` / `orcarouter` `IQ3_XXS` or `AtomicChat` `IQ4_XS` (`47–53 GiB` Shard 1) | No (`>36 GiB` resident needed) | Rejected | Exceeds `32 GB` RAM + `24 GB` VRAM, forcing slow SSD expert streaming (`~15–20 tok/s`). |

### 2.2 Uncensored Inkling 2 Mini (`Model 2`, English + `1M` Context + Zero CCP Alignment)

| Approach | Fit on 24 GB VRAM + 32 GB RAM | Decision | Rationale |
| --- | --- | --- | --- |
| **`nokeepai/Inkling-2-Mini-Unbound-GGUF` (`v2-Q4_K_M.gguf`, `17.78 GB` + `Inkling-2-Mini-Vision.strata` `0.64 GB`)** on `strata-inkling 0.1.66` | Yes (`100%` experts in VRAM; `~4–12.75 GiB` mapped host KV via `k8v4` auto-clamped to free RAM) | **Selected (Executed)** | US-trained (`0.3%` PRC framing vs Qwen `22%`; [Nous eval](https://pasqualepillitteri.it/en/news/31090/arete-27b-nous-research-qwen-western-values)). `78.7–83.4%` non-dodging compliance ([Unbound HF](https://huggingface.co/nokeepai/Inkling-2-Mini-Unbound)), `0.0346` KLD (`95.1%` top-1), and `GGUF-Q4_K` CUDA kernels ran cleanly at `98.9 tok/s` on `sm_89` (`HF-NVFP4-MXFP8` `safetensors` passed `--cpu` but returned gibberish `token 123268` on `0.1.66` `sm_89` CUDA). |
| `nokeepai/Inkling-2-Mini-Unbound` (`safetensors` `HF-NVFP4-MXFP8`, `19.5 GB`) | Yes (`100%` VRAM) | Rejected (0.1.66 CUDA bug) | Coherent on `--cpu`, but `strata-inkling 0.1.66` `sm_89` CUDA output gibberish on `HF-NVFP4-MXFP8` (`32K fp16` & `1M k8v4`) and lacked `dspark.` prefix in `ink_safetensors_loader.h:412`. |
| Stock `Inkling-2-Mini-SQ-20` / `AtomicChat` / `Heretic` | Yes | Rejected | Stock `SQ-20` refuses `96%` of sensitive prompts; `AtomicChat` (`51.6–58.8%`) and `Heretic` (`17.3–26.7%`) trail `Unbound v2` (`78.7–83.4%`). |

---

## 3. Target Architecture & Resource Budget

### 3.1 File Layout

```text
D:\workspace\github\devxplay-forks\
├── Strata\                                   (git repo, clean working tree)
│   ├── MACHINE_OPTIMIZATION_NOTE.md          (git-tracked host optimization log)
│   ├── docs\specs\2026-11-23-dual-model-uncensored-setup.md
│   ├── strata-sc117-iq2_xs.json              (gitignored, Model 1 config)
│   ├── run-sc117-iq2_xs.bat                  (gitignored, Model 1 launcher -> :8081)
│   ├── strata-inkling-unbound.json           (gitignored, Model 2 config)
│   └── run-inkling-unbound.bat               (gitignored, Model 2 launcher -> :8081)
└── Strata-data\                              (outside git, ~89–134 GB total after Phase 1+2)
    ├── models\
    │   ├── sc117-IQ2_XS\                     (69.38 GB: IQ2_XS shards 1 & 2 + mmproj BF16)
    │   └── inkling-2-mini-unbound-gguf\      (18.42 GB: v2-Q4_K_M.gguf + Inkling-2-Mini-Vision.strata)
    ├── packs\
    │   ├── sc117-iq2_xs\                     (~37 GB: packed experts + dense + tokenizer)
    │   └── inkling-2-mini-unbound\tokenizer\ (~26 MB: exported o200k tokenizer + chat_template)
    ├── mtp\rt\                               (6.93 GB: reused Qwen MTP runtime)
    └── expert-profile-learned.bin            (196 KB: learned Qwen expert profile)
```

Runtime engine binaries live in `%LOCALAPPDATA%\Strata\components\`:
- Qwen engine: `engine\0.1.66-windows-x86_64-cuda-cuda13\strata.exe` + `strata-vision.exe` (already installed)
- Inkling shape engine: `shapes\inkling2\engine\0.1.66-windows-x86_64-cuda-cuda13\strata-inkling.exe` (`417 MB`, installed in Phase 2 without modifying `strata.exe`)

### 3.2 Model Configurations & Memory Contracts

| Parameter | Model 1: `SC117 IQ2_XS` | Model 2: `Inkling 2 Mini Unbound v2` |
| --- | --- | --- |
| `exe` / `arch` | `0.1.65\strata.exe` (`v0.2.19` binary under `0.2.20` codebase; `0.1.66` hits non-`#6704` `0xC0000005` on `SC117`) | `"component:strata-inkling"` (`0.1.66`) / `"inkling2_loop"` |
| Context (`--max-context`) | `262144` (`256K`) | `1048576` (`1M` requested, auto-clamped by `ink_runner.cuh` to `~571,520` on `32 GB` RAM) |
| KV Cache (`--kv`) | `k8v4` | `k8v4` (`12.75 KiB/tok` global via `--kv-offload auto`) |
| Vision | CPU `0.1.65\strata-vision.exe` + `mmproj-Qwen3.8-Flash-Next-BF16.gguf` (`0.91 GB`) | In-process `--vision` pointing to `Inkling-2-Mini-Vision.strata` (`0.64 GB` lossless BF16) |
| Speculative Draft | `--spec 4 --mtp ...\mtp\rt` (`--spec-min-p 0.70`, `--pcie-frac 0.35`, `--adapt-every 1 --adapt-swaps 80 --adapt-decay 0.97`) | `"drafting": "off"` (external `dspark` lacks `dspark.` tensor prefix in `0.1.66`) |
| Measured Speed (RTX 4090) | **`102.5–117.7 tok/s`** (`0.1.65` `CAL.run`, `981 s`) | **`98.9 tok/s`** decode, `534.0 ppts` prefill (`--bench`) |

### 3.3 Critical Invariants & Risks

1. **China-Gov Political Framing on `SC117` (`arXiv:2610.29342v2`):** Weight abliteration drops `SC117 IQ2_XS` refusals to `0/9`, but residual PRC political framing persists (`115/243` scripted openings) without a system prompt. **Mitigation:** Configure a short explicit neutral/uncensored script-writing system prompt in the client (`+0.93..+0.97` score in paper).
2. **`SC117` Thinking Length:** `SC117` uses base `Qwen3.8-Flash-Next` weights (2–3× longer thinking traces than `Swift 1.5`). **Mitigation:** Set thinking `off` or `low` for standard script drafting.
3. **`SC117` Pack Isolation:** Never reuse `packs\swift-iq2_xs\native_experts.txt` (`ArenaExpertSource` mismatch). Always run `terminal/tools/iq_pack.py` on `SC117` shards.
4. **Inkling Split-Flag Trap (`Niko1221/Strata#6716`):** Do **not** include `--vram-experts-gb` or `--ram-experts-gb` in `strata-inkling-unbound.json`, ensuring `ink_runner.cuh` keeps `100%` of experts in VRAM (`15.68 GB`).
5. **Inkling `sm_89` DSpark & Chinese Limits:** Test `--draft dspark` vs `--draft off` on `sm_89` (`#6716` showed `dspark` slowed prose on `sm_86`). Use Inkling primarily for English scripts; its model card warns of clipped phrasing and mixed languages in Chinese.

---

## 4. Phased Implementation Plan (Next Execution Session)

### Phase 1: Uncensored Qwen (`SC117 IQ2_XS`) First

1. **Pre-cleanup (reclaim `~106 GB` before download):**
   - Remove `D:\workspace\github\devxplay-forks\Strata-data\models\swift-IQ2_XS\`, `models\mmproj-Swift-Qwen3.8-Flash-Next-BF16.gguf`, `packs\swift-iq2_xs\`, and root `strata-swift-iq2_xs.json`, `run-swift-iq2_xs.bat`, `strata-swift-iq2_xs.log`.
   - Keep `D:\workspace\github\devxplay-forks\Strata-data\mtp\rt\` (`6.93 GB`) and `expert-profile-learned.bin` (`196 KB`, or reset if re-profiling).
2. **Download `SC117` `IQ2_XS` (`69.38 GB`):**
   - Download `IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-abliterated-IQ2_XS-00001-of-00002.gguf` (`39.67 GB`), `-00002-of-00002.gguf` (`28.80 GB`), and `mmproj-Qwen3.8-Flash-Next-BF16.gguf` (`0.91 GB`) from `SC117/Qwen3.8-Flash-Next-GSQ-RCO-abliterated-GGUF` into `D:\workspace\github\devxplay-forks\Strata-data\models\sc117-IQ2_XS\`.
3. **Pack & Configure:**
   - Run `python terminal\tools\iq_pack.py --gguf D:\workspace\github\devxplay-forks\Strata-data\models\sc117-IQ2_XS\Qwen3.8-Flash-Next-GSQ-RCO-abliterated-IQ2_XS-00001-of-00002.gguf --out D:\workspace\github\devxplay-forks\Strata-data\packs\sc117-iq2_xs`.
   - Create `strata-sc117-iq2_xs.json` (`"exe": "component:strata-engine"`, `--max-context 262144`, `--kv k8v4`, `--vision`, CPU `0.1.66\strata-vision.exe`, `--spec 4 --mtp ...\mtp\rt`, `"port": 8081`, `0.1.66` opt-in `"env"` flags) and `run-sc117-iq2_xs.bat`.
4. **Calibrate & Verify:**
   - With `>= 25.8 GiB` free RAM on idle PC, run `terminal/tools/calibrate.py` (`CAL.run(...)` with `"component:strata-engine"` resolved in-memory and `CAL.apply(args, {})` baseline) and write winning `--pcie-frac`, `--spec-min-p`, and `--adapt-*` flags into `strata-sc117-iq2_xs.json`.
   - Launch `.\run-sc117-iq2_xs.bat`, verify Golden Health lines, test `/v1/chat/completions` on `http://127.0.0.1:8081/v1` (Chinese uncensored script + neutral system prompt + Vision image check).

### Phase 2: Uncensored Inkling 2 Mini (`Unbound v2`) Second

1. **Install `strata-inkling` (`0.1.66`):**
   - Fetch `strata-inkling` (`windows-x86_64-cuda-cuda13`, `417 MB`) via `strata_control.shape_engine.ensure('inkling2', ...)` into `%LOCALAPPDATA%\Strata\components\shapes\inkling2\engine\0.1.66-windows-x86_64-cuda-cuda13\strata-inkling.exe` and verify `--version`.
2. **Download `nokeepai/Inkling-2-Mini-Unbound` v2 (`~20.2 GB`) & Export Tokenizer:**
   - Download `model-00001-of-00001.safetensors` (`19.5 GB`), `mtp.safetensors` (`0.64 GB`), `config.json`, `processor_config.json`, `special_tokens_map.json`, `tokenizer.json`, `tokenizer_config.json`, and `chat_template.jinja` into `D:\workspace\github\devxplay-forks\Strata-data\models\inkling-2-mini-unbound\`.
   - Export tokenizer directory to `D:\workspace\github\devxplay-forks\Strata-data\packs\inkling-2-mini-unbound\tokenizer\` via `terminal/tools/inkling_tokenizer.py`.
3. **Configure & Benchmark:**
   - Create `strata-inkling-unbound.json` (`"exe": "component:strata-inkling"`, `"arch": "inkling2_loop"`, `"port": 8081`, `"drafting": "dspark"`, `--native ...\model-00001-of-00001.safetensors`, `--vision ...\model-00001-of-00001.safetensors`, `--dspark ...\mtp.safetensors`, `--max-context 1048576`, `--kv k8v4`, `--kv-offload auto`, omitting `--vram-experts-gb` / `--ram-experts-gb`) and `run-inkling-unbound.bat`.
   - Run `strata-inkling.exe --bench` A/B (`--draft dspark` vs `--draft off`, `--kv k8v4` vs `--kv fp8`) and update `"drafting"` to whichever wins on `sm_89`.
4. **Verify & Document:**
   - Stop Qwen (`Ctrl+C`), launch `.\run-inkling-unbound.bat`, verify `1M` (or WDDM-clamped max) context + English uncensored script + Vision on `http://127.0.0.1:8081/v1`.
   - Update `MACHINE_OPTIMIZATION_NOTE.md` with both finalized model configs, measured speeds, and switching workflow.
