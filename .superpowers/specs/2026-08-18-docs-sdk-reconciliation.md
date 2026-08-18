# ZeroSettle docs ↔ ZeroSettleKit source reconciliation

**Date:** 2026-08-18
**Scope:** `docs.zerosettle.io` (`~/dev/docs`, 76 `.mdx`) vs `ZeroSettleKit` 1.5.2
(`~/dev/zerosettle/ZeroSettleKit`, ~22k lines Swift) vs the AI integration prompt
(`~/dev/zerosettle-site/frontend/src/utils/generateIntegrationPrompt.ts` and
`~/dev/zerosettle-site/integration-prompt.md`).
**Status:** findings only — no doc or SDK edits applied.

Read as a developer would: could I integrate from these docs alone, and would the
code I copied compile and do what the page says?

---

## P0 — the iOS samples do not compile

### 1. `Product` is not a ZeroSettleKit type

Every Swift sample refers to the SDK's product model as `Product`. The type is
`ZSProduct`, and there is **no `typealias Product`** anywhere in
`Sources/ZeroSettleKit` (verified: `grep -rn "typealias Product" Sources/` →
no matches; `Exports.swift`'s alias list does not include it).

The modifier signature is `checkoutSheet(item: Binding<ZSProduct?>, …)`
(`UI/CheckoutSheet+API.swift:563`) and the cached catalog is
`public private(set) var products: [ZSProduct]` (`ZeroSettle.swift:447`).

So `@State private var selectedProduct: Product?` either fails to resolve, or —
worse, in any file that also does `import StoreKit` — silently binds to
`StoreKit.Product` and fails at the `.checkoutSheet(item:)` call with a
confusing type error.

Affected: `iap/quickstart.mdx` (4 sites: 183, 184, 208, 210, 317, 377, 379),
`iap/payment-sheet.mdx` (23, 24, 48, 50), `iap/storekit-integration.mdx`
(88, 89), `iap/revenuecat-integration.mdx` (168), `iap/best-practices.mdx` (1189).

The inconsistency is visible *within* single pages: `storekit-integration.mdx`
uses `Product` in the hybrid-flow sample and then explains the
`ZSProduct` model three paragraphs later.

**Fix:** `Product` → `ZSProduct` throughout the Swift tabs. (Kotlin samples using
`Product` are correct for that SDK — leave them.)

### 2. The stated toolchain floor is wrong

`iap/installation.mdx` and `iap/quickstart.mdx` both say **Swift 5.9+ / Xcode
15.0+**. `Package.swift` is `// swift-tools-version: 6.1` and declares a `traits:`
block — that needs **Swift 6.1 / Xcode 16.3+**. A developer on the documented
floor cannot resolve the package at all.

### 3. "No third-party dependencies" is not true

`iap/installation.mdx`: *"Requires iOS 17.0+ … No third-party dependencies."*
`Package.swift` declares `.package(url: "https://github.com/stripe/stripe-ios.git",
"23.0.0"..<"25.0.0")`. It is gated to the `NativePay` trait at the *target* level,
but the dependency itself is unconditional, so SwiftPM resolves and clones
stripe-ios for every adopter. Say so, and say the trait is what decides whether
it links.

Related: the `NativePay` trait, `appleMerchantId`, and `applePaySetupBehavior`
have no documentation page at all (see P2).

---

## P1 — documented behavior the SDK does not implement

### 4. The 5-minute entitlement cache does not exist

`iap/subscription-state.mdx` — *"The SDK will update the cache if it's older than
5 minutes when you call `restoreEntitlements()`…"* and *"It's safe to call
`restoreEntitlements()` frequently … the network request is lightweight."*

`_restoreEntitlementsImpl` (`ZeroSettle.swift:2232`) reads local StoreKit
entitlements and then **unconditionally** calls `backend.getEntitlements(userId:)`.
There is no TTL, timestamp, or staleness check anywhere in `Sources/ZeroSettleKit`
(`grep -rn "TTL\|ttl\|maxAge\|cacheAge\|lastFetch"` → nothing).

Either implement the TTL or delete the paragraph. As written it invites adopters
to call it on every premium-content view.

### 5. `Identity.anonymous()` / `Identity.deferred()` are not callable

`iap/user-identity.mdx` writes them with parentheses in the mode table, section
headings, prose, and `iap/subscription-state.mdx`. In Swift they are enum cases:
`.anonymous`, `.deferred` (`ZeroSettle.swift:291`). The Flutter form *is*
`Identity.anonymous()` — the docs merged the two. The Swift code fences are
right; the prose around them is wrong, which is worse, because prose is what
gets copied into an LLM prompt.

### 6. `Identity.anonymous`'s doc comment promises an API that was never shipped

`ZeroSettle.swift:281`: *"they can be reconciled into a real user account later
via `completeIdentification(.user(...))` (added in PR C2 — backend reconciliation
endpoint)."*

`grep -rn "completeIdentification" Sources/` matches only that comment. There is
no such method. This is load-bearing: it is the sentence that makes `.anonymous`
look like a safe default, and without it **anonymous → identified is a one-way
door** (the SDK docs elsewhere say the anonymous UUID is simply lost). Either
ship the method or strike the sentence, and state plainly in
`iap/user-identity.mdx` that promoting an anonymous install to a real user id
does not carry its purchase history.

### 7. `Entitlement` is documented at ~⅓ of its real surface

`iap/user-identity.mdx` lists six properties: `id`, `productId`, `source`,
`isActive`, `expiresAt`, `purchasedAt`.

`Models/Entitlement.swift` is public for all of: `status` (`Status` — `.active`,
`.paused`, `.expired`, `.revoked`, `.cancelled`, `.gracePeriod`, `.pastDue`,
`.unknown(String)`), `isTrial`, `trialEndsAt`, `trialDaysRemaining`, `willRenew`,
`isCancelled`, `cancelledAt`, `isPaused`, `pausedAt`, `pauseResumesAt`,
`storekitOriginalTransactionId`, `originalPurchaseDate`.

This is the single largest documentation gap by adopter value. Everything a
subscription app actually renders — "trial ends in 3 days", "cancels at period
end", "payment failed, in grace period" — is present in the SDK and absent from
the docs. It is also the exact data an analytics-only adopter is integrating
*for*.

Also in that table: `source` is typed `EntitlementSource` and lists a
`.playStore` case. On iOS the type is `Entitlement.Source` and it has exactly two
cases, `.storeKit` and `.webCheckout` (`Models/Entitlement.swift:24-26`).

---

## P2 — public API with zero documentation

Method: enumerated the public surface out of `Sources/ZeroSettleKit`, then
grepped each symbol across `iap/*.mdx` + `index.mdx` +
`api-reference/introduction.mdx`.

**Functions:** `trackEvent(_:productId:screenName:metadata:)`, `reportOfferViewed(…)`,
`recommendedAppAccountToken()`, `newConsumableEntitlements(excluding:)`,
`setCustomer(name:email:)`, `dismissDemoModeAlert()`.

**Types:** `FunnelEventType`, `CheckoutType`, `CheckoutMode`, `CheckoutRoute`,
`CheckoutPresentation`, `ApplePayAvailability`, `ApplePaySetupBehavior`,
`RemoteConfig`, `ResolvedOffer`, `StoreKitSubscriptionStatus`, `ZSTrialFacts`,
`ZSDemoMode`, `OfferImpressionInteraction`.

**Properties:** `checkoutType`, `isApplePayOnly`, `applePayAvailability`,
`applePaySetupBehavior`, `effectiveJurisdiction`, `sessionId`, `sdkVersion`,
`statePublisher`, `debugStateOverride`, `onCheckoutFailure`, `isSandbox`,
`monthlyEquivalent`, `trialEndsAt`, `trialDaysRemaining`.

Three of these deserve their own pages rather than a line in a table:

- **`ZeroSettle.trackEvent(_:productId:screenName:metadata:)`** with `FunnelEventType`
  (`.paywallViewed`, `.checkoutStarted`, `.checkoutAbandoned`). It is
  `nonisolated static`, fire-and-forget, and it is the only funnel-analytics
  entry point in the SDK. The endpoint (`POST /iap/events`) is in the OpenAPI
  spec; the Swift call is documented nowhere. An adopter reading the docs cannot
  discover that paywall-funnel tracking exists.
- **`ZSTrialFacts`** (`ZSProduct.trial`) — trial `mode` (`.free` / `.paid` /
  `.authHold`), `upfrontAmountCents`, `holdAmountCents`, `validatesCard`. Paid
  trials and auth-hold trials are a *product feature* with no docs page.
- **Apple Pay**: `ApplePayAvailability`, `ApplePaySetupBehavior`,
  `presentApplePaySetup()`, `isApplePayOnly`, the `appleMerchantId` config field,
  and the `NativePay` trait. There are two design specs for this in
  `ZeroSettleKit/docs/superpowers/` and not one public page.

---

## P3 — deprecated APIs still actively taught

`grep -rn "@available(\*, deprecated"` finds 41 deprecations in ZeroSettleKit,
almost all of them the pre-`identify()` explicit-`userId` overloads slated for
removal in 2.0.

### 8. The AI integration prompt teaches `bootstrap(userId:)` — the worst instance

`~/dev/zerosettle-site/frontend/src/utils/generateIntegrationPrompt.ts` is what
the dashboard hands new adopters (rendered by
`frontend/src/components/onboarding/steps/StepConfigureTest.tsx`). Its Swift
lane is built around `bootstrap()`:

- line 100: `### 3. Fetch products and bootstrap`
- line 114: *"Call `bootstrap()` once after authentication — it fetches products
  AND restores entitlements in one call"*
- lines 163, 175: `try await ZeroSettle.shared.bootstrap(userId: …)`
- line 199: `.checkoutSheet(item: $selectedProduct, userId: userId)`

`bootstrap(userId:name:email:)` is `@available(*, deprecated, message: "Use
identify(.user(id:…)) instead. bootstrap() will be removed in ZeroSettleKit
2.0.")` (`ZeroSettle.swift:1350`). Kotlin (252), Flutter (282) and React Native
(307) lanes teach the same deprecated shape; RN is genuinely still on it, the
other three are not.

Net effect: every adopter who uses the dashboard's AI prompt starts on an API
with a removal date, and gets a deprecation warning on their first build.

### 9. `~/dev/zerosettle-site/integration-prompt.md` is worse and appears unused

The standalone 1,468-line prompt teaches `bootstrap(userId)` **and**
`ZSPaymentSheet` (deprecated alias for `CheckoutSheet`, `Exports.swift:139`) as
core concepts in its `<context>` block. It does not appear to feed the `.ts`
generator — they have drifted independently. Decide which one is canonical and
delete or regenerate the other.

### 10. Deprecated symbols recommended in the docs proper

- `MigrationTipView` / `MigrationManager` are class-level deprecated in favour of
  `OfferTipView` / `ZSOfferManager` (`UI/MigrationTipView.swift:11`), yet appear
  as live guidance in `iap/migration-manager.mdx`, `iap/campaigns.mdx`,
  `iap/dashboard.mdx`, `iap/promotions.mdx`, `iap/switch-and-save.mdx`.
- `markCheckoutSucceeded()` and `ZSOfferManager.present()` are deprecated —
  bookkeeping has been automatic since 1.3.5 — but `iap/offer-manager.mdx` and
  `iap/migration-manager.mdx` still instruct adopters to call them.
- `freeTrialDays:` is deprecated *and ignored* ("Free trials are configured
  server-side"), still present in `iap/promotions.mdx`.

(`iap/migration-guide.mdx` mentioning old names is correct — that's its job.)

---

## P4 — patterns worth simplifying

### 11. `@ObservedObject var iap = ZeroSettle.shared`

Used in `iap/quickstart.mdx` and `iap/subscription-state.mdx`. It works —
`ZeroSettle` is both `@Observable` and `ObservableObject`, and the SDK manually
fires `objectWillChange.send()` in the three mutation points that need it
(`ZeroSettle.swift:1072, 1082, 1091`) precisely because the `@Observable` macro
does not publish to `objectWillChange`.

But it teaches the wrong thing twice over: `@ObservedObject var x = <object>`
(rather than `@StateObject`) is the classic SwiftUI ownership bug shape, and on
iOS 17+ none of it is needed — reading `ZeroSettle.shared.entitlements` directly
in `body` is tracked automatically. Recommend showing the plain read, and keep
the `ObservableObject` conformance documented only as a back-compat note.

### 12. `configure()` inside `.task { }`

`iap/quickstart.mdx` says *"`configure()` is synchronous — call it before
anything else"* and then puts it in `.task {}`, which runs after the first
render. Given `setActiveUserId` is what starts the `Transaction.updates`
listener (`ZeroSettle.swift:994`, the documented fix for "the canonical 1.3.x
StoreKit-attribution bug class"), the sample should configure in `init()` or at
the App level and only `identify()` in `.task`.

### 13. Async work inside non-async completion closures

`iap/revenuecat-integration.mdx:175` has `try? await Purchases.shared
.customerInfo()` inside `.checkoutSheet(item:) { result in … }` — the closure is
`@escaping (Result<CheckoutTransaction, Error>) -> Void`, not async. Needs a
`Task { }`.

### 14. Four broken cross-links

`/sdk/android/offers`, `/sdk/android/sync-queue`, `/sdk/android/pending-actions`,
`/sdk/android/acknowledgement-model` are linked from `iap/switch-and-save.mdx`
and neighbours. No `/sdk/**` route exists — `docs.json`'s navigation and the
`.mdx` file set match each other exactly, and neither contains one.

---

## P5 — the missing use case: analytics-only / observer mode

There is no documented configuration for *"use ZeroSettle to instrument
StoreKit purchases, trials and entitlements; don't use it for checkout or
gating"* — which is how a large share of prospects will trial the product,
because it is exactly how RevenueCat is commonly deployed.

The docs offer two shapes and neither is it:

1. `iap/storekit-integration.mdx` — StoreKit as a *fallback* alongside web
   checkout, `syncStoreKitTransactions: true`.
2. `iap/revenuecat-integration.mdx` + `iap/user-identity.mdx` — RevenueCat
   present, therefore `syncStoreKitTransactions: **false**`, and ZeroSettle only
   sees its own web checkouts.

Reading (2), an adopter who wants StoreKit analytics from ZeroSettle is told to
turn off the very listener that produces them. The "conflicts" the warning cites
are unspecified; from source the concrete overlap is only that both SDKs call
`transaction.finish()` and both iterate `Transaction.updates` (a broadcast
sequence — multiple listeners are supported).

**It does work.** Verified from source:

- `configure(.init(publishableKey:))` + `identify(.anonymous)` is sufficient; no
  associated domains, no universal-link handler, no dashboard catalog required.
- `identify()` calls `setActiveUserId` → `StoreKitManager.startListening` →
  `syncCurrentTransactions(userId:)` **before** it fetches products, so the
  StoreKit sync happens even when the later `fetchProducts` /
  `restoreEntitlements` legs throw (`_runBootstrap`, `ZeroSettle.swift:1451`).
- The backend accepts a product it has never seen: `create_storekit_transaction`
  does `product_name = prod.name if prod else product_id`
  (`transaction_service.py:334`), and price comes from the Apple-signed JWS
  (`payload["price"]` / `payload["currency"]`, `iap_views.py:12371`) rather than
  the catalog — so revenue is real, not zero, without any Stripe mapping.
- Trials are detected server-side from the JWS (`detect_apple_trial`), so
  `isTrial` / `trialEndsAt` populate with no app-side work.

**Two real gaps for that adopter:**

- **No "record a purchase I completed myself" API.** `Transaction.updates` does
  not redeliver a transaction returned directly from `Product.purchase()`, so an
  app that runs its own StoreKit purchase flow — which is every observer-mode
  adopter — is invisible to the listener until the *next* `identify()` picks it
  up via `syncCurrentTransactions`. RevenueCat solves this with
  `recordPurchase(_:)`. ZeroSettleKit needs a public equivalent, e.g.
  `ZeroSettle.shared.recordStoreKitPurchase(_ transaction: StoreKit.Transaction)`.
  Today the only workaround is to re-`identify()` after a purchase, which re-runs
  the whole bootstrap.
- **`recommendedAppAccountToken()` is undocumented** but effectively mandatory
  for anyone calling `Product.purchase()` themselves. Without stamping it into
  `options:`, `StoreKitManager.handleVerifiedTransaction` logs a loud
  `appAccountToken mismatch` error and cross-account ownership transfer silently
  stops working for that transaction (`StoreKitManager.swift:515-541`). The
  SDK's own error message names the API; the docs never do.

Recommended: a short `iap/analytics-only.mdx` page — configure, `identify`,
stamp `recommendedAppAccountToken()`, what lands in the dashboard, and an
explicit statement that entitlement authority stays with the app.

---

# Part 2 — second pass (full read of the remaining iOS pages)

## P0 (continued)

### 15. `ZeroSettleError` is not `Equatable`, but two samples compare it with `==`

`iap/best-practices.mdx:429-431` and `610-611`:

```swift
if let zsError = error as? ZeroSettleError, zsError == .cancelled { return }
```

`public enum ZeroSettleError: Error, LocalizedError` (`ZeroSettle.swift:82`) — no
`Equatable`, and several cases carry non-`Equatable` payloads (`underlyingError:
Error`), so it cannot be synthesised. Does not compile.

The SDK ships `ZeroSettleError.isCancellation(_:)` precisely for this, and
`iap/checkout-errors.mdx` uses it correctly. Fix best-practices to match, or add
`Equatable` to the enum (see Framework changes below).

### 16. `.networkError` does not exist

`iap/best-practices.mdx` lists `.networkError` in its iOS error table and matches
`case .networkError(let underlying):` at line 1120. There is no such case. Network
and server failures surface as `.apiError(APIErrorDetail)`.

That whole table is a stale duplicate of `iap/checkout-errors.mdx`, which is
**accurate and complete** — it matches the 19 real cases one-for-one. Delete the
duplicate and link to it. Duplicated reference tables are how this drift happened.

## P1 (continued)

### 17. The PaymentIntent cache TTL is 30 minutes, not 5

Stated as 5 minutes in five places — `iap/preloading.mdx` (Layer 1 step 3,
Cleanup, and the Defaults Summary table) and `iap/best-practices.mdx` (lines 518
and 1454). `CheckoutResponseCache.swift:32`: `fileprivate let ttl: TimeInterval =
1800 // 30 minutes`, aligned to the backend's
`Transaction.checkout_config_expires_at`.

### 18. `appAccountToken` is not gated on the userId being a UUID

`iap/entitlement-ownership.mdx`: *"The SDK sets this token automatically (since
ZeroSettleKit 1.1.5 / Android 0.15.0) when your `userId` is a valid UUID."*

`StoreKitManager.swift:216` derives it for **any** non-empty userId via
`AppAccountToken.derive(userId:bundleId:)` — a UUIDv5 over the ZeroSettle root
namespace. The source comment at line 210 says so explicitly: *"for non-UUID
`userId` formats (Firebase, Privy, Auth0, ...)"*.

As written the doc pushes adopters to reshape their user IDs for no reason.

More importantly, the same page's claim that this "happens automatically" is only
true on the `purchaseViaStoreKit()` path. An app calling `Product.purchase()`
itself stamps nothing, and the backend's `apple_verified_current_owner` check
then can't confirm ownership. That caveat belongs on this page, next to
`recommendedAppAccountToken()`.

### 19. External-purchase token minting is documented as unshipped, and it shipped

`iap/external-purchase-reporting.mdx` §7: *"Starting with the next ZeroSettleKit
release, the SDK handles the device side…"* — `ExternalPurchaseTokenProvider` is
in 1.5.2 and wired at `Backend.swift:298`.

### 20. `iap/overview.mdx` contradicts `iap/installation.mdx` on React Native

Overview: *"supports Swift, Kotlin, and Flutter, with React Native coming soon"*,
and its Platform Support table lists React Native as "Coming soon". Installation
documents an installable npm package (`react-native-zerosettle@^1.1.0`), and
`releases/react-native.mdx` exists. Overview also repeats the wrong Swift 5.9 /
Xcode 15.0 floor (see finding 2).

### 21. The debug-logging filter in `iap/troubleshooting.mdx` is fiction

It tells you to filter the console for `[ZeroSettle]`, `[ZSCheckout]`,
`[ZSMigrateTipView Business Logic]`, `[ZSStoreKit]`.

`ZSLogger` uses `OSLog(subsystem: "com.zerosettle.kit", category:)` with
categories `General`, `Checkout`, `CancelFlow`, `Entitlements`, `DeepLinks`,
`Network`, `Migration`. None of those bracket prefixes is a thing.

Worse, the page omits the fact that actually matters: `ZSLogger.log` uses
`%{public}@` **only in DEBUG** — Release builds log `%{private}@`, so a
TestFlight or App Store build shows `<private>` in Console.app and in any
sysdiagnose. Anyone debugging a production integration needs to know that before
they go looking.

## P4 (continued)

### 22. `troubleshooting.mdx` puts `isWebCheckoutEnabled` on the wrong object

*"Check `isWebCheckoutEnabled` on the product"* — it is
`ZeroSettle.shared.isWebCheckoutEnabled` (`ZeroSettle.swift:593`), not a
`ZSProduct` member. The same paragraph tells you to inspect `jurisdictionConfig`;
no such public property exists (the type `JurisdictionCheckoutConfig` does).

### 23. `fatalError` in a published payment sample

`iap/best-practices.mdx:1117`: `case .notConfigured: fatalError("SDK not
configured")`. Don't ship a crash as the recommended handling of a misconfigured
payment SDK.

### 24. Identity guidance omits the best iOS answer

`iap/best-practices.mdx` § *Use Stable Identifiers* offers backend UUID or
Firebase UID as "good", and `identifierForVendor` / email as "bad". It never
mentions **Sign in with Apple's `ASAuthorizationAppleIDCredential.user`** — a
stable, opaque, non-guessable, team-scoped identifier that survives reinstall,
is identical across the user's devices, and is scoped to the same Apple ID that
owns the StoreKit purchases. For an iOS app with no backend it is the single best
available answer, and it is what this SDK's own docs should lead with on iOS.

## Framework changes (ZeroSettleKit)

### F1. `objectWillChange` fires on three mutations, not "every mutation"

`ZeroSettle.swift:1063-1071` claims: *"We do both on every mutation to guarantee
downstream re-renders regardless of which observation style the app chose."*

`objectWillChange.send()` appears at exactly four sites — `logout()` (858),
`updateEntitlements` (1072), `addPendingClaim` (1082), `removePendingClaim`
(1091). It does **not** fire for:

| Property | Mutated at | Consequence for an `@ObservedObject` reader |
|---|---|---|
| `products` | 1647 | Catalog loads, list never re-renders |
| `pendingCheckout` | ~14 sites from 1747 | Spinner never appears or never clears |
| `isConfigured` | 1235 | Gating on it never updates |
| `currentOffer` | 440 | Offer banner never appears |
| `entitlements` | 1418 (bootstrap reset) | Stale entitlements survive a user switch |

The two docs samples that use `@ObservedObject` are precisely the broken ones:
the quickstart's Complete Example (`ForEach(iap.products)`) and best-practices'
Show Loading State (`iap.pendingCheckout`). Under `@Observable` tracking both
work; under the `ObservableObject` conformance the docs teach, neither does.

Recommended: route every observable mutation through a `mutating { }` helper that
sends first, in 1.x. Switch the docs to plain `@Observable` reads now, and drop
the `ObservableObject` conformance in 2.0 — the iOS 17 floor makes it dead weight.

### F2. `recordStoreKitPurchase(_:)` — the observer-mode gap

See P5 in Part 1. Needed by any adopter who runs their own `Product.purchase()`.

### F3. `ZeroSettleError: Equatable`

Adding it makes `== .cancelled` work as adopters keep trying to write it
(twice in our own docs). Cases carrying `any Error` need a hand-written `==` that
compares case identity. Cheap, and it removes a whole class of adopter confusion.

### F4. `completeIdentification(.user(...))` — ship it or strike the comment

`Identity.anonymous`'s doc comment sells a reconciliation path that does not
exist. Until it does, anonymous → identified loses the install's purchase history.

### F5. `Identity.anonymous()` / `.deferred()` in prose

Not a framework change, but if the Flutter/Swift shapes keep colliding in the
docs, consider adding `public static func anonymous() -> Identity` so both spellings
compile. Probably not worth it — fixing the prose is cheaper.

### F6. ZeroSettleKit ships no privacy manifest

`find Sources -name PrivacyInfo.xcprivacy` → nothing, while the SDK uses
`UserDefaults` in six files across 27 sites (the anonymous session UUID, the
StoreKit sync queue, offer dismissal state, Apple Pay availability caching).

`UserDefaults` is on Apple's required-reason API list
(`NSPrivacyAccessedAPICategoryUserDefaults`, reason `CA92.1` for an SDK reading
and writing only its own app-scoped defaults). A third-party SDK is expected to
declare its own; without one, every adopter either absorbs the declaration into
their app's manifest — which they cannot do accurately, because they cannot see
inside the SDK — or ships without it and collects ITMS-91053 "Missing API
declaration" on upload.

This is a payments SDK, which is the category Apple looks at hardest. Shipping
`Sources/ZeroSettleKit/PrivacyInfo.xcprivacy` (declaring the UserDefaults
category, and `NSPrivacyTracking` false, and the data types the backend
receives) removes a recurring adopter surprise for a few lines of plist.

Worth checking the same for the Android, Flutter, and React Native packages —
the Flutter and React Native ones vendor ZeroSettleKit and inherit the gap.

Separately, and not ZeroSettle's problem: **Pawprints itself ships no
`PrivacyInfo.xcprivacy`** either, and it uses `UserDefaults` heavily. That
pre-dates this integration, but it is on the same App Store submission path.
