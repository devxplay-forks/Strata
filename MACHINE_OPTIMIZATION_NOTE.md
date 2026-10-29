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

### 2026-10-29 — parking reverted: single session only; timestamps in the log

**Decision:** back to one session. `--conversation-cache-mib 0` (explicitly **zero**, never absent),
`--conversation-cache-slots` and `--conversation-cache-min-free-mib` removed, `STRATA_PARK_ASYNC` removed,
`STRATA_RESIDENT_HEADROOM_GIB` 10 → **6**. Args 47 → 43, env 16 → 15.

**Why: the park ceiling is ~28K tokens, so parking cannot hold an agent session.**

| Metric                  | Value                                        |
| ----------------------- | -------------------------------------------- |
| `skip parking` failures | 900 (492 RAM admission, 408 snapshot budget) |
| Park events             | 593                                          |
| Parks ≥ 1,000 tokens    | **4** — 9,985 / 10,662 / 26,615 / 28,179     |
| Restores ≥ 1,000 tokens | **2** — 9,861 / 10,558                       |
| A cold Pi session       | 25,450–27,746 tokens, `reused 0`             |

The largest park ever achieved (28,179) leaves only ~400–1,600 tokens above a bare Pi system prompt, so
parking held a greeting and never a working conversation.

**The real cost was the headroom raised for parking.** At 10 GiB the full 19.56 GiB complement no longer
fitted, and the engine said so on 45 of 55 boots: `resident complement 19.56 GiB exceeds available RAM
(24.95 GiB) minus the 10 GiB safety headroom`. Resident experts fell from **19.66–20.58 GiB** (27 boots at
6 GiB headroom) to **13.43–15.98 GiB** (18 boots at 10 GiB): **−3.6 to −7.2 GiB (−18% to −35%)**. File blob
reads went from 2 to as many as 6,833,263 in one session, and the expert-cache hit rate fell from ~96.5% to
~90%.

**After the revert** (boot 2026-10-29 16:18:37, 27.77 GiB free at launch):

| Reading                    | Before (10 GiB) | After (6 GiB) |
| -------------------------- | --------------: | ------------: |
| Resident experts           |       14.33 GiB | **18.97 GiB** |
| `room for the complement`  |       14.30 GiB | **18.97 GiB** |
| Drive reads per 32K prompt |        ~34.9 GB |   **~4.1 GB** |
| `conversation cache` lines |       900 skips |         **0** |

**Still 0.59 GiB short.** Available RAM was 25.22 GiB and the full complement needs ≥ 25.81 GiB at 6 GiB
headroom, so the warning persists. That is a boot-time RAM limit, not the constant: launch with background
applications closed, or test headroom 5 (fits at ≥ 24.81 GiB). Headroom 5 has **not** been tested.

**Decode: no proven change.** 600-token generations at short context measured 128.9 / 158.6 / 152.9 tok/s
(median 152.9); at ~24K context 94.1–184.0 (median 111.2). Both sit inside the historical band (122.4 at
19.02 GiB; 126.6 deep-gen), so the +4.64 GiB of resident experts is **not demonstrated in tok/s**. The
proven wins are fewer drive reads and zero parking overhead.

**Timestamps in the log.** Progress lines carried only elapsed seconds. `serve/server.py` now has `_stamp()`
(`2026-10-29T16:22:10+08:00`: local date, time and UTC offset, DST-aware, no new imports) and `_say()`, which
prints and also mirrors into the engine log — the launcher pipes this process's stdout for the console alone,
so these lines never reached the log before. The `engine started:` line uses the same format.

```
[strata] 2026-10-29T16:22:09+08:00 reading the prompt: 62 of 67 tokens, 1 s so far
[strata] 2026-10-29T16:22:10+08:00 thinking: 71 of max 200 tokens, 76.1 tok/s, 2 s
```

Verified: 3 stamped progress lines, **0 unstamped**. The tests
`ThinkingBudget.test_a_progress_line_carries_the_local_date_time_and_utc_offset` and
`ThinkingBudget.test_the_done_line_carries_the_local_date_time_and_utc_offset` each fail without the stamp and
pass with it.

