# Pippin Net — thin-client browser architecture

A homebrew web browser for the Pippin, built on a deliberate decision: **do not clone an old browser**. A full modern browser on a 66 MHz 603 with ~5 MB of RAM is impossible if everything runs on the Pippin. The answer is a **thin client** — the heavy work (TLS, HTML/CSS/JS, layout) happens on a modern proxy server, and the Pippin receives content already digested into a trivial wire format ([PPF](ppf-format.md)) which it just displays.

This also became the strategic fallback after the Netscape image verdict (see [`../dingusppc/networking-recon.md`](../dingusppc/networking-recon.md)): a browser we fully control, carrying a format we define over the proven-reliable serial link, sidesteps Netscape + PPP + MacTCP entirely.

---

## The architecture

```
Pippin (thin client)                Modern proxy server
─────────────────────               ───────────────────
small C app          ── URL ──▶      TLS + HTML/CSS/JS + layout
parse PPF            ◀─ PPF ──        fetch real page, transcode to PPF
render + interact
```

Two flavours of proxy were envisaged:

- **Text (easy):** the server is a readability proxy — it returns reflowed text + links of today's web as PPF. This is what was prototyped.
- **Screenshot (WRP-style "wow"):** the server renders the page headless, scales it to the 16-bit display, and sends a bitmap + an imagemap of link rectangles. This reuses a fast 16-bit blit pipeline already proven in an earlier graphics demo on this platform.

The point of the split is that **TLS and modern web complexity never touch the Pippin** — it only ever sees simple, pre-digested content.

## The client (verified)

The client is a real Mac application built with Retro68 (PowerPC, `powerpc-apple-macos`) and run on the emulated Pippin. Verified on screen: paginated text, links that work both by **mouse** (click) and **keyboard** (number keys), scrolling, and a Back history.

Implementation notes:

- **Rendering** reuses the off-screen-GWorld + `CopyBits` pipeline from an earlier 3D demo — drawing to a 16-bit off-screen world and blitting, so there's no flicker on scroll or navigation.
- **Layout:** word-wrap via `TextWidth` at the window width; bold headings, body text, blue underlined links shown as `[n]`. Two passes per redraw — measure (content height → clamp scroll), then draw.
- **Links:** URLs stored by number in `g_linkurl[]`; on-screen `Rect`s in `g_linkrect[]` for mouse hit-testing.
- **Event loop:** classic `WaitNextEvent` — `keyDown`/`autoKey`, `mouseDown` with `FindWindow` (`inGoAway`/`inDrag`/`inContent`), `updateEvt`. Keys 1–9 follow links, arrows/space scroll, `B` goes back (history stack), `Esc`/`Q` quits.
- **Window:** `documentProc` with go-away + drag, sized to the screen (capped ~580×380).

## The key design property — transport-ready

The single most important structural choice: **transport and presentation are separate.** In the prototype, "transport" is a stub —

```c
static const char* fetch(const char* url);   // routes 3 in-memory PPF pages
```

— returning canned PPF pages from memory. **When real networking arrives, only `fetch()` changes** (from in-memory pages to a read over the serial link / Open Transport). The PPF parser, the renderer, the link handling, the scrolling and the history all stay identical.

This is why the browser could be built and fully validated **offline**, in parallel with the emulator-side networking work — the two tracks only meet at `fetch()`.

## Building the client

Retro68 via the prebuilt Docker image (`ghcr.io/autc04/retro68:latest`, which carries a working `powerpc-apple-macos-gcc`), since the from-source MSYS2 build of Retro68 is a dead end on this host. A `CMakeLists.txt` with `add_application(...)` produces a MacBinary `.bin`, which a small Python importer (`machfs` + a MacBinary parser) places onto a homebrew-built HFS disc image that boots in DingusPPC. That whole pipeline — C → PowerPC → HFS ISO → Pippin — is owned end to end.

## Where this goes next

The chosen next step after the Netscape verdict is to make this browser the real path online: replace the stub `fetch()` with a serial read, and add the one piece the format will need for images — a **from-scratch GIF (LZW) decoder** on the client (~200–300 lines of C; GIF87a/89a is well documented) — paired with a proxy that transcodes both pages and images into the simple wire format. That carries the whole experience over the serial link that's already proven reliable for modest transfers, with no Netscape, no PPP and no MacTCP in the path.
