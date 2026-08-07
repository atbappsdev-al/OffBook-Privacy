# Pro pricing / sale badge — publishing guide

`pricing.json` in this repo controls the **LIMITED-TIME OFFER badge** on the in-app
Pro paywall in both OffBook apps. Editing it turns the badge on or off. **No app
update or store release is needed.**

Served from GitHub Pages at:

```
https://atbappsdev-al.github.io/OffBook/pricing.json
```

> **This repo is public.** Anything written here is world-readable. That is fine for
> prices — they're on the store listing anyway — but don't add notes about unreleased
> plans or promotions you haven't announced.

**It does not set the price.** The price on the paywall always comes from the App
Store / Google Play, whatever you have configured there. This document only tells the
apps what **full price** is, so they can tell whether today's price is a discount.

---

## Contents

- [Why this file exists](#why-this-file-exists)
- [How to run a sale](#how-to-run-a-sale)
- [How to end a sale](#how-to-end-a-sale)
- [How long until users see it](#how-long-until-users-see-it)
- [Field reference](#field-reference)
- [Traps worth remembering](#traps-worth-remembering)
- [Pre-publish checklist](#pre-publish-checklist)
- [Where the code lives](#where-the-code-lives)

---

## Why this file exists

OffBook Pro is a **single non-consumable** purchase. Neither StoreKit nor Play
Billing has any notion of "regular price vs current price" for one — they report what
the product costs *today* and nothing else. So the only way an app can know a price is
**reduced** is if we publish what full price is.

Before this file, both apps compared the live price against `4.99` baked into the
binary. That was wrong twice over:

- **In the UK it badged full price as a sale.** On iOS the baseline was written as a
  `Decimal` float literal, which is really 4.990000000000001024 — fractionally above
  the store's exact £4.99 — so "is the price below the baseline?" was true at full
  price.
- **Everywhere else it compared against the wrong currency.** A bare `4.99` measured
  against €4.49 or CA$6.99 says nothing at all; €4.49 is an ordinary full-price tier,
  and it would have worn a sale badge permanently.

Hence: full price per currency, published here, with a master switch.

---

## How to run a sale

1. **Change the price on the stores first** (App Store Connect / Play Console) and let
   it go live. This file describes reality; it doesn't create it.
2. Make sure `baseline_prices` lists the **regular** price for every currency you care
   about — the price you're discounting *from*, not the sale price. Leave those values
   alone during the sale.
3. Set `"sale_active": true`.
4. Commit and push. GitHub Pages redeploys in a minute or two.
5. Verify the live file loads and is valid JSON:
   ```
   curl -s https://atbappsdev-al.github.io/OffBook/pricing.json | jq .
   ```

### Copy-paste template

```json
{
  "schema_version": 1,
  "sale_active": true,
  "baseline_prices": {
    "GBP": "4.99",
    "USD": "6.99",
    "CAD": "6.99"
  }
}
```

A user sees the badge only when **both** are true:

- `sale_active` is `true`, **and**
- the store's price for **their** storefront is **strictly below** the
  `baseline_prices` entry for **their** currency.

So a storefront where you didn't actually drop the price shows no badge, even with the
switch on — which is the point. A currency with no entry never shows the badge.

---

## How to end a sale

Set `"sale_active": false` and push. Leave `baseline_prices` as it is — those are
your regular prices, ready for next time.

Do this **at the same time as** (or before) restoring the price on the stores. If the
price goes back up while the switch is still on, nothing lies to the user — the badge
disappears on its own because the price is no longer below the baseline — but flip it
back anyway so the document reflects reality.

**If you permanently change a regular price**, update that currency's entry here to
match. A stale baseline that's *above* the new regular price would badge full price as
a discount for as long as it's wrong.

---

## How long until users see it

**The next time anyone opens a screen that shows the price.** Unlike
`announcements.json` and `endpoints.json`, which each app fetches at most once every
6 hours on a cold start, this document is re-fetched **every time a price is about to
be displayed** — the paywall, and the locked Statistics / confidence cards. A stale
model order is invisible; a stale sale badge is a price claim, so it isn't allowed to
lag.

- Both apps bypass their own HTTP cache **and** the GitHub Pages CDN (via a
  per-second query value), so the change reaches devices as soon as Pages has
  redeployed — not up to ten minutes later, which is what the Pages
  `Cache-Control: max-age=600` would otherwise impose.
- The fetch is capped at **2.5 seconds** so it can never hold up a paywall. If it
  times out or fails, the **last-known-good cached copy** decides (installed at app
  launch), and ultimately "no sale".
- Several price surfaces opening at once share a single request.

Practical upshot: push the change, wait for Pages to redeploy (a minute or two), then
reopen the paywall — you should see the badge appear or disappear immediately. There
is no cache window to wait out.

---

## Field reference

| Field | Required | Notes |
|---|---|---|
| `schema_version` | **Yes** | Must be exactly `1`. Any other value → the whole document is discarded and no badge shows |
| `sale_active` | No | `true` to allow the badge, `false` (or omitted) to suppress it everywhere |
| `baseline_prices` | No | Object of **ISO 4217 currency code → regular price**. Codes are case-insensitive; three letters only. Omitted or empty → no badge anywhere |

Prices must be written as **strings** (`"4.99"`), digits and at most one `.` with up to
six decimal places. Whole numbers may be unquoted (`800`). Anything else — `4,99`,
`"$6.99"`, `"1e3"`, a negative, or a fractional **unquoted** number like `4.99` — is
dropped, and that currency simply never badges.

> Why strings? An unquoted `4.99` can only reach the apps through a binary floating
> point value, whose nearest value to 4.99 is *above* 4.99. That is the exact
> imprecision that badged full price as a sale in the first place, so the parsers
> refuse to read fractional numbers.

Unknown fields are ignored by both apps, which is why `_readme` in the live file is
harmless.

---

## Traps worth remembering

- **`baseline_prices` is the full price, not the sale price.** Putting the discounted
  price in makes the badge vanish (nothing is below it).
- **Numbers must be quoted.** `"GBP": 4.99` is silently dropped; `"GBP": "4.99"` is
  correct. Run the checklist below.
- **A currency you don't list never badges.** Add every storefront you want the badge
  in — the apps deliberately don't guess from other currencies.
- **Equal isn't below.** A price *equal* to the baseline is full price by definition.
- **Nothing here is user-visible copy.** The badge wording lives in the apps; this
  document can't change it.
- **Pages redeploy is the only wait.** The apps don't cache-lag this document, but
  GitHub still needs a minute or two to publish your commit. If `curl` shows the old
  content, the app will too.
- **Everything fails closed.** A malformed document, a wrong `schema_version`, a
  missing file, or no network on a fresh install all mean "no badge" — never a badge
  claiming a discount that isn't real. A broken document therefore looks exactly like
  "no sale", with no error surface anywhere. Always run the `jq` check.

---

## Pre-publish checklist

- [ ] The store price has actually changed (or is about to), in the storefronts you're
      badging.
- [ ] `schema_version` is `1`.
- [ ] `sale_active` is the value you intend.
- [ ] Every price is a **quoted string**, and is the **regular** price.
- [ ] Every currency code is three letters and matches the storefronts you care about.
- [ ] `curl -s https://atbappsdev-al.github.io/OffBook/pricing.json | jq .` returns the
      document (after Pages redeploys).
- [ ] Sanity-check on a device: the paywall shows your storefront's real price, and the
      badge is present/absent as intended (allow for the 6-hour cache).

---

## Where the code lives

| Platform | Files |
|---|---|
| iOS | `OffBookKit/Sources/OffBookKit/Pricing/ProPricingConfig.swift` (model + parser), `OffBook/Services/ProPricingService.swift` (fetch/cache), `OffBook/Services/StoreManager.swift` (`remotePricing`, `priceInfo()`), `OffBook/Features/Paywall/PaywallSheet.swift` (the badge) |
| Android | `app/src/main/java/com/abdev/offbook/pricing/` (`ProPricingConfig`, `ProPricingParser`, `ProPricingFetcher`, `ProPricingManager`), `BillingManager.getProPriceInfo` (the sale decision), `ui/paywall/PaywallBottomSheet.kt` (the badge) |

Both platforms share this document byte-for-byte and have parser tests pinned to the
shape above (`ProPricingParserTests` / `ProPricingParserTest`). If you change the
schema, change both apps and bump `schema_version` — but note that a bump makes every
already-shipped build discard the document, so prefer adding fields.

The same fail-closed, cache-first pattern powers `announcements.json` (see
[ANNOUNCEMENTS.md](ANNOUNCEMENTS.md)) and `endpoints.json` — but on a 6-hour cadence.
This document alone refreshes per price display, deliberately: see
[How long until users see it](#how-long-until-users-see-it).
