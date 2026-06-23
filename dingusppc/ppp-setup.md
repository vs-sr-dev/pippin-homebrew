# Host-side modem + PPP chain — putting the Pippin on the Internet

Once DingusPPC's serial port carries real bytes over a host socket (see [`dbdma-rx-fix.md`](dbdma-rx-fix.md)), the host has to look like a **modem** and then like an **ISP**. This document records that chain: a Python Hayes virtual modem, a WSL `pppd` + NAT server, the Mac-side configuration, the build recipe for DingusPPC on Windows, and the operational gotchas that cost real time.

The end-to-end path that carried the emulated Pippin onto the live Internet:

```
Pippin MacTCP → @WORLD-Dialer (MacPPP) → port .A → ESCC → DBDMA
  → socket 1232 → vmodem → 127.0.0.1:1234 → WSL socat → pppd → ppp0 → NAT → Internet
```

---

## DingusPPC build recipe (Windows, MSYS2/MinGW — no vcpkg/MSVC)

Reusable, since the official Windows path assumes vcpkg/MSVC. MSYS2 mingw64 already has g++/cmake/ninja/SDL2.

1. Clone the repo; `git submodule update --init --recursive thirdparty/cubeb` (capstone not needed; the 68k debugger is off by default).
2. Patch `CMakeLists.txt`: add an `elseif (WIN32 AND MINGW)` branch that does `find_package(SDL2 REQUIRED)` + `SDL_MAIN_HANDLED` (skipping the forced vcpkg of the plain `WIN32` branch), and link `ws2_32`.
3. Configure: `cmake .. -G Ninja -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ -DCMAKE_BUILD_TYPE=Release -DUSE_JACK=0`. **`-DUSE_JACK=0` is mandatory** — MSYS2 ships `jack/jack.h`, so cubeb tries to build its JACK backend, which needs `dlfcn.h` (absent on Windows).
4. Build with **`ninja -j2`** (the per-core default hits `cc1plus: out of memory` at ~32 MB/process). Output: `build-mingw/bin/dingusppc.exe`.
5. Select the backend at runtime: `--serial_backend socket` (every property registers as `--<name>`).

Run two CDs on the MESH bus (Kickstart first as chainloader, then the homebrew volume), separated by `:`, with **relative paths** (the list parser splits on `:`, so a `d:\…` drive letter aborts):

```
dingusppc.exe -b ..\..\kinka13.rom -m pippin --cdr_img2 ..\..\isos\ks.iso:..\..\isos\ciao.iso -r
```

## The virtual modem (`vmodem.py`)

A Hayes modem in Python that connects as a **client** to the emulator's listener (`127.0.0.1:1232` — DingusPPC is the server):

- **Command mode:** `AT`, `ATZ`, `ATE`, `ATH`, `AT&F`, `ATS…` → `OK`; `ATE1` echo.
- **Dial:** `ATDT host:port` opens a TCP connection (tcpser-style); `ATDT0` is a loopback echo for testing. In practice `resolve_dial` maps *any* dialed number to the PPP server, because the Pippin's dialer is configured for a dead @World number and the read-only disc can't be reconfigured — so whatever it dials, it connects.
- **Data mode:** transparent byte bridge (carries the PPP HDLC frames). `+++` escape with guard-time (CR/LF treated as neutral so a stray Enter from a terminal doesn't trip it); `ATH` → `NO CARRIER`.

A standalone self-test exercises the whole `AT` surface (dial loopback + TCP, data, `+++`, hangup) with no emulator in the loop.

## The PPP server (WSL)

`pppd` + `socat` in WSL2 (Ubuntu). The chain: vmodem connects to `127.0.0.1:1234` → WSL `socat` → `pppd` on a pty → `ppp0` → `iptables` MASQUERADE NAT → Internet. `pppd` assigns server `192.168.7.1`, Pippin `192.168.7.2`, `noauth`. **Verified:** connecting raises an `LCP ConfReq` (HDLC `7e ff 7d23 c021 …`), `pppd` brings up `ppp0`.

**DNS:** the Pippin's hard-coded @World DNS servers are dead and the disc is read-only, so instead of reconfiguring the Mac, a WSL `iptables` rule DNATs all port-53 traffic on `ppp0` to `1.1.1.1` (`PREROUTING … --dport 53 -j DNAT --to 1.1.1.1:53`, UDP + TCP). This makes name resolution Just Work without touching the guest.

### WSL gotchas (each cost time)

- `sudo` prompts non-interactively under WSL → use `wsl.exe -u root -e bash -lc "…"`.
- Install `ppp socat iptables` (Ubuntu 24.04 doesn't ship them; iptables is nf_tables).
- Export `PATH=/usr/sbin:/sbin:…` — `pppd`/`iptables` live in `/usr/sbin`.
- `socat EXEC` with spaces fails ("wrong number of parameters") → put `pppd` in a wrapper script and `EXEC:/tmp/pppd_wrap.sh,pty,ctty,raw,echo=0`.
- Use **`TCP4-LISTEN`** (plain `TCP-LISTEN` binds IPv6) so WSL2 exposes `127.0.0.1:1234` to Windows.
- Keep the server **foreground inside a background task** — `nohup &` dies when the WSL session returns.
- `/dev/ppp` exists in the WSL2 6.6 kernel; PPP is supported.

## The Mac side

The Japanese disc's System Folder already has everything: **MacTCP**, the **@WORLD-Dialer** extension (a MacPPP-family `APPP`-creator dialer), and **Netscape**. Configuration: MacTCP → select the PPP link, "Server" addressing (PPP assigns the IP); the dialer dials via modem port `.A`. The Pippin dials → vmodem `CONNECT` → PPP negotiates LCP/IPCP → Pippin gets `192.168.7.2` → online. (The Mac `ProtRej`s CCP and IPv6CP, which is normal.)

## The result

- **`ping 1.1.1.1` → 10/10 packets, 0% loss, ~0.10 s.** A 1996 Pippin, emulated, on the real Internet.
- **Netscape 1.1N loaded `frogfind.com` from the live web**, rendered on screen.

For text browsing this is fully working. For the image-loading caveat — which is **not** in this chain but in the browser itself — see the verdict in [`networking-recon.md`](networking-recon.md), and the homebrew browser that sidesteps the whole vintage stack in [`../ppnet/browser-architecture.md`](../ppnet/browser-architecture.md).

## A recurring operational trap

`CharIoSocket` accepts **one** client. On Windows, killing a backgrounded Python by its bash `$!` PID often misses the real process, so ghost `vmodem` instances accumulate and contend for the socket — symptom: "data mode doesn't echo". Rule: before each launch, kill stray Pythons by real Windows PID, keep exactly one vmodem, and verify `netstat -ano | grep 127.0.0.1:1232` shows a single `ESTABLISHED` pair. If the listener backlog jams, restart the emulator to clear the socket.
