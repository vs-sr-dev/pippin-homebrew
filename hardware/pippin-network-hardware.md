# Pippin networking hardware

How a 1996 Bandai Pippin actually got onto the Internet. This is poorly documented anywhere, so it's worth stating plainly: the Pippin had **no native Ethernet** and reached the Internet over a **modem on its serial/GeoPort connector**, driven by a Japanese "Internet Kit". Understanding this chain is a prerequisite for the emulation work in the rest of this repository.

---

## What the Pippin had

- **No built-in Ethernet.** Wired networking existed only via a rare expansion dock with an RJ45 10BASE-T port (MSRP ~$139) — a prototype/near-unreleased accessory, very scarce. For practical purposes, the Pippin has no Ethernet.
- **A GeoPort serial connector** (Mini-DIN-9). This is the modem path and the one that mattered.
- **An AAUI network slot** and standard Mac ADB/serial I/O.

## How it went online — the modem path

Internet access was a **modem hanging off the GeoPort serial port**, connected to a phone line over RJ11:

- an **internal 14.4k** modem, or
- external modems — 28.8k ModemSURFR, 33.6k Atmark — on the serial port.

The "Netscape" found on the Japanese discs is the **Internet Kit**: Netscape Navigator 1.12 (later 2.01) running on Apple's Mac networking stack plus a PPP layer ("Pip PPP", a MacPPP-family dialer). The full software chain, top to bottom, is:

```
Netscape  →  MacTCP / Open Transport  →  Pip PPP (MacPPP family)  →  modem  →  phone line  →  ISP
```

This is an ordinary mid-90s Mac dial-up Internet stack, which is exactly why it can be reconstructed under emulation: every layer is a known Mac OS component.

## Why this shapes the emulation work

Because the Pippin's only realistic network path is "serial port → modem → PPP", emulating it does **not** require emulating a network card. It requires:

1. an emulator serial backend that carries bytes off the emulated machine (DingusPPC's ESCC channel A is the Mac modem port — see [`../dingusppc/networking-recon.md`](../dingusppc/networking-recon.md));
2. a **virtual modem** on the host that answers Hayes `AT` commands and bridges to TCP, plus a **PPP server** to give the Pippin an IP and route it to the Internet (see [`../dingusppc/ppp-setup.md`](../dingusppc/ppp-setup.md));
3. the Mac-side stack (MacTCP + a PPP dialer + a browser), all already present on the existing Pippin discs.

The Mac side already had everything needed: inspecting the Japanese disc's System Folder shows **MacTCP**, an **@WORLD-Dialer** extension (a MacPPP-family `APPP`-creator dialer), and **Netscape**. So real Internet access on the emulated Pippin was feasible with the *existing* disc image — the work was entirely on the emulator and host sides.

## ROM revisions and homebrew (context)

Three consumer ROM revisions exist (1.0, 1.2, 1.3). **ROM 1.3 has no authentication check** and boots unsigned homebrew discs directly. On 1.0/1.2, an unsigned disc is rejected unless you chainload through Keith Kaisershot's signed **Pippin Kickstart** (GPLv2), which boots as the first CD and hands off to the first bootable unsigned Mac volume. This matters here because the networking discs used in testing were built by adding apps (a TCP utility, the dialer, the browser) to a homebrew-built HFS volume.
