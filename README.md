# HFY NEKROCODEXXX

A standalone, self-hosted reader and ebook maker for serialized fiction published on Reddit. What started as a dedicated reader for r/HFY has grown into a general-purpose tool that can fetch story updates from any subreddit, build offline archives from JSONL data dumps, and export complete series as EPUB ebooks for personal use.

---

![Version](https://img.shields.io/badge/version-FX45v31-FF003C?style=for-the-badge)
![Platform](https://img.shields.io/badge/platform-Android%20%7C%20Windows%20%7C%20Web-00FFF0?style=for-the-badge)
![Chapters](https://img.shields.io/badge/chapters-72%2C248-FFE600?style=for-the-badge)
![Series](https://img.shields.io/badge/series-5%2C533-9D00FF?style=for-the-badge)
![CORS Proxies](https://img.shields.io/badge/CORS%20proxies-386-00FF88?style=for-the-badge)
![License](https://img.shields.io/badge/license-Personal%20Use-FF8800?style=for-the-badge)

---

## Table of Contents

- [What Is This](#what-is-this)
- [How the App Evolved](#how-the-app-evolved)
- [How It Works](#how-it-works)
- [Features](#features)
- [Platform Support](#platform-support)
- [Quick Start](#quick-start)
- [How to Add JSONL Files](#how-to-add-jsonl-files)
- [Where to Get JSONL Files](#where-to-get-jsonl-files)
- [Using the App with Other Subreddits](#using-the-app-with-other-subreddits)
- [Creating Ebooks](#creating-ebooks)
- [How the Android APK Was Built](#how-the-android-apk-was-built)
- [How the App Was Made](#how-the-app-was-made)
- [Credits and Acknowledgments](#credits-and-acknowledgments)
- [Known Bugs and Call for Contributors](#known-bugs-and-call-for-contributors)
- [Forking and Personal Use](#forking-and-personal-use)
- [Legal Notice — Ebook Distribution Prohibited](#legal-notice--ebook-distribution-prohibited)
- [System Requirements](#system-requirements)
- [Troubleshooting](#troubleshooting)
- [API Reference](#api-reference)
- [Configuration](#configuration)
- [File Structure](#file-structure)

---

## What Is This

HFY NEKROCODEXXX is a local-first application that runs a Node.js server on your device and serves a React-based user interface to your browser. It is designed for people who read multi-chapter serial fiction on Reddit and want a better experience than the official Reddit client provides.

The application ships with a bundled catalog of 5,533 series containing 72,248 chapters from the r/HFY wiki. This catalog loads into memory at startup, so you can browse, search, and sort immediately without any internet connection. When you open a chapter, the app fetches the content from Reddit using a multi-strategy approach that works around rate limiting. Once fetched, the content is cached locally so you can read it offline.

The app also supports JSONL data dumps — large files containing every post from a subreddit. When you index these files, the app becomes fully offline-capable: all content, scores, and metadata are served from local files, and fetching is nearly instant. You can browse hundreds of thousands of posts, discover new series automatically, and export complete series as EPUB ebooks.

No accounts, sign-ups, analytics, telemetry, or advertising are present. The app does not phone home, does not report usage statistics, and does not require any personal information. It is a tool for reading stories, and that is all it does.

---

## How the App Evolved

This project started with a simple goal: build a better reader for r/HFY stories. The Reddit mobile client is poorly suited for multi-chapter serial fiction. It collapses long comments, injects ads between posts, and provides no sequential chapter navigation. The first versions of this app were strictly an r/HFY reader — they bundled the HFY wiki series index, fetched chapter content from Reddit, and displayed it in a clean, customizable reader.

As development progressed, several things happened that pushed the app beyond its original scope:

**The rate-limiting problem.** Reddit aggressively limits API access. A single account gets 600 queries per minute, which sounds like a lot until you try to C-FETCH (content-fetch) a 900-chapter series. The app evolved to support up to 9 Reddit accounts with automatic round-robin token rotation, giving up to 5,400 QPM. When that was not enough, a pool of 386 public CORS proxies was added as a fallback, followed by Wayback Machine integration and hfy.foundation as additional content sources.

**The offline problem.** Even with all the rate-limit mitigations, the app still needed an internet connection to read anything. If Reddit went down, or if a post was deleted, the content was gone. The solution was JSONL archive integration — if you have a data dump file containing every post from a subreddit, the app can index it and serve all content locally. This turned the app from a network-dependent reader into a fully offline-capable archive reader. The JSONL integration was implemented in 14 parts, covering folder access, indexing, content fetching, score pre-loading, chapter discovery, new UI tabs, download and merge, persistence, and file replacement detection.

**The multi-subreddit realization.** The JSONL indexing engine was designed to work with any subreddit's data dump, not just r/HFY. When you index a file from r/nosleep, r/WritingPrompts, r/sexyspacebabes, or any other subreddit, the app automatically detects the subreddit, adds it to the filter dropdown, and makes all its posts browsable in the Archive tab. The cross-subreddit search and chapter chain following work regardless of which subreddit the posts come from. This means the app can be used to track and read serial fiction from any subreddit, not just HFY.

**The ebook export.** The download and merge features were originally designed for personal offline reading. But once the JSONL integration made content retrieval instant, merging a 1,000-chapter series into a single EPUB went from taking 10+ minutes to taking under 10 seconds. This made the app a practical ebook creation tool for personal use. You can now take any subreddit's JSONL dump, discover all series in it, and export complete ebooks for your own e-reader.

The app went overboard, in the best way possible. What started as an HFY reader became a general-purpose Reddit serial fiction archive reader and ebook maker. The name still says HFY because that is where it started, but the app works with any subreddit.

---

## How It Works

### Architecture

The app follows a client-server architecture within a single Node.js process:

1. **The server** (port 3000) is a Next.js 16 standalone application that serves static assets and handles API requests. It runs on your device — no external server is involved.

2. **The JSONL batch handler** (port 3001) is a lightweight HTTP server that handles bulk operations like saving thousands of scores or chapter contents in a single request. This exists because the main Next.js server processes requests individually, which is too slow for batch operations.

3. **The client** is a React single-page application served to your browser. It contains all the UI components, state management, and the JSONL indexing engine. Most of the heavy lifting happens client-side.

4. **The filesystem** is the persistent data store. All your library, reading progress, favorites, downloaded chapters, and cached scores are stored as JSON files in the `data/` directory. On Android, this is `getFilesDir()/hfy-data/` which survives app restarts.

### Content Fetching — The 9-Strategy Cascade

When you open a chapter, the app tries 9 strategies in sequence to fetch the content. The first successful strategy wins:

1. **JSONL local archive** — If the post exists in your indexed JSONL files, the content is read from the local file using random-access byte-offset reading. This takes less than 1 millisecond and requires no network connection.

2. **OAuth JSON** — Fetches from `oauth.reddit.com` using a Bearer token from your connected Reddit accounts. Each account gets 600 queries per minute.

3. **Cookie-based fetch** — Uses Reddit session cookies you inject manually. This bypasses OAuth rate limits.

4. **Browser simulation** — Sends complete Chrome browser headers (Sec-Fetch-*, Sec-Ch-Ua, Accept-Encoding) with the OAuth token to mimic a real browser visit.

5. **Direct RSS** — Fetches the `.rss` Atom feed for the post. RSS feeds are not subject to the same IP-level restrictions as JSON endpoints.

6. **CORS proxy pool** — Tries up to 10 of the 386 public CORS proxies in rotation. Proxies that fail are marked dead with a 5-minute cooldown.

7. **Direct .json fallback** — Last resort: fetches the `.json` endpoint directly without OAuth.

8. **Wayback Machine** — Queries `web.archive.org` for archived versions of the post.

9. **hfy.foundation** — Queries the community index at `hfy.foundation` for the post.

User-Agent strings rotate across 8 browser profiles (Chrome, Firefox, Safari, Edge on Windows, macOS, and Linux) to reduce fingerprinting.

### JSONL Indexing Engine

When you select a folder containing `.jsonl` files, the indexing engine:

1. Scans the folder for `.jsonl` files using the File System Access API
2. Streams each file line by line using `file.stream().getReader()`
3. Parses each line as JSON and builds four in-memory indices:
   - **Primary index** — Map of postId to post metadata (title, author, score, date, byteOffset, lineLength)
   - **Author index** — Map of author to their posts, total score, and activity dates
   - **Subreddit index** — Map of subreddit to its posts and statistics
   - **File metadata** — Map of fileName to size, modification date, and post count (for change detection)
4. Pre-loads all scores to the server in a single batch POST
5. Reports progress every 1,000 posts

Indexing 162,000 posts takes about 30 seconds. Once indexed, content retrieval is nearly instant because the app uses `file.slice(byteOffset, byteOffset + lineLength)` to read only the needed bytes.

### Chapter Discovery

The app uses several methods to discover chapters in a series:

**Chapter chain following** — Starting from a known chapter, the app parses the post content for links that match "next chapter" patterns (next, next chapter, part N+1, chapter N+1, etc.). It follows the first match, fetches that chapter, and repeats. This is based on the approach used by hfy2epub.

**Gap filling** — When a series has missing chapter numbers, the app searches the JSONL index for posts with titles matching the missing number and the series name.

**Auto-discover pipeline** — An 8-stage pipeline that tries: cache check, Wayback Machine, hfy.foundation, BFS crawl, Wayback crawl, Reddit JSON, Reddit search, and WebSocket parser. Each stage can be enabled or disabled.

**PostId matching** — Cross-references JSONL postIds with the app's existing chapters to find posts that belong to a series but are not yet in the chapter list.

### State Management and Persistence

The app uses Zustand for state management with a three-tier persistence pattern:

1. **localStorage** — Fast, synchronous reads. Used for all state. On Android, this is cleared on restart.
2. **IndexedDB** — Persists across restarts. Used for large data like chapter caches and directory handles.
3. **Server filesystem** — Durable, unlimited size. The authoritative source of truth. Data is stored as JSON files in `data/app-store/`, `data/user-chapters/`, and `data/chapter-cache/`.

On startup, if localStorage is empty (Android restart scenario), the app asynchronously restores state from the server. If any store was restored, the page reloads once to ensure consistency. A sessionStorage flag prevents infinite reload loops.

### C-FETCH System

C-FETCH (Content Fetch) is the system for pre-fetching chapter content and upvote scores:

- **REHASH** — Re-fetches all chapters in a series, even already-cached ones. Useful for detecting when an author has edited a chapter.
- **CONTINUE** — Fetches only unfetched chapters, then crawls forward from the last known chapter to discover new chapters.
- **STOP** — Uses `AbortController` to immediately cancel the HTTP connection.

Chapters are fetched in batches (configurable 1-500, default 5) with a 1-second delay between batches. Progress is streamed to the client as NDJSON (newline-delimited JSON).

---

## Features

### Core Features

- **Bundled catalog** — 5,533 series with 72,248 chapters from the r/HFY wiki, available offline immediately on launch
- **9-strategy content fetching** — JSONL local, OAuth JSON, cookies, browser simulation, RSS, CORS proxy pool, direct JSON, Wayback Machine, hfy.foundation
- **Multi-account OAuth rotation** — Connect up to 9 Reddit accounts for 5,400 QPM (9 x 600)
- **CORS proxy pool** — 386 public proxies with health checking and automatic rotation
- **JSONL archive integration** — Index data dumps from any subreddit for fully offline reading
- **C-FETCH system** — Batch pre-fetch chapter content and scores (REHASH, CONTINUE, STOP)
- **Auto-discover pipeline** — 8-stage chapter discovery with resume capability
- **Chapter chain following** — Automatically follows "next" links to discover all chapters in a series
- **Download and merge** — Export individual chapters or complete series as EPUB or TXT
- **Custom series creation** — Create your own series from any posts in the archive
- **Discovered series** — Auto-detected series from JSONL data that are not in the bundled catalog

### Reader Features

- Adjustable font size (3px to 48px)
- 30+ bundled Google Fonts plus custom font upload
- 22 reader themes (Void, Blood, Matrix, Analog, Ice, AMOLED, etc.)
- 25 app themes (Night City, Arasaka Net, Corpo Gold, Netrunner, etc.)
- Swipe navigation between chapters
- Chapter preloading (configurable 1-100 chapters ahead)
- Automatic progress tracking
- Markdown rendering of chapter content
- Comment content collection (for stories continued in comments)

### Archive Features

- Browse all posts from indexed JSONL files
- Filter by subreddit, author, date, score, and search query
- Sort by top (score), newest, or oldest
- Pagination (20 posts per page)
- Full post view with markdown rendering
- Add any post to an existing or custom series
- Author browser with statistics

### Persistence Features

- Triple storage (localStorage + IndexedDB + server filesystem)
- Atomic file writes (temp file + rename) to prevent corruption
- Persistence audit and repair system
- File replacement detection (automatic re-indexing when JSONL files change)
- Cross-subreddit support (unified index covers all JSONL files)

### Android-Specific Features

- Foreground service with WiFi lock and wake lock
- 250 kbps keepalive to maintain network activity
- Auto-restart on crash (3-second delay)
- Persistent storage at `getFilesDir()/hfy-data/`
- Native folder picker for downloads and JSONL files

---

## Platform Support

### Windows Standalone

The Windows package includes `node.exe` (Node.js runtime, 80 MB) and a `.cmd` launcher that starts the server and opens a standalone Chrome/Edge window in `--app` mode. No installation required — just extract and run.

### Android APK

The Android APK wraps the webapp in a native Android shell using Capacitor. It bundles a native Node.js binary for ARM64, runs as a foreground service with WiFi and wake locks, and displays the app in a WebView. The APK is built by replacing `assets/server/` in a base Capacitor APK with the latest webapp build, zip-aligning, and signing with `apksigner`.

### Webapp (Cross-Platform)

The webapp ZIP can run on any operating system with Node.js v22+. Extract, run `node server.js` in the `server/` directory, and open `http://localhost:3000` in a browser. No `npm install` is required — all dependencies are bundled.

---

## Quick Start

### Windows

1. Download the latest release ZIP
2. Extract to any folder (for example, `C:\HFY\`)
3. Double-click `start.bat`
4. Wait 5-10 seconds for the server to start
5. Your browser opens automatically to `http://localhost:3000`
6. Go to Settings, select your JSONL data folder, and wait for indexing to complete

### Webapp

1. Download the webapp ZIP
2. Extract it
3. Open a terminal in the `server/` directory
4. Run: `node server.js`
5. Open `http://localhost:3000` in your browser

### Android

1. Download the APK
2. Uninstall any previous version first (different versions may use different signing keys)
3. Enable installation from unknown sources in Android settings
4. Install the APK
5. Open the app and wait 10-30 seconds for the catalog to load

---

## How to Add JSONL Files

JSONL (JSON Lines) files contain Reddit post data — one JSON object per line. The app can index these files to build a fully offline archive.

### Step-by-Step

1. **Obtain JSONL files** — See [Where to Get JSONL Files](#where-to-get-jsonl-files) below.

2. **Put the files in a folder** — Create a folder on your computer (for example, `C:\HFY-Data\`) and place all `.jsonl` files there. You can mix files from different subreddits.

3. **Start the app** — Double-click `start.bat` (Windows) or run `node server.js` (webapp).

4. **Open Settings** — Click the SETTINGS tab at the bottom of the screen.

5. **Scroll to SUBREDDIT JSONL DATA** — Look for the card with the folder icon.

6. **Click SELECT FOLDER** — A file picker dialog appears. Select the folder containing your JSONL files.

7. **Wait for indexing** — The app streams through each file, parses every line, and builds the index. Progress is shown as a percentage. Indexing 162,000 posts takes about 30 seconds.

8. **Browse the Archive** — Once indexing is complete, go to the ARCHIVE tab to browse all posts. Use the subreddit filter, author filter, search, and sort options to find content.

9. **Discover series** — Go to the DISCOVER tab to see series that the app auto-detected from the JSONL data. You can create custom series from these.

### File Format

Each line in a `.jsonl` file should be a valid JSON object representing a Reddit post. The app expects the following fields (most are optional, but `id`, `title`, and `selftext` are important):

```json
{
  "id": "ciqsdj",
  "title": "Chapter Title",
  "author": "username",
  "score": 424,
  "created_utc": 1564320000,
  "permalink": "/r/HFY/comments/ciqsdj/chapter_title/",
  "subreddit": "HFY",
  "num_comments": 15,
  "over_18": false,
  "is_self": true,
  "selftext": "The full text content of the post...",
  "selftext_html": "<!-- SC_OFF --><div class=\"md\">...</div>",
  "url": "https://www.reddit.com/r/HFY/comments/ciqsdj/chapter_title/",
  "link_flair_text": null,
  "upvote_ratio": 0.95
}
```

### File Replacement Detection

The app automatically detects when you replace, add, or remove JSONL files:

- **Replaced file** (same name, different size or date) — The old entries are removed and the new file is re-indexed
- **New file added** — Indexed alongside existing files
- **File removed** — Its entries are removed from the index

To trigger re-indexing, click the RE-INDEX button in Settings, or simply restart the app (it scans the folder on startup).

---

## Where to Get JSONL Files

### From the Arctic Shift Website

[Arctic Shift](https://github.com/ArthurHeitmann/arctic_shift) is a project that makes Reddit data accessible to everyone. It provides three ways to get data:

**Option 1: Download Tool (easiest for specific subreddits)**

Visit the [Arctic Shift download tool](https://arctic-shift.photon-reddit.com/download-tool). Enter the subreddit name, check both "download posts" and "download comments", and save the files to your JSONL folder. This is the best option for smaller subreddits or when you only want data from one subreddit.

**Option 2: API (for programmatic access)**

The Arctic Shift API at `https://arctic-shift.photon-reddit.com` provides endpoints for searching and retrieving posts and comments. You can use it to download data for a specific subreddit, author, or time period. See the [API documentation](https://github.com/ArthurHeitmann/arctic_shift/blob/main/api/README.md) for details.

Example API call:
```
GET https://arctic-shift.photon-reddit.com/api/posts/search?subreddit=HFY&sort=asc&limit=100
```

**Option 3: Torrent Dumps (for bulk data)**

Arctic Shift provides monthly data dumps via Academic Torrents. These contain every post and comment from every subreddit for that month. Visit the [download links page](https://github.com/ArthurHeitmann/arctic_shift/blob/main/download_links.md) for torrent links.

The dumps are in `.zst` (Zstandard) compressed format. You need to decompress them before the app can read them:

```bash
zstd -d r_hfy_posts.zst -o r_hfy_posts.jsonl
```

You can download Zstandard from [the official site](https://facebook.github.io/zstd/) or install it via your package manager (`apt install zstd`, `brew install zstd`, etc.).

### From GitHub

Some Reddit data archives are also available on GitHub:

**Arctic Shift repository:** The [arctic_shift GitHub repo](https://github.com/ArthurHeitmann/arctic_shift) contains the data schemas, helper scripts, and API documentation. The actual data dumps are hosted on Academic Torrents (see above).

**Other archives:** Search GitHub for "reddit jsonl" or "subreddit archive" to find other community-maintained data dumps. Make sure the format matches what the app expects (one JSON object per line with the fields listed above).

### From the App's Own Download Link

The app's README includes a link to pre-packaged JSONL files for four subreddits:
- r_hfy_posts.jsonl
- r_natureofpredators_posts.jsonl
- r_sexyspacebabes_posts.jsonl
- r_TheCryopodToHell_posts.jsonl

These are hosted on GoFile and can be downloaded directly.

### Creating Your Own JSONL Files

If you want to create a JSONL file for a subreddit that is not available in any archive, you can use the Arctic Shift API or the easy-reddit-downloader tool to fetch posts and save them in JSONL format:

```python
# Using the Arctic Shift API with Python
import requests, json

posts = []
after = None
while True:
    params = {"subreddit": "YOUR_SUBREDDIT", "limit": 100, "sort": "asc"}
    if after:
        params["after"] = after
    r = requests.get("https://arctic-shift.photon-reddit.com/api/posts/search", params=params)
    data = r.json()["data"]
    if not data:
        break
    posts.extend(data)
    after = data[-1]["created_utc"] + 1

with open("r_your_subreddit_posts.jsonl", "w") as f:
    for post in posts:
        f.write(json.dumps(post) + "\n")
```

---

## Using the App with Other Subreddits

Although the app was originally built for r/HFY, it works with any subreddit that has serial fiction. Here is how to use it with other subreddits:

### Step 1: Get the JSONL Data

Download or create a JSONL file for the subreddit you want to read. See [Where to Get JSONL Files](#where-to-get-jsonl-files) above.

### Step 2: Index the Data

Put the `.jsonl` file in your JSONL folder and select it in Settings. The app will index it and the subreddit will appear in the Archive tab's subreddit filter.

### Step 3: Browse and Discover

- Go to the **ARCHIVE** tab to browse all posts from the subreddit
- Use the subreddit filter to focus on one subreddit at a time
- Go to the **DISCOVER** tab to see series that the app auto-detected from the data
- The auto-detection groups posts by author and title patterns (Part N, Chapter N, Episode N, etc.)

### Step 4: Create Custom Series

If the app does not auto-detect a series, you can create one manually:

1. In the ARCHIVE tab, find the first chapter of the series
2. Click **ADD TO SERIES** and choose "Create New Series"
3. Enter the series name
4. Find subsequent chapters and add them to the series
5. You can also use multi-select mode to add multiple posts at once

### Step 5: Read and Export

- Open the series from the BROWSE tab (custom series appear with a CUSTOM badge)
- C-FETCH the chapters to cache content and scores
- Use the merge feature to export the entire series as an EPUB ebook

### Cross-Subreddit Series

The app supports series that span multiple subreddits. If an author posts chapters in different subreddits (for example, main chapters in r/HFY and side stories in r/WritingPrompts), the unified JSONL index covers all subreddits. The chapter chain following will follow links across subreddits automatically.

---

## Creating Ebooks

The app can export chapters and complete series as EPUB or TXT files for offline reading on e-readers, phones, or tablets.

### Individual Chapter Download

1. Open a series and find the chapter you want to download
2. Click the download icon next to the chapter
3. The chapter is saved as an EPUB file (web) or TXT file (Android)
4. Files are named with zero-padded chapter numbers for correct sort order

### Download All Chapters

1. Open the series detail page
2. Click the download all button
3. All chapters are downloaded as individual files
4. Parallel download count is configurable (1-1000) in Settings

### Merge into Single Ebook

1. Open the series detail page
2. Click the merge button
3. The app checks that all chapters have been fetched (merge safety check)
4. If any chapters are unfetched, the merge is blocked with a warning
5. If all chapters are fetched, a single EPUB is generated with:
   - A table of contents with chapter numbers and titles
   - Zero-padded filenames for correct sort order
   - All chapter content concatenated

### Merge from JSONL (Instant)

If you have JSONL data indexed, merging is nearly instant. The app reads chapter content directly from the local JSONL file using random-access byte-offset reading. Merging 1,000 chapters takes under 10 seconds instead of 10+ minutes.

### Ebook Format

- **EPUB** — Compatible with Kindle, Kobo, Apple Books, Google Play Books, and most e-readers. Generated using JSZip on the client side.
- **TXT** — Plain text format, used on Android with the native interface. Each chapter is separated by a divider.

### File Naming

Files are named with zero-padded chapter numbers:
```
001. Series Name - Chapter Title.epub
002. Series Name - Chapter Title.epub
...
100. Series Name - Chapter Title.epub
```

This ensures correct sort order in file managers and e-readers.

---

## How the Android APK Was Built

The Android APK is the most complex build target. It wraps the webapp in a native Android shell that runs Node.js on the device.

### Prerequisites

- JDK 17 or later
- Android SDK (platform-34, build-tools 36.0.0)
- Gradle 8.14 or later
- Bun (for building the webapp from source)
- apksigner and zipalign (from build-tools)
- A keystore for signing

### Build Steps

**Step 1: Build the Webapp**

```bash
bun install
bunx next build --webpack
cp -r .next/static .next/standalone/.next/
cp -r public .next/standalone/
```

This produces a self-contained server in `.next/standalone/` that includes `server.js`, the compiled `.next/` directory, `public/` assets, and a minimal `node_modules/`.

**Step 2: Build the Base APK with Capacitor**

The Android project uses Capacitor 8.4 to wrap the webapp:

```bash
npx cap init com.nekro.hfycyberdeck HFYNEKROCODEXXX
npx cap add android
npx cap copy android
cd android
./gradlew assembleRelease
```

This produces a base APK with the Java bridge components (MainActivity, NodeForegroundService, NodeRunner) and the native Node.js binary for ARM64.

**Step 3: Replace Server Assets**

The base APK's `assets/server/` directory is replaced with the latest webapp build:

```bash
cp base.apk HFY-v250fx13.apk
cd /tmp/apk-update
zip -0 HFY-v250fx13.apk \
    assets/server/.next/server/chunks/4034.js \
    assets/server/.next/static/chunks/app/page-94aba2608e3f02d1.js \
    assets/server/.next/static/chunks/9365-d1a77d1fb931ad63.js \
    assets/server/.next/BUILD_ID \
    assets/server/package.json
```

The `-0` flag stores files uncompressed, which is required for Node.js memory-mapping on Android.

**Step 4: Zip-Align and Sign**

```bash
zipalign -v 4 HFY-v250fx13.apk HFY-v250fx13-aligned.apk
apksigner sign --ks my-release-key.jks \
    --ks-key-alias my-key-alias \
    --ks-pass pass:my-password \
    --key-pass pass:my-password \
    --v2-signing-enabled true \
    --v3-signing-enabled true \
    HFY-v250fx13-aligned.apk
```

### APK Architecture

The APK contains three Java components:

**MainActivity.java** — Hosts a WebView that loads `http://localhost:3000`. Handles the Android back button and lifecycle events.

**NodeForegroundService.java** — Runs as a foreground service to prevent the OS from killing the Node.js process. It acquires:
- WiFi lock (`WIFI_MODE_FULL_HIGH_PERF`) — keeps the WiFi radio in maximum-throughput mode
- Wake lock (`PARTIAL_WAKE_LOCK`) — keeps the CPU active when the screen is off
- 250 kbps keepalive — downloads and uploads 250 KB per second to maintain network activity

The service also implements auto-restart via `onTaskRemoved`, `onDestroy`, and `START_STICKY`.

**NodeRunner.java** — Manages the native Node.js process:
- Launches the Node.js binary (ARM64) via `ProcessBuilder`
- Sets `HFY_DATA_DIR` to `getFilesDir()/hfy-data/` (persistent storage)
- Monitors the process and auto-restarts after a 3-second delay if it crashes
- Uses `volatile Process` and `volatile boolean keepRunning` for thread safety

### Native Libraries

The APK includes 10 native libraries:
- `libsqlite3.so`
- `libssl.so`
- `libcrypto.so`
- ICU (International Components for Unicode) libraries
- c-ares library
- `libnode.so` (the Node.js binary)

### Permissions

The APK requests the following permissions:
- `INTERNET`, `ACCESS_WIFI_STATE` — network access and WiFi lock
- `FOREGROUND_SERVICE_DATA_SYNC` — Android 14+ foreground service type
- `WAKE_LOCK` — CPU wake lock
- `MANAGE_EXTERNAL_STORAGE` — file access for downloads
- `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` — battery optimization bypass
- `VIBRATE` — haptic feedback on long-press

---

## How the App Was Made

### Technology Stack

The app is built with:

- **Next.js 16** (App Router, standalone output) — web framework
- **React 19** — UI library
- **TypeScript 5** — type system (with `ignoreBuildErrors: true` for standalone build)
- **Zustand 5** — state management (7 stores with persist middleware)
- **Tailwind CSS 4** — styling
- **Framer Motion 12** — animations (splash screen, screen transitions)
- **JSZip 3.10** — EPUB generation (client-side)
- **react-markdown 10** — chapter content rendering
- **Capacitor 8.4** — Android APK wrapping
- **Puppeteer-Core 25** — real browser login for OAuth
- **lucide-react** — icons
- **Sonner** — toast notifications
- **@dnd-kit** — drag-and-drop for chapter reordering

### Build Configuration

The app is built with Next.js standalone output mode:

```json
{
  "output": "standalone",
  "swcMinify": false,
  "webpack": { "optimization": { "minimize": false } },
  "typescript": { "ignoreBuildErrors": true }
}
```

- `output: "standalone"` — produces a self-contained server that does not require `npm install` on the target machine
- `swcMinify: false` — prevents TDZ (Temporal Dead Zone) errors in bundled code
- `webpack.optimization.minimize: false` — same reason; prevents variable hoisting issues
- `typescript.ignoreBuildErrors: true` — ships despite TypeScript errors (the code works at runtime)

### File Structure

```
HFY-NEKROCODEXXX/
├── HFY-NEKROCODEXXX.cmd        # Windows launcher
├── start.bat                    # Fallback launcher
├── stop.bat                     # Stop script
├── README.txt                   # Quick start guide
├── node/
│   └── node.exe                 # Node.js runtime (80 MB)
└── server/
    ├── server.js                # Next.js standalone server
    ├── aggregate-counts.js      # Periodic aggregation script
    ├── jsonl-handler.js         # JSONL batch handler (port 3001)
    ├── package.json             # Node.js package config
    ├── .next/                   # Compiled Next.js app
    │   ├── static/              # Client-side chunks
    │   │   └── chunks/
    │   │       └── app/
    │   │           └── page-94aba2608e3f02d1.js  # Main client bundle (2.7 MB)
    │   └── server/              # Server-side chunks + API routes
    │       └── chunks/
    │           └── 4034.js      # Reddit scraper with 9 fetch strategies
    ├── public/                  # Static assets (icons, images, fonts)
    ├── data/                    # User data directory
    │   ├── app-store/           # Key-value JSON store
    │   ├── user-chapters/       # Series chapter lists
    │   └── chapter-cache/       # Fetched chapter content
    └── node_modules/            # Minimal dependencies (45 MB)
```

### Minimal node_modules

The distribution package includes a minimal `node_modules` (45 MB) created using `@vercel/nft` (Node File Tracing). Only the 17 packages actually required by the server at runtime are included. `.map` files, `.d.ts` files, README/LICENSE files, and platform-specific native binaries are excluded to reduce size.

---

## Credits and Acknowledgments

This project stands on the shoulders of giants. The following open-source projects and their authors made this app possible:

### easy-reddit-downloader

**Repository:** [josephrcox/easy-reddit-downloader](https://github.com/josephrcox/easy-reddit-downloader)  
**Author:** Joseph R. Cox  
**License:** MIT

This Node.js Reddit post downloader demonstrated that Reddit's public JSON API can be accessed without OAuth by simply appending `.json` to URLs. The app's authless `.json` fetch strategy (Tier R in the Downloader Cascade) is directly inspired by this project. The URL building patterns, post type classification system (self post, media post, link post, gallery, poll), and file name sanitization logic were all adapted from easy-reddit-downloader's `lib/utils.js` and `lib/api.js` modules.

Joseph's clean, well-documented code made it straightforward to understand how Reddit's public API works and how to build URLs for subreddits, user profiles, and search queries. Without this project as a reference, the app's fallback fetch strategy would not exist.

### hfy2epub

**Repository:** [hacst/hfy2epub](https://github.com/hacst/hfy2epub)  
**Author:** hacst  
**License:** MIT

This browser-based tool converts r/HFY post series into EPUB files. The app's chapter chain following algorithm is directly based on hfy2epub's `findSeriesParts` function, which recursively follows "next" links to discover all chapters in a series. The heuristic for detecting "next" links (matching link text against regex patterns like "next", "next chapter", "part N+1") was adapted from hfy2epub's `findNextURL` function.

Additionally, hfy2epub's approach to handling Reddit's URL shortener (redd.it), its comment content collection heuristic (detecting when an author continues the story in the comments), and its author opt-out mechanism (the `[NOEPUB]` tag) all influenced the app's design. The concept of respecting author preferences and not distributing generated ebooks comes directly from hfy2epub's README.

### arctic_shift

**Repository:** [ArthurHeitmann/arctic_shift](https://github.com/ArthurHeitmann/arctic_shift)  
**Author:** Arthur Heitmann  
**Website:** [arctic-shift.photon-reddit.com](https://arctic-shift.photon-reddit.com)

Arctic Shift is the project that makes Reddit data accessible to everyone. It provides large data dumps (via Academic Torrents), a REST API, and a web search interface. The data spans from 2005 to the present, with monthly dumps available.

The app's JSONL archive integration is built entirely around Arctic Shift's data format. The field names (`id`, `title`, `author`, `score`, `created_utc`, `permalink`, `subreddit`, `selftext`, etc.), the JSONL line format, and the `_meta` field tracking (deleted, edited, initially unavailable) all come from Arctic Shift's data schema. The app's JSONL indexing engine was inspired by Arctic Shift's `fileStreams.py` module, which streams through compressed JSONL files line by line.

The app also calls the Arctic Shift API as a fallback when content is not available in the local JSONL archive. The `/api/posts/search` and `/api/posts/ids` endpoints are used to find and retrieve specific posts.

Arthur's decision to make all this data freely available, with no login required and no rate limits, is what made the app's offline archive feature possible. Without Arctic Shift, there would be no JSONL files to index.

### ArcticZim

**Repository:** [IMayBeABitShy/ArcticZim](https://github.com/IMayBeABitShy/ArcticZim)  
**Author:** IMayBeABitShy  
**License:** Open source

ArcticZim is a tool for converting Arctic Shift data into ZIM files (offline web archives readable by Kiwix). While this app does not produce ZIM files, several UI patterns and architectural concepts from ArcticZim were adapted.

The Archive tab's post card layout (showing score, title, author, subreddit, date, content length, comment count, and NSFW badge) is based on ArcticZim's `postsummary.html.jinja` template. The author browser concept, the pagination approach (20 posts per page with Previous/Next buttons), and the batch processing pattern for importing large datasets all come from ArcticZim.

ArcticZim's clean separation between data import, media download, and rendering stages influenced the app's architecture. The concept of building a structured index from raw JSONL data and then querying it efficiently was directly inspired by ArcticZim's SQLAlchemy-based approach.

### Additional Thanks

- **The r/HFY community** — For writing the stories that this app is designed to read. Without the authors who pour their time and creativity into HFY fiction, none of this would matter.
- **The Reddit team** — For maintaining a public API and RSS feeds that make tools like this possible.
- **The Wayback Machine** — For archiving the internet and providing a fallback when content is deleted.
- **hfy.foundation** — For maintaining a community index of HFY content.
- **The open-source community** — For the hundreds of libraries that this app depends on.

---

## Known Bugs and Call for Contributors

This app is a personal project built by one person. It has bugs. I am aware of that, and I need help fixing them.

### Known Issues

1. **Memory usage on low-RAM devices** — The app can use 150-300 MB of RAM. On devices with less than 4 GB RAM, the server may crash when the browser loads the page simultaneously. The JSONL indexing engine adds another ~56 MB for 216K posts.

2. **Reddit rate limiting** — Even with 9 accounts and 386 CORS proxies, Reddit can still rate-limit the server IP. The app retries with backoff, but persistent rate limiting can make C-FETCH slow.

3. **Chapter chain following false positives** — The "next" link detection uses regex patterns that can sometimes match non-chapter links. This can cause the crawl engine to follow irrelevant links.

4. **Large series merge memory** — Merging a series with 1,000+ chapters without JSONL data can consume significant memory because all chapter content is held in memory before generating the EPUB.

5. **Service worker caching** — The service worker can serve stale HTML when the app is updated. Users may need to hard-refresh (Ctrl+Shift+R) after updating.

6. **Android folder picker duplicates** — On some Android devices, the native folder picker can create duplicate folders with "(1)" suffixes when writing files. The app has a workaround, but it may not work on all devices.

7. **OCS (Out of Cruel Space) chapter gaps** — The old Reddit search returns limited results, so some chapters in very long series (1700+ chapters) may not be discovered. The JSONL archive integration fixes this, but only if the JSONL file contains those chapters.

8. **Hardcoded paths** — Some paths in the server code are hardcoded to specific directories. These may need adjustment when running on non-standard configurations.

### How to Help

If you want to contribute:

1. **Fork the repository** and make your changes
2. **Test thoroughly** — Start the server, open the app in a browser, and verify that your fix works
3. **Document your changes** — Explain what you fixed and why
4. **Submit a pull request** — I will review and merge

Areas that need the most help:
- Bug fixes and stability improvements
- Support for additional subreddits (testing and fixing subreddit-specific issues)
- Performance optimization (especially for large JSONL files)
- UI/UX improvements
- Documentation improvements
- Translation to other languages

I cannot offer monetary compensation, but I will credit all contributors in the README and in the app's About section.

---

## Forking and Personal Use

This project is open for forking. You are free to:

- Fork the repository
- Modify the code for your own use
- Add support for additional subreddits
- Change the UI, themes, and fonts
- Add new features
- Build and distribute your own version (subject to the legal notice below)

You are encouraged to use the app however you wish for your personal reading. If you make improvements that would benefit others, please consider submitting a pull request so everyone can benefit.

---

## Legal Notice — Ebook Distribution Prohibited

### Personal Use Only

The stories accessible through this application are the intellectual property of their original authors, published by those authors on Reddit. Reddit's User Agreement permits users to access and consume this content through Reddit's platform and authorized interfaces.

This application is intended strictly for **personal, private reading**. It is not a distribution tool.

### I Do Not Condone Sharing Ebooks

The application includes features to download chapters and merge them into EPUB or TXT files. These features exist for the convenience of the individual user — to read content offline, on e-readers, or in a preferred format.

**I do not condone, encourage, or support the sharing, distribution, republishing, selling, hosting, or making available of any ebook or text file generated by this application.** This includes but is not limited to:

- Uploading merged EPUBs or TXT files to file-sharing services, torrent sites, or public repositories
- Posting downloaded chapters on other websites, forums, or social media
- Selling or commercially exploiting any content fetched through this application
- Republishing author content in any form without the original author's explicit written permission

Doing so constitutes copyright infringement against the original authors and may expose you to legal liability. The authors of Reddit stories have not granted permission for their work to be redistributed in this manner.

If you enjoy a story, support the original author by upvoting their posts on Reddit, sharing direct links to the Reddit thread, or contacting them directly regarding licensing or republication.

### Disclaimer of Warranty

This software is provided "as is," without warranty of any kind. The developer is not responsible for any misuse of the application, any consequences of redistributing copyrighted content, or any actions taken by Reddit or content authors against users who violate these terms.

---

## System Requirements

### Windows
- Windows 10/11 64-bit
- 2 GB RAM minimum (4 GB recommended for JSONL indexing)
- 500 MB disk space (5 GB recommended for app + JSONL files)
- Chrome or Edge browser (for standalone window mode)

### Android
- Android 10 (API 29) or later
- ARM64 architecture
- 4 GB RAM recommended
- 500 MB internal storage (5 GB for app + JSONL files)

### Webapp
- Node.js v22 or later
- 2 GB RAM minimum
- 500 MB disk space
- Any modern browser

---

## Troubleshooting

### The app does not open (Windows)

- Check `server.log` for error messages
- If the log shows "Cannot find module 'next'", the `node_modules` directory is missing. Re-download the latest release.
- If antivirus or SmartScreen blocks execution, select "More info" then "Run anyway"
- Use `start.bat` as an alternative — it runs the server directly
- The `server/` folder must be in the same directory as the launcher

### The app is stuck at "INITIALIZING... 0%"

This means the server is not running. Check that:
- `node.exe` exists in the `node/` folder
- `server/server.js` exists
- `server/node_modules/next` exists
- Port 3000 is not already in use by another application

### Port 3000 is already in use

Edit `start.bat` and change `PORT=3000` to another port (for example, `PORT=3001`). Then open `http://localhost:3001` in your browser.

### Indexing is slow

- Use SSD storage for JSONL files
- Close other memory-intensive applications
- The first indexing takes longer; subsequent startups are faster if files have not changed

### C-FETCH is slow or fails

- Connect Reddit accounts in Settings for higher rate limits (600 QPM per account)
- The CORS proxy pool may be degraded; check `/api/proxy-stats` for proxy health
- Try smaller batch sizes (Settings, C-FETCH Batch Size)
- If you have JSONL data indexed, C-FETCH should be nearly instant

### Chapters fail to download

- The chapter may be deleted from Reddit
- Reddit may be rate-limiting your IP
- The app retries failed chapters 3 times with exponential backoff
- If you have JSONL data, the chapter should be available locally

### Favorites or reading progress is lost

- This was a bug in earlier versions. Make sure you are using FX45 v31 or later.
- On Android, data is stored at `getFilesDir()/hfy-data/` which survives restarts
- Use the Settings, Audit Persistence button to check for data integrity issues

---

## API Reference

The app exposes 41 API endpoints. Here are the most important ones:

### Store API
- `GET/POST/DELETE /api/store/[key]` — Generic key-value store (filesystem-backed)
- `GET /api/store-scores-all` — Get all series scores
- `GET /api/store-chapter-counts-all` — Get all chapter counts

### Series API
- `GET /api/series` — Get series list
- `GET /api/series/[slug]` — Get series detail
- `GET/POST/DELETE /api/series/[slug]/chapters` — Chapter CRUD
- `POST /api/series/[slug]/add-chapter` — Add chapter
- `POST /api/series/[slug]/fetch-cache` — C-FETCH (NDJSON stream)
- `POST /api/series/[slug]/continue-fetch` — Continue crawl
- `POST /api/series/[slug]/fetch-missing` — Fetch missing chapters

### Content API
- `GET /api/chapter/[id]` — Get chapter content
- `POST /api/download` — Download chapters
- `POST /api/crawl` — RSS crawl

### Search API
- `GET /api/search` — Search Reddit
- `GET /api/search-series` — Search series index
- `GET /api/hfy-foundation-search` — Search hfy.foundation

### Reddit Auth API
- `GET /api/reddit-auth` — Start OAuth flow
- `GET/POST /api/reddit-multi-token` — Multi-account token rotation
- `GET/POST /api/reddit-cookies` — Cookie management

### Auto-Discover API
- `POST /api/auto-discover` — Start auto-discover job
- `GET /api/auto-discover/[jobId]` — Get job status

### JSONL Handler (port 3001)
- `POST /batch-scores` — Batch save scores
- `POST /batch-cache` — Batch save chapter content
- `GET /stats` — Index statistics

---

## Configuration

### Environment Variables
- `PORT` — Server port (default 3000)
- `HOSTNAME` — Bind address (default 0.0.0.0)
- `NODE_ENV` — Set to `production`
- `HFY_DATA_DIR` — Data directory for persistent storage (Android: `getFilesDir()/hfy-data/`)
- `KEEP_ALIVE_TIMEOUT` — HTTP keep-alive timeout

### Settings (in-app)
- **C-FETCH Batch Size** — 1-500 (default 5). Controls how many chapters are fetched per batch.
- **Parallel Downloads** — 1-1000 (default 5). Controls concurrent download count.
- **Reader font size** — 3px to 48px
- **Reader font family** — 30+ bundled fonts or custom upload
- **App theme** — 25 themes
- **Reader theme** — 22 themes
- **Auto-update interval** — 1h, 24h, 72h. Automatically runs series discovery for library series.

---

## File Structure

```
server/
├── server.js                    # Next.js standalone server entry point
├── aggregate-counts.js          # Periodic aggregation script (runs every 120s)
├── jsonl-handler.js             # JSONL batch handler (port 3001)
├── package.json                 # Node.js package configuration
├── .next/
│   ├── static/
│   │   └── chunks/
│   │       └── app/
│   │           └── page-94aba2608e3f02d1.js  # Main client bundle (2.7 MB)
│   └── server/
│       ├── app/
│       │   └── api/             # 41 API route files
│       └── chunks/
│           └── 4034.js          # Reddit scraper (9 fetch strategies)
├── public/                      # Static assets
│   ├── bg-images/               # 23 background images
│   ├── icon-*.png               # App icons
│   ├── sw.js                    # Service worker
│   └── manifest.json            # PWA manifest
├── data/
│   ├── app-store/               # Key-value JSON store
│   ├── user-chapters/           # Series chapter lists
│   └── chapter-cache/           # Fetched chapter content
└── node_modules/                # Minimal dependencies (45 MB)
```

---

## License

The application code is provided for personal use and modification. You are free to modify, extend, and rebuild the application for your own use.

The content accessed through the application (stories, chapters, and associated text) is owned by the original authors who posted it to Reddit. No rights to the content are granted by this application beyond the personal reading access described in the Legal Notice above.

Redistribution of the application itself, with or without modification, is permitted provided that this README, including the Legal Notice, is included unmodified.
