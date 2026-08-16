# Pseudonyx

A background **Windows agent that pseudonymizes GDPR-sensitive data before it leaves your machine.**
It protects three paths:

- **Clipboard** — when you copy text, sensitive spans (names, emails, IBANs, health details, …) are
  replaced in place with stable tokens like `john.doe@example.com` or `[PERSON_2]` before you paste.
- **File paste / drag-drop** — files pasted into a protected destination are redacted first; a
  `name.redacted.*` copy is substituted so the original values never travel.
- **AI-website attachments** — the browser extension takes a file **as you attach it** through the file
  picker on a protected site, has the desktop app redact it, and substitutes the redacted copy before the
  page ever sees the original. Host-independent, so it does not go stale when a service moves its upload
  endpoint. If redaction cannot be guaranteed — the file is over the size cap, has no text layer, or the
  app is not running — the **attachment is refused** rather than sent unredacted, and Pseudonyx says why.
  Dragging a file onto the composer and pasting one in are covered too. Where a site will not accept a file
  it did not get from the operating system — some read an API that only a real OS drag can satisfy — the
  redacted text goes into the message box instead, or waits for one click on the site's own attach button.
  Either way you are told which happened, and the original never leaves your machine.
- **AI-website uploads (proxy)** — an optional local TLS-intercepting proxy redacts uploads to a small,
  precise set of upload endpoints. It is what covers native desktop AI apps; for browsers, the attach-time
  path above is the primary protection, because an endpoint outside the proxy's list is not merely
  uninspected — it never reaches Pseudonyx at all.
- **Downloads (reverse)** — documents downloaded back from an AI site that still contain your tokens are
  restored locally into a `name_unredacted.ext` copy, and the original (still-redacted) download is
  renamed `name_Redacted.ext` so you can tell them apart. **Detection is folder-agnostic (v17):** the true
  Downloads/Desktop/Documents folders and each browser's download directory are watched **recursively** and
  the decision is made by the download's Mark-of-the-Web origin **or** our own token/[BIO_2] — so a
  redacted file saved to any of those (or a subfolder) is caught with zero setup. There is no manual
  "watched folders" page; a tray **"Un-redact a file…"** action covers anything saved elsewhere.
- **Images (reverse, true original) — v17** — when we redact an image we keep its **byte-exact original**
  for 5 days in a **DPAPI-encrypted vault**, keyed by a perceptual [BIO_2] of the redacted output.
  When that image comes back (downloaded or pasted from an AI site) we restore the **actual original** —
  surviving rename and re-encoding — instead of the older lossy OCR repaint (which is now only a fallback).
  Those 5 days are a deliberate limit, not an oversight: keeping copies of your pictures longer than needed
  would work against the point of the product. **Bring the same image back after the window and Pseudonyx
  says so in a Windows notification** — it names the file and the number of hidden values still showing, so a
  file that arrives still covered in tokens reads as the retention promise being kept rather than as a bug.
- **Images (everywhere)** — GDPR-sensitive text *inside images* is redacted with on-device Windows OCR,
  painting tokens over the sensitive words. Wired into **all** paths: a copied screenshot pasted into an
  AI site, image files pasted/dragged, and image uploads — using the same engine as document uploads.
- **Faces (everywhere)** — human faces in images are covered with an opaque block before the image
  leaves your machine, on the same paths (copy-paste, file paste/drag, uploads), using the on-device
  Windows face detector (no cloud, no model to ship). When the same image returns from an AI site it is
  restored to the **byte-exact original** via the [BIO_2] vault — the same reverse path as text.
  On by default; toggle in **Settings ▸ "Face redaction in images"**.
- **Images (reverse)** — a redacted image coming back from an AI site is restored too: paste a copied
  image into a trusted app, or download one from a companies.json site, and the originals are painted
  back. A quick OCR token check makes it easy to tell whether an image needs un-redaction.

