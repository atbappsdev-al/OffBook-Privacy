# In-app announcements — publishing guide

`announcements.json` in this repo is the **live in-app announcement banner** for both
OffBook apps. Editing it changes what users see on the Home screen. **No app update
or store release is needed.**

Served from GitHub Pages at:

```
https://atbappsdev-al.github.io/OffBook/announcements.json
```

> **This repo is public.** Anything written here — including this file and every
> draft announcement — is world-readable. Don't put unreleased plans, internal
> URLs, or anything you wouldn't publish in the JSON or in the commit message.

---

## Contents

- [How to publish](#how-to-publish)
- [How long until users see it](#how-long-until-users-see-it)
- [Field reference](#field-reference)
- [Text formatting (markup)](#text-formatting-markup)
- [Splitting the audience](#splitting-the-audience)
- [Targeting free vs Pro (the announcements array)](#targeting-free-vs-pro-the-announcements-array)
- [Not currently possible](#not-currently-possible)
- [Retiring an announcement](#retiring-an-announcement)
- [Traps worth remembering](#traps-worth-remembering)
- [Pre-publish checklist](#pre-publish-checklist)
- [Testing before you publish](#testing-before-you-publish)
- [Where the code lives](#where-the-code-lives)

---

## How to publish

1. Edit `announcements.json` on `main` in this repo.
2. **Give it a new `id`.** This is the single most important rule — see
   [Traps](#traps-worth-remembering).
3. Commit and push. GitHub Pages redeploys in a minute or two.
4. Verify the live file loads and is valid JSON:
   ```
   curl -s https://atbappsdev-al.github.io/OffBook/announcements.json | jq .
   ```

If the JSON is malformed, or `schema_version` isn't `1`, or `id`/`title` are missing
or blank, **both apps silently show nothing**. There is no error surface anywhere —
a broken document looks exactly like "no announcement". Always run the `jq` check.

### Copy-paste template

```json
{
  "schema_version": 1,
  "id": "2026-08-14-new-feature",
  "title": "🎭 **Something new** in OffBook",
  "message": "We've added a thing.\n\n- It does **this**\n- And __this__\n\nEnjoy! 🎉",
  "platforms": ["android", "ios"],
  "min_app_version": 1,
  "max_app_version": 999,
  "cta_action": "url",
  "cta_text": "See what's new",
  "cta_url": "https://offbook.abdev.dev/whats-new",
  "expires_at": "2026-09-14T00:00:00Z",
  "priority": "normal"
}
```

---

## How long until users see it

- Each app fetches the document **at most once every 6 hours**, and only on a
  **cold start** (not on resume, not on navigation).
- Between fetches the **cached copy** decides what shows.
- So worst case is roughly *6 hours, and then the user's next cold start*. Fresh
  installs fetch immediately.
- A failed fetch (offline, timeout, non-200) deliberately doesn't update the
  timestamp, so the next cold start retries.
- The show/hide decision is made **once per app process**. Navigating around won't
  make a banner appear or disappear mid-session.

Practical upshot: don't publish something time-critical and expect it within
minutes. Assume most users see it within a day.

---

## Field reference

The document is a single JSON object — **one announcement at a time**.

| Field | Required | Notes |
|---|---|---|
| `schema_version` | **Yes** | Must be exactly `1`. Any other value → the whole document is discarded |
| `id` | **Yes** | Non-blank. The per-user "already dismissed" key. **Bump it for every new announcement** |
| `title` | **Yes** | Non-blank. Supports [markup](#text-formatting-markup) |
| `message` | No | Body text, supports markup. Omit or leave blank and only the title shows |
| `platforms` | No | `["android"]`, `["ios"]`, both, or omit/empty for all. Case-insensitive |
| `min_app_version` | No | Inclusive lower build-number bound. **Platform-local — see [Splitting](#splitting-the-audience)** |
| `max_app_version` | No | Inclusive upper build-number bound. Same caveat |
| `cta_action` | No | `"share"`, `"url"` or `"paywall"`; anything else → no button. See below |
| `cta_text` | No | Button label. **Blank or missing = no button, regardless of `cta_action`** |
| `cta_url` | No | Required when `cta_action` is `"url"`; missing → no button. Not used by the other actions |
| `share_text` | No | Overrides the share-sheet body when `cta_action` is `"share"`. Omit to use the built-in app copy |
| `expires_at` | No | ISO-8601 UTC. Hidden once the device clock reaches it. Omit = never expires |
| `priority` | No | **Parsed but completely unused.** Reserved. Setting it does nothing |
| `audience` | No | `"all"`, `"free"`, `"pro"`, `"trial"`. Omit = everyone. See [Targeting free vs Pro](#targeting-free-vs-pro-the-announcements-array) |
| `announcements` | No | Array of targeted entries. When present and non-empty it **replaces** the top-level announcement for builds that understand it. Same section |

Unknown fields are ignored by both apps, so adding notes-style keys is harmless
(but remember the repo is public).

### How the CTA button resolves

| Action | Tap behaviour |
|---|---|
| `"url"` | Opens `cta_url` in the browser |
| `"share"` | Opens the system share sheet (`share_text`, or the built-in app copy) |
| `"paywall"` | Opens the **in-app Pro paywall** — for offer/upsell announcements. Needs only `cta_text`, no URL |

The button degrades to "no button" (dismiss-only) rather than erroring, whenever:

- `cta_action` is missing, empty, or an unrecognised value, **or**
- `cta_text` is blank or missing, **or**
- `cta_action` is `"url"` but `cta_url` is blank or missing.

This is why the current live document shows no button — its `cta_text` is `""`.

> ⚠️ **`"paywall"` needs app builds that know the action** (added to both apps
> July 2026 — check it has shipped in the store builds you care about). Older
> builds treat it as unrecognised and show the banner with **no button** —
> harmless, but the offer is unreachable for them, so consider a
> `min_app_version` bound (per platform). A user who already owns Pro never
> gets the paywall — their tap is just a dismissal — but the cleaner pattern is
> to pair the paywall CTA with `audience: "free"` or `"trial"` so Pro users
> don't see the upsell banner at all (see the example file). Note that as of
> July 2026 audience targeting has shipped on **iOS only** — Android builds
> still read just the top-level fields, so an offer that must reach Android
> needs the paywall CTA at the top level; the app's own Pro check still keeps
> the paywall away from Pro users who tap it.

Both **Dismiss** and the **CTA** mark the announcement as seen, and the dismissal is
saved *before* the CTA runs — so a cancelled share sheet or a failed link can never
make the banner come back.

---

## Text formatting (markup)

`title` and `message` both run through a deliberately tiny markup tokenizer.
It is **not** Markdown.

| Syntax | Result |
|---|---|
| `**text**` | **Bold** |
| `__text__` | <u>Underline</u> (two underscores, not one) |
| `**__text__**` | Both — they nest fine, in either order |
| `- ` at the very start of a line | Bullet point (`•` + hanging indent) |
| `\n` | Line break (escaped, as JSON requires) |
| Emoji | Passed straight through — 🎭 🎉 🙌 all fine |

### What is NOT supported

- ❌ *Italics* — no `*text*` or `_text_`
- ❌ Links — a bare URL renders as plain text, it is not tappable. Use `cta_url`
- ❌ Headers, blockquotes, code, tables, images
- ❌ Escaping — there is no way to show a literal `**` or `__`
- ❌ Numbered lists — `1. ` is just text

### Formatting rules that catch people out

- **Styles never cross a line break.** `"**bold\nover two lines**"` will *not* work —
  the marker on each line is unpaired and gets stripped. Style each line separately.
- **Unpaired markers are silently removed.** `"a ** b"` renders as `a  b`, not
  `a ** b`. You never get stray asterisks, but you also get no warning.
- **The bullet prefix must be exactly `- ` at column zero.** `"-x"` (no space) and
  `" - x"` (leading space) are both plain text, not bullets.
- **Blank lines are preserved**, so `\n\n` gives you a paragraph gap.
- Bullets technically work in `title` too, but it looks odd — keep them in `message`.

### Worked example

```json
"message": "**What's new:**\n- Faster **script scanning**\n- __Improved__ cue detection\n\nThanks for rehearsing with us 🎭"
```

Renders as:

> **What's new:**
> • Faster **script scanning**
> • <u>Improved</u> cue detection
>
> Thanks for rehearsing with us 🎭

---

## Splitting the audience

Three dimensions work **right now**, with no app changes. They combine with AND —
every gate must pass for the banner to show.

### By platform

```json
"platforms": ["ios"]
```

iOS only. Use `["android"]` for Android only, `["android", "ios"]` or omit for
everyone.

### By app version

```json
"min_app_version": 12,
"max_app_version": 40
```

⚠️ **These are platform-local build numbers.** Android compares its `versionCode`;
iOS compares its `CFBundleVersion`. They are two completely independent counters, so
`min_app_version: 12` means a *different release* on each platform.

**Only use version bounds together with a single-platform `platforms` value**,
otherwise the window means something different on each store. If you need a version
window on both platforms at once, you can't — see
[Not currently possible](#not-currently-possible).

### By time

```json
"expires_at": "2026-09-14T00:00:00Z"
```

Hidden from that instant onward. Accepts `Z`-suffixed timestamps with or without
fractional seconds, and explicit offsets like `+01:00`.

There is **no** `starts_at`. Publishing is the start — you can't schedule ahead.

### Per user, automatically

Each user is shown a given `id` **once**. Dismissing it (or tapping the CTA) records
the id permanently on that device.

---

## Targeting free vs Pro (the announcements array)

> ⚠️ **Requires app builds with audience targeting.** Older builds ignore the
> `announcements` array entirely and read the **top-level fields** instead. Always
> leave a sensible generic announcement at the top level — it is what those users
> see. Check that the change has actually shipped in the store builds you care
> about before relying on targeting.

A single document can only carry one announcement, so different copy per segment
needs the optional `announcements` **array**. Each entry takes every normal field
plus `audience`, and clients evaluate entries **in order, first match wins**.

`announcements.example.json` in this repo is a complete, valid four-way split
(iOS/Android × free/Pro). Copy its shape. Its free entries show the natural
offer pattern: `audience: "free"` + `cta_action: "paywall"`, so the button
opens the in-app Pro paywall directly.

```json
{
  "schema_version": 1,

  "id": "2026-08-01-generic",
  "title": "🎭 **OffBook** has news",
  "message": "Generic copy that OLDER builds see.",

  "announcements": [
    { "id": "…-ios-pro",      "platforms": ["ios"],     "audience": "pro",  "title": "…" },
    { "id": "…-ios-free",     "platforms": ["ios"],     "audience": "free", "title": "…" },
    { "id": "…-android-pro",  "platforms": ["android"], "audience": "pro",  "title": "…" },
    { "id": "…-android-free", "platforms": ["android"], "audience": "free", "title": "…" }
  ]
}
```

### What each audience value means

| Value | Matches |
|---|---|
| `"all"` (or omitted) | Everyone |
| `"pro"` | Users who own OffBook Pro |
| `"free"` | Users who do **not** own Pro — **including** users mid-trial, since they don't own it yet and upsell copy still applies |
| `"trial"` | The narrower segment: non-Pro users with trial sessions still remaining |

### Rules that matter

- **Order is precedence.** List the most specific entries first. A user sees at most
  **one** banner — the first entry passing every gate.
- **Every entry needs its own unique `id`.** Dismissal is tracked per id.
- **Entries do not carry `schema_version`.** It stays once, at the top level.
- **An unrecognised `audience` matches nobody.** `"prro"` shows to no one rather
  than leaking to everyone — it fails closed, deliberately. So does a value a
  future publisher invents that current builds don't know.
- **An invalid entry is dropped individually** (missing `id`/`title`, or not an
  object) without invalidating the others.
- **An array that is present but yields no usable entry shows nothing** — it does
  *not* fall through to the top-level fallback. Only an **absent or empty** array
  does that.
- **Entries are independent for dismissal.** A free user who dismisses the free
  variant and then buys Pro will match the Pro entry on a later cold start and see a
  second banner. If you don't want that, give the two entries copy that reads
  sensibly in sequence, or publish only one of them.

### Checking your split before publishing

`jq` can confirm each segment resolves to the entry you intended:

```
jq -r '.announcements[] | "\(.platforms[0])\t\(.audience)\t\(.id)"' announcements.json
```

Every platform × audience combination you care about should appear exactly once.

---

## Not currently possible

Documented here so it's clear what needs code, not just a JSON edit.

| Want | Status |
|---|---|
| **Scheduling a future start** | ❌ No `starts_at` field. Publishing is the start |
| **Two banners at once** | ❌ One user sees at most one entry; the rest are skipped, not queued |
| **Queueing / explicit ordering** | ❌ `priority` exists in the schema but no client code reads it. Array order is the only precedence |
| **A version window meaning the same thing on both platforms** | ❌ Build numbers are platform-local — use one entry per platform |
| **Targeting anything else** (locale, install age, production count, streak) | ❌ Only platform, build number, time and entitlement are wired into the decision |

---

## Retiring an announcement

Any of these stops the banner. Pick by intent:

- **Cleanest — expire it.** Set `expires_at` to a past timestamp, e.g.
  `"2020-01-01T00:00:00Z"`. This is what the current live document does. The file
  stays readable as a record of what was last published.
- **Blank the title.** A blank or missing `title` invalidates the whole document, so
  nothing shows. Blunter, and looks like a mistake later.
- **Publish a replacement.** New `id` + new content simply supersedes it.

⚠️ Deleting `announcements.json` entirely is **not** a good way to retire one. The
fetch fails, and each app falls back to its **cached copy** — so users keep seeing
the old banner until their cache is replaced. Always leave a valid, expired file in
place.

---

## Traps worth remembering

1. **Editing content without changing the `id` reaches nobody who already dismissed
   it.** The id *is* the dismissal key. Every new announcement needs a fresh id —
   this is the mistake that's easiest to make and hardest to notice, because it looks
   fine on a clean install and does nothing on a real device.
2. **A typo in `expires_at` means "never expires", not "expired".** Unparseable
   timestamps are treated as no expiry. It fails open — the banner runs forever.
   Check the format when you set it.
3. **`cta_text: ""` silently removes the button** even with a perfectly good
   `cta_action` and `cta_url`.
4. **`schema_version` must stay `1`.** Bumping it to `2` makes every currently
   shipped app discard the document permanently. There is no graceful upgrade path
   for existing builds — add optional keys instead of bumping. This is exactly why
   audience targeting arrived as an optional `announcements` array at version 1
   rather than a version 2 document.
5. **Older builds ignore the `announcements` array.** They read the top-level fields
   instead, so a document whose top level is blank or expired shows them nothing at
   all. Targeting does not reach users until the supporting build has shipped.
6. **The in-app review prompt wins.** If the review prompt has been triggered in the
   current session, the banner is suppressed entirely so two "please engage" surfaces
   never stack. This is expected behaviour, not a bug — worth remembering when a
   banner doesn't appear during testing.
7. **Everything fails silently for users.** No error state, no analytics — a broken
   document is indistinguishable from "no announcement in flight". There *is*
   debug-build logging (`Log.d` with tags `AnnouncementFetcher` / `AnnouncementManager`
   on Android, `print` on iOS), so attach a debug build if you need to see why
   something was rejected. Otherwise work through the checklist above.

---

## Pre-publish checklist

- [ ] `schema_version` is `1`
- [ ] `id` is **new** and unlike any previously published id
- [ ] Valid JSON (`jq .` passes)
- [ ] `title` non-blank; `\n` escaped properly in `message`
- [ ] Every `**` and `__` is paired **within its own line**
- [ ] If there's a button: `cta_text` non-blank, and `cta_url` set when
      `cta_action` is `"url"`
- [ ] `expires_at` in the **future** and in valid ISO-8601 (or omitted)
- [ ] Version bounds only used alongside a single-platform `platforms`
- [ ] Read the rendered text once more — no unintended asterisks, no italics assumed
- [ ] Nothing confidential (public repo)

If you're using the `announcements` array as well:

- [ ] Every entry has its own **unique, new** `id`
- [ ] `audience` is exactly one of `all` / `free` / `pro` / `trial` — a typo shows
      the entry to **nobody**
- [ ] No `schema_version` inside the entries (top level only)
- [ ] Most specific entries listed first
- [ ] Each segment you care about resolves to one entry (`jq` check above)
- [ ] The **top-level** fields still hold sensible generic copy — that's what older
      builds show

---

## Testing before you publish

Both apps have a debug switch that shows a hardcoded banner exercising every markup
feature, without touching this file:

- **Android** — `DEBUG_FORCE_ANNOUNCEMENT` in `announce/AnnouncementManager.kt`
- **iOS** — `debugForceAnnouncement` in `OffBook/Services/AnnouncementService.swift`

Both must **never** ship as `true`. While forced, the banner reappears on every
launch (dismissal isn't persisted).

To re-test a *real* announcement you've already dismissed, either bump the `id` or
clear the stored id:

- **Android** — key `last_seen_announcement_id` in the `offbook_trial_prefs`
  SharedPreferences file (clear app data works too)
- **iOS** — `UserDefaults` key `offbook.announcements.lastSeenId`

The cached document lives alongside it (`announcement_cached_json` /
`offbook.announcements.cachedJSON`) with its fetch timestamp — clear those to force
an immediate re-fetch instead of waiting out the 6-hour window.

---

## Where the code lives

**Android** (`OffBook` repo, `app/src/main/java/com/abdev/offbook/`)

| File | Role |
|---|---|
| `announce/AnnouncementFetcher.kt` | Downloads the document (5 s timeout, best-effort) |
| `announce/AnnouncementParser.kt` | Tolerant JSON → model |
| `announce/AnnouncementDecision.kt` | Pure show/hide gates |
| `announce/AnnouncementManager.kt` | Cache, 6-hour window, dismissal, exposes the banner |
| `util/AnnouncementMarkup.kt` | Markup tokenizer |
| `ui/widget/AnnouncementBannerView.kt` | The banner UI |

**iOS** (`OffBookiOS` repo)

| File | Role |
|---|---|
| `OffBook/Services/AnnouncementService.swift` | Fetch, cache, 6-hour window, dismissal |
| `OffBookKit/Sources/OffBookKit/Announcements/AnnouncementModels.swift` | Model + tolerant parser |
| `OffBookKit/Sources/OffBookKit/Announcements/AnnouncementDecision.swift` | Pure show/hide gates |
| `OffBookKit/Sources/OffBookKit/Announcements/AnnouncementMarkup.swift` | Markup tokenizer |
| `OffBook/Components/AnnouncementBanner.swift` | The banner UI |

The parsers, decision logic and markup tokenizers are **deliberate 1:1 twins** with
byte-identical test fixtures. If you change the schema, change both — and update this
file.

`endpoints.json` in this repo works the same way for the remote Gemini model order.
