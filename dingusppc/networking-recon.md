# DingusPPC networking — recon, the road online, and the falsifying verdict

This is the narrative thread of the Pippin networking work: what state DingusPPC's serial path was in, how it was brought from "unimplemented" to an emulated Pippin **pinging the live Internet and loading a real web page**, and the carefully-instrumented detective work that ended in a result which **falsifies** the obvious explanation for the one remaining failure.

The deep emulator fix is documented separately in [`dbdma-rx-fix.md`](dbdma-rx-fix.md); the host-side modem/PPP chain in [`ppp-setup.md`](ppp-setup.md). This file is the map that connects them.

DingusPPC is GPLv3 — see the repository [README](../README.md) for the licensing implications of the code changes referenced here.

---

## Starting point: networking "currently unimplemented", but the architecture is clean

DingusPPC's manual lists networking under *Currently Unimplemented* — no NIC, no modem. The only way bytes leave the emulated machine is the **serial backend**. The good news from the recon: that serial architecture is clean and the blockage was small and localised.

- `CharIoBackEnd` (`devices/serial/chario.h`) is a 6-method virtual interface (`xmit_char` / `rcv_char` / `rcv_char_available` / …). Backends: `CharIoNull`, `CharIoStdin`, `CharIoSocket`.
- `EsccController` (`devices/serial/escc.cpp`) models the Zilog 8530 SCC with two channels. **Channel A's backend is configurable** via the `serial_backend` property; on a classic Mac, **channel A is the modem port** — exactly where the Pippin's modem lives.
- The data path `send_byte → chario → xmit_char` (and the reverse) is backend-agnostic: if a backend produces and consumes bytes, the Mac OS Serial Driver sees a real serial stream on the modem port.

So the Pippin already had a modem *port*; what it lacked was anything on the other end of it, on Windows.

## The road online, in stages

### Stage 1 — a Windows socket backend for the serial port

On Windows there were three `#ifdef _WIN32` gates that disabled the socket backend (an empty SOCKET block in `chario.cpp`, and two in `escc.cpp` that excluded "socket" from the property and from `attach_backend()` — the latter producing `ESCC_A: unknown backend ID 2, using NULL` at runtime). With those implemented, `CharIoSocket` became a Winsock2 TCP listener on `127.0.0.1:1232` (bidirectional byte bridge, `MSG_PEEK` disconnect detection). **Verified:** `--serial_backend socket` → the log shows the listener up, a host TCP client connects, and the Pippin's modem port is now a host socket. (Build recipe for DingusPPC under MSYS2/MinGW is in [`ppp-setup.md`](ppp-setup.md).)

### Stage 2 — making the serial DMA actually move bytes

A socket existed, but the Mac Serial Driver is **interrupt- and DMA-driven**, and DingusPPC's ESCC neither raised interrupts nor ran its DMA. This was the real work, and it is documented in full in [`dbdma-rx-fix.md`](dbdma-rx-fix.md). The short version: the ESCC is wired to real **DBDMA** channels through GrandCentral, but the MMIO routing for those channels had been stubbed out. Re-enabling it (plus an RX FLUSH handler and the right `resCount` bookkeeping) produced a complete bidirectional chain — `Serial Driver ↔ ESCC ↔ DBDMA ↔ socket ↔ host` — verified on screen by a serial-terminal homebrew echoing "PIPPIN" back and forth.

### Stage 3 — a virtual modem and a PPP uplink

With raw bytes flowing, the host side needed to look like a modem and an ISP. A Python Hayes **virtual modem** (`AT` command set, dial, data mode, `+++` escape) connects to the emulator's socket; a **WSL `pppd` + NAT** server terminates PPP and routes to the Internet. Full details and gotchas in [`ppp-setup.md`](ppp-setup.md).

### The milestone — the Pippin reaches the Internet

With the chain complete, the emulated Pippin:

- **pinged `1.1.1.1` — 10/10 packets, 0% loss, ~0.10 s**, over `Pippin MacTCP → @WORLD-Dialer (MacPPP) → ESCC → DBDMA → socket → vmodem → WSL pppd → NAT → Internet`;
- and **Netscape 1.1N loaded `frogfind.com` from the live web**, rendered on screen.

(FrogFind, by Action Retro, is a proxy that transcodes modern sites into vintage-friendly HTML — so via it the Pippin can effectively reach the whole modern web. DNS was handled by a WSL `iptables` redirect to `1.1.1.1`, since the Pippin's hard-coded @World DNS servers are long dead and the read-only disc can't be reconfigured.)

This is the headline: a 1996 Bandai Pippin, emulated, on the real 2026 Internet, browsing a live web page.

---

## The one thing that still failed — and the verdict that falsifies the obvious cause

Text pages loaded. **Embedded images did not** — `frogfind.com`'s logo stayed a black box, and FrogFind's search returned "Document contains no data". The obvious suspect was the emulator's freshly-built RX path losing bytes on a burst. That suspicion was **wrong**, and proving it wrong took careful instrumentation.

The investigation is detailed in [`dbdma-rx-fix.md`](dbdma-rx-fix.md); the conclusion belongs here. Across a long live session with `tcpdump` on the PPP interface and RX tracing in the emulator, **five hypotheses were falsified**:

1. **DBDMA interrupt coalescing / pacing** — a real, diagnosed accuracy bug (back-to-back buffer completions collapsing into one level-based IRQ), but pacing it made **no difference** to the image.
2. **"Never fill the buffer" cap** — made things **worse** (suppressed completion IRQs); reverted.
3. **VJ TCP/IP header compression** (`novj`) — no difference.
4. **System RAM** — expanding to ~37 MB let Netscape and a TCP utility coexist, but the image still failed.
5. **MacTCP per-connection receive buffer overflow on the burst** — falsified by extreme stop-and-wait pacing (`tc tbf` at ~one segment/second): the image **still** failed even when packets arrived one at a time, spaced out.

The decisive evidence: with packets paced to arrive singly, the capture shows the server send the `HTTP/1.1 200` header → the Mac **ACK** it → the first GIF data segment arrive → and the Mac send a **TCP RST** ~23 ms later. A host sends RST only **after** receiving a *valid* packet — so the bytes reached the Mac's TCP stack **intact**, and then **Netscape aborted on the image content itself**. Serving the same GIF in multiple header variants (HTTP/1.0 no-Content-Length vs HTTP/1.1 with it), and even a clean freshly-generated GIF87a, all failed identically — so it is the content path in the browser, not headers, not transport.

**Verdict:** the image failure is **not** the emulator and **not** the network. It is **Netscape 1.1N** (March 1995): it reads the GIF's width/height from the header, draws the placeholder box, and does not complete the decode. Netscape 1.1N supported GIF and inline JPEG, so "wrong format" is excluded; the defect is in this specific browser's handling of the image content as delivered.

This is documented at length deliberately. The DBDMA serial fixes are a real, upstreamable contribution; the image symptom that motivated half the debugging turned out to live somewhere else entirely, and a falsification backed by `tcpdump` is worth more than a plausible story. The chosen next step is the homebrew thin-client browser (see [`../ppnet/browser-architecture.md`](../ppnet/browser-architecture.md)), which sidesteps Netscape, PPP and MacTCP altogether by carrying a simple wire format — and a from-scratch GIF decoder — over the proven-reliable serial link.