**A second gap, found in the first live run (2026-10-29 17:21):** the `[strata] done:` summary — the line a long
run is actually followed by — was still bare, because the first pass covered only `_progress`, a different call
site. `done:` and "the reply reached max tokens while still thinking" now also go through `_say()` + `_stamp()`,
so they reach the engine log as well. About 120 further `[strata]` prints stay bare deliberately: startup
notices (`engine priority`, `layer split`, `loading the model`) and error paths are not the lines a run is
followed by. Ask if those should be stamped too.

**Recalibration after the revert** (2026-10-29, ~10 min, 27.96 GiB free, server stopped). The 2026-10-28 sweeps
all ran with `--conversation-cache-mib 8192` and 10 GiB headroom, so they measured a ~14 GiB-resident machine.
Re-run at 18.97 GiB, `START-HERE.bat --calibrate --no-start --yes` returned:

| Flag            | 2026-10-28 | 2026-10-29 | Verdict                                           |
| --------------- | ---------: | ---------: | ------------------------------------------------- |
| `--pcie-frac`   |       0.00 |   **0.00** | unchanged                                         |
| `--adapt-every` |          1 |          1 | unchanged                                         |
| `--adapt-swaps` |        160 |    **160** | unchanged — the guess that it had moved was wrong |
| `--adapt-decay` |       0.97 |       0.97 | unchanged                                         |
| `--spec-min-p`  |       0.70 |   **0.50** | sweep chose 0.50 today; pinned to 0.70 by hand    |

**`--spec-min-p` must be pinned by hand because the calibrator cannot resolve it.** The flag is not skipped:
`SPEC_MIN_PS = (0.3, 0.5, 0.7)` (`tools/calibrate.py:51`) is swept at the PCIe share just chosen (`:335`). Two
things make the answer unstable. First, `:310` (`apply(base_args, {})`) resets all seven calibrated flags to
product defaults before measuring, so a tuned value never survives as the starting point. Second, `pick()`
(`:186-192`) keeps a winner only when it beats the default pair by more than `MIN_GAIN = 3%` (`:49`), and
`:360-362` then writes `--pcie-frac` and `--spec-min-p` **together**. The 0.50↔0.70 gap sits exactly on that 3%
line: 2026-10-26 measured 0.70 = 125.2 against 0.50 = 121.5, only **+3.05%**; on 2026-10-29 at the same
`--pcie-frac 0.00` the sweep chose 0.50 instead. Same share, opposite answer — the difference is the size of the
noise. So 0.70 is pinned for **determinism, not because it is proven faster**. Deciding it would take an
interleaved A/B (0.50, 0.70, 0.50, 0.70, …), which a single sequential sweep cannot do.

**The recalibration did not touch the revert:** `--conversation-cache-mib 0`, `STRATA_RESIDENT_HEADROOM_GIB=6`
and the other 14 `env` entries survived (43 args / 15 env before and after, nothing added or removed,
`user_args` still `--vram-reserve-mib 1600`). `apply_env` manages only `STRATA_SH_STREAM`
(`calibrate.py:514`), so the revert is safe across a calibration — verified by diff, not assumed.

**Interleaved A/B, 2026-10-29** (harness kept outside the repo; 6 engine boots, one warm-up plus 3 × 600-token
generations per boot, every arm at `--pcie-frac 0.00` and `--conversation-cache-mib 0`).

| Arm                                 | Medians per boot |   Overall | Resident        | `does not fit` |
| ----------------------------------- | ---------------: | --------: | --------------- | -------------- |
| `--spec-min-p 0.70`, 6 GiB headroom |     221.6, 230.1 |     225.8 | 19.14–19.15 GiB | yes            |
| `--spec-min-p 0.70`, 5 GiB headroom |     205.8, 253.5 | **229.7** | 20.25–20.45 GiB | **no**         |
| `--spec-min-p 0.50`, 5 GiB headroom |     248.5, 224.7 | **236.6** | 20.25–20.45 GiB | no             |

