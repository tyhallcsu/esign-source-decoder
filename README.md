# esign-source-decoder

> Decode ESign `source[…]` share blobs into plain URLs in the browser, and import each one into SideStore or AltStore with a single tap.

<p>
  <img alt="Top language" src="https://img.shields.io/github/languages/top/tyhallcsu/esign-source-decoder">
  <img alt="Repo size" src="https://img.shields.io/github/repo-size/tyhallcsu/esign-source-decoder">
  <img alt="Last commit" src="https://img.shields.io/github/last-commit/tyhallcsu/esign-source-decoder">
  <img alt="No build required" src="https://img.shields.io/badge/build-none%20required-informational">
</p>

A single static HTML page (`index.html`) that takes an ESign source-list export — the `source[…]` blob ESign produces when you tap **Share** on your source list — and decodes it client-side into the list of original source URLs. Every URL is rendered as a tappable button that opens the SideStore or AltStore URL scheme on iOS, so moving a full ESign source list into SideStore/AltStore is a matter of opening the page on your device and tapping each entry.

No build step, no backend, no dependencies. The page runs offline after first load.

---

## Contents

- [What this is](#what-this-is)
- [How the decoding works](#how-the-decoding-works)
- [Usage](#usage)
- [Running locally](#running-locally)
- [ESign JSON → AltStore format converter](#esign-json--altstore-format-converter)
- [Credits](#credits)
- [Disclaimer](#disclaimer)

---

## What this is

ESign is an iOS IPA-signing app that lets users collect a list of "sources" (AltStore-style JSON repos of sideloadable apps). Its **Share** button exports that list as an opaque-looking string of the form:

```
source[5GHxhb1U7Lc5jIMpumASbN2teg9dyK5EAazzwnfm1/g=]
```

The only existing consumer of this format is ESign itself — paste it into ESign's importer and it expands back into URLs. This tool does the same expansion, in any browser, without ESign installed, and then hands the result to SideStore or AltStore instead.

What the page does, in order:

1. Extracts the base64 payload from inside `source[ … ]`.
2. XOR-decrypts it against the 5055-byte static key embedded in the page.
3. UTF-8 decodes the result and splits on `\n` to get one URL per line.
4. Renders each URL with **Add to SideStore** / **Add to AltStore** / **Copy** buttons.
5. Offers two "Open all" buttons that walk the list and fire each import URL scheme 1.2 seconds apart.

---

## How the decoding works

The `source[…]` format is **not** AES, RC4, or any real cipher. It is a straight byte-for-byte XOR against a fixed 5055-byte key baked into the ESign iOS app binary.

Given ciphertext bytes `C[0…n]` and key bytes `K[0…5054]`:

```
P[i] = C[i] XOR K[i mod 5055]
```

The plaintext is a newline-separated list of source URLs. That's the whole scheme.

### How the key was recovered

The key and algorithm are published in [magic-bdg/Backdoor](https://github.com/magic-bdg/Backdoor) — another open-source iOS signing app that reads ESign's share format for interop. Two files there describe it:

- [`Shared/Magic/esign/ESignRepoParser.swift`](https://github.com/magic-bdg/Backdoor/blob/main/Shared/Magic/esign/ESignRepoParser.swift) — regex-extracts the base64 from `source\[(.*?)\]`, loops `encryptedByte ^ key[keyIndex]` with `keyIndex = (keyIndex + 1) % keyLength`, decodes UTF-8, splits on `\n`.
- [`Shared/Magic/esign/ESignRepoKey.swift`](https://github.com/magic-bdg/Backdoor/blob/main/Shared/Magic/esign/ESignRepoKey.swift) — a `[UInt8]` literal of the 5055-byte key.

This repo's `index.html` embeds the same 5055 bytes as a base64 constant (`ESIGN_KEY_B64`) and reimplements the parser in ~10 lines of JavaScript:

```js
function decodeEsignBlob(input) {
  const m = input.match(/source\[([\s\S]*?)\]/);
  const b64 = (m ? m[1] : input).replace(/\s+/g, "");
  const ct = b64ToBytes(b64);
  const pt = new Uint8Array(ct.length);
  const keyLen = ESIGN_KEY.length;
  for (let i = 0; i < ct.length; i++) pt[i] = ct[i] ^ ESIGN_KEY[i % keyLen];
  return new TextDecoder("utf-8").decode(pt);
}
```

Because XOR is symmetric and the key is static, the same function also encrypts (if you ever want to produce a `source[…]` blob yourself: URL-list → `\n`-join → UTF-8 bytes → XOR with the key → base64 → wrap in `source[ ]`).

---

## Usage

### On your phone (the intended flow)

1. Open `index.html` on your iOS device in Safari. Either host it anywhere static (GitHub Pages, any web server) or see [Running locally](#running-locally).
2. Paste your ESign `source[…]` blob into section 1 and tap **Decode**.
3. Section 2 fills with the decrypted URLs.
4. Tap **Add to SideStore** (or **Add to AltStore**) next to each URL. iOS will hand off to the app.
5. Alternatively tap **Open all in SideStore** — the page will fire each import link 1.2 s apart so SideStore can process them in order.

### From the browser on any platform

Open `index.html` directly (double-click or `file://` load). Paste a blob, decode, and use the **Copy** buttons to grab plain URLs for any other tool.

---

## Running locally

No toolchain. Any static file server works. From the repo root:

```bash
python3 -m http.server 8765
```

Then open `http://<your-LAN-IP>:8765/` from any device on the same network. macOS shortcut to find the IP:

```bash
ipconfig getifaddr en0
```

Or just double-click `index.html` — the `file://` load works for everything except the JSON-converter's remote-fetch feature (which needs an HTTP origin for CORS).

---

## ESign JSON → AltStore format converter

Section 3 of the page (collapsed under a `<details>`) is an in-browser port of [asdfzxcvbn/altSourceConverter](https://github.com/asdfzxcvbn/altSourceConverter). That Nim tool converts an "ESign-style" apps JSON (flat `apps` array, one entry per version of each app) into AltStore's shape (one entry per app with a nested `versions` array, grouped by `bundleIdentifier`). The JS port does the same grouping:

- Fetches a URL you paste (or accepts pasted JSON, for CORS-unfriendly hosts).
- Buckets every app by `bundleIdentifier`, preserving discovery order.
- Emits a `.altsource.json` you can download and host.

This is a completely separate pipeline from the `source[…]` decoder — it operates on already-plaintext JSON.

---

## Credits

- **[magic-bdg/Backdoor](https://github.com/magic-bdg/Backdoor)** — `ESignRepoParser.swift` and `ESignRepoKey.swift` provided the algorithm and the 5055-byte key used here. Without that repo the format would still be opaque.
- **[asdfzxcvbn/altSourceConverter](https://github.com/asdfzxcvbn/altSourceConverter)** — the ESign-JSON → AltStore-JSON grouping logic in section 3 is a direct port from this Nim source.

---

## Disclaimer

This tool reads a publicly shareable export format produced by a third-party iOS app. It does not bypass any DRM, modify ESign, or interact with ESign's servers; it only reimplements a documented XOR decoding over data the user already possesses. Use it with source lists you exported yourself.

No license is attached to this repo yet. Treat it as "all rights reserved" by default until one is added.
