# Results: Galaxy Z Fold 8

These notes come from Termux on a Galaxy Z Fold 8 (`SM-F971U`), measured 2026-10-27 to 2026-10-30.
The phone has a Snapdragon 8 Elite Gen 5 (`SM8850`), an Adreno 840 GPU, and a Hexagon NPU (HTP
arch **v81**). It runs Samsung's Android 16 firmware and is **not rooted**.

The model was NVIDIA Nemotron-3.5-Bolt-E4B, a Gemma-4-E4B finetune (`gemma4` architecture), in
**Q4_0**: `prithivMLmods/NVIDIA-Nemotron-3.5-Bolt-E4B-GGUF`,
`NVIDIA-Nemotron-3.5-Bolt-E4B.Q4_0.gguf`, 5,185,934,336 bytes.

## NPU (`HTP0`): works

Samsung's firmware accepts unsigned DSP code: there was no `createUnsignedPD` refusal, and the
output is correct with HMX on. The setup is in the [README](../README.md): the build's own `lib/`
holds the copied vendor libraries, and `/vendor/lib64` is never on the path.

`llama-bench`, pp512 / tg128, 3 reps, `-dev HTP0 -ngl 99`, llama.cpp @ 7ac5727d, toolchain v0.8:

| `GGML_HEXAGON_POWER_MODE` | pp512 t/s | tg128 t/s |
|---|---|---|
| **burst** | **1354.4 ± 13.2** | **19.57 ± 0.04** |
| sustained_high_performance | 1150.2 ± 3.6 | 16.05 ± 0.21 |
| balanced | 974.6 ± 5.7 | 13.32 ± 0.10 |

For comparison, Qualcomm's GenieX demo app runs the same Q4_0 at 12–13 t/s on this phone.

`llama-server` (burst, `-c 4096 --jinja`):

- It loads in about 7 s and streams at 18.6 t/s.
- The prompt cache works: a second turn of the same conversation reads only its new tokens.
- Peak RSS (VmHWM) was 2.11 GB, but the real cost is larger: MemAvailable drops by about 5–6 GB
  during the load. About 2.76 GB of weights sit in the NPU's buffers, which no process's RSS counts.
  Another 2.73 GB stay on the CPU side as memory-mapped per-layer embeddings, which the system can
  drop and read again.

KV cache:

- f16 costs 16 KB per token: 1,024 MB at 64K, 512 MB at 32K. That works out because only 4 of the
  model's 42 layers keep a full-length cache.
- `-ctk q8_0 -ctv q8_0 -fa on` works and runs at the same speed as f16, using about 544 MB at 64K.
- q5_0 and q4_0 KV abort at load, because the Hexagon backend can't run SET_ROWS into those types.
- The model's trained context is 64K (`gemma4.context_length` 65536).

**Memory is the real limit.** About 5.3 GB was available with nothing else running, so with this
model loaded and another heavy process beside it, Android's low-memory killer took Termux.

### Quants with NPU kernels

`libggml-htp-v81.so` (read from GenieX's build, which uses the same upstream kernels) has kernels for
**Q4_0, Q4_1, Q8_0, IQ4_NL, MXFP4, Q5_K, Q6_K, Q3_K, Q2_K**, plus f16 and f32. It has none for
**Q4_K, Q5_0/Q5_1, IQ3_XXS or any IQ2**. So unsloth's `UD-Q*_K_XL` files, which are mostly Q4_K, fit
the NPU badly. Requantize to Q4_0, or use IQ4_NL or MXFP4.

## GPU (`GPUOpenCL`): doesn't work here

`ggml_opencl: platform IDs not available`, so `GPUOpenCL` never shows up as a device. Android 16's
linker namespace won't let Termux load `/vendor/lib64/libOpenCL.so` (with `LD_PRELOAD` it fails as
`not accessible for the namespace`). Copying the vendor libraries into Termux crashed with a
segfault, and the `libOpenCL.so` this README copies into `lib/` still finds no platform. On Android 15 the same approach worked
([discussion #23736](https://github.com/ggml-org/llama.cpp/discussions/23736)).

Vulkan (a separate build with `-DGGML_VULKAN=ON`) runs, but gemma4 output on Adreno is garbage, even
with `GGML_VK_DISABLE_INTEGER_DOT_PRODUCT=1`. Setting `GGML_VK_DISABLE_F16=1` crashes the loader.

## CPU

- 19.8 t/s pp512 on 6 threads with this build and the same model. The CPU also runs hot and draws a
  lot of power; the NPU is the backend worth using.
- Termux's own `llama-cpp` package (0.7.0) targets the baseline ARM architecture: its
  `libggml-cpu.so` has no `sdot`/`smmla` instructions. On a Q3_K_XL model it was 4–5× slower than a
  source build with `-DGGML_NATIVE=OFF -DGGML_CPU_ARM_ARCH=armv8.6-a+dotprod+i8mm+bf16+fp16`.
  `-mcpu=native` reports "generic" on this SoC, so you have to name the architecture yourself.
- Samsung keeps Termux on 4 cores (the `moderate` cpuset) while it's in the background and gives it
  all 8 while it's on screen. 6 threads generated faster than 8.

## Not measured

Power and temperature: `/sys/class/power_supply/battery` is permission-denied from Termux on this
firmware.
