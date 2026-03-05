# CAEN A3818 Linux Driver — Patched

Patched version of the CAEN A3818 PCI Express CONET2 optical link driver for Linux.

The original driver (v1.6.12) from [CAEN SpA](https://www.caen.it/products/a3818/) contains several bugs that cause kernel panics and system freezes under high-throughput DAQ workloads. This repository documents and fixes those bugs.

## Quick Start

```bash
# Build
make

# Install (replaces system driver)
sudo rmmod a3818 2>/dev/null
sudo insmod src/a3818.ko

# Persistent install
sudo cp src/a3818.ko /lib/modules/$(uname -r)/updates/dkms/a3818.ko
sudo depmod -a
sudo modprobe a3818
```

## Bugs Fixed

### v1.6.12-delila1 (2026-02-24)

| # | Severity | Bug | Impact |
|---|----------|-----|--------|
| 1 | CRITICAL | `app_dma_in` buffer overflow (1MB too small) | Kernel panic (memcpy hits guard page in IRQ context) at >524K events/s |
| 2 | HIGH | No bounds check in `a3818_dispatch_pkt` before `memcpy` | Heap corruption on oversized packets |
| 3 | HIGH | Off-by-one in interrupt handler loop | Reads uninitialized memory for last link |
| 4 | MEDIUM | Semaphore leak in `IOCTL_SEND` error path | Deadlock on subsequent IOCTL calls |
| 5 | MEDIUM | Unbalanced `up()` in `IOCTL_RECV` | Semaphore count corruption |
| 6 | HIGH | No PCIe failure detection (`0xFFFFFFFF` read) | `writel` to dead device, undefined behavior |

### v1.6.12-delila2 (2026-03-05)

| # | Severity | Bug | Impact |
|---|----------|-----|--------|
| 7 | CRITICAL | `spin_lock` + `msleep` in `a3818_open` | `BUG: scheduling while atomic` → system freeze |

## Bug 7 Detail: scheduling-while-atomic

`a3818_open()` acquires `spin_lock(&CardLock)` then calls `a3818_reset_onopen()`, which internally calls `msleep(10)` and `msleep(1)`. Calling `msleep()` (which invokes `schedule()`) while holding a spinlock is a kernel bug — spinlocks disable preemption, and sleeping in that state triggers `BUG: scheduling while atomic`.

**Production trigger:** An optical link failure causes all digitizer readers to disconnect simultaneously. The subsequent reconnection storm (12 concurrent `a3818_open()` calls at 1-second intervals) amplifies the bug, causing repeated kernel BUGs and system freeze.

**Fix:** Replace `spin_lock(&CardLock)` with a dedicated `mutex_lock(&GTPResetMutex)` for the GTP reset section. The mutex allows safe sleeping. Double-checked locking on the `GTPReset` flag prevents redundant resets.

`CardLock` cannot simply be converted to a mutex because it is also used in the IRQ handler (`a3818_interrupt`).

## Full Analysis

See [docs/analysis.md](docs/analysis.md) for detailed analysis of all bugs, including lock architecture, threshold calculations, and dmesg evidence.

## Environment

Developed and tested in the [DELILA DAQ](https://github.com/aogaki/delila-rs) project:
- Hardware: CAEN A3818 (4-link PCIe optical controller) + VX1730B/VX2730 digitizers
- OS: Ubuntu 22.04 LTS, kernel 6.8.0
- Workload: 12 digitizers, 5+ MHz aggregate event rate, 10+ hour runs

## License

Original driver copyright CAEN SpA. Patches are provided for the benefit of the community.