The reversible mapping is kept **encrypted on your machine (Windows DPAPI)**, so tokens can be
rehydrated back to the originals when you copy an AI answer back. Each mapping is retained on a **rolling
5-day window** measured from its last use: a placeholder stays reversible as long as you keep using it.
After five idle days the mapping is removed and its ordinal is permanently retired, never reassigned to a
different value. If you try to un-redact content whose placeholders have aged out, Pseudonyx shows a
notification that it can no longer be restored.

**Also in v16:** a **timed "Pause protection"** snooze (5 / 15 / 60 min) that auto-resumes so you're
never left unprotected by accident; a one-click **GDPR "Record of Processing" export** (HTML + CSV,
counts/categories only — no raw data); and a read-only tray **Info** window (user manual + About).

**Latest (v19–v22):** a **48-hour (2 full days) free trial** with a license key to unlock unlimited use; a **daily
new-version check** that, when an update is online, shows a minimalistic dark popup with a **Download**
button (v21); an **automatic companies.json update** — the list of covered AI websites refreshes
itself in the background, applies live with no restart, and shows a click-to-explain "Protection list
updated" notice; a **first-run installer** that walks you through consent, installs per-user (no admin),
runs quick health checks and turns on **always-on auto-start** (v22); and **optional usage analytics** —
off by default and opt-in, pseudonymous metadata about *how* the app is used (never your content),
end-to-end encrypted, withdrawable any time in Settings ▸ Privacy. All outbound
calls are fail-soft: being offline never revokes access
or changes your data, and none of them ever carries the personal content you redact.

[GENE_672]
[GENE_673]

---

## Why

Pasting or uploading real data into third-party AI tools is a common source of GDPR leaks. Pseudonyx
sits quietly in the tray and neutralizes that risk at the moment data moves, within a sub-second
latency budget, so your workflow is unchanged.

## How it works

The clipboard is watched via a message-only window (`AddClipboardFormatListener`). On each change the
text runs through a tiered detection pipeline, sensitive spans are swapped for tokens, and the result
is written back. Files are redacted through the **same** engine wherever they enter: at paste time by the
paste hook, at attach time by the browser extension (streamed to the app in 512 KB chunks over native
messaging and substituted back into the page once verified), and in flight by the proxy for the upload
endpoints it decrypts. Reversal (rehydration) maps tokens back to values.

Every path that hands bytes to a third party **fails closed**: if the pipeline cannot vouch for the whole
file, the bytes are withheld and you are told, rather than sent as-is. The local paste path keeps the
opposite default, because blocking a user from moving their own data around costs privacy nothing.

**Pre-processing on copy (destination-aware paste).** The moment you copy, redaction runs in the
**background, in parallel**, and the result is cached in memory so the paste itself is instant — text
via `PreRedactionCache`, and **images** (OCR text **and** faces) via `ImagePreRedactionCache`. Nothing
sensitive is committed to disk. What lands depends on where you paste: a paste into a **non-AI**
window gets the **original** content, while a paste into an **AI** destination gets the already-redacted
content. If you paste into an AI site before background processing has finished, the paste simply
**waits** for it to complete, then delivers the redacted result (and keeps it cached for further AI
pastes). Copying anything else clears the cached redaction so nothing stale is kept.

### Detection tiers

1. **Structured (regex + validation)** — always on, high precision, sub-millisecond. Email, phone,
   IBAN (mod-97), card (Luhn), SSN / passport / tax ID / driver's license, IPv4/IPv6, MAC, IMEI,
   advertising/cookie IDs, precise geolocation, raw DNA/RNA, and economic identifiers.
2. **Context-gated topic lexicons** — opt-in per category. Article 9 special categories (health,
   religion, political opinion, racial/ethnic origin, [UNION_3], [SEXLIFE_139]/orientation, criminal,
   biometric) and children's data. A term only matches when a supporting cue is nearby.
3. **Name/City gazetteers** — fast hash-set lookup (fallback when NER is off), and the tier that makes the
   European claim real. 896 given names covering English, Dutch/Flemish, French and German; 583 towns
   covering the Benelux in both language variants ([CITY_1331]/[CITY_1332], [CITY_1333]/[CITY_1334], [CITY_1335]/[CITY_1336]) plus the major
   French, German, Austrian and Swiss cities. A name that is also an ordinary word ("amber", "iris", "jean",
   Dutch "rein", and "jan" as the month) is excluded from the case-insensitive matcher and still redacts when
   written as a name, so ordinary Dutch and French prose stays readable.