**5 GiB headroom is not faster: +1.7%, inside the noise** (individual 600-token rates span 146.9–275.0, about
±30%). What it does buy is measurable elsewhere: **+1.1–1.3 GiB of resident experts and the `does not fit`
warning disappears**. Its cost is 1 GiB less of the only out-of-memory cushion this machine has. **Decision:
5 GiB**, for the larger resident pool and the clean log, with the decode gain explicitly disclaimed.

**`--spec-min-p` stays unresolved.** 0.50 measured 236.6 against 229.7 for 0.70 (+3.0%), but the two rounds
disagree in direction (round 1 favours 0.50, round 2 favours 0.70) and the spread inside a single arm is wider
than the gap. **Decision: keep 0.70.** Do not re-litigate without many more samples.

**Correction to the decode figures above.** The `128.9 / 158.6 / 152.9` rates were taken through ad-hoc HTTP
calls _after_ a 23,870-token prefill had filled the page cache. With a matched harness the same 6 GiB
configuration measures **225.8**, so that figure was a measurement-condition artifact, not a headroom penalty.
**226 is also not a real agent decode rate**: this harness reads a ~20-token prompt and decodes 600 highly
predictable tokens, so it overstates the 122.4 tok/s a real 40–200K-token agent turn produces. The harness is
valid **only** for comparing arms against each other, which is how it was used here.

**Agent-shaped A/B at ~58K context, 2026-10-29** (3 interleaved rounds, a 57,875-token prompt and a 300-token
answer, 6 engine boots; decode taken from the engine's own stamped `answering:` lines rather than from
wall-clock, which removes HTTP and serialisation overhead).

| Arm   |   R1 |   R2 |   R3 |  Overall | Resident        | `does not fit`  |
| ----- | ---: | ---: | ---: | -------: | --------------- | --------------- |
| 5 GiB | 64.8 | 62.8 | 58.6 | **62.8** | 20.31–20.73 GiB | never           |
| 6 GiB | 62.8 | 62.6 | 66.6 | **62.8** | 19.20–19.73 GiB | once (at 19.20) |

**Identical at 58K context: 62.8 against 62.8.** Prefill tied as well (57,870 tokens in 32 s against 33 s, about
1,808 against 1,754 tok/s). Together with the short-prompt result (+1.7%), **5 GiB headroom has no measurable
throughput benefit at either context length.** It is kept only for the larger resident pool and the clean log,
with the cost stated plainly: 1 GiB less out-of-memory cushion. If the desktop ever feels starved, set it back
to 6 — that costs nothing measurable either.

**The real agent numbers, for the record:** about **63 tok/s decode and 1,800 tok/s prefill at a 58K prompt**.
That context penalty (~3.5× against the 226 a 20-token prompt yields) is exactly why the short-prompt harness
must never be quoted as a real decode rate.

### 2026-10-28 — engine `0.1.50` → `0.1.52`; conversation parking enabled for 2 sessions (superseded 2026-10-29)

