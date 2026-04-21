# Docs UIKit Coverage — Phase 1 Design Spec

**Date:** 2026-04-21
**Scope:** Tier 1 (critical path) iOS docs only
**Phases:** This is Phase 1 of 3. Phases 2-3 cover Tier 2 (lifecycle surfaces) and Tier 3 (reference).

---

## Problem

ZeroSettle's iOS docs are written SwiftUI-first throughout. UIKit apps — which make up a significant share of production iOS apps — have no clear integration path for the handler installation, quickstart flow, or checkout presentation. A developer question surfaced this gap concretely: the `.zeroSettleHandler()` modifier has no documented UIKit equivalent.

---

## Decisions Made

| Decision | Choice |
|---|---|
| Format | Mintlify `<Tabs>` per code block (SwiftUI \| UIKit) |
| Prose style | SwiftUI-first; UIKit divergences get inline `<Note>` callouts |
| Audit scope | All three tiers (Phase 1 covers Tier 1) |
| Redesign latitude | Full — new articles, nav changes, prose rewrites permitted |
| Phasing | Phase 1 first; defer Phases 2–3 until Phase 1 ships |

---

## Scope Boundaries

### In scope — Phase 1
- `iap/installation.mdx` — Swift section only
- `iap/quickstart.mdx` — full rebuild
- `iap/payment-sheet.mdx` — split + rebuild
- `iap/preloading.mdx` — cleanup, absorbs Performance & Caching
- `iap/universal-links.mdx` — trim + tabs conversion
- `docs.json` — nav updates for new articles and renamed quickstart

### Out of scope — Phase 1
- `iap/quickstart-android.mdx`, `iap/quickstart-flutter.mdx` — other platforms untouched
- Android/Kotlin sections of `payment-sheet.mdx` — moved or deleted (see below), not rewritten
- All Tier 2 docs (`offer-tip-view`, `migration-manager`, `offer-manager`, `upgrade-offer`, `cancel-flow`, `custom-cancel-flow`, `customer-portal`, `switch-and-save`, `campaigns`)
- All Tier 3 docs (`overview`, `best-practices`, `storekit-integration`, `subscription-state`, `user-identity`, `troubleshooting`, `sample-app`)
- `api-reference/*`, `releases/*` — untouched
- SDK source code — docs only

### Guardrails
- No *.mdx* file is deleted in Phase 1 — only created or rewritten. Sections *within* rebuilt files (e.g., the Android block inside `payment-sheet.mdx`) may be removed as part of the rebuild.
- All nav changes go through `docs.json` — no broken links
- Every new article and every structural move is listed in this spec with a rationale

---

## New Information Architecture

### Getting Started (nav group — unchanged structure)

| File | Change | Notes |
|---|---|---|
| `iap/account-setup.mdx` | None | |
| `iap/installation.mdx` | **Effectively untouched** | Confirmed: no `zeroSettleHandler` or framework-specific snippet in this file. SPM install and Stripe setup are already framework-neutral. At most a minor prose note that SwiftUI/UIKit are both supported. |
| `iap/quickstart.mdx` | **Full rebuild** | Renamed "Quickstart (iOS)". See per-article spec below. |
| `iap/quickstart-android.mdx` | None | |
| `iap/quickstart-flutter.mdx` | None | |

### Checkout (nav group — new articles added)

| File | Change | Notes |
|---|---|---|
| `iap/payment-sheet.mdx` | **Split + rebuild** | iOS only, ≤400 lines. See per-article spec below. |
| `iap/preloading.mdx` | **Absorbs section + tabs** | Absorbs "Performance & Caching" from old payment-sheet. |
| `iap/checkout-errors.mdx` | **NEW** | Extracted "Error Handling" section from payment-sheet (~144 lines). |
| `iap/promotions.mdx` | **NEW** | Extracted "Promotions" section from payment-sheet (~123 lines). |
| `iap/universal-links.mdx` | **Trim + tabs** | Remove "Step 4: Handle Checkout Results" (duplicated elsewhere). |

### Android content from `payment-sheet.mdx`

The current file has ~300 lines of combined Kotlin/Android content. During implementation, diff this against `iap/quickstart-android.mdx`:
- **(a) Delete** if it's a duplicate of quickstart-android (default assumption)
- **(b) Move** to a new `iap/payment-sheet-android.mdx` if it has unique material not in the quickstart

Implementer reports finding in commit message. Phase 1 nav assumes option (a); if (b), add `iap/payment-sheet-android` to the Checkout group in `docs.json`.

---

## Per-Article Spec

### `iap/quickstart.mdx` — Full rebuild

**Current state:** 285 lines, titled "Quickstart (Swift)", SwiftUI throughout. No UIKit path.

**Target:** ≤350 lines, titled "Quickstart (iOS)". Single linear flow with SwiftUI/UIKit tabs at every code block where the frameworks diverge.