4. **Addresses in continental forms** — three street shapes, because the house number is not always in the
   same place: `[ADDRESS_3240]`, `10 Rue de Rivoli`, and the glued Dutch/German form with the
   number last, `[ADDRESS_3241]`. Postal codes carry their own evidence: US 5/5+4 and Dutch `[ADDRESS_3242]` on
   shape alone, and a bare Belgian or Dutch four-digit code only when a known town follows it (`[ADDRESS_3243] [CITY_1331]`)
   or a label announces it (`postcode`, `PLZ`, `code postal`) — because four digits are a year, an order
   number and a price far more often than an address.
5. **ONNX NER (optional)** — a quantized [PERSON_2883] token classifier for open-vocabulary PERSON/CITY.
   Governed by the **Redaction quality** slider in Functions: **Fast** (no heavy model, lists only),
   **Balanced** (NER on shorter text), **Maximum** (NER on more/longer text). Picking a NER tier offers a
   one-time on-device model download if it isn't installed yet. The BIO decoder captures **whole words**:
   because the model labels only a word's first WordPiece, `##` continuation subwords inherit their word's
   entity and extend the span, so multi-piece names/addresses redact in full ("Jenniffer", not "Jen").

### Live typing awareness (in-app + optional browser extension)

Beyond copy/paste and uploads, Pseudonyx can flag sensitive text **as you type** into supported AI sites,
so you see what will be redacted before you send. Two independent paths, both **fully on-device**:

- **In-app (default, no add-on):** a UI-Automation watcher reads the focused compose field and shows a
  click-through overlay. Honest limit — it cannot reliably rewrite web-chat SPA fields at send time, so it
  is **preview/awareness**, governed by `EnableLiveTypingAwareness`.
- **Browser extension (optional):** a minimal cross-browser **MV3** extension (`extension/`) talks to the
  app over **native messaging** — the app runs the *same* detection engine and returns the sensitive spans
  and the stable tokens they map to. This reaches in-page compose fields the app path can't. It redacts the
  compose field on send, and (proactively, on a pause) only after it is **certain a multi-word subject is
  finished** — so a first name isn't tokenized before you finish typing the surname. Committed placeholders
  in the transcript are **restored on screen** with a dwell-then-animate, one-word-at-a-time reveal that
  fires **once per message** (no flicker or re-animation when the AI answer re-renders while streaming).
  Its popup is intentionally **minimal** — just a live *connected / not connected* status and a **Refresh
  connection** button; every real setting (which sites are protected, detection options) is owned by the
  desktop app.
  If the app becomes unreachable while you're on a protected site, the extension shows a **red in-page
  connection-lost banner** (with a Reconnect button + restart instructions), and the desktop app raises a
  matching warning toast. Host coverage spans the major AI-chat providers — ChatGPT, Gemini, Claude,
  Copilot, Perplexity, Poe, Mistral, DeepSeek, [PERSON_2833] Chat, Character.AI, Meta.AI, Grok, You.com and
  Pi. It works **load-unpacked today**; the enterprise self-hosted Chromium package is generated by
  `tools/package-extension.ps1` and served by the installed app on loopback for managed force-install.
  Firefox still requires an AMO-signed XPI before its enterprise policy can be claimed as effective. See
  [`docs/HUMAN-ACTIONS.md`](docs/HUMAN-ACTIONS.md) §6.

  ```powershell
  # Register the app as the native-messaging host (writes host manifest + Chrome/Edge/Firefox reg keys)
  Pseudonyx.exe --register-native-host --allow-extension <UNPACKED_EXTENSION_ID>
  ```

  Then load `extension/` unpacked (see [`extension/README.md`](extension/README.md)). All extension config
  lives in the `Guardian` section of `appsettings.json` (`EnableBrowserExtensionAutoInstall`,
  `ChromeExtensionId`, `EdgeExtensionId`, `FirefoxAddonId`, `FirefoxXpiUrl`, `NativeMessagingHostName`).

