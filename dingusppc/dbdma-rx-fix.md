# DingusPPC serial-over-DBDMA — the fix, and the RX coalescing analysis

This is the deep technical contribution of the Pippin networking work, and the part intended for **upstreaming to `dingusdev/dingusppc`**. It documents how DingusPPC's serial path went from carrying zero bytes to a fully bidirectional `Serial Driver ↔ ESCC ↔ DBDMA ↔ socket` chain — enough to put the emulated Pippin on the Internet — and a root-cause analysis of an RX interrupt-coalescing accuracy bug found along the way.

> **License:** DingusPPC is **GPLv3**. The changes below are modifications to GPLv3 source (`devices/serial/escc.{h,cpp}`, `devices/ioctrl/grandcentral.cpp`, `devices/common/dmacore` / `dbdma.{h,cpp}`). They are described here in prose; the source is not redistributed in this repository. Upstreaming carries GPLv3 as required.

The classic-Mac serial stack here is the Zilog 8530 SCC (DingusPPC's `EsccController`) wired through Apple's **GrandCentral** I/O controller to **DBDMA** (descriptor-based DMA) channels. On the Pippin, ESCC **channel A is the modem port**.

---

## Why nothing moved: the data path existed, the routing was stubbed

The Mac OS Serial Driver does not poll — it drives transmission and reception through **DBDMA**, started by writing the `RUN` bit in the channel's MMIO registers. The initial symptom was that an app's `FSWrite` to the serial port simply **hung**: the driver issued an asynchronous write and waited for a DMA completion that never came.

The investigation passed through two wrong layers before finding the right one — instructive enough to record:

1. **ESCC interrupts.** First hypothesis: the ESCC never raises IRQs (`escc.cpp` literally had `TODO: implement interrupt`). SCC interrupt emulation was added (per-channel Tx/Rx/Ext interrupt-pending, `RR3` aggregation, the `WR0`/`WR1`/`WR9` enable logic). Correct and upstreamable in itself — **but the wrong layer for this driver**: a runtime trace showed the driver writing `WR1` with `TxIE=0`, using the Wait/DMA-Request bits, not Tx interrupts. The Mac driver transmits via **DBDMA**, not SCC interrupts.
2. **The real layer — DBDMA routing.** GrandCentral connects the ESCC to real DBDMA channels (`escc_a_tx_dma`, etc.). The data path was all present — `EsccChannel::xfer_to`/`xfer_from` → `send_byte`/`receive_byte` → `chario` → socket — and the DBDMA engine itself (`dbdma.cpp`: `interpret_cmd`/`xfer_to_device`/`finish_cmd`/`update_irq`) was complete. **The one missing piece** was the MMIO routing: in `grandcentral.cpp`, the `MIO_GC_DMA_ESCC_A_XMIT/RCV/B…` register cases were **commented out** ("Stubbed out due to serial emulation being unfinished") from a time when `xfer_to`/`xfer_from` didn't yet exist. So the driver's writes to the channel `RUN` bit fell into the void.

## The TX fix (keeper)

Re-enable the four read + four write DMA-ESCC MMIO cases in `grandcentral.cpp` so they route to the channels' `reg_read`/`reg_write`. With that, the chain completes on transmit. **Verified on screen:** a serial-terminal homebrew sends "PIPPIN" (7 bytes, `50 49 50 50 49 4E 0D`) and the host echo process receives it — `Serial Driver → ESCC → DBDMA → socket → host`. The DMA completion IRQ propagates correctly (`OUTPUT_LAST` with the interrupt bit set → `ack_dma_int` for `SCCA_Tx` → reaches the CPU), so `FSWrite` returns.

## The RX fix (keeper)

Receive was harder because the Mac modem driver uses a specific **FLUSH-poll** protocol on its RX DBDMA ring, and two things were wrong:

- A spurious **Rx interrupt** (from an earlier interrupt experiment's `rx_poll`) froze the app the instant data arrived — the driver services RX via DMA, not via an Rx ISR, so an unexpected Rx IRQ wedged it. Fix: `rx_poll` no longer raises an Rx IRQ; instead, each cyclic tick (500 µs) it **drives the RX DMA channel** (`rx_pull`), emulating the SCC asserting W//REQ to feed the buffer between FLUSHes.
- The **FLUSH handler was a no-op.** The driver's RX program is `[NOP][INPUT_MORE req=768 → buffer]`; roughly every 18 ms it issues `FLUSH → abort(RUN=0) → reload CommandPtr → RUN=1 → PAUSE → resume`, using the FLUSH as "tell me how many bytes you have", then reading `resCount` from the descriptor. The handler wrote no count, so the driver re-armed (resetting the queue) before it could read the real value, and the app saw zero. Fix: on FLUSH of an active `INPUT`/`DIR_FROM_DEV` channel, pull the pending bytes (`dev_obj->xfer_from`) and **publish `resCount` + `xferStatus` into the descriptor** (`cmd_desc[12]/[14]`) exactly where the driver reads after a FLUSH. Plus `xfer_from` now respects the descriptor `len` (it was overrunning the buffer), and `xfer_from_device` handles partial transfers and writes back `resCount`.

**Verified on screen:** 5 sends → 35 bytes received (7 bytes each), the app shows "PIPPIN" on every press. The full bidirectional chain is closed, and this is what made the Pippin's PPP uplink — and reaching the live Internet — possible.

These keeper changes (`grandcentral.cpp` MMIO routing, the `dbdma.cpp` FLUSH/`resCount`/partial-transfer handling, the `escc.cpp` `rx_poll` DMA-driving) are the upstreamable serial-DBDMA enablement.

---

## The deeper accuracy bug: RX interrupt coalescing under burst

After the Pippin was online and text pages loaded, large pages and images proved unreliable: a precise, repeatable pattern where the second ~512-byte segment of a response was lost and retransmitted. `768 B` is exactly the DBDMA RX `INPUT` buffer size, which pointed at the buffer/re-arm cycle.

Forensics on a 6.8 GB runtime log found the real structure. The Mac driver does **not** use one fixed buffer — it runs a **4-buffer auto-looping DBDMA ring** (`NOP; INPUT_MORE req=768; STORE_QUAD (i-bit completion marker); … branch → loop`). RX is delivered both via the FLUSH-poll (partials) and via **DBDMA completion IRQs** (full buffers, from the `STORE_QUAD`).

The bug: DingusPPC's synchronous DMA ops (`start`/`resume`/`xfer_from_device`, mutually recursive through a trailing `interpret_cmd()`) **fill and complete several ring buffers within a single op** when a burst ≥ 1 buffer arrives — observed in the log as one `resume()` cascading through multiple `INPUT → STORE_QUAD → branch` cycles. Each completion fires `ack_dma_int(SCCA_Rx)`, but `ack_int_common` is **level-based**: two or more completions before the ISR clears the line collapse into **one** interrupt. The driver services one buffer; the others coalesce and their 768-byte payloads are lost → TCP retransmit → large transfers fail. On real hardware the completions are spaced at the baud rate and never arrive back-to-back, so the bug is emulation-specific.

**Candidate fix (pacing):** mark `escc_a_rx`/`escc_b_rx` as paced; track a `rx_completed_in_op` flag reset at the start of each synchronous op and set when a buffer completes; in `xfer_from_device`, if paced and already completed this op, **early-return**, leaving the freshly-armed buffer parked. That guarantees **at most one completion per synchronous op**, so the guest runs its completion ISR between buffers — no coalescing. The parked buffer is filled on the next `rx_poll` tick or FLUSH; order preserved, throughput still ample (768 B/op, many ops/ms). Non-paced channels (disk/SCSI) are untouched.

## Honest status of the pacing fix

The pacing change is documented as a **diagnosed accuracy bug with a candidate fix, not a shipped one.** In the live investigation it did **not** resolve the motivating symptom (images), because — as established with `tcpdump`, see [`networking-recon.md`](networking-recon.md) — the image failure was ultimately **Netscape 1.1N**, not byte loss. The pacing (and a related "don't fill the buffer" cap, which made things worse) were therefore **reverted to baseline** to keep the keeper fixes clean. The coalescing analysis stands on its own as a real emulator-accuracy issue worth addressing upstream; it simply was not the cause of the image symptom it was chased for.

## Summary for an upstream PR

| Change | File(s) | Status |
|---|---|---|
| Re-enable DMA-ESCC MMIO routing (4 read + 4 write) | `grandcentral.cpp` | **Keeper** — enables serial TX/RX |
| RX FLUSH handler: pull bytes + publish `resCount`/`xferStatus` | `dbdma.cpp` | **Keeper** |
| `xfer_from` respects `len`; `xfer_from_device` handles partials + `resCount` writeback | `dbdma.cpp` | **Keeper** |
| `rx_poll` drives RX DMA channel instead of raising a spurious Rx IRQ | `escc.cpp` | **Keeper** |
| SCC interrupt emulation (Tx/Rx/Ext IP, RR3, WR0/WR1/WR9) | `escc.{h,cpp}` | Correct, but unused by this driver — optional |
| RX completion **pacing** (anti-coalescing) | `dbdma.{h,cpp}`, `grandcentral.cpp` | Candidate — diagnosed bug, reverted from baseline (didn't fix the Netscape symptom) |

Before any PR, the temporary instrumentation must be removed: the verbose `RXDBG`/`DBDMA_RX_TRACE` logging (the 6.8 GB log came from logging every `ch_ctrl` write at INFO — now gated behind a `#define`, default off), the temporary `CHARIO_RX` byte counter, and the elevated `update_irq`/`ack_dma_int` INFO lines (restored to `LOG_F(9)`). Note for reproduction: in interpreter mode without `--log_to_stderr`, loguru INFO goes to `dingusppc.log` in the working directory, not stdout.
