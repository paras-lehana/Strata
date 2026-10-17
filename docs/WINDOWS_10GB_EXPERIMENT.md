# Windows RTX 3080 10 GB experiment

Based on upstream v0.1.45 (`477ac13cbe8e582c20484871ce459d20d82df09d`).
This experimental fork accompanies [Local Coding Lab](https://github.com/paras-lehana/local-coding-lab).
The upstream README recommends 12 GB; its setup also permits smaller cards.
There is no hard 12 GB gate removed by this fork.

## Changes

1. `3e0b27d`: accept a Qwen3.6 GGUF without the optional
   `qwen35moe.nextn_predict_layers` key as zero MTP layers. Present values still
   receive numeric/range validation. Geometry/layout tests cover both cases.
2. `b0513be`: `STRATA_Q36_PAGEABLE_EXPERTS=1` selects the existing malloc arena
   fallback without first attempting to pin the full routed-expert arena.
   Default allocation is unchanged; pageable transfers may be slower.
3. `6ef632f`: `STRATA_Q36_TRIM_GGUF=1` on Windows asks the OS to remove already
   copied read-only GGUF pages from this process's working set. It uses
   `VirtualUnlock`; `ERROR_NOT_LOCKED` is the documented result for unlocked
   pages. It does not unmap, change file bytes, or flush the system file cache.
   Without the variable, the helper returns immediately. Other platforms do
   not perform the Windows call.

## Measured so far

Windows11, RTX3080 10GB, Ryzen7 5800X, 64GB DDR4; ordinary apps remain open.
Typically 25–26GiB host RAM and 6.5–6.9GiB device memory were available before
loading. CUDA13.3/MSVC14.50 source build, sm86, portable AVX2 CPU kernels.
The same model artifact used by llama.cpp was not requantized or edited:
`Qwen3.6-35B-A3B-UD-Q4_K_M.gguf`, SHA256
`ac0e2c1189e055faa36eff361580e79c5bd6f8e76bffb4ce547f167d53e31a61`.

The stock loader rejected its omitted MTP metadata. The patched host test
passes28 assertions and validates all733 real tensors. Source CUDA builds pass.
With the GGUF trimming option, a 32K synthetic benchmark completes with both
pinned and pageable arenas. The pinned run's three256-token prompt rates are
96.2/129.7/129.4tok/s; its64-token decode rates are21.9/21.0/21.2tok/s.
These tiny runs do not measure coding quality or establish a llama.cpp speedup.

With the owned downloader stopped in both runs, disabling the trim option
hits the host2.5GiB free-RAM floor during load; enabling it completes.
An earlier downloader retained6.50GiB committed memory even while suspended.
Removing that interference was separate from the source patch.

The initial32K API greeting answers correctly. A subsequent multi-file request
hits the600MiB free-VRAM floor and is stopped, unscored. Serving with a smaller
prompt chunk and graphs disabled is being evaluated. Do not treat a native
benchmark pass as evidence that an editor conversation fits.

The successful native smoke uses:

```powershell
$env:STRATA_Q8_PACKED = '0'
$env:STRATA_Q36_NO_MMQ = '1'
$env:STRATA_Q36_TRIM_GGUF = '1'
# Optionally: $env:STRATA_Q36_PAGEABLE_EXPERTS = '1'
.\strata-q36.exe --native PATH_TO_VERIFIED_GGUF --max-context 32768 `
  --prefill 128 --mtp 0 --expert-vram 0.25 --cpu-threads 8 `
  --bench-pp 256 --bench-tg 64
```

Run under independent RAM, commit, VRAM and temperature monitoring. This is
an experiment command, not an automatically safe profile for another PC.
The engine reports9 CPU threads for the8-worker option. The Qwen path allocates
fp32 attention KV on GPU and a complete18.23GiB expert arena on CPU; increasing
GPU expert slots does not remove the same weights from the host arena.
Flash-Next's host-budget and KV-streaming flags belong to another engine path.

## Limits

No completed coding score, 128K Qwen serving result, real8GB hardware test,
HIP/SYCL build, broad numerical-equivalence test, or upstream PR review is claimed
at this checkpoint. Only CUDA was built here. Native defaults were not changed.
Do not adopt this fork as a replacement for a working long-context server based
on the short throughput figures. The lab report retains every failed trial and
will distinguish later completed answers from these initial measurements.