All requirement and architecture documents live in [`docs/`](docs/):

- Exact 24-category/token/channel contract and browser boundaries: [`docs/PERSONAL-DATA-COVERAGE.md`](docs/PERSONAL-DATA-COVERAGE.md).
- Full GDPR Article 4/9 mapping: [`docs/GDPR-AND-COMPLIANCE.md`](docs/GDPR-AND-COMPLIANCE.md).
- File-redaction scope and GDPR conformance: [`docs/FILE-REDACTION-REQUIREMENTS.md`](docs/FILE-REDACTION-REQUIREMENTS.md).
- Exhaustive client design reference: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).
- Cloud, payments & website architecture (Cloudflare, Stripe, GitHub, Google OAuth): [`docs/CLOUD-WEBSITE-ARCHITECTURE.md`](docs/CLOUD-WEBSITE-ARCHITECTURE.md).

## Project layout

```
src/
  Pseudonyx.Core/       Detection engine, detectors, mapping store, pseudonymize/rehydrate
  Pseudonyx.Service/    Windows worker: clipboard monitor, file redaction, upload proxy,
                        tray UI, consent gate, config, gazetteers
  Pseudonyx.Images/     Isolated image text-redaction (WinRT OCR + result cache); depends only on Pseudonyx.Core
extension/            Optional cross-browser MV3 extension (live-typing bridge over native messaging)
tests/
  Pseudonyx.Tests/      xUnit tests for detection, files, images, proxy, stats, settings, consent,
                        config, licensing/update-check and companies.json remote sync
  Pseudonyx.E2E/        End-to-end harness, one entry point (run-all.ps1), three tiers:
                        tier 1 the local editor zoo, tier 2 the live AI-site sweep,
                        tier 3 real Windows input via FlaUI. Asserts by reading text back
                        off screenshots with OCR, so it survives site redesigns.
docs/                          All requirement + architecture docs (client, GDPR, cloud/website)
consent/CONSENT-AGREEMENT.md   First-launch/update consent agreement (shipped with the exe)
settings.json                  App-layer settings (theme, strings, window sizes, notifications, logs)
appsettings.json (in Service)  Detection-engine tunables (the "Guardian" section)
companies.json                 Company → websites/title-cue seed for the redact/block policy
```

## Requirements

To **build** it:

- Windows (x64)
- [.NET 8 SDK]([OID_10169])

To **run** it, minimum: Windows 10 x64 (build [ADDRESS_3244]), 2 cores, **4 GB RAM**, **1 GB** free disk. No .NET
runtime needed, it is bundled. Recommended: 4 cores and 8 GB, which is what the higher-accuracy name model
wants. On 4 GB, set redaction quality to **Fast**: every pattern-based and checksum-based detection still
runs, and you save about 90 MB per process and 130 ms per prompt.

[`docs/SYSTEM-REQUIREMENTS.md`](docs/SYSTEM-REQUIREMENTS.md) has the measured detail: CPU time and RAM at
all 27 slider positions, at 1 to 10,000 redactions, for images and documents, plus why the data folder
stops growing after 5 days. Re-measure it on any machine with:

```powershell
powershell -File tools/run-capacity-test.ps1 -KillTray
```

## Build & test

```powershell
dotnet build Pseudonyx.sln
dotnet test  Pseudonyx.sln      # 2875 tests, the release gate
```

The optional browser extension has its own Node test suite (`extension/`, jsdom): **423** fast
unit/integration tests. See [`extension/README.md`](extension/README.md).

```powershell
cd extension; npm install; npm test
```

The end-to-end harness lives in `tests/Pseudonyx.E2E` and has a single entry point. Tier 1 is the local
editor zoo and runs in seconds against a bundled Chromium with the extension loaded; tier 3 drives real
Windows input. Tier 2 walks the live AI sites and needs browser profiles you signed into once by hand,
so it is opt in.

