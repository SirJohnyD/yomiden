# Yomiden

> **読殿 · "Hall of Reading"** — one Android app for manga, manhwa, manhua **and** light/web novels, with natural-sounding offline narration that can read both.

Yomiden is a fork of **[Mihon](https://github.com/mihonapp/mihon)** that adds novel support inspired by **[LNReader](https://github.com/LNReader/lnreader)**, novel sources from **[LNReader plugins](https://github.com/LNReader/lnreader-plugins)** and **[Lightnovel Crawler](https://github.com/lncrawl/lightnovel-crawler)**, a redesigned library, and a rebuilt neural TTS narrator.

| Source project | What Yomiden takes from it |
|---|---|
| **Mihon** (Kotlin, Compose, Apache-2.0) | App foundation — this repo is a direct fork |
| **LNReader** (React Native, MIT) | Novel reader UX ideas; compatibility with its JS plugin ecosystem |
| **Lightnovel Crawler** (Python, GPL-3.0) | Its catalog of novel source crawlers |

> **Status:** planning. This README is the build spec. Nothing below is implemented yet beyond the Mihon base.

---

## 1. Goals & non-goals

**Goals**
- Single native Android app (Kotlin + Compose) for manga, manhwa, manhua, light novels and web novels.
- Keep Mihon's polish: library, updates, history, downloads, backups, trackers.
- A library that shows at a glance what each title is, and whether you have it in more than one format.
- One Play button that narrates **whatever is on screen** — novel text or manga pages.
- Narration that keeps going chapter after chapter without the user touching the phone.
- Reproducible APK builds from GitHub Actions.

**Non-goals (v1)**
- iOS / desktop. Google Play distribution.
- Hosting or bundling content. Yomiden ships with **no sources**; users add repositories.

---

## 2. Architecture

**Base: this Mihon fork + new modules.** No React Native code is merged.

```
┌──────────────────────────────── Yomiden (Kotlin / Compose) ────────────────────────────────┐
│  UI: Library (redesigned, §4) · Updates · History · Browse · More                          │
├────────────────────────────────────────────────────────────────────────────────────────────┤
│  Readers:  Manga reader (Mihon)          Novel reader (new, LNReader-inspired, §7)          │
│                 └──────────── Narrator (one TTS service for both, §6) ───────────┘          │
│                               ├── text path   (novel chapter text)                          │
│                               └── OCR path    (manga page → ML Kit → speech bubbles)        │
├────────────────────────────────────────────────────────────────────────────────────────────┤
│  Domain: Entry { contentType, format } · SeriesLink (same story, multiple formats) · Chapter│
├────────────────────────────────────────────────────────────────────────────────────────────┤
│  Sources: Mihon extensions · LNReader plugin host (QuickJS) · lncrawl bridge               │
├────────────────────────────────────────────────────────────────────────────────────────────┤
│  Data: SQLDelight · download cache · backup (protobuf, extended)                            │
└────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Tech stack

| Area | Choice |
|---|---|
| Language / UI | Kotlin, Jetpack Compose, Material 3 (as in Mihon) |
| DB | SQLDelight (Mihon), new tables/columns below |
| Networking / parsing | OkHttp (Mihon `NetworkHelper`, Cloudflare interceptor), Jsoup |
| JS plugin runtime | QuickJS (`app.cash.quickjs` or `quickjs-kt`) + polyfill layer |
| Python crawler runtime | Chaquopy — `full` build flavor only |
| TTS | sherpa-onnx Android AAR; **Kokoro-82M** and **Piper/VITS** voices, user-selectable |
| OCR (manga narration) | Google ML Kit Text Recognition v2, on-device (Latin, Japanese, Korean, Chinese models) |
| Audio | Media3 `MediaSessionService` (background playback, lock screen, Bluetooth) |
| SDK / ABI | minSdk 26; `arm64-v8a` primary |
| App ID | `com.sirjohnyd.yomiden` (replaces `app.mihon`) |

---

## 4. Library & details redesign

### 4.1 Format model
Every entry gets a **format**, shown as a badge:

| Format | Badge label | Badge color (token) | How it's detected |
|---|---|---|---|
| Manga | `MANGA` | `badgeManga` (red) | Source metadata / Japanese origin / genre tags |
| Manhwa | `MANHWA` | `badgeManhwa` (blue) | Korean origin, "manhwa"/"webtoon" tags |
| Manhua | `MANHUA` | `badgeManhua` (amber) | Chinese origin, "manhua" tag |
| Light novel | `LN` | `badgeNovel` (green) | Novel source + "light novel" tag |
| Web novel | `WN` | `badgeNovel` (green, outlined) | Novel source, no LN tag |

- Detection runs when an entry is added; the user can always override it from the details screen (⋮ → Set format).
- DB: `manga.content_type INTEGER` (`0 = COMIC`, `1 = NOVEL`) and `manga.format INTEGER` (enum above, nullable = unknown).

### 4.2 Series links ("I have both")
The same story often exists as a light novel **and** a manga/manhua adaptation. Yomiden links them.

- New table `series_link(link_id, entry_id, is_primary)` — entries sharing a `link_id` are the same story.
- **Auto-suggest:** after adding an entry, fuzzy-match title + alt titles (+ tracker IDs from AniList/MAL when available) against the library. Show a snackbar: *"Link with <title> (Light Novel)?"*.
- **Manual:** details screen ⋮ → *Link another format* → pick from library or search sources.
- In the library, linked entries collapse into **one card** (the primary), unless the user turns off *Settings → Library → Merge linked formats*.

### 4.3 Library card (grid)
```
┌───────────────────────────┐
│ ┌──┐            ┌───────┐ │   top-left:  unread count badge (unchanged Mihon behaviour)
│ │12│            │ MANGA │ │   top-right: format badge(s), stacked vertically
│ └──┘            ├───────┤ │              one per linked format, e.g. MANGA + LN
│                 │  LN   │ │
│                 └───────┘ │
│        (cover art)        │
│                           │
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│
│ Title of the series       │   title on gradient scrim (Mihon "compact grid")
└───────────────────────────┘
```
- **Left:** unread count (sum across linked formats; long-press shows the split, e.g. "Manga 8 · LN 4").
- **Right:** format bubble(s). Single format → one pill. Linked formats → one pill per format, stacked, max 3 visible (`+1` overflow).
- Pills: 10–11 sp bold caps, 4 dp corner radius, semi-opaque background so they read on any cover. Respect Mihon's existing badge toggles (Settings → Library → Badges → *Show format badges*).
- List mode: same badges shown inline after the title.
- **Filter sheet:** new *Format* section with tri-state chips (Manga · Manhwa · Manhua · LN · WN · Has multiple formats).

### 4.4 Details screen
Tapping a card opens the details screen (Mihon's layout: cover, title, author, status, source, description, genres).

Under the description, the chapter header row becomes:

```
  128 chapters      [ Light Novel | Manhua ]          ⇅  ⋮
```

- The **format switcher** is a segmented button shown only when the entry has linked formats. Labels use the real format (`Light Novel`, `Web Novel`, `Manga`, `Manhwa`, `Manhua`).
- Switching swaps the chapter list, chapter count, download state, and the *Continue/Start reading* button to that format, without leaving the screen. Last-selected format is remembered per series.
- Single-format entries show the chapter count and a small static format chip instead of the switcher.
- *Continue reading* opens the right reader (manga reader or novel reader) for the selected format.
- Optional progress hint under the switcher: *"LN ch. 42 ≈ Manhua ch. 118"* if the user sets a chapter mapping (stretch goal).

---

## 5. Sources

One `SourceManager`, three backends, all mapped to Mihon's `SManga` / `SChapter` with `contentType` set.

```kotlin
interface NovelSource : Source {
    suspend fun getPopular(page: Int): MangasPage
    suspend fun getLatest(page: Int): MangasPage
    suspend fun search(query: String, page: Int, filters: FilterList): MangasPage
    suspend fun getDetails(entry: SManga): SManga
    suspend fun getChapterList(entry: SManga): List<SChapter>
    suspend fun getChapterText(chapter: SChapter): String   // sanitized HTML
}
```

### 5.1 Mihon extensions
Unchanged. Manga/manhwa/manhua sources.

### 5.2 LNReader plugin host (novels, primary)
- User adds plugin repo URLs (format `…/plugins/<tag>/.dist/plugins.min.json`); plugins download as `.js`.
- Run in **QuickJS**, one context per plugin, 30 s per-call timeout.
- Polyfills: `fetchApi`/`fetchText` → OkHttp (shared cookies + Cloudflare), `cheerio` → bundled JS build or Jsoup shim, `dayjs`, `qs`, `urlencode`, `Storage`/`LocalStorage`, `FilterTypes`, `NovelStatus`.
- Stable source ID: `hash("lnr:" + plugin.id)`.

### 5.3 lncrawl bridge (novels, extended catalog)
- **Remote mode (all flavors):** talk to a self-hosted lncrawl server (Docker image, web server on port 8181).
- **Embedded mode (`full` flavor):** Chaquopy + lncrawl `sources/` + `requests`/`beautifulsoup4`/`lxml`; Kotlin adapter calls `search_novel`, `read_novel_info`, `download_chapter_body`. Browser-dependent crawlers disabled.
- `scripts/sync_lncrawl.py` pulls `sources/` at build time and writes `lncrawl_index.json`.
- Duplicate domains across backends: user picks a preferred backend per domain.

---

## 6. Narrator (TTS)

One narrator service, two input paths, two neural engines.

### 6.1 Engines — user picks Kokoro or Piper
**Settings → Narrator → Engine:** `Kokoro` · `Piper` · `System TTS`

| Engine | Quality | Speed | Notes |
|---|---|---|---|
| **Kokoro-82M** | Best, audiobook-like | Slow on budget phones | Many voices / accents; first choice on recent flagships |
| **Piper / VITS** | Natural | Fast on any phone | First choice on older/mid phones; huge voice catalog |
| **System TTS** | Device-dependent | Fast | Fallback; also exposes any installed engine |

- Both Kokoro and Piper are always available; neither is hidden behind the other.
- **First-run picker:** quick benchmark (synthesize one sentence with each installed engine), then recommend one — but the user chooses.
- **Voice picker** per engine: list, *Hear sample*, download, delete, set default.
- **Overrides:** per-series engine/voice (e.g. Kokoro for novels, Piper for manga OCR), set from the details screen ⋮ → *Narrator voice*.
- Mid-chapter engine switch is allowed: the narrator re-synthesizes from the current sentence.
- Models download at runtime (60–350 MB each), never bundled. Local import of sherpa-onnx models supported.

### 6.2 "Play reads what's on screen"
The Play button lives in both readers' bottom bars and does the right thing for the content:

| Reader | What gets read | How |
|---|---|---|
| **Novel reader** | Chapter text from the first visible paragraph | Text path |
| **Manga reader** | Speech bubbles / captions on the current page(s) | OCR path |

**Text path (novels):** chapter HTML → cleaner (strip ads, watermarks like "read at xyz.com", optional translator notes) → language-aware sentence splitter (≤ 300 chars, never splits inside quotes) → synthesis queue.

**OCR path (manga/manhwa/manhua):**
1. Grab the decoded page bitmap(s) currently displayed (or upcoming pages for look-ahead).
2. ML Kit text recognition → text blocks with bounding boxes.
3. **Group blocks into bubbles** (merge blocks whose boxes overlap/are vertically adjacent).
4. **Order bubbles by format:** manga = right-to-left, top-to-bottom; manhwa/manhua/webtoon = top-to-bottom, left-to-right. Honors the reader's direction setting if the user changed it.
5. Filter noise: drop very short strings, page numbers, watermark text, and (optional) sound effects.
6. Feed bubble text into the same synthesis queue.
7. When the page's last bubble finishes → **auto-turn the page** (paged mode) or **auto-scroll** to the next bubble cluster (webtoon mode).
- Highlight: draw a translucent outline on the bubble being read.
- OCR results cached per page so replays and rewinds are instant.
- Known limits (documented in-app): stylized/handwritten fonts and art-text may be missed; works best on typeset scanlations.

### 6.3 Continuous reading across chapters
LNReader stops at the end of a chapter. Yomiden doesn't have to.

**Settings → Narrator → At end of chapter:**
- `Continue to next chapter` (default)
- `Stop`
- `Continue for N chapters` (1–20), then stop
- Works with the **sleep timer** (minutes, or end of chapter / after N chapters).

Behavior when continuing:
- Preload the next chapter (download or fetch text / page images) when the current one is ~80 % narrated.
- Switch the reader to the next chapter **in the background**, mark the finished chapter read (respecting Mihon's "mark read" settings), update history and trackers, and keep speaking — no tap needed, works with the screen off.
- Short audible cue between chapters (optional) + announces the new chapter title.
- If the next chapter is unavailable (offline, source error, not yet released): speak *"No next chapter available"*, stop, and post a notification.
- Never jumps formats (finishing a manhua chapter does not continue into the light novel).

### 6.4 Playback & controls
- Look-ahead synthesis (default 4 sentences/bubbles, adaptive) so Kokoro doesn't stall; optional "pre-render whole chapter while charging".
- Media3 foreground service: notification, lock-screen and headset controls, audio-focus ducking.
- Controls: play/pause, previous/next sentence (or bubble), previous/next chapter, speed 0.5–3.0× without pitch change, pitch, pause length between sentences/paragraphs/pages.
- Pronunciation dictionary (global + per series, regex supported) for names and invented terms.
- Resume from the exact sentence / bubble (`reading_position.tts_index`).
- Stretch: separate dialogue voice for quoted text.

### 6.5 Acceptance criteria
- Starts speaking ≤ 2 s after Play on a mid-range phone with Piper.
- No gaps between sentences with Kokoro on a Snapdragon 7-class phone after the first buffer.
- Narrates ≥ 3 consecutive chapters with the screen off and no interaction when *Continue to next chapter* is on.
- Manga OCR path reads bubbles in the correct order on a typical typeset manga page and a typical manhwa strip.
- Switching between Kokoro and Piper takes effect from the current sentence.

---

## 7. Novel reader
- Sanitized HTML in a WebView with app-controlled CSS (matches LNReader, handles complex markup).
- Themes (light, dark, AMOLED, sepia, custom), fonts (bundled/system/user `.ttf`), size, line height, margins, alignment.
- Scroll and paged modes; volume-key paging; auto-scroll.
- Chapter-end prompt, swipe between chapters, bookmarks, per-chapter progress.
- Long-press: copy, translate (share intent), **Narrate from here**.
- Offline chapter storage; EPUB export (stretch).

---

## 8. Data model summary
| Change | Purpose |
|---|---|
| `manga.content_type` | Comic vs novel |
| `manga.format` | Manga / Manhwa / Manhua / LN / WN badge |
| `series_link` | Links formats of the same story |
| `novel_chapter_text(chapter_id, html, fetched_at)` | Offline novel chapters |
| `reading_position(chapter_id, progress, char_offset, tts_index)` | Exact resume for reading and narration |
| `narrator_override(entry_id, engine, voice_id)` | Per-series engine/voice |
| Backup proto | Adds all of the above; restores stock Mihon backups; LNReader backup import (stretch) |

---

## 9. Repository layout (target)
```
yomiden/
├── app/                  # Mihon app module → com.sirjohnyd.yomiden
├── core/ data/ domain/ presentation-*/ source-api/ ...   # Mihon modules
├── novel/
│   ├── reader/           # novel reader UI
│   ├── lnreader-host/    # QuickJS runtime, polyfills, plugin repos
│   └── lncrawl-bridge/   # Chaquopy adapter + remote client
├── narrator/
│   ├── engine/           # sherpa-onnx: Kokoro + Piper, model manager
│   ├── ocr/              # ML Kit, bubble grouping & ordering
│   └── service/          # Media3 service, queue, continuous-chapter logic
├── scripts/sync_lncrawl.py
├── LICENSING.md · NOTICE · THIRD_PARTY_NOTICES.md
└── .github/workflows/    # build.yml (debug APK), release.yml (signed APK)
```

---

## 10. Building the APK (GitHub Actions)
| Flavor | Contents |
|---|---|
| `lite` | Everything except embedded Python; lncrawl via remote server |
| `full` | `lite` + embedded lncrawl crawlers (Chaquopy) |

- `build.yml`: on push/PR → JDK 17 → Gradle → (full) Python + `sync_lncrawl.py` → `assembleLiteDebug assembleFullDebug` → upload APKs as workflow artifacts.
- `release.yml`: on tag `v*` → decode keystore from secrets → signed release APKs → GitHub Release.
- Secrets: `SIGNING_KEYSTORE_BASE64`, `SIGNING_KEYSTORE_PASSWORD`, `SIGNING_KEY_ALIAS`, `SIGNING_KEY_PASSWORD`.
- Mihon's existing workflows are kept until Phase 0 replaces them.

---

## 11. Licensing
See **[LICENSING.md](LICENSING.md)**. Short version: this fork is Apache-2.0 today (inherited from Mihon). Yomiden will be released as **GPL-3.0-or-later** once lncrawl code is bundled.

---

## 12. Roadmap
| Phase | Deliverable |
|---|---|
| **0 — Rebrand** | App ID `com.sirjohnyd.yomiden`, name/icon, licensing files, CI producing a debug APK |
| **1 — Formats & library** | `content_type`, `format`, badges, filter chips, details-screen format chip |
| **2 — Novels** | Novel reader, novel chapter storage, LNReader plugin host |
| **3 — Series links** | `series_link`, merged library cards with stacked badges, details format switcher |
| **4 — Narrator v1** | Kokoro + Piper engines with picker, model manager, Media3 service, novel text path, continuous chapters |
| **5 — Narrator v2** | Manga OCR path, bubble highlight, pronunciation dictionary, per-series voice, sleep timer |
| **6 — lncrawl** | Remote mode, then `full` flavor |
| **7 — Polish** | Paged novel mode, LNReader backup import, EPUB export, signed releases |

---

## 13. Rules for whoever builds this (human or AI agent)
1. One phase at a time; each ends with CI green and an installable APK.
2. Keep Mihon changes minimal and marked `// Yomiden:`; new code goes in `novel/` and `narrator/`.
3. Never commit content, credentials, or voice models.
4. Keep every upstream copyright header. Add the Yomiden notice only to new files.
5. Tests for: plugin/crawler result mapping, sentence splitter, text cleaner, bubble ordering, series-link matching, continuous-chapter state machine, backup round-trip.

---

*Yomiden is a SirJohnyD project, based on Mihon. Not affiliated with Mihon, LNReader, Lightnovel Crawler, or any content provider.*