**Why the jump:** 0.1.52 is the newest **Qwen** package (release commit `a42cc3d7`: "every Qwen and DeepSeek
package"). It matters for this machine: the five conversation-cache fixes are dated 2026-10-26 and the 0.1.50
build commit (`1aff6d14117b`) is dated 2026-10-24, so 0.1.50 predates all of them — including `f8ff3420`
(#1503), a **correctness** bug where one conversation could be parked with another's retained K/V.

**Problem being solved:** two Pi sessions alternating re-read their whole history on every turn. Root cause is
structural, not a setting: `include/strata/program/conv_cache.hpp:6-10` — the KV cache is _one arena that
holds one branch of history at a time_, and retained checkpoints form a **chain**, not a tree. So
`--prompt-cache 6` is six checkpoints on one branch; a second conversation invalidates the first. The boot log
of 2026-10-28 shows it plainly: `190978`, `68882`, `193940`, `69137`, `195506` tokens all with
`reused 0`, at ~100 s per re-read.

**Why parking never worked before:** it was on by default the whole time and refused every attempt — the
auto budget (`auto_conversation_cache_mib`, 40% of RAM above the floor, `conversation_memory.hpp:70-78`)
collapsed to **1024 MiB** while real snapshots needed **1033–3338 MiB**, and `MemAvailable` was 200–400 MiB
because 19.5 GiB of resident experts own the RAM.

**How it was installed:** the manual §7 procedure. `strata.py update --no-control` fetched and test-started
0.1.52 (loaded the model, answered one token) into the component store; it was copied over `engine/` and
`BUILD.json` `"source": "f-027-rel"` rewritten to `"prebuilt"`. `strata.exe` 206,539,776 → 272,727,040
bytes. `STRATA_PARK_ASYNC` and `STRATA_PARK_VRAM_MIB` are present in the 0.1.52 binary and absent in 0.1.50.

**Config changes (all four landed, `args` 41 → 47, `env` 15 → 16):**

| Change                              | From        | To      | Why                                                       |
| ----------------------------------- | ----------- | ------- | --------------------------------------------------------- |
| `STRATA_RESIDENT_HEADROOM_GIB`      | `6`         | `10`    | free host RAM for parking; resident 19.54 → **14.33 GiB** |
| `STRATA_PARK_ASYNC`                 | —           | `1`     | 0.1.52 only: park in the background                       |
| `--conversation-cache-mib`          | auto 1024   | `8192`  | fits two ~3.3 GiB snapshots                               |
| `--conversation-cache-slots`        | 4           | `2`     | `1` is wrong: B's park would evict A                      |
| `--conversation-cache-min-free-mib` | 2560 → 1024 | **512** | see below — this is what made it work                     |

**The floor was the real blocker.** At `1024` the admission check still refused the 10K conversation:
`need 351 MiB plus 1024 MiB floor, 1090 MiB available`, and A's return re-read in **4.39 s**. At `512`
the same conversation parked (368 MB snapshot) and restored in **39.4 ms**. Exact log evidence:

```
conversation cache: parked 10662 tokens in 120.3 ms; snapshot_bytes=368278792
conversation cache: restored 10558 tokens (checkpoint) in 39.4 ms
prompt 10575 tokens = 10558 reused + 17 read in 211 ms (80.5 tok/s)
```

Single-session cold reads for scale: `prompt 10563 tokens = 0 reused + 10563 read in 11671 ms`.
A returning session now re-reads **17 tokens instead of 10,563**.

**Re-tune (calibration, 27.6 GiB free, idle machine).** The old values were fitted at 19.54 GiB resident on
0.1.50 and were stale. The new optimum **flipped**:

| Sweep         | Result                                                                                                        |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| PCIe share    | **0.00 = 120.1** (best), 1.00 = 119.9, 0.9 = 119.3, 0.35 = 118.7, 0.2 = 112.7, **0.75 = 111.0**, 0.55 = 109.7 |
| Draft floor   | **0.7 = 125.2** (best), 0.5 = 115.4, 0.3 = 112.0                                                              |
| CPU workers   | **7 = 124.4** (best), 5 = 114.2, 4 = 102.1, 2 = 83.7 → keep the engine default                                |
| Adaptive tier | default = 84.7, 80 swaps = 95.3, **160 swaps = 98.3** → keep 160                                              |

`--pcie-frac` 0.75 → **0.00** is the headline: with 5.2 GiB fewer resident experts the missed-expert split
moves to the CPU. `--spec-min-p 0.70` was confirmed unchanged. `carry_user_args` preserved every parking
setting through the rewrite (`args` 47, `env` 16 unchanged) — calibration is safe to re-run now.

**Decode after the change:** warm 103–122 tok/s at `--pcie-frac 0.00` on short prompts, and 120–168 tok/s on
the expert cache in general — at or above the 122 tok/s baseline. A direct 0.75-vs-0.00 probe disagreed with
the sweep at n=3; the 9-prompt, 3-round sweep is the authoritative method, so 0.00 stands.

**Known limits.** Parking is RAM-marginal on 32 GB: two ~10K conversations fit comfortably, but the two
sessions still run **one at a time** (parking preserves history, it does not decode concurrently). A 190K
conversation needs ~3.3 GiB per snapshot and will still be refused at headroom 10 — 64 GB of RAM remains the
fix for that. Snapshots are not kept across restarts (`DETAILS.md:639`), so every boot starts cold.

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
