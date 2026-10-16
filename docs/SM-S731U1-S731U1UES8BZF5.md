# Galaxy S25 FE SM-S731U1 / S731U1UES8BZF5

**Status: offline port and exact-kernel audit complete; not hardware-tested.**

This profile ports the S25 FE (`r13s`) payload to the BZF5 firmware and pairs
it with ReSukiSU `v4.2.0-rc3`. No device was connected, flashed, or boot-tested
during this port.

## Firmware identity

| Field | Value |
| --- | --- |
| Model | `SM-S731U1` |
| CSC | `XAA` |
| Codename / product | `r13s` / `r13sue` |
| AP build | `S731U1UES8BZF5` |
| CSC build | `S731U1OYM8BZF5` |
| Four-part FUS version | `S731U1UES8BZF5/S731U1OYM8BZF5/S731U1UES8BZF5/S731U1UES8BZF5` |
| Display build | `BP4A.251205.006.S731U1UES8BZF5` |
| System fingerprint | `samsung/r13sue/essi:16/BP4A.251205.006/S731U1UES8BZF5:user/release-keys` |
| Kernel release | `6.1.157-android14-11` |
| Kernel build | `#1 SMP PREEMPT Thu Jul 2 07:41:33 UTC 2026` |

The FOTA archive contains `ro.system.build.fingerprint` but not
`ro.build.fingerprint`; the recorded fingerprint is the exact system property
from that archive. In this firmware the `meta-data/fota.zip` file is nested
inside the AP tar, rather than being a top-level firmware ZIP member. It was
extracted from the AP tar and passed through the Device-Porter FOTA parser.

## Kernel recovery

The kernel was extracted from the BZF5 AP package using the Device-Porter
pipeline:

| Object | Size | SHA-256 |
| --- | ---: | --- |
| `boot.img` | 67,108,864 | `b7f0848a5fe40c84a7a698075439251a3bf24b2c6fe53dccce0c4a58a2d98a3c` |
| Raw kernel Image | 38,832,640 | `827219dcdbd6c4961ef565838fe7a8d9c15c6f67e713682e12433e94a2e7e3d0` |
| Recovered `vmlinux.elf` | 44,366,383 | `ccc21cadedf084f1f9a8bf5b0b0c3cef3b00ab8072333d8af99cbb545722db9d` |
| Raw BTF | 5,998,390 | `ce67f63f0d8b9311caba368ccb3f2c4fa8693b33b5c855725a8cd13d9122f4a8` |

`vmlinux-to-elf` recovered the ELF load base as `0xffffffc008000000`.
`llvm-nm` produced 115,669 symbol lines. The raw BTF interval in the kernel
Image is `[0x188b248, 0x1e4397e)`. The BTF structure dump and exact-kernel
derivation are retained in the Device-Porter work directory; the committed
derived/scaffolded breakdown is
[`src/targets/r13s-S731U1UES8BZF5/port-report.md`](../src/targets/r13s-S731U1UES8BZF5/port-report.md).

The BZF5 kernel has the same recovered offsets as the BZF3 S25 FE scaffold
for all symbol/BTF-derived values checked by the generator. Two cases needed
explicit review:

- `worker_thread` has `bl schedule` at `0xffffffc0080dbd98`; the following
  instruction is `0xffffffc0080dbd9c`. The second apparent match is
  `schedule_timeout_interruptible`, not `schedule`.
- The BZF5 kallsyms name is `ashmem_miscs`; its recovered offset is
  `0x02484c20`. BTF reports `miscdevice.fops` at `0x10`, so
  `ASHMEM_MISC_FOPS_OFF` is `0x02484c30`.

The original BZF3 header had an incorrect DRAM ceiling. This profile uses the
documented S25 FE device-tested 8-GiB ceiling
`0xffffff8a00000000` for both
`APP_KERNEL_PAGE_KSNITCH_IDENTITY_END` and `APP_RECLAIM_MAX_DIRECT_BASE`, with
`APP_KERNEL_PAGE_KSNITCH_EXACT_PARTITION=1`. The existing BZF3 profile was not
changed.

The physical P0 mapping/load values, tracefs event ID, pselect word shift, and
fake-task/waiter layout remain scaffolded from the BZF3 profile. The Device
Porter report lists them; they still need hardware validation on BZF5.

## Release payload

The app payload was built with Android NDK r28c:

```sh
ANDROID_NDK_HOME=/path/to/android-ndk-r28c \
  make TARGET=r13s-S731U1UES8BZF5 release
```

| Artifact | Size | SHA-256 |
| --- | ---: | --- |
| `artifacts/r13s-S731U1UES8BZF5/cve-2026-43499-app.so` | 104,128 | `6f13459360e5f893d796f67e051ae4def2958b0755af29e224c2b8a4787c9380` |

The output is exactly at the Device-Porter release size limit.

## ReSukiSU pair

The existing ReSukiSU `v4.2.0-rc3` pair for the Exynos 2400 S921B
(`S921BXXSFDZE1`) was reused, not rebuilt. Both source artifacts were copied
unchanged and renamed for the BZF5 profile. Reuse was gated by exact-target
checks against this firmware's recovered kernel:

- Module `vermagic` is exactly `6.1.157-android14-11`.
- The module's `__versions` section is zero-length for its manual-relocation
  loader. `audit_module_against_target.py --manual-relocation` found 220
  undefined imports, zero missing target symbols, 53 imports resolved through
  kallsyms, zero module-version entries, and zero target CRC mismatches.
- `verify_pair.py` accepted the module release and AArch64 daemon.
- The daemon identifies itself as ReSukiSU `4.2.0-rc3 (uapi: 4)`.

| Artifact | Size | SHA-256 |
| --- | ---: | --- |
| `kernelsu/android14-6.1_kernelsu-rsksu-r13s-S731U1UES8BZF5-kdp.ko` | 485,128 | `8fb778fef9f61f5c9a2a770315e2e43c1cd0ba44f26f8e317f78f0ca4254eae8` |
| `kernelsu/ksud-rsksu-r13s-S731U1UES8BZF5-kdp` | 4,387,176 | `f034b18238d6bfdccf2ef01da7d7a531383b3501b83615b306947b925073e8e3` |

The support entry is
`r13s-S731U1UES8BZF5-rsksu420`, declares flavor `resukisu`, version
`4.2.0-rc3`, and requires a fresh P0 session.

## Validation and remaining work

- Release payload built and size-gated.
- Exact firmware kernel release, BTF, ELF base, target symbol offsets, module
  vermagic, manual relocations, and ReSukiSU pair metadata were checked.
- Device-Porter `validate-feed`, `validate-port`, and `validate-analysis`
  gates are run as part of this port.
- No `adb`/hardware validation was available. Do not treat this offline port
  as device-confirmed; test the exploit chain, ReSukiSU module load, root
  control channel, SELinux state, and recovery behavior on an owned device
  before promoting the feed entry.