**Structure:**
1. **What You'll Build** — same as today (framework-neutral prose)
2. **Prerequisites** — same, add one `<Note>` that UIKit apps need SceneDelegate for the link handler
3. **Step 1: Configure** — `ZeroSettle.configure(publishableKey:)`. Framework-neutral. No tabs (identical code).
4. **Step 2: Install the handler** — `<Tabs>` showing `.zeroSettleHandler()` (SwiftUI) vs SceneDelegate `handleUniversalLink` + cold-start `willConnectTo` (UIKit). `<Note>` explaining UIKit also needs an Associated Domains entitlement (same as SwiftUI, just clarifying it's not modifier-delivered).
5. **Step 3: Fetch products** — `ZeroSettle.shared.products()`. Framework-neutral. No tabs.
6. **Step 4: Present checkout** — `<Tabs>` showing `.checkoutSheet()` modifier (SwiftUI) vs `CheckoutSheet.present(from:)` imperative call (UIKit).
7. **Step 5: Verify entitlement** — framework-neutral. No tabs.
8. **Complete Example** — `<Tabs>` with a full SwiftUI file and a full UIKit `UIViewController` file side by side.
9. **Next Steps** — same links as today, no changes.

**Key UIKit callouts (inline `<Note>`):**
- Step 2: "Preloading requires WebViews to be in the view hierarchy. If you want pre-warmed checkout (faster first open), host a 1pt `UIHostingController(rootView: Color.clear.zeroSettleHandler())` as a child of your root VC." Link to `iap/preloading`.
- Step 4: "`CheckoutSheet.present(from:)` uses `UIWindowScene` lookup internally — it does not need to be called from a specific view controller."

---

### `iap/payment-sheet.mdx` — Split + rebuild

**Current state:** 1260 lines. Contains: SwiftUI (330 lines), UIKit (90 lines), Kotlin/Android (90 lines), Presentation API (12 lines), Performance & Caching (215 lines), Android (219 lines), How It Works (13 lines), Error Handling (144 lines), Promotions (123 lines).

**Target:** ≤400 lines. iOS only. Covers what the payment sheet is, how to present it in both frameworks, customization basics. Everything else extracted.

**Sections extracted to new articles:**
- "Presentation API" (12 lines) → folded into rebuilt payment-sheet under the Presentation section (it describes the `item:` binding pattern — belongs in the main article, just consolidated)
- Performance & Caching → `iap/preloading.mdx`
- Error Handling → `iap/checkout-errors.mdx`
- Promotions → `iap/promotions.mdx`
- Kotlin/Android + Android → deleted or moved (see Android content decision above)

**Structure:**
1. **Overview** — what `CheckoutSheet` is, when to use it (1 paragraph)
2. **Presentation** — `<Tabs>` SwiftUI modifier (`.checkoutSheet()`) vs UIKit imperative (`CheckoutSheet.present(from:)`). Show both the basic call and the purchase completion handler.
3. **Customization** — header, footer, tint. `<Tabs>` where the API differs by framework. (Currently scattered — consolidated here.)
4. **Preloading** — 2-sentence description + link to `iap/preloading`
5. **Error handling** — 2-sentence description + link to `iap/checkout-errors`
6. **Promotions** — 1-sentence description + link to `iap/promotions`
7. **Next Steps**

**UIKit `<Note>` callouts:**
- Under Presentation: "If you're presenting from inside a SwiftUI view embedded in UIKit, prefer the `.checkoutSheet()` modifier — it resolves `UIWindowScene` from the containing hierarchy automatically."

---

### `iap/preloading.mdx` — Absorbs section + tabs

**Current state:** 196 lines. Covers preloading config, how it works, methods, performance, defaults.

**Target:** ~320 lines (absorbs ~130 lines of Performance & Caching from old payment-sheet, drops the rest that was duplicated there).

**Additions:**
- Absorb "Performance & Caching" section from old `payment-sheet.mdx` after deduplication
- Add SwiftUI/UIKit tabs to any snippet that references `CheckoutPreloaderPool` or the warmup pattern
- Add `<Note>` explaining the UIKit hosting requirement for preloading to work (the 1pt UIHostingController pattern). This is the most important UIKit-specific callout in the entire docs.

**UIKit `<Note>` for preloading:**
```
**UIKit:** The preloader hosts WebViews in the SwiftUI view hierarchy via `.zeroSettleHandler()`.
In UIKit apps, add a 1pt invisible child to your root view controller so WebKit can render off-screen:

    let host = UIHostingController(rootView: Color.clear.zeroSettleHandler())
    addChild(host)
    host.view.frame = CGRect(x: 0, y: 0, width: 1, height: 1)
    host.view.isUserInteractionEnabled = false
    view.addSubview(host.view)
    host.didMove(toParent: self)

Without this, the preloader still fetches checkout URLs but WebKit can't render — the payment
sheet opens with a spinner and waits for the ready signal (typically 1–2s).
```

---

### `iap/checkout-errors.mdx` — NEW

**Content source:** "Error Handling" section from current `payment-sheet.mdx` (lines ~984–1128).

**Structure:**
1. Brief intro (1 sentence — what errors ZeroSettle checkout can surface)
2. Error types table — every `ZeroSettleError` case, description, typical cause, recovery path
3. Catching errors — code snippet (SwiftUI/UIKit tabs — same catch syntax, but UIKit shows it in the imperative `present` callback rather than async modifier)
4. Retry patterns (brief, if covered in current content)

---

### `iap/promotions.mdx` — NEW

**Content source:** "Promotions" section from current `payment-sheet.mdx` (lines ~1128–1251).

**Structure:**
1. What promotion codes are (1 paragraph)
2. How to apply a code — SwiftUI/UIKit tabs
3. How ZeroSettle validates and applies the discount
4. Testing promotions in sandbox

---

### `iap/universal-links.mdx` — Trim + tabs

**Current state:** 574 lines. Already has UIKit SceneDelegate and AppDelegate sections but they're free-standing, not tabbed with SwiftUI equivalents.

**Target:** ≤400 lines.

**Changes:**
- Delete "Step 4: Handle Checkout Results" (~90 lines) — this is duplicated in payment-sheet and quickstart. Replace with a 2-sentence summary + cross-link to payment-sheet.
- Convert "Step 3: Handle the Callback" so SwiftUI and UIKit options are side-by-side `<Tabs>` (SwiftUI `.onContinueUserActivity` vs SceneDelegate). Currently they're sequentially buried under nested headings.
- Consolidate the two UIKit sections ("UIKit (SceneDelegate)" and "UIKit (AppDelegate - iOS 12 and earlier)") into one tab, with AppDelegate as a brief note inside it ("If you're on iOS 12 or earlier and don't use SceneDelegate...").
- Keep "Troubleshooting", "Testing Universal Links", "Fallback Handling" — these are good reference material, no cuts.
- Keep "Android Deep Links" section — out of Phase 1 scope, untouched.

---

### `iap/installation.mdx` — Minor edit

**Current state:** 191 lines. Framework-neutral SPM install steps. No SwiftUI assumptions except possibly in an example snippet.

**Target:** Same 191 lines ± 20. Only change: if any snippet shows `.zeroSettleHandler()` in an app entry point context, wrap in `<Tabs>` showing the SwiftUI App protocol vs AppDelegate pattern. Stripe setup section is neutral — untouched.

---

## `docs.json` Changes

```diff
- "iap/quickstart"              // label from "Quickstart (Swift)"
+ "iap/quickstart"              // title in frontmatter changes to "Quickstart (iOS)"

// In Checkout group, after "iap/preloading":
+ "iap/checkout-errors"
+ "iap/promotions"

// If Android option (b) chosen:
+ "iap/payment-sheet-android"
```

Nav group structure and ordering otherwise unchanged.

---

## Framework Convention Reference

These rules apply to Phase 1 and become the standard for Phases 2–3.

### When to use `<Tabs>`
Use `<Tabs>` (with exactly the labels `SwiftUI` and `UIKit`) whenever the Swift code differs by framework. If the code is identical — single block, no tabs.

### When to use `<Note>`
Use `<Note>` inline, adjacent to the relevant code block, when there is a **non-code** behavioral or architectural difference UIKit users need to know about. Do not use `<Note>` as a rote UIKit disclaimer on every tab group — only where the difference is load-bearing.

Common triggers:
- Preloader requires a view hierarchy host in UIKit
- SwiftUI `.task` vs UIKit `viewDidAppear` for async calls (Phase 2)
- `UIWindowScene` timing differences (Phase 2)

### Tab labels
Always exactly `SwiftUI` and `UIKit`. Never "Option A / B", version suffixes, or platform names (iOS is assumed for both).

### Prose style
Procedure steps are written from the SwiftUI perspective. UIKit readers follow the same steps but use the UIKit tabs. Don't write two copies of prose — write once, fork only at the code.

### Platform callouts vs framework callouts
`<Tip>` pointing Android/Flutter users elsewhere = platform signal, stays. `<Note>` for UIKit = framework signal, new. These are distinct — don't conflate.

---

## Open Questions (Resolved During Implementation)

1. **Android content in `payment-sheet.mdx`** — diff against `quickstart-android.mdx` and choose option (a) delete or (b) move. Implementer decides and notes in commit.
2. **`iap/installation.mdx` app-entry snippet** — check whether file actually shows `.zeroSettleHandler()` in context. If not, no tabs needed and the file may be untouched.

---

## Out of Scope for Phase 1 (Backlog for Phases 2–3)

- UIKit `UIHostingController` story for rendered tip/migration/offer views — these ship SwiftUI `View`s
- SwiftUI `.task` vs UIKit `viewDidAppear` lifecycle pattern (shows up in many Tier 2 docs)
- Standalone "UIKit Integration Guide" landing page (evaluate after Phase 1 ships)
- Android deeplink tab in `universal-links.mdx`
- `iap/subscription-state.mdx`, `iap/user-identity.mdx`, `iap/storekit-integration.mdx` cleanup