```powershell
cd tests/Pseudonyx.E2E
npm install; npm run install-browsers
powershell -Command "& '.\run-all.ps1' -Tier @(1,3)"   # local gate, no sign-in needed
powershell -File .\run-all.ps1 -Login                  # one time, sign the profiles in by hand
powershell -Command "& '.\run-all.ps1' -Tier @(1,2,3)" # full sweep
```

Pass the tier list as `@(1,3)`, not `1,3`. PowerShell cannot bind a bare comma list to the script's
`[int[]]` parameter, so `-Tier 1,3` silently arrives empty and every tier is skipped while the run still
writes a report.

To guard against config drift (settings.json ↔ AppSettings), run the gate manually or install it as a
git pre-push hook:

```powershell
powershell -File tools/validate-config.ps1   # build (-warnaserror) + tests, incl. config-drift checks
powershell -File tools/install-hooks.ps1      # opt-in: run the gate before every push
```

## Run

```powershell
dotnet run --project src/Pseudonyx.Service
```

Or launch the built executable detached (recommended for day-to-day use). Pass `--no-setup` during
development so the first-run installer doesn't trigger from the build folder:

```powershell
Start-Process -FilePath "src/Pseudonyx.Service/bin/Debug/net8.0-windows10.0.[ADDRESS_3244].0/win-x64/Pseudonyx.exe" -ArgumentList "--no-setup" -[PERSON_2884]
```

## Release package (share with a tester)

Build a single, self-contained exe that needs **no .NET runtime** on the target PC:

```powershell
powershell -File tools/publish.ps1            # -> dist\Pseudonyx.exe (+ settings.json, companies.json, data\, models\)
```

