# llama-snapdragon

Upstream [llama.cpp](https://github.com/ggml-org/llama.cpp) built for arm64 Android with its
Snapdragon backends, the Hexagon NPU (`HTP0`) and the Adreno GPU (`GPUOpenCL`), to run from Termux.
The build follows llama.cpp's own
[Snapdragon docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/backend/snapdragon/README.md)
in Qualcomm's public toolchain image (`ghcr.io/snapdragon-toolchain/arm64-android`); nothing is
patched. The workflow is started by hand, and each run replaces the
[`latest` release](../../releases/tag/latest): `llama.cpp-snapdragon-android.tar.xz` and its
`sha256sums.txt`. The llama.cpp commit and toolchain are in the release notes and in the tarball
(`LLAMA_COMMIT`, `TOOLCHAIN`).

## In Termux

Never put `/vendor/lib64` on `LD_LIBRARY_PATH`. An app can't open `/dev/fastrpc-cdsp`, so
`libcdsprpc.so` asks Qualcomm's DSP service for it, and that lookup fails with `/vendor/lib64` on the
path (FastRPC err 114, session error 0x72; llama.cpp#27677). Copy the vendor libraries it needs into
the build's own `lib/` instead and put only that on the path:

```sh
curl -fLO https://github.com/Dmage22/llama-snapdragon/releases/download/latest/llama.cpp-snapdragon-android.tar.xz
curl -fLO https://github.com/Dmage22/llama-snapdragon/releases/download/latest/sha256sums.txt
sha256sum -c sha256sums.txt
mkdir -p ~/llama-snapdragon && tar -xJf llama.cpp-snapdragon-android.tar.xz -C ~/llama-snapdragon --strip-components=1
for l in libcdsprpc.so vendor.qti.hardware.dsp@1.0.so vendor.qti.hardware.dsp-V1-ndk.so libvmmem.so libOpenCL.so; do
  cp /vendor/lib64/$l ~/llama-snapdragon/lib/ 2>/dev/null
done
LD_LIBRARY_PATH=~/llama-snapdragon/lib ADSP_LIBRARY_PATH=~/llama-snapdragon/lib \
  ~/llama-snapdragon/bin/llama-bench -m model-Q4_0.gguf -dev HTP0 -ngl 99
```

`Q4_0` is the quant the NPU backend is built around. The DSP has to accept unsigned code for this to
work, which is a property of the phone's firmware, not of the build.

## Tested devices

- Galaxy Z Fold 8 (SM8850, Hexagon v81): the NPU works, about 19.6 t/s generating on a 4B-class
  Q4_0 model; GPUOpenCL doesn't. Numbers, memory, and which quants have NPU kernels are in
  [docs/galaxy-z-fold8.md](docs/galaxy-z-fold8.md).
