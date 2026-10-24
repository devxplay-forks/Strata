# Machine Profile & Performance Setup Guide

> **Hardware Baseline (as of 2026-10-24):**
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

> **Critical Prerequisite:** On Windows with 32 GB RAM, the default Windows Commit Limit (~42 GB) is too small. Strata requires ~55–65 GB of commit space to allocate its 14.4 GB page-locked resident RAM complement. Without this, Strata refuses resident RAM and falls back to slow SSD mmap. **Configure this first before installing models.**

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
   "sibling"
   ```

### Step 3: Hardware Calibration

```powershell
.\START-HERE.bat --calibrate --family swift --model IQ2_XS --no-start
```

_(Runs on-card sweeps across PCIe share, speculative drafting depth, and adaptive swaps to lock in best tok/s)._

### Step 4: Run Server

```powershell
.\START-HERE.bat --host 0.0.0.0 --api-key sk-strata-local --port 8081
```

---

## 2. Why Each Setting Was Chosen

| Setting                                | Value                          | Why / Benefit                                                                                                                                                                              |
| -------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--vision cpu`                         | CPU reader (8 threads)         | **Frees ~1.4 GB VRAM**. Offloading vision to CPU allows the GPU to maximize expert slots.                                                                                                  |
| `--draft-vocab en`                     | English/code head              | **Saves ~110 MiB VRAM & +1-2% decode**. Drops unneeded CJK tokens from draft head.                                                                                                         |
| `--vram-reserve-mib 1600`              | Fixed VRAM headroom            | **Eliminates WDDM page eviction cliff**. Prevents Windows 11 DWM from dropping speed from 50 tok/s to 2 tok/s on deep 100K-262K context.                                                   |
| `--kv k8v4`                            | Mixed INT8 K + 4-bit V         | **Halves VRAM consumption for KV cache** over standard INT8, allowing full 262,144 context without blowing out 24 GB VRAM.                                                                 |
| `context 262144`                       | Exact trained window           | Native context length; avoids RoPE scaling / YaRN degradation.                                                                                                                             |
| `STRATA_PF_FUSED=1`                    | Fused int8 prompt kernels      | **+12% prompt speed at 4K, +3% at 32K** (`DETAILS.md:26-27`) by running prompt experts on tensor-core GEMMs.                                                                               |
| `--mtp-window 8192`                    | Bounded MTP attention          | Caps attention span during speculative draft verification to keep decode overhead low at long contexts.                                                                                    |
| `--adapt-async 1`                      | Async expert swapping          | **+7% to +11% decode speed** (`DETAILS.md:202`) by asynchronously overlapping RAM-to-VRAM expert swaps with token decoding.                                                                |
| `--host-core sibling`                  | SMT host core isolation        | Moves host scheduler to Core 0 SMT sibling (Logical Processor 1), leaving Processor 0 free for Windows 11 GPU driver interrupts (+3-5%).                                                   |
| `STRATA_PIN_BY_NEED=1` / `PCIE_MIN1=1` | PCIe miss activation           | Fixes `pcie_share 0.0` symptom; forces PCIe lane to activate even for low miss counts (+5-7% prompt speed).                                                                                |
| `STRATA_RESIDENT_HEADROOM_GIB=6`       | RAM headroom guard             | Prevents Windows 11 working set memory trimming on 32 GB RAM systems.                                                                                                                      |
| `STRATA_SPEC_COUPLED=1`                | Coupled draft sampling         | Couples speculative draft RNG with target model distribution, raising draft acceptance rates to 68-75%.                                                                                    |
| `--pcie-frac 1.00`                     | Empirical PCIe share           | Swept by calibration (105-113 tok/s across all share rates, 1.00 peaked highest).                                                                                                          |
| `--spec-min-p 0.70`                    | Acceptance threshold           | Swept by calibration (best draft threshold on 14.4 GB resident RAM; 113.6-120.6 tok/s bursts).                                                                                             |
| `--adapt-swaps 160`                    | Adaptive tier swaps            | Swept by calibration (measured an all-time record **91.9 tok/s** in the 512-token sustained stress test).                                                                                  |
| `STRATA_SPEC_GUMBEL=1`                 | Gumbel-Max noise coupling      | Replaces softmax draft coupling with Gumbel-max noise matching, raising speculative draft acceptance rates with zero VRAM/RAM cost.                                                        |
| `STRATA_RAM_ADAPT=2`                   | Dynamic resident RAM sync      | Swaps out dormant experts in the 18.22 GiB RAM pool with active conversation experts; cuts repeated SSD reads per token from 14 to ~2.7.                                                   |
| `STRATA_RAM_ADAPT2=1`                  | Async RAM promotion I/O        | Stages RAM expert promotions in the background on a dedicated I/O thread, avoiding GPU decode stalls during RAM updates.                                                                   |
| `STRATA_DEQUANT_WIDE=1`                | Ada wide dequantization        | Bypasses hardcoded `sm_86` default to enable 16-bit loads / 16-byte stores on RTX 4090 (Ada sm_89); cuts **11% to 15% off TTFT**.                                                          |
| `STRATA_PROMPT_ATTN_IMMA=1`            | INT8 Tensor Core Attention     | Routes `--kv k8v4` attention matrix products directly into Ada 4th-Gen INT8 Tensor Cores; **+2.5% prompt prefill throughput**.                                                             |
| `STRATA_ATTN_DAILY=1`                  | Warp-synchronous decode attn   | Replaces upstream chunked decode attention with warp-synchronous reduction kernel (+1.5% to +3% decode).                                                                                   |
| `STRATA_GU_SWIGLU=1`                   | Fused gate/up + SwiGLU launch  | Fuses gate/up projection, SwiGLU activation, and Q8_1 quantization into a single CUDA kernel launch, cutting kernel launch overhead.                                                       |
| `STRATA_GR_UP75=1`                     | Staged group-reduction up-proj | Optimizes multi-token verification window group-reduction up-projection via sm_75+ tensor-core staging (+1% to +1.5% decode).                                                              |
| `STRATA_QSA_ROWS=1`                    | Low-query QSA block scores     | Optimizes QSA block score calculations for small query counts (NQ <= 4), reducing warp stalls during expert routing.                                                                       |
| `STRATA_SIDE_MIXER=2`                  | Async side mixer execution     | Runs GDN gate projections and QSA indexing asynchronously on a secondary CUDA stream alongside layer projections.                                                                          |
| `expert_profile_save`                  | Learned profile persistence    | Automatically saves the active expert routing table to `data/expert-profile-learned.bin` every 10 min and on exit; future launches load this file to eliminate the cold-start warmup ramp. |

