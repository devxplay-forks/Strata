# Machine Profile & Performance Setup Guide

> **Hardware Baseline** — the engine version currently in use, and every measurement behind these settings, are recorded in **§8 Version Log**.
>
> - **GPU:** NVIDIA GeForce RTX 4090 (24 GB VRAM, PCIe Gen4 x16)
> - **CPU:** AMD Ryzen 7 3700X (8 Cores, 16 Threads, Zen 2, AVX2)
> - **RAM:** 32 GB DDR4
> - **OS:** Windows 11 (CUDA 13 environment)
> - **Target Model:** `swift` (Swift 1.5) / `IQ2_XS` (2-bit i-quant)

---

## 1. Quick Re-setup Commands

When re-installing from scratch, switching branches, or rebuilding on this machine, follow these steps in order:

### Step 0: Mandatory OS-Level Prerequisite — Virtual Memory (Pagefile)

> **Why this is mandatory:** On Windows with 32 GB RAM, the default commit limit (~42 GB) is too small. Strata requires ~55–65 GB of commit space to allocate its ~19 GB page-locked resident RAM expert complement. Without this, Strata refuses resident RAM and falls back to slow SSD mmap (dropping speed to 17–32 tok/s). **Configure this first before installing models or running scripts.**

#### Step-by-Step Configuration Guide (English / 中文輔助)

1. Press `Win + R`, type `sysdm.cpl` and press Enter.
2. Navigate to **Advanced** tab -> under **Performance**, click **Settings...**
   _(中文：切換至 **「進階」** 分頁 -> 在 **「效能」** 區塊點擊 **「設定...」**)_
3. Navigate to **Advanced** tab -> under **Virtual memory**, click **Change...**
   _(中文：切換至 **「進階」** 分頁 -> 在 **「虛擬記憶體」** 區塊點擊 **「變更...」**)_
4. **Uncheck** "Automatically manage paging file size for all drives".
   _(中文：**取消勾選**「自動管理所有磁碟分頁檔案大小」)_
5. Select your fastest NVMe SSD drive (e.g. `C:` or `D:`), choose **Custom size**:
   _(中文：選取最快的 NVMe SSD 磁碟機代號，選擇 **「自訂大小」**)_
   - **Initial size (MB) / 起始大小 (MB):** `32768` (32 GB)
   - **Maximum size (MB) / 最大值 (MB):** `65536` (64 GB)
6. **CRITICAL:** Click the **Set** button! (The numbers in the drive list above will update).
   _(中文：**務必點擊** 右側的 **「設定」** 按鈕！確認磁碟機清單內的數字更新)_
7. Click **OK** -> **OK** -> **OK**, then restart your computer.
   _(中文：依序點擊 **「確定」**，並依提示重新啟動電腦)_

---

### Step 1: Initial Setup

```powershell
.\START-HERE.bat --family swift --model IQ2_XS --context 262144 --kv k8v4 --vision cpu --draft-vocab en --vram-reserve-mib 1600 --port 8081 --yes --no-start
```

### Step 2: Custom Optimizations (Manual JSON Edit)

Open `strata-swift-iq2_xs.json` and ensure these entries are present:

1. At top-level:
   ```json
   "expert_profile_save": "data/expert-profile-learned.bin",
   "expert_profile_save_every": 10,
   ```
2. In `"env"`:
   ```json
   "env": {
     "STRATA_PF_FUSED": "1",
     "STRATA_RESIDENT_HEADROOM_GIB": "6",
     "STRATA_SPEC_COUPLED": "1",
     "STRATA_PIN_BY_NEED": "1",
     "STRATA_PCIE_MIN1": "1",
     "STRATA_SPEC_GUMBEL": "1",
     "STRATA_RAM_ADAPT": "2",
     "STRATA_RAM_ADAPT2": "1",
     "STRATA_DEQUANT_WIDE": "1",
     "STRATA_PROMPT_ATTN_IMMA": "1",
     "STRATA_ATTN_DAILY": "1",
     "STRATA_GU_SWIGLU": "1",
     "STRATA_GR_UP75": "1",
     "STRATA_QSA_ROWS": "1",
     "STRATA_SIDE_MIXER": "2"
   }
   ```
