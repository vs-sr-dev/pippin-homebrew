# pippin-homebrew

Technical documentation for networking the **Bandai Pippin @WORLD** (1996) under emulation — including a set of upstreamable fixes to the **DingusPPC** emulator that put the emulated Pippin **on the real Internet**, and a homebrew thin-client browser for it.

This is a preservation-research effort. The Pippin is barely documented and barely emulated; its networking path had, as far as we can tell, never been brought up in an emulator before this work. The repository records the hardware facts, the emulator fixes (with their GPLv3 obligations made explicit), and an honest account of where the investigation ended — including a result that **falsifies** the obvious explanation.

---

## What is the Bandai Pippin?

The Pippin is a CD-ROM multimedia console designed by Apple and sold by Bandai in 1995–96 (Japanese "ATMARK", US "@WORLD"). It was, in effect, a stripped-down Macintosh: it runs a fork of Mac OS 7.5.2 and uses the standard Mac Toolbox. It was a commercial failure, selling a tiny fraction of projections.

Hardware essentials:

- **CPU:** PowerPC 603 @ 66 MHz.
- **RAM:** 6 MB onboard on the Bandai units (~5 MB usable to applications), expandable.
- **GPU:** "Taos" 8/16-bit VGA, derived from Apple's LC line.
- **ROM:** 4 MB, removable. Three consumer revisions — **ROM 1.3 has no authentication protection**, so it accepts unsigned homebrew discs directly (the others require a signed disc or a chainloader).
- **OS:** Pippin OS, a fork of Mac OS 7.5.x, with an AppleJack gamepad driver.
- **I/O:** a GeoPort serial/modem slot, an AAUI network slot, ADB, serial. **No built-in Ethernet and no internal hard disk.**

Emulation today is via **DingusPPC** (GPLv3), which has had a working `pippin` machine since 2024 — it boots Pippin OS to the Finder and starts retail titles. What it did *not* have was any working networking.

## Why this documentation exists

The Pippin's path to the Internet is unusual and undocumented: it went online over a **modem on the GeoPort serial port**, driven by a Japanese "Internet Kit" (Netscape + a Mac PPP stack). DingusPPC lists networking as "currently unimplemented". Bringing the emulated Pippin online therefore required working at three layers at once — the emulator's serial/DMA hardware, a virtual modem + PPP host stack, and the Mac-side software — none of which was documented anywhere.

This repository captures all of it: the hardware facts, the emulator fixes (upstreamable to DingusPPC), the host-side modem/PPP chain, a homebrew thin-client browser built as an alternative to the fragile vintage stack, and the empirical detective work — including the final, falsifying verdict on why retail Netscape still couldn't load images even after the Pippin was demonstrably online.

---

## What's in this repository

| Path | Contents |
|---|---|
| [`hardware/pippin-network-hardware.md`](hardware/pippin-network-hardware.md) | The Pippin's networking hardware: GeoPort modem, no native Ethernet, the Japanese Internet Kit, and the full software chain from Netscape down to the phone line. |
| [`dingusppc/networking-recon.md`](dingusppc/networking-recon.md) | The recon: DingusPPC's serial architecture (ESCC / `CharIoBackEnd`), the missing Windows socket backend, and the step-by-step path from "unimplemented" to a Pippin pinging `1.1.1.1` and loading a live web page. |
| [`dingusppc/dbdma-rx-fix.md`](dingusppc/dbdma-rx-fix.md) | **The upstreamable contribution.** The serial-over-DBDMA fixes that gave DingusPPC working bidirectional serial I/O, plus a root-cause analysis of DBDMA RX interrupt coalescing. GPLv3. |
| [`dingusppc/ppp-setup.md`](dingusppc/ppp-setup.md) | The host-side chain: a Python Hayes virtual modem + a WSL `pppd`/NAT server, and the Mac-side MacTCP/MacPPP configuration that together carried the Pippin onto the Internet. |
| [`ppnet/ppf-format.md`](ppnet/ppf-format.md) | "Pippin Page Format" — the line-oriented wire format for a thin-client browser, trivially parseable on a 66 MHz 603. |
| [`ppnet/browser-architecture.md`](ppnet/browser-architecture.md) | The thin-client browser architecture: transport/render separation, why it sidesteps TLS and the fragile vintage stack, and how it was prototyped offline. |