The script verifies the package is complete and that the shared analytics public key shipped. Zip the
whole `dist\` folder and send it. When the tester runs `Pseudonyx.exe` the **first-run installer** opens:

1. A clean dark **welcome** and the **consent** agreement (tick each point).
2. **Boost redaction quality? (opt-in)** — offers the advanced on-device NER model (higher-quality
   name/place detection) and explains the extra memory cost. If accepted, the **model is installed
   first**, before the app, with its own progress bar — by default **downloaded** (~100 MB quantized
   INT8) from a public [PERSON_2833] URL (`NerModelUrl`/`NerVocabUrl`), or copied from the package when
   bundled next to the installer; if declined, nothing extra is installed and the fast built-in
   word lists are used. The choice is saved (Settings ▸ "Higher-accuracy name detection").
3. **Install** — copies the program per-user to `%LOCALAPPDATA%\Pseudonyx\app` (no admin) with an
   animated progress bar.
4. **Health checks** — verifies the config loads, the engine actually redacts a sample, the analytics
   key is present, storage is writable and auto-start is registered.
5. **Always-on auto-start** — an HKCU `Run` entry **and** a logon-triggered Scheduled Task with
   restart-on-failure, so protection returns after reboot, logon or a crash.
6. A friendly **done** screen, then it hands off to the installed copy in the tray.

Run `Pseudonyx.exe --uninstall` to remove the install and its auto-start entries.

### First launch — consent gate

On first launch (and after an app or agreement update) a dark **consent screen** blocks the UI until
you accept each clause. Acceptance is stored as a signed, tamper-evident `consent.json` in
`%LOCALAPPDATA%\Pseudonyx\` (HMAC-SHA256, keyed by a DPAPI-protected device key) and is re-verified on
every start.

### Tray + one merged window

A **shield icon** appears in the system tray. **Left-click opens the main window** directly (lands on
Insights); **right-click** shows the menu: **Pseudonyx info**, **Report a bug…**, **Un-redact a file…**,
**Pause protection ▸** (5 / 15 / 60 min timed snooze that auto-resumes — there is no indefinite pause),
and **Exit**. Pausing and Exit require the admin password (default `adminadmin` — a tamper gate, not
strong security). The window is dark, resizable and minimizable.

Everything lives in **one window** (no separate Settings/Info split, no password to open). Its tabs sit
on a **single horizontal row** with **‹ › overflow arrows** that appear only when they don't all fit the
window width (it never wraps to a second row): the freely-viewable tabs first (**Insights · About · How to
use · Privacy · License · Activity · Redactions**), then the edit-locked tabs **last**, each marked with a
padlock (**Functions · Data types · Companies · Blocking**). The locked tabs can be viewed freely; the
first edit prompts once for the admin password and unlocks all four for the session. When a newer release
is available, a **Download <version></version>** button appears in the top-right of the header.

The UI follows the Pseudonyx website's design system: an **absolute-black + electric-blue** palette, a
static blue **aurora** behind every window with translucent cards on top, restrained motion (toast
slide-in, slider knob, stat count-up), region-rounded buttons, and a slim rounded custom scrollbar.

- **Insights** — period dropdown (**Last hour / Today / This week / This month / This year**) with a
  logarithmic "redactions over time" graph, 4 stat cards, and "most redacted companies / data types /
  file types" charts.
- **About** — usage graph, plain-language pitch, contact block, and the processing-record export.
- **Privacy** — share-analytics opt-in (blue button to enable, grey to disable), review/withdraw
  consent, delete history, delete reversible "digital twins", and the privacy notice.
- **License** — trial status with "Get a license" on the progress-bar line, key entry, refresh, and
  **Check for updates** (shows a Download popup when a new version is found). The "Your license key" ribbon
  is outlined for attention when action is needed: red when the trial/license has expired, amber when a
  license is within two weeks of expiry, and the accent while on the free trial. Team/enterprise licenses
  also get a **Team access** section here (see Licensing tiers below).
- **Functions** — master protection switch (click the ribbon to pause/resume), a past-day 5-minute-
  granularity activity graph, a **Redaction strength** slider (Soft / Medium / Full, which maps to how
  broad a set of data categories is redacted), and per-feature toggles.
- **Data types** — toggle GDPR categories on/off (un-ticking asks for confirmation).
- **Companies** — search (first) + add-company form (last); tick companies to redact and manage each
  company's websites. Hovering shows the sites as one natural, alphabetically-ordered sentence. Deleting
  a company or website shows a 60-second floating **undo** button with a countdown-timeline background.
- **Blocking** — tick a company to block its sites on this device.
- **Activity / Redactions** — paginated log viewers: the 50 most-recent entries load first, and each
  time you scroll near the bottom the next 50 load (with a brief "loading more…" line). An **Export**
  button writes one interactive HTML bundling **all** activity and redactions (with charts) plus copies
  of the raw logs.

### Notifications

Tray toasts report when data was hidden before a destination, when an upload was blocked, and when
tokens copied back from an AI answer were **restored** (count + source company). Each toast has a
top-right close **X** for instant dismissal, pauses its timer while the pointer is over it, dismisses
faster once the pointer leaves, and otherwise stays ~25% longer than the base duration. Clicking a
"data hidden" toast opens a window explaining what was redacted, how, and why. When an app update is
available the user is reminded twice a day to download it within two weeks; when a license is within
two weeks of expiry the user is reminded twice a day plus once at startup.

### Trial-ended / license-expired state

When the free trial runs out or a license expires, protection stops and **all activity recording stops**
too, and the UI makes this unmistakable: graphs, oversight bars, toggles, company/blocking/data-type rows
and the trial ribbon all turn red; the Companies, Blocking, Data-types and Functions pages become
non-interactive (clicking any control, including add/remove/search, prompts you to get a license); and the
Activity and Redactions pages show a see-through "all activity/redactions stopped" overlay centred over
the (dimmed but still visible) list. A reminder recurs every six hours until a valid key is entered.

### Licensing tiers & settings access

The license carries a **plan** and **role** that decide how settings are protected:

- **Personal** (and the free trial): **no password anywhere.** Every setting is editable directly, with no
  padlocks and no unlock step.
- **Team / Enterprise**: settings are gated by a rolling **5-minute team access code**. One or two people
  hold an **administrator** license and read the live `PSX-XXXX-XXXX` code on their **Pseudonyx web
  dashboard** (the desktop app links to it). A colleague **without** an admin license enters their work
  email + that code in the app to change settings; the app validates it server-side
  (`POST /team/validate-code`), which checks the code is current. Non-admins clicking "Generate team
  password" are told an administrator license is required.
- **Coming soon, Personal, [PERSON_2885]**: everything an agentic AI pulls from your PC autonomously is
  redacted first, so the AI only ever sees redacted data (future release; shown as "coming soon").

[GENE_674]
[GENE_675]
[GENE_676]
[GENE_677]

### Always-on protection (anti-kill watchdog)

Because Pseudonyx protects data continuously, the **installed** app can't simply be terminated by another
process: it runs a small sibling watchdog that relaunches it if it is killed, and the app relaunches the
watchdog if the watchdog is killed. The only clean way to stop the pair is the **password-gated tray
Exit**. (This is inert for local development builds, so it never interferes with building or testing.)

### Try it

With the agent running, copy this line, then paste it somewhere:

```
Contact Dr. [PERSON_2886] at [EMAIL_4152], SSN [NID_11033], salary is [ECON_3636]
```

You'll paste back something like:

```
[PERSON_1] Dr. [PERSON_2] at john.doe@example.com, SSN [NID_1], salary is eviction
```

## Configuration

Two JSON files, both editable and hot-loaded at startup:

- **`appsettings.json`** (`Guardian` section) — detection-engine tunables: `EnabledCategories`,
  `TokenTemplate`, `EnableNer`/`Ner*`, `EnableGazetteers`, `EnableTopicLexicons`, `PersistMappings`,
  `DebounceMilliseconds`, file-redaction flags, and proxy hosts/port.
- **`settings.json`** — app-layer constants: theme palette (RGB arrays), window sizes, all
  user-facing strings/titles, notification and confirmation text, hero lines, tab/period labels,
  the admin password, **log-rotation limits** (v17 default: trim logs back to 450 MB once they exceed
  500 MB), `protection` (snooze durations), `images` (`ocrMaxEdge`, `ocrCacheSize`, and the v17 vault
  knobs `vaultRetentionDays`/`vaultMaxBytes`/`fingerprintGrid`/`fingerprintMaxHamming`), `storage` (the
  v17 janitor sweep interval + redacted-copy caps), `reporting`, `info`, and the v19 `licensing`
  (trial/update-checker + endpoint/master-key) and `companySync` (versioned companies.json remote sync)
  sections. Missing keys fall back to safe code defaults, and
  `ConfigConsistencyTests` keep this file in sync with the code.

## Local files (per-user, under `%LOCALAPPDATA%\Pseudonyx\`)

| File                                    | Contents                                                                       |
| --------------------------------------- | ------------------------------------------------------------------------------ |
| `mappings.bin`                        | DPAPI-encrypted reversible token↔original mapping                             |
| `runtime.json`                        | feature toggles (incl. NER on/off + quality-max flag)                          |
| `categories.json`                     | enabled GDPR data categories                                                   |
| `counters.json`                       | v33: lifetime redaction tallies by channel (survive log trimming)              |
| `redaction-mode.txt`                  | redaction strength (Soft / Medium / Full)                                      |
| `sites.json` / `sites-deleted.json` | per-company redact/block policy                                                |
| `activity.log`                        | copy/paste provenance (redacted previews only) — rotated at 500→450 MB (v17) |
| `image-vault/`                        | v17: DPAPI-encrypted originals of redacted images (3-day TTL, byte-capped)     |
| `consent.json` / `consent.key`      | signed consent record + DPAPI-protected signing key                            |
| `redacted/`                           | temporary redacted file copies (size-bounded)                                  |

The raw redaction audit ledger (`Pseudonyx-redactions.log`, in your user profile) contains original
values in plaintext for your own GDPR record-keeping and is likewise rotated.

## Performance

Event-driven, so **CPU is ~0% when idle** (measured ~0.1%). Memory and latency depend on whether the
optional ONNX **NER** tier is enabled (open-vocabulary names/locations):

| Path                             | Gazetteer only (NER off — default when no model is installed) | NER on (fp32`dslim/bert-base-NER`)                             |
| -------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------- |
| Text redaction (avg end-to-end)  | ~115 ms · 1.66 MB/s                                           | **~1180 ms · 0.16 MB/s** (detect = 96–99% NER inference) |
| Image redaction (avg end-to-end) | ~89 ms                                                         | ~310 ms                                                          |
| Working-set RAM                  | ~0.6 GB (Debug; ~65–80 MB Release, model absent)              | **~1.37 GB** (411 MB fp32 model resident)                  |
| Idle CPU                         | ~0.1%                                                          | ~0.1%                                                            |

[GENE_678]
[GENE_679]
> **quantized INT8** model or the smaller **multilingual DistilBERT** (`tools/fetch-ner-model.ps1 -Model "Davlan/distilbert-base-multilingual-cased-ner-hrl"`) — ~15–80 ms and ~+200–400 MB; (2) leave
[GENE_680]
[GENE_681]
[GENE_682]
>
[GENE_683]
[GENE_684]
[GENE_685]

```powershell
powershell -File tools/run-speed-test.ps1   # text+image throughput/latency -> analytics/speed-test-*
powershell -File tools/measure-usage.ps1     # CPU/RAM/latency under load  -> analytics/usage-report.html
```

## Privacy & security notes

- Redaction is local: the personal data you redact never leaves the machine. Pseudonyx sends license/update checks and the companies.json sync, and — **only if you opt in** (off by default; toggle in Settings ▸ Privacy) — periodic **pseudonymous usage analytics** (usage/configuration metadata only, never your redacted content; the insights payload is end-to-end encrypted). See `consent/PRIVACY-NOTICE.md` for the full privacy notice, and [`docs/GDPR-AND-COMPLIANCE.md`](docs/GDPR-AND-COMPLIANCE.md) for the impact assessment.
- The reversible mapping, the redaction audit ledger and the consent signing key are DPAPI-encrypted (per-user, per-machine). Settings ▸ Privacy also offers a **"Verify a file's redaction"** check (the automated copy-paste test) and one-click deletion of history, reversible data and consent.
- The reversible mapping and the consent signing key are DPAPI-encrypted (per-user, per-machine).
- Detection is heuristic — it reduces exposure but is not a guarantee. Review before sharing highly
  sensitive material and tune `EnabledCategories`/lexicons for your data.
- Download un-redaction only ever acts on files you actually downloaded from the web (identified by the
  Windows Mark-of-the-Web); files created locally by other programs (for example a background AI agent
  writing documents) are never touched. When your trial ends or a license expires, all redaction and
  un-redaction stops completely — no processing and no notifications.
- Diagnostics and bug reports you send include your operating system and version, device name and hardware
  specs (CPU, cores, RAM, architecture) so the Pseudonyx team can troubleshoot remotely. They never include
  your redacted content, license key, or the reversible mapping.
- The upload proxy is opt-in and only decrypts a precise allow-list of upload endpoints; every other
  host is blind-tunneled, and a crash/exit leaves the internet working (PAC `; DIRECT` fallback). The flip
  side of that narrowness, stated plainly: an upload to an endpoint outside the list does not reach the
  proxy at all, so it is neither inspected nor logged. That is why browser attachments are protected by the
  extension at attach time instead of by the host list.

## License

Personal project. No license granted for redistribution unless stated otherwise by the owner.

## Uninstall

The cleanest way to remove an installed copy is:

```powershell
"%LOCALAPPDATA%\Pseudonyx\app\Pseudonyx.exe" --uninstall
```

This removes the program files (`%LOCALAPPDATA%\Pseudonyx\app\`) and both auto-start entries (the HKCU
`Run` value and the `Pseudonyx` Scheduled Task). Everything else lives under `%LOCALAPPDATA%\Pseudonyx\`
(mappings, toggles, logs, redacted temp copies, the DPAPI-encrypted `image-vault\`, the install marker)
plus the home-dir audit ledger `%USERPROFILE%\Pseudonyx-redactions.log`; delete `%LOCALAPPDATA%\Pseudonyx\`
(and the ledger if you kept one) to erase the rest. Nothing is written elsewhere; a background **storage
janitor** keeps those files capped while the app runs, so a fleet install never bloats a profile.