---

## 3. Performance Ceiling & Upstream Upgrade Guide

### Why is speed bounded at ~100–113 tok/s peak / ~50–60 tok/s sustained?

- **32 GB RAM bottleneck:** Swift IQ2_XS requires ~36–40 GB to hold all experts in memory. On 32 GB RAM, un-cached experts (~5-8 GB) must stream from SSD (`--resident-experts`).
- **Zen 2 memory bus:** Host-to-GPU expert transfers over DDR4 are bounded by CPU memory bandwidth (~25.6 GB/s), not the RTX 4090.
- _Only hardware upgrade that radically shifts this:_ 64 GB RAM (eliminates SSD streaming entirely, unlocking 120-150+ tok/s).

---

## 4. Resizable BAR (ReBAR) Status & BIOS Enable Guide

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

### Checking New Upstream Releases (`git pull` / newer Strata)

When upgrading Strata to a newer engine version later, check:

1. **New fused kernels:** See if newer CUDA GEMM/FP4 or INT4 fused kernels support IQ2_XS without manual switches.
2. **Dynamic memory management:** Check if new release supports hybrid SSD caching improvements or reduced working-set locks on Windows.
3. **Re-calibration:** Always run `START-HERE.bat --calibrate --family swift --model IQ2_XS --no-start` after an engine update to re-tune `--pcie-frac` and `--adapt-swaps`.
