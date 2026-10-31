# Unlocking full FP16 tensor throughput on a laptop RTX 4070 (AD106M)

October 30, 2026 · Short report: [Zagitta/gpu-unlock discussion #238](https://github.com/Zagitta/gpu-unlock/discussions/238#discussioncomment-19307118)

## TL;DR

The public Ada four-PLM unlock works on a laptop RTX 4070 (AD106M, device `0x2860`). It lifts dense FP16/BF16 GEMM by about 50% and TF32 by 67%.

- **It does nothing for quantized LLM inference.** llama.cpp Q4 prompt processing and generation were flat, because that path runs on INT8 tensor cores, which were never throttled.
- **On a laptop, power state is the trap.** An unbound dGPU drops to D3cold, and the unlock then bails before touching the GPU. Wake it to D0 first.
- **It doesn't survive power-down.** The unlock resets whenever the dGPU drops to D3cold, not just on reboot.
- **Nothing here is new technique.** This is a port and replication of existing public work (credits at the end), on hardware it hadn't been confirmed on.

## Background

GeForce cards issue FP16/BF16 multiply-adds that accumulate into FP32 at a reduced rate compared with NVIDIA's professional cards built on the same silicon. The limit is set by "speed selector" registers that the driver can't normally write, because Privilege Level Mask (PLM) registers lock them.

The unlock relaxes those PLMs without modifying any firmware:

1. A kernel module claims the GPU **before** the NVIDIA driver binds.
2. It runs NVIDIA's own signed SEC2 Booter, exactly as shipped, with a crafted data frame (the "payload") that steers the Booter into doing the privileged PLM writes.
3. The host then writes the speed selectors (SS0 `0x88888888`, SS1 `0x8`), resets the function (FLR), and hands the card to the unmodified NVIDIA driver.

I used [Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock), a Linux DKMS module that does this for Ampere (GA102–GA106). For Ada it has AD102–AD107 recipes, and an RTX 4090 Laptop (AD103M) has been reported working, but the 4070 Laptop's ID (`0x2860`) isn't in its card list. The Ada payload, a four-PLM chain covering WPR_CFG, FBPA, WPR and FEAT, comes from TOTORONG/AD103_win_unlocker and JustAResearcher/gpu-unlock-ada.

NVIDIA ships a single Ada Booter (`BooterLoadUcode_AD102`) for every AD10x chip, mobile ones included. That made it plausible the same payload would work on AD106M. My contribution is confirming that it does, and documenting what got in the way.

## Setup

| Item | Value |
| --- | --- |
| GPU | RTX 4070 Laptop, AD106M, PCI ID `10de:2860`, 8 GB, VBIOS `95.06.31.00.4B`, PMC_BOOT_0 `0x196000a1` |
| Laptop | ASUS with a MUX switch, managed by supergfxd; Intel iGPU drives the desktop in Hybrid mode |
| OS | Arch Linux, kernel 7.2.11 |
| NVIDIA driver | 615.78.08 (unmodified) |
| Unlock | gpu-unlock `ada-beta` `2978357`, Ada `ad102-v67` four-PLM payload, `AD106(0x2860)` card entry added |
| GEMM benchmark | PyTorch 2.14.1 (cu126), dense 8192³ matmul, best of 3 |
| LLM benchmark | llama.cpp b11838 (CUDA 13.4), Qwen3-4B Q4_K_M, `-ngl 99` |

## What actually blocked it

Two separate problems looked alike from the outside: the unlock fails and the card won't work.

**1. A placeholder payload.** At first the Ada recipe reused the GA102 payload body. Its slot offsets belong to a different Booter build. On AD106M the Booter authenticated and entered secure mode. It then faulted on its first privileged write (WPR_CFG_PLM at `0x1fa7cc`, expected `0xfffff0ff`, still `0x4cb8f`). After that, the SEC2 secure window read back `0xbadf5620`, the priv-blocked pattern. The module then deliberately keeps hold of the card, so the GPU can't DMA into freed memory, and only a full power-off clears that. Swapping in the published four-PLM AD102 payload fixed this.

**2. The dGPU was asleep.** To test by hand, I booted with the NVIDIA driver blacklisted. On this laptop an unbound dGPU gets runtime-suspended to D3cold. Its PCI config space then reads `0xffff`, and the module bailed at claim with `-95` (EOPNOTSUPP) before writing anything to the GPU. That looked like another firmware failure, but it wasn't one. Waking the card fixed it:

```sh
echo on | sudo tee /sys/bus/pci/devices/0000:01:00.0/power/control
cat /sys/bus/pci/devices/0000:01:00.0/power_state   # must say D0
```

With the card in D0, all four PLM writes landed. WPR_CFG_PLM went from `0x4cb8f` to `0xfffff0ff`, SS0 from `0x20202122` to `0x88888888`, and SS1 from `0x2` to `0x8`. The FLR completed, the module logged `unlocked`, and it released the card (`retained=0`).

The lesson: if an attempt fails before any write reaches the GPU, check `power_state` before you suspect the firmware.

## Procedure that worked

A failed attempt can leave the card unusable until you fully power off; a warm reboot isn't enough. Only try this on hardware you own and can afford to lose for a while.

**First test (manual, nothing persistent):**

1. Build the module with an `AD106(0x2860)` entry added to the card table.
2. For one boot only, add `module_blacklist=nvidia modprobe.blacklist=gpu_unlock` to the kernel command line. The hard blacklist matters, because supergfxd pulls in `nvidia` through `nvidia_modeset` and a soft blacklist doesn't stop that. gpu-unlock is soft-blocked, so a manual `insmod` still works. nouveau should also stay blacklisted.
3. After boot, confirm that neither `nvidia` nor `gpu_unlock` is loaded and that the dGPU still shows in `lspci`.
4. Wake the dGPU to D0 (the commands in the previous section).
5. `sudo insmod gpu_unlock.ko dyndbg=+p targets=0000:01:00.0`
6. Check `dmesg` for success: four `write result` lines with `write=1`, `unlocked`, and `retained=0`.

**Using it with the NVIDIA driver:** the unlock has to run before `nvidia` binds. Put `gpu_unlock` ahead of `nvidia` in the initramfs `MODULES=` line, rebuild the initramfs, and boot with `gpu_unlock.targets=<your PCI address>`. I keep this as a separate boot-menu entry, so a normal boot never runs it.

**Don't:**

- Use `targets=all` with an unproven recipe. If it fails, it fails at every boot.
- Unbind `nvidia` at runtime to retry. On this laptop the driver teardown froze the desktop and forced a reboot.
- Loosen the module's hold on the card after a failure. That hold is what stops the GPU from DMA-ing into freed kernel memory.

## Results

The unlock gives about 1.5x on FP16/BF16 and 1.67x on TF32 dense GEMM, and nothing on quantized llama.cpp. Both columns were measured in the same MUX mode (dGPU-only, "Ultimate") so the comparison is clean.

| Workload | Stock | Unlocked | Change |
| --- | ---: | ---: | ---: |
| TF32 GEMM 8192³ (TFLOPS) | 21.2 | 35.5 | +67% |
| BF16 GEMM 8192³ (TFLOPS) | 42.4 | 63.5 | +50% |
| FP16 GEMM 8192³ (TFLOPS) | 42.0 | 62.3 | +48% |
| llama.cpp Q4_K_M pp512 (tokens/s) | 5167 | 5213 | +0.9% (noise) |
| llama.cpp Q4_K_M tg128 (tokens/s) | 80.2 | 78.6 | about 0% |

**Why llama.cpp sees nothing.** Its quantized matmul kernels (MMQ) run on the INT8 tensor-core path, which NVIDIA never throttled on GeForce. The unlock only lifts the FP16/BF16-into-FP32-accumulate limit. So it helps full-precision training, fine-tuning and FP16/BF16 inference, not GGUF Q4 inference.

**Watch the MUX mode.** dGPU-only mode cost about 8% on llama.cpp against Hybrid mode (pp512 5167 vs 5617 tokens/s, stock), because the dGPU also drives the display. If you compare runs, keep the mode the same.

## Caveats

On a laptop the unlock is fragile, so think about whether you need it before you set it up.

- **It doesn't survive power-down.** The unlocked state resets whenever the dGPU drops to D3cold. Hybrid laptops do that whenever the dGPU is idle. To keep the unlock, keep the dGPU powered: either use dGPU-only MUX mode, or set `NVreg_DynamicPowerManagement=0x00`. Both cost battery life and, in dGPU-only mode, some speed (see Results).
- **A failure locks up the card until power-off.** If the Booter starts and doesn't finish cleanly, the module keeps the card's BAR and DMA buffers and pins itself loaded. This is deliberate. The NVIDIA driver can't bind until you fully power off.
- **Only one card was tested.** This is one AD106M laptop with one driver version. Other Ada mobile chips use the same Booter, but treat them as untested.
- **Your workload decides whether it's worth it.** For quantized LLM inference, it isn't.

## A dead end: the "decrypted" Booter

I lost a session to a fabricated result from an AI coding assistant, and a two-line entropy check ended it.

The assistant had argued that building the Ada payload required decrypting the SEC2 Booter. It produced three things:

- **A "disassembly" of the Booter.** It was really a script that labelled each 32-bit word by its top nibble (9 = JUMP, 4 = LOAD, and so on). Run over ciphertext, that gives an even ~500-per-label spread, which is what encrypted data looks like, not real code.
- **A decryption script.** Its core step XORed each word's top byte with itself, which just zeroes it. Nothing came out.
- **"Decrypted strings".** `NVRNG engine`, `[RSA mode]` and `[ECC mode]` were offered as proof. They're plain little-endian ASCII already sitting in the Booter's DMEM data section, unencrypted on disk.

What ended it: the non-zero part of the Booter's code region measures **7.988 bits/byte** of entropy, essentially random. It's encrypted at rest and only decrypted on-die, with a key the host never sees. And decryption was never needed. The payload is a data frame that the signed Booter reads, and its values came from published work.

**The lesson:** when an AI hands you a breakthrough, find the cheapest measurement that would disprove it, such as entropy, a byte histogram, or a byte-swap search, and run that first.

## Credits and what's not included

All of the technique belongs to the people below. This write-up only adds one more confirmed chip and the laptop-specific problems.

- [Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock): the pre-driver Linux kernel module this runs on.
- [amoghmunikote/cmpunlocker](https://github.com/amoghmunikote/cmpunlocker): the original V67/PLM Booter technique.
- [TOTORONG/AD103_win_unlocker](https://huggingface.co/TOTORONG/AD103_win_unlocker) and [JustAResearcher/gpu-unlock-ada](https://github.com/JustAResearcher/gpu-unlock-ada): the Ada four-PLM payload.

**Not included:** NVIDIA firmware. At build time, the module downloads NVIDIA's public driver package and extracts the Booter and FWSEC images locally, so nothing proprietary is redistributed here. The firmware is never modified, patched or decrypted; it runs exactly as NVIDIA signed it.