3. In `"args"` (append to list):
   ```json
   "--mtp-window",
   "8192",
   "--adapt-async",
   "1",
   "--host-core",
   "sibling",
   "--prompt-cache-spread"
   ```

### Step 3: Hardware Calibration

> **Precondition — do not skip this.** A calibration is only valid if the machine is idle. The resident complement equals `available RAM − 6.25 GiB` (capped at 19.54), so a boot with other programs open measures a **different machine** and can pick the wrong values. **Require `>= 25.8 GiB` free RAM** (= 19.54 + 6.25) before running. A run below that threshold has produced a different `--pcie-frac` on this same machine; the two runs are compared in §8.

```powershell
.\START-HERE.bat --calibrate --family swift --model IQ2_XS --no-start
```

_(Sweeps the PCIe share, the draft floor, the CPU worker count and the adaptive tier. Takes ~10–30 min and 8–20 engine restarts. It applies values as a **per-request API parameter** (`tools/calibrate.py:210`, `sampling["strata_tune"]`), so the swept values never appear in the `engine started` log line — do not grep for them there. A value is kept only if it beats the engine's own default by more than 3% (`MIN_GAIN`).)_

### Step 4: Run Server

```powershell
.\START-HERE.bat --host 0.0.0.0 --api-key sk-strata-local --port 8081
```

---

## 2. Daily Routine & Verification (After Reboots)

### Daily Startup Command

You do **NOT** need to rerun Setup (Step 1) or Calibration (Step 3) after reboots. Simply start the server:

```powershell
.\START-HERE.bat --host 0.0.0.0 --api-key sk-strata-local --port 8081
```

### Golden Health Verification Checklist

When the server boots up, check the terminal / `strata-swift-iq2_xs.log` for these 4 golden lines:

1. `resident RAM mode: 19.5x–20.5x GiB of experts in RAM (locked)` — confirms the resident experts are locked in RAM with 0 SSD file stalls. **The number is not fixed:** it equals `available RAM − 6.25 GiB` (6 GiB `STRATA_RESIDENT_HEADROOM_GIB` + 0.25 margin), capped at 19.54 GiB. At ~26 GiB free you get the full ~20.5 GiB; at ~22 GiB free only ~15.7 GiB, and decode drops with it. **If you see 15.x, another program is holding RAM — close it and restart before judging speed.** This is RAM pressure, not an engine fault; the matched-RAM proof is in §8.
2. `expert cache 10068 slots, 13.48 GiB of VRAM` — confirms GPU VRAM cache is fully populated (10067 or 10068; the draft head moves it by one).
3. `the host thread on 1 (draining too) (--host-core sibling)` — confirms the host scheduler moved to Core 0 Thread 1, leaving Thread 0 free for Windows 11 GPU driver interrupts.
4. `--expert-profile data/expert-profile-learned.bin` — confirms your real-world coding expert routing distribution is pre-loaded.

### Config-Change Verification (Prompt Reuse / Conversation Cache)

After changing any `--prompt-cache*` flag, confirm it reached the engine and measure the effect:

1. **Flag arrived:** `Select-String -Path strata-swift-iq2_xs.log -Pattern "engine started" | Select-Object -Last 1` — the line must end with `--prompt-cache-spread`.
2. **Zero-reuse rate — the flag does not move this number; keep it anyway.** Count `= 0 reused` against the total `strata serve: prompt` lines. `--prompt-cache-spread` does **not** lower it on this machine, because the misses are dominated by **conversation switches** and cold starts, not the history-rewrite case the flag targets. The root cause is `--prompt-cache 6` being **one global checkpoint chain shared across all conversations** — working in several chats prunes each other’s checkpoints. Fixing that needs conversation parking, which needs 64 GB RAM (§5). Measurements are in §8.
3. **Decode must not move:** 105–127 tok/s. Spread changes checkpoint retention only, never the decode path.

---

## 3. What to NEVER Touch (Traps & False Optimizations)

When researching forums, GitHub issues, or AI suggestions months later, avoid these 7 known traps:

- ❌ **Do NOT enable `--kv-resident`**: On a 32 GB RAM machine, offloading KV cache to host RAM steals budget from resident experts, triggering massive SSD re-reads.
- ❌ **Do NOT set `--conversation-cache-mib` > 0**: Parking a conversation needs its full snapshot (**1,036–3,124 MiB** at 100K–187K context) _plus_ the 2,560 MiB floor to fit in free RAM _at park time_. This machine has only **121–1,464 MiB** free then (19 GiB of experts locked + 6 GiB headroom), so every park is refused — one log shows `exceeds available budget` ×192 and `evicted 0, 0 still parked` ×49. Only 149–547-token conversations ever parked. Revisit only after the 64 GB upgrade.
- ❌ **Do NOT hunt for a conversation-cache idle timeout**: There is none. `idle|ttl|expire` matches nothing in `serve/` or `include/`, and every refusal line reads `evicted 0` — nothing was ever cleared because nothing was ever stored. Waiting 10 minutes vs 10 hours changes nothing; reuse is lost when the client **rewrites its history** so no exact token-prefix match survives. `--prompt-cache-spread` is the fix for that, not a timer setting.
- ❌ **Do NOT set `--prefill auto:32768`**: Official repo benchmarks (#834 #669) prove 32K prompt chunks run **~3x slower** on 32 GB RAM. Keep `--prefill auto` (8,192 chunk).
- ❌ **Do NOT switch to `--draft dflash2`**: Requires an unbuilt model pack and consumes 1.2 GB VRAM; MTP with Gumbel noise already achieves 78%–88% acceptance on this card.
- ❌ **Do NOT raise power limit to 510W (`nvidia-smi -pl 510`)**: Decode consumes only ~266W on the RTX 4090; raising power adds heat and fan noise with 0.0 tok/s gain.
- ❌ **Do NOT run `UPDATE.bat` expecting a newer engine**: `setup.py:5371` returns early whenever the installed engine is `>= MIN_ENGINE`, which it already is on this machine, so `UPDATE.bat` prints "already installed" and fetches nothing. Use the §7 procedure instead.

## 4. Why Each Setting Was Chosen

| Setting                                | Value                          | Why / Benefit                                                                                                                                                                                                                                                                              |
| -------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--vision cpu`                         | CPU reader (8 threads)         | **Frees ~1.4 GB VRAM**. Offloading vision to CPU allows the GPU to maximize expert slots.                                                                                                                                                                                                  |
| `--draft-vocab en`                     | English/code head              | **Saves ~110 MiB VRAM & +1-2% decode**. Drops unneeded CJK tokens from draft head.                                                                                                                                                                                                         |
| `--vram-reserve-mib 1600`              | Fixed VRAM headroom            | **Eliminates WDDM page eviction cliff**. Prevents Windows 11 DWM from dropping speed from 50 tok/s to 2 tok/s on deep 100K-262K context.                                                                                                                                                   |
| `--kv k8v4`                            | Mixed INT8 K + 4-bit V         | **Halves VRAM consumption for KV cache** over standard INT8, allowing full 262,144 context without blowing out 24 GB VRAM.                                                                                                                                                                 |
| `context 262144`                       | Exact trained window           | Native context length; avoids RoPE scaling / YaRN degradation.                                                                                                                                                                                                                             |
| `STRATA_PF_FUSED=1`                    | Fused int8 prompt kernels      | **+12% prompt speed at 4K, +3% at 32K** (`DETAILS.md:26-27`) by running prompt experts on tensor-core GEMMs.                                                                                                                                                                               |
| `--mtp-window 8192`                    | Bounded MTP attention          | Caps attention span during speculative draft verification to keep decode overhead low at long contexts.                                                                                                                                                                                    |
| `--adapt-async 1`                      | Async expert swapping          | **+7% to +11% decode speed** (`DETAILS.md:202`) by asynchronously overlapping RAM-to-VRAM expert swaps with token decoding.                                                                                                                                                                |
| `--host-core sibling`                  | SMT host core isolation        | Moves host scheduler to Core 0 SMT sibling (Logical Processor 1), leaving Processor 0 free for Windows 11 GPU driver interrupts (+3-5%).                                                                                                                                                   |
| `STRATA_PIN_BY_NEED=1` / `PCIE_MIN1=1` | PCIe miss activation           | Fixes `pcie_share 0.0` symptom; forces PCIe lane to activate even for low miss counts (+5-7% prompt speed).                                                                                                                                                                                |
| `STRATA_RESIDENT_HEADROOM_GIB=6`       | RAM headroom guard             | Prevents Windows 11 working set memory trimming on 32 GB RAM systems.                                                                                                                                                                                                                      |
| `STRATA_SPEC_COUPLED=1`                | Coupled draft sampling         | Couples speculative draft RNG with target model distribution, raising draft acceptance rates to 68-75%.                                                                                                                                                                                    |
| `--pcie-frac 0.75`                     | Empirical PCIe share           | Swept by calibration on an idle machine; 0.75 beat every other value, including 1.00. Full medians, and the two superseded RAM-starved picks, are in §8.                                                                                                                                   |
| `--spec-min-p 0.70`                    | Acceptance threshold           | Swept by calibration; the highest value in the grid won. Measurements in §8.                                                                                                                                                                                                               |
| `--adapt-swaps 160`                    | Adaptive tier swaps            | Confirmed by calibration: the adaptive tier is worth roughly +10% over leaving it off, and 160 swaps beat 80. Measurements in §8.                                                                                                                                                          |
| `STRATA_SPEC_GUMBEL=1`                 | Gumbel-Max noise coupling      | Replaces softmax draft coupling with Gumbel-max noise matching, raising speculative draft acceptance rates with zero VRAM/RAM cost.                                                                                                                                                        |
| `STRATA_RAM_ADAPT=2`                   | Dynamic resident RAM sync      | Swaps out dormant experts in the 18.22 GiB RAM pool with active conversation experts; cuts repeated SSD reads per token from 14 to ~2.7.                                                                                                                                                   |
| `STRATA_RAM_ADAPT2=1`                  | Async RAM promotion I/O        | Stages RAM expert promotions in the background on a dedicated I/O thread, avoiding GPU decode stalls during RAM updates.                                                                                                                                                                   |
| `STRATA_DEQUANT_WIDE=1`                | Ada wide dequantization        | Bypasses hardcoded `sm_86` default to enable 16-bit loads / 16-byte stores on RTX 4090 (Ada sm_89); cuts **11% to 15% off TTFT**.                                                                                                                                                          |
| `STRATA_PROMPT_ATTN_IMMA=1`            | INT8 Tensor Core Attention     | Routes `--kv k8v4` attention matrix products directly into Ada 4th-Gen INT8 Tensor Cores; **+2.5% prompt prefill throughput**.                                                                                                                                                             |
| `STRATA_ATTN_DAILY=1`                  | Warp-synchronous decode attn   | Replaces upstream chunked decode attention with warp-synchronous reduction kernel (+1.5% to +3% decode).                                                                                                                                                                                   |
| `STRATA_GU_SWIGLU=1`                   | Fused gate/up + SwiGLU launch  | Fuses gate/up projection, SwiGLU activation, and Q8_1 quantization into a single CUDA kernel launch, cutting kernel launch overhead.                                                                                                                                                       |
| `STRATA_GR_UP75=1`                     | Staged group-reduction up-proj | Optimizes multi-token verification window group-reduction up-projection via sm_75+ tensor-core staging (+1% to +1.5% decode).                                                                                                                                                              |
| `STRATA_QSA_ROWS=1`                    | Low-query QSA block scores     | Optimizes QSA block score calculations for small query counts (NQ <= 4), reducing warp stalls during expert routing.                                                                                                                                                                       |
| `STRATA_SIDE_MIXER=2`                  | Async side mixer execution     | Runs GDN gate projections and QSA indexing asynchronously on a secondary CUDA stream alongside layer projections.                                                                                                                                                                          |
| `expert_profile_save`                  | Learned profile persistence    | Automatically saves the active expert routing table to `data/expert-profile-learned.bin` every 10 min and on exit; future launches load this file to eliminate the cold-start warmup ramp.                                                                                                 |
| `--prompt-cache-spread`                | Geometric checkpoint spacing   | Keeps the 6 checkpoints thinned geometrically back from the end instead of clustered in the last ~2K tokens. An agent that rewrites history N tokens back re-reads ~N instead of the whole prompt (6K back: 8K not 200K; 42K back: 43K not 200K). **Zero extra RAM** — same 6 checkpoints. |

---

## 5. Performance Ceiling & Hardware Upgrade Guide

### Real-World Measured Performance (Cycle 11 Verified)

- **Prompt Ingestion:** ~1,400 – 1,870 tok/s
- **Sustained Decode:** ~105 – 120 tok/s across deep contexts
- **Tested Maximum Context:** **197,583 tokens (~200K)** with zero memory errors or throttling.

### Why is speed bounded at ~110–120 tok/s?

- **32 GB RAM Hardware Limit:** Swift IQ2_XS requires ~33–36 GB to hold all experts in memory. On 32 GB RAM, ~19 GB is locked in RAM and ~13.5 GB is in VRAM. The remaining uncached tail experts stream over DDR4 dual-channel memory (~25.6 GB/s).
- **The Only Hardware Upgrade That Breaks This:** Upgrading to **64 GB RAM (2×32GB DDR4)** will allow 100% of the model weights to stay resident in system RAM, completely eliminating the SSD file tier and boosting decode throughput to **120–150+ tok/s**.

---

## 6. Resizable BAR (ReBAR) Status & BIOS Enable Guide

_(One-time motherboard BIOS setup: expands GPU BAR1 window from 256 MiB to 32 GB so CPU addresses full 24 GB VRAM directly)._

- **Hardware Status:** BAR1 Total is now **32,768 MiB (32 GB)** — **Resizable BAR is ACTIVE and VERIFIED**!
- **Motherboard Identified:** `Gigabyte X570 AORUS ELITE` (BIOS Version: `F32`).
- **Benefit Realized:** The CPU can now address the entire 24 GB VRAM in a single address space without 256 MiB window switching overhead.

### Exact Step-by-Step BIOS Guide for Phone/Video Call Guidance

When guiding over a phone or video call, walk through these exact screens and keys:

1. **Restart the PC and Enter BIOS:**
   - Restart Windows. As soon as the screen goes black, press the **Delete** key repeatedly until the Gigabyte AORUS BIOS screen appears.
2. **Switch to Advanced Mode:**
   - If the screen is in "Easy Mode", press **F2** on the keyboard to switch to **Advanced Mode** (上方會顯示分頁標籤).
3. **Step A: Disable CSM Support (Boot Tab):**
   - Use the arrow keys or mouse to click the **Boot** tab (中文：**開機**).
   - Find **CSM Support** (中文：**CSM 支援**).
   - Change it to: **`Disabled`** (中文：**關閉**).
     _(Note: Because the C: drive is GPT, disabling CSM is completely safe and required for ReBAR)._
4. **Step B: Enable Above 4G & Re-Size BAR (Settings Tab):**
   - Click the **Settings** tab (中文：**設定**).
   - Select **IO Ports** (中文：**I/O 埠**).
   - Find **Above 4G Decoding** (中文：**大於 4G 解碼**):
     - Change to: **`Enabled`** (中文：**開啟**).
   - Once Above 4G is Enabled, the hidden option **Re-Size BAR Support** (中文：**Re-Size BAR 支援**) will appear right below it:
     - Change to: **`Auto`** (or **`Enabled`**).
5. **Step C: Save and Reboot:**
   - Press **F10** on the keyboard (Save & Exit Setup / 儲存並離開).
   - Select **Yes** (是) and hit Enter. The PC will reboot into Windows.
6. **Verification in Windows:**
   - In Windows, open NVIDIA Control Panel -> click **Help** (說明) -> **System Information** (系統資訊).
   - Look for **Resizable BAR**: it will now show **`Yes`** (是).

---

## 7. Upgrading the Engine (Procedure)

This section is the **procedure only** — it works for any future version. What was measured on each actual
upgrade is recorded in **§8 Version Log**.

### 7.1 Why the two versions drift

`git pull` updates the **source code only**. It never touches the engine binary: `engine/` is gitignored
(`.gitignore:50`). Source and binary therefore move independently, and that is normal. Always check both:

| What       | How to check                                   | Meaning                         |
| ---------- | ---------------------------------------------- | ------------------------------- |
| Source     | `grep 'project(strata VERSION' CMakeLists.txt` | what the Python and docs expect |
| **Binary** | `.\engine\strata.exe --version`                | **what actually runs**          |

The binary is the one that matters for performance. A source tree newer than the binary changes nothing until
the binary is replaced.

### 7.2 Find out what is available

```powershell
python strata.py update --no-control --check
```

Read-only; downloads nothing. In the JSON output, `engine.offered.version` is the newest package for this PC
and `engine.installed` is what the updater currently manages. Two things to expect:

- `engine.installed: null` on a source checkout — normal, see §7.4.
- The newest _source_ version may be **higher** than the newest _engine package_. Engine packages are
  published per family; a version bump can apply to one family and not another. Trust `offered`, not
  `CMakeLists.txt`.

### 7.3 Procedure

```powershell
# 0. Stop the server — an update refuses while one is serving (exit code 7)
taskkill /PID <server-pid> /T /F

# 1. Record the old version, then back up the engine AND the tuned config
$old = (Get-Content engine\BUILD.json -Raw | ConvertFrom-Json).version
Rename-Item engine "engine-$old-backup"
Copy-Item strata-swift-iq2_xs.json strata-swift-iq2_xs.json.bak

# 2. Download, verify, test-start into the components store
python strata.py update --no-control

# 3. See which package folder the store now holds
Get-ChildItem "$env:LOCALAPPDATA\Strata\components\engine" -Directory | Select-Object Name

# 4. Copy it into engine\   (PowerShell, NOT cmd: %LOCALAPPDATA% does not expand there)
$src = "$env:LOCALAPPDATA\Strata\components\engine\<folder-from-step-3>"
New-Item -ItemType Directory -Force -Path engine | Out-Null
Copy-Item "$src\*" engine\ -Recurse -Force

# 5. Fix BUILD.json: the store copy says "source": "<branch>"; setup needs "prebuilt"
$b = Get-Content engine\BUILD.json -Raw | ConvertFrom-Json
$b.source = "prebuilt"
$b | ConvertTo-Json -Depth 8 | Set-Content engine\BUILD.json -Encoding utf8

# 6. Verify the binary really changed (must print the NEW version)
.\engine\strata.exe --version

# 7. Re-calibrate — the engine identity changed (see the Step 3 precondition)
.\START-HERE.bat --calibrate --family swift --model IQ2_XS --no-start
```

**Why step 5 is required:** `setup.py:5182` treats an engine whose `BUILD.json` does not say
`"source": "prebuilt"` as one compiled on this machine, and will try to rebuild it.

### 7.4 Why the copy in step 4 cannot be skipped

`strata update` installs into `%LOCALAPPDATA%\Strata\components\`, which **this config never reads** — its
`"exe"` points at this checkout's `engine\strata.exe`. That is why `--check` reports `"installed": null`.

`setup.py` does contain a bridge that performs the copy automatically (`engine_from_feed`,
`setup.py:4353-4401`), but it is unreachable: `get_prebuilt` returns early at `setup.py:5191` whenever the
installed engine is `>= MIN_ENGINE`. There is no flag to force it, so the copy stays manual.

### 7.5 Never use Step 1 Setup to "update"

Setup rewrites `"args"` from its own answers and would reset `--spec-min-p` back to its own value (it is in
`SETUP_CHOICE_FLAGS`, `setup.py:6311`). Other hand edits survive via `user_args` only because they appear
**0 times** in `setup.py`. Use calibration, not setup.

### 7.6 Rollback

Keep `engine-<oldver>-backup\` until the new engine has run one full real session. To go back: stop the
server, `Rename-Item engine "engine-<newver>"`, then `Rename-Item "engine-<oldver>-backup" engine`.
`strata components rollback` only covers engines installed through the feed, not a source checkout.

---

## 8. Version Log (What Each Upgrade Actually Did)

Append-only record. Newest first. Keep the procedure in §7 clean — evidence lives here.

### 2026-10-26 — engine `0.1.48` → `0.1.50` (source `0.1.51`)

**Why the jump:** `git pull` brought source to 0.1.51. The newest published **Qwen** engine package was
0.1.50; 0.1.51 applied to DeepSeek and Control only (commit `562a0c2e`: "the Qwen engine packages and Metal
stay 0.1.50"). So 0.1.50 was the correct target, not 0.1.51.

**How it was installed:** the manual §7 procedure. `UPDATE.bat` refused (installed 0.1.48 was already
`>= MIN_ENGINE` 0,1,46). Package source folder was `0.1.50-windows-x86_64-cuda-cuda13`; its `BUILD.json`
shipped `"source": "f-0243-rel"` and had to be rewritten to `"prebuilt"`.

**Verification that it was a real new binary:** `--version` → `0.1.50`; `strata.exe` 206,539,776 bytes
(was 206,359,040); the new `--adapt-async 0|1` help text appeared (absent in 0.1.48).

**Was it faster?** No measurable change, and none was expected. The only Qwen-side change was the
`--adapt-async` auto-tune, which is **double-gated off** on this machine: it requires the _expert arena_
(we use `--resident-experts`, a different source) and it returns early when `--adapt-async` is given
explicitly (`adapt_async_default.hpp:43,48`). The rest of the Qwen diff was AVX-512 code, and this CPU
(Zen 2) has no AVX-512.

**Resident RAM was briefly misread as a regression.** After the upgrade a boot showed 15.73 GiB instead of
~20 GiB. It was not the engine: resident = `available RAM − 6.25 GiB`, and that boot had only 21.98 GiB
free. At matched RAM the two engines were identical (0.1.48 → 15.70 GiB, 0.1.50 → 15.73 GiB).

**First calibration was invalid.** It ran with 21.98 GiB free and chose `--pcie-frac 0.35`. Re-run with
25.94 GiB free it chose `0.75`. Same command, same engine, different answer — the machine was not idle.
This is why Step 3 now carries a hard RAM precondition.

**Valid calibration results** (2026-10-26, resident avg 19.42 GiB, median of 3):

| Sweep         | Result                                                                                                   |
| ------------- | -------------------------------------------------------------------------------------------------------- |
| PCIe share    | **0.75 = 123.6** (best), 0.2 = 122.5, 0.35 = 119.5, 0.0 = 118.5, 1.00 = 117.7, 0.55 = 117.6, 0.9 = 110.9 |
| Draft floor   | **0.7 = 122.5** (best), 0.5 = 121.5, 0.3 = 113.4                                                         |
| CPU workers   | 7 = 120.6, 5 = 120.7 → engine default kept (no >3% gain)                                                 |
| Adaptive tier | off = 88.8, 80 swaps = 97.8, **160 swaps = 98.2** → keep 160 (**+10.6%**)                                |

**Net config change:** `--pcie-frac` 1.00 → **0.75**. Everything else unchanged.

### 2026-10-25/26 — `--prompt-cache-spread` measured (no effect)

**Why it was added:** `--prompt-cache 6` keeps the checkpoints clustered in the last ~2K tokens, so an agent
client that rewrites its history a few thousand tokens back matches none of them and re-reads the whole
prompt. `--prompt-cache-spread` spaces them geometrically instead. Zero extra RAM — same 6 checkpoints.

**What was measured:**

| Window                           | Prompts | Zero-reuse | Weighted reuse |
| -------------------------------- | ------- | ---------- | -------------- |
| Before the flag                  | 432     | 66 (15.3%) | 81.9%          |
| After the flag                   | 26      | 4 (15.4%)  | 69.9%          |
| After, minus cold start/switches | 22      | —          | **81.5%**      |

**Result: no change.** The raw weighted-reuse drop (81.9% → 69.9%) is an artifact of 4 misses in a 26-prompt
sample; excluding them, reuse is 81.5% vs 81.9% (noise). **None of the 4 misses was the case the flag
fixes:** 3 were conversation switches (prompt size dropped: 121K → 88K, 97K → 30K) and 1 was the cold start
after a reboot.

**One hint of benefit, not established:** the mediocre 50–79% reuse band collapsed (47.7% → 15.4%) while the
80–94% band grew (28.7% → 53.8%), consistent with a better-placed checkpoint. At n=26 with a different
conversation mix this is suggestive only.

**Decision: keep the flag.** Zero RAM cost, no measured harm, and it is the only lever that helps the
distant-rewrite case should it occur. Reverting buys nothing.
