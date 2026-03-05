# Changelog

## v1.6.12-delila2 (2026-03-05)

### Fixed
- **[CRITICAL] scheduling-while-atomic in `a3818_open()`**: `spin_lock(&CardLock)` was held while calling `a3818_reset_onopen()`, which internally calls `msleep()`. This triggers `BUG: scheduling while atomic` and causes system freeze under concurrent reconnection. Fixed by using a dedicated `mutex` (`GTPResetMutex`) with double-checked locking on the `GTPReset` flag.

### Changed
- `a3818.h`: Added `struct mutex GTPResetMutex` to `a3818_state`
- `a3818.c`: `a3818_open()` uses `mutex_lock`/`mutex_unlock` instead of `spin_lock`/`spin_unlock`
- `a3818.c`: `a3818_init_board()` initializes `GTPResetMutex` in both RAW and normal board paths

## v1.6.12-delila1 (2026-02-24)

### Fixed
- **[CRITICAL] Buffer overflow in `a3818_dispatch_pkt()`**: `app_dma_in` buffer was 1MB, insufficient for >524K events/s at 100ms readout cycle. Increased to 16MB (`APP_DMA_IN_SIZE`).
- **[HIGH] Missing bounds check in `a3818_dispatch_pkt()`**: Added check before `memcpy` to prevent heap corruption on oversized packets.
- **[HIGH] Off-by-one in interrupt handler**: Loop used `i = NumOfLink` instead of `i = NumOfLink - 1`, reading uninitialized memory.
- **[MEDIUM] Semaphore leak in `IOCTL_SEND`**: `err_send` error path did not release semaphore, causing deadlock on subsequent calls.
- **[MEDIUM] Unbalanced `up()` in `IOCTL_RECV`**: Extra semaphore release corrupted semaphore count.
- **[HIGH] No PCIe failure detection**: `recv_pkt` did not check for `0xFFFFFFFF` (PCIe read from dead device), leading to `writel` to non-existent hardware.

## v1.6.12 (CAEN original)

- Original release from CAEN SpA
