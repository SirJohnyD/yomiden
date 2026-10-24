# Licensing Yomiden

*Not legal advice — a practical checklist based on how these licenses are normally applied.*

## Where things stand

| Component | License | How Yomiden uses it |
|---|---|---|
| Mihon (this fork's base) | Apache-2.0 | Code is in this repo |
| LNReader | MIT | Ideas/UX; any code ported must keep its MIT notice |
| LNReader plugins | Their repo's license | Downloaded by users at runtime, **not** bundled |
| Lightnovel Crawler (lncrawl) | GPL-3.0 | Bundled only in the `full` flavor; contacted over HTTP in remote mode |
| sherpa-onnx | Apache-2.0 | Library dependency |
| Kokoro-82M model | Apache-2.0 | Downloaded at runtime |
| Piper voices | Varies per voice | Downloaded at runtime; show each voice's license in the voice picker |
| Google ML Kit | Google's ML Kit terms (proprietary SDK) | Library dependency — see note below |

## Decision: release Yomiden as GPL-3.0-or-later

Bundling lncrawl's Python code makes the shipped app a combined work under GPL-3.0. Apache-2.0 and MIT code may be included in a GPL-3.0 project, so the whole app can be GPL-3.0. The alternative (stay Apache-2.0) only works if Yomiden never bundles lncrawl code.

## What to do

1. **Keep Mihon's license.** Do not delete or edit the existing `LICENSE` text for Mihon's code. Move it to `licenses/APACHE-2.0-mihon.txt` and keep every existing copyright header in Mihon's files.
2. **Add the GPL.** Put the full GPL-3.0 text in `LICENSE` (copy from https://www.gnu.org/licenses/gpl-3.0.txt) when the first GPL code (lncrawl) lands — or now, if you want to commit to GPL from day one.
3. **Header for new files** you write:
   ```
   // Copyright (C) 2026 Kagen Systems
   // SPDX-License-Identifier: GPL-3.0-or-later
   ```
   (Use `Apache-2.0` instead if you choose to keep new code Apache-licensed; GPL-3.0 accepts Apache-2.0 code.)
4. **Mark changes** to Mihon files (Apache-2.0 §4b): a `// Yomiden: <what changed>` comment where you modify them is enough.
5. **NOTICE & third-party notices:** keep `NOTICE` and `THIRD_PARTY_NOTICES.md` up to date and show them in-app under *More → About → Open-source licenses* (Mihon's About screen already has a licenses page — add the new libraries and models there).
6. **Source availability (GPL):** every APK you publish must have its matching source available. Publishing APKs from this public repo's GitHub Releases, built from a tagged commit, satisfies this.
7. **Name & branding:** don't use the Mihon, Tachiyomi, or LNReader names/logos as Yomiden's branding. Apache-2.0 doesn't grant trademark rights. Change the app name, icon, and `applicationId` (`app.mihon` → `com.kagensystems.yomiden`) in Phase 0.
8. **ML Kit:** it's a proprietary Google SDK. Distributing a GPL app that links it is common but debated, and it blocks F-Droid. To stay clean, build the OCR module behind an interface and offer a `foss` flavor using Tesseract (Apache-2.0) instead.
9. **Content disclaimer:** keep the "no content bundled, not affiliated with any source" statement in the README and the app's About screen.

## Files in this repo for licensing
- `LICENSING.md` — this file
- `NOTICE` — attribution summary
- `THIRD_PARTY_NOTICES.md` — per-component credits and licenses
