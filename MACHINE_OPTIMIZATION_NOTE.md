# Machine Profile & Performance Setup Guide

> **Hardware Baseline (as of 2026-10-24):**
> - **GPU:** NVIDIA GeForce RTX 4090 (24 GB VRAM, PCIe Gen4 x16)
> - **CPU:** AMD Ryzen 7 3700X (8 Cores, 16 Threads, Zen 2, AVX2)
> - **RAM:** 32 GB DDR4
> - **OS:** Windows 11 (CUDA 13 environment)
> - **Target Model:** `swift` (Swift 1.5) / `IQ2_XS` (2-bit i-quant)

---

## 1. Quick Re-setup Commands

When re-installing from scratch, switching branches, or rebuilding on this machine, run these steps in order:

### Step 1: Initial Setup
```powershell
.\START-HERE.bat --family swift --model IQ2_XS --context 262144 --kv k8v4 --vision cpu --draft-vocab en --port 8081 --yes --no-start
```

### Step 2: Custom Optimizations (Manual JSON Edit)
Open `strata-swift-iq2_xs.json` and ensure these two entries are present:
1. In `"env"`:
   ```json
   "env": {
     "STRATA_PF_FUSED": "1"
   }
   ```
2. In `"args"` (append to list):
   ```json
   "--mtp-window",
   "8192"
   ```

### Step 3: Hardware Calibration
```powershell
.\START-HERE.bat --calibrate --family swift --model IQ2_XS --no-start
```
*(Runs on-card sweeps across PCIe share, speculative drafting depth, and adaptive swaps to lock in best tok/s).*

### Step 4: Run Server
```powershell
.\START-HERE.bat --host 0.0.0.0 --api-key sk-strata-local --port 8081
```

---

## 2. Why Each Setting Was Chosen

| Setting | Value | Why / Benefit |
|---|---|---|
| `--vision cpu` | CPU reader (8 threads) | **Frees ~1.4 GB VRAM**. GPU vision encoder holds a 700+ MiB fixed reserve, shrinking expert cache. Offloading vision to CPU allows the GPU to fit 10,580+ expert slots. |
| `--draft-vocab en` | English/code head | **Saves ~110 MiB VRAM & +1-2% decode**. Drops unneeded CJK tokens from the draft head, fitting cache tighter. |
| `--kv k8v4` | Mixed INT8 K + 4-bit V | **Halves VRAM consumption for KV cache** over standard INT8, allowing full 262,144 context without blowing out 24 GB VRAM. |
| `context 262144` | Exact trained window | Native context length; avoids RoPE scaling / YaRN degradation. |
| `STRATA_PF_FUSED=1` | Fused int8 prompt kernels | **+12% prompt speed at 4K, +3% at 32K** (`DETAILS.md:26-27`) by running prompt experts on tensor-core GEMMs. |
| `--mtp-window 8192` | Bounded MTP attention | Caps attention span during speculative draft verification to keep decode overhead low at long contexts (`STRIX_HALO.md:144`). |
| `--pcie-frac 0.55` | Empirical PCIe share | Swept by calibration (tested 0.0 to 1.0; 0.55 produced best 108+ tok/s bursts). |
| `--spec-min-p 0.70` | Acceptance threshold | Swept by calibration (best draft acceptance ratio for Zen 2 host threads). |
| `--adapt-swaps 80` | Adaptive tier swaps | Swept by calibration (produced **82.6 tok/s** vs 69.3 tok/s at 160 swaps). |

---

## 3. Performance Ceiling & Upstream Upgrade Guide

### Why is speed bounded at ~80–85 tok/s?
- **32 GB RAM bottleneck:** Swift IQ2_XS requires ~36–40 GB to hold all experts in memory. On 32 GB RAM, un-cached experts must stream from SSD (`--resident-experts`).
- **Zen 2 memory bus:** Host-to-GPU expert transfers over DDR4 are bounded by CPU memory bandwidth, not the RTX 4090.
- *Only hardware upgrade that radically shifts this:* 64 GB RAM (eliminates SSD streaming).

### Checking New Upstream Releases (`git pull` / newer Strata)
When upgrading Strata to a newer engine version later, check:
1. **New fused kernels:** See if newer CUDA GEMM/FP4 or INT4 fused kernels support IQ2_XS without manual switches.
2. **Dynamic memory management:** Check if new release supports hybrid SSD caching improvements or reduced working-set locks on Windows.
3. **Re-calibration:** Always run `START-HERE.bat --calibrate --family swift --model IQ2_XS --no-start` after an engine update to re-tune `--pcie-frac` and `--adapt-swaps`.
