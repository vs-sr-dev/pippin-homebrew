# PPF — Pippin Page Format

PPF is the line-oriented wire format for the homebrew thin-client browser ("Pippin Net"). It is deliberately trivial: one directive per line, so the parser on a 66 MHz PowerPC 603 is a few lines of C and never has to deal with HTML, CSS or a tag soup. The heavy lifting (TLS, HTML/CSS/JS, layout) happens on a modern proxy server; the Pippin receives content already digested into PPF and just displays it.

See [`browser-architecture.md`](browser-architecture.md) for the surrounding thin-client design and why the format looks like this.

---

## The format

One directive per line, distinguished by the first character:

| Line starts with | Meaning | Rendering |
|---|---|---|
| `@` | Page title | Goes to the window title bar |
| `#` | Heading | Bold |
| `>` | Link: `>Label\|url` | Numbered, clickable link |
| *(blank line)* | Spacer | Vertical space |
| *(anything else)* | Paragraph text | Word-wrapped body text |

That's the whole grammar. A link line carries a display label and a target URL separated by `|`; the renderer assigns it a sequential number so it can be followed by keyboard (press the number) as well as by mouse.

### Example

```
@FrogFind Results
#Search results for "pippin"
This is a paragraph of body text. It word-wraps to the
window width and needs no markup of its own.

>Apple Pippin - Wikipedia|http://frogfind.com/read.php?a=...
>Pippin (computer) - everything2|http://frogfind.com/read.php?a=...
```

## Why this shape

- **One directive per line** means the parser is a `switch` on the first byte — no lookahead, no nesting, no state machine. On the target CPU that matters.
- **Links carry their URL inline** (`label|url`), so the client keeps a flat `url[]` array indexed by link number and an on-screen `rect[]` array for mouse hit-testing. No DOM.
- **Everything degrades to a paragraph.** Unknown or malformed lines render as body text rather than breaking the page — robust against a sloppy proxy.

## Where the format lives

PPF is the contract between two halves of the browser:

- the **proxy server** (modern host) fetches a real URL, does all the parsing/layout, and emits PPF;
- the **Pippin client** parses PPF and renders it.

Because the contract is this small, the client was built and verified **before** any networking existed — the proxy side was a stub returning canned PPF pages from memory. When real transport arrived, only the fetch function changed. See [`browser-architecture.md`](browser-architecture.md).

## Natural extensions

The format is intentionally open to growth without breaking the one-directive-per-line rule: an image directive (e.g. `!alt|url` paired with a server-side transcode to a simple bitmap the client can blit), more heading levels, or an editable address-bar directive. Each is a new leading character with its own handler; old clients ignore what they don't recognise because unknown lines fall through to paragraph rendering.