---

## A note on the headline result, stated honestly

The investigation's motivating goal — *get the emulated Pippin loading real web pages* — was reached: the Pippin pings the live Internet and **Netscape 1.1N loaded a real web page (frogfind.com) from the live web**. Along the way, the DBDMA serial path in DingusPPC was made to work, which is a genuine, upstreamable emulator contribution.

But the *specific* symptom that drove much of the deep debugging — embedded **images** not loading — turned out, after five alternative hypotheses were falsified with `tcpdump`, **not** to be the emulator's fault, nor the network's. The final verdict places it in **Netscape 1.1N itself**. That falsification is documented in full in [`dingusppc/networking-recon.md`](dingusppc/networking-recon.md), because being wrong in a well-instrumented, reproducible way is part of the contribution.

---

## Part of the Silicon Relics project

This is one of three documentation repositories produced as part of **Silicon Relics**, a long-running cross-console preservation and homebrew effort focused on obscure and failed vintage platforms:

- **[hyperscan-homebrew](https://github.com/vs-sr-dev/hyperscan-homebrew)** — Mattel HyperScan / S+Core 7
- **[ngpc-homebrew-notes](https://github.com/vs-sr-dev/ngpc-homebrew-notes)** — SNK Neo Geo Pocket Color / TLCS-900H
- **[pippin-homebrew](https://github.com/vs-sr-dev/pippin-homebrew)** — Bandai Pippin @WORLD / DingusPPC networking (this repo)

GitHub profile: **[@vs-sr-dev](https://github.com/vs-sr-dev)**

---

## Attribution

This research is a human + LLM collaboration. Stating that plainly is the honest thing to do — and it is also the point, because the interesting result here is what the collaboration itself makes possible.

**This work would not exist without Claude as a technical bridge. The human brings domain knowledge and direction; Claude provides implementation across architectures the human could not otherwise approach.** Neither side reaches these results alone. A domain expert does not, by hand, patch an emulator's GrandCentral DMA routing, diagnose interrupt coalescing in a DBDMA engine from a 6.8 GB log, and stand up a PPP/NAT stack to carry a 1996 console online. And a language model, on its own, does not know the Pippin reached the Internet over a GeoPort modem, that DingusPPC stubbed its serial DMA, or that loading a live web page on emulated Netscape is a result worth the weeks it took.

The split, concretely:

- **Human:** decades of encyclopaedic knowledge of the retro-hardware ecosystem; the judgement to recognise a genuine first *because* the field is known; project direction — what to look for, why it matters, where it's worth digging; running the live emulator and the GUI navigation a headless process can't; and the ground-truth calls.
- **Claude (Anthropic) — Opus 4.x family, via Claude Code:** translating that direction into implementation — the DingusPPC emulator patches, the virtual modem and PPP host tooling, the homebrew browser, the log forensics, and the writing of this documentation.

The human author is a professional technical translator whose field has been heavily affected by AI, and who therefore takes a deliberate position: AI used for **net-new work that displaces no one** — such as bringing an abandoned 1996 console online, which nobody was doing — is the case worth defending. The collaboration is named openly for that reason; visibility is the point, not a disclaimer.

Findings were verified empirically; nothing here is asserted from the model's prior knowledge alone.

---

## License

The original documentation in this repository is **MIT** — see [LICENSE](LICENSE).

**Important — DingusPPC is GPLv3.** The emulator fixes described in [`dingusppc/dbdma-rx-fix.md`](dingusppc/dbdma-rx-fix.md) are modifications to DingusPPC (`dingusdev/dingusppc`), which is licensed **GPLv3**. This repository documents those changes in prose; it does **not** redistribute DingusPPC source files. The changes themselves are intended for upstreaming as a pull request to `dingusdev/dingusppc`, where they carry GPLv3 as required. Any repository that redistributes the modified DingusPPC source must be GPLv3 with complete corresponding source and attribution.
