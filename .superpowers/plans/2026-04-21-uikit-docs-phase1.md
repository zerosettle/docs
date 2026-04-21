# UIKit Docs Coverage — Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add full UIKit/SwiftUI parity to the five Tier-1 iOS docs (quickstart, payment-sheet, preloading, universal-links, installation), extract two new articles from the bloated payment-sheet page, and update nav.

**Architecture:** SwiftUI-first prose with `<Tabs title="SwiftUI">` / `<Tab title="UIKit">` on every code block where the frameworks differ. UIKit architectural caveats (preloader hosting, SceneDelegate lifecycle) get inline `<Note>` callouts adjacent to the code they explain. New articles (`checkout-errors`, `promotions`) are extracted from `payment-sheet.mdx` so the main article drops from 1260 lines to ~400. All work is on branch `uikit-docs-phase1` in worktree `.worktrees/uikit-docs-phase1/`.

**Tech Stack:** Mintlify MDX, `<Tabs>` / `<Tab>` / `<Note>` / `<CodeGroup>` / `<Steps>` components, Swift code samples.

---

## Convention Reference (read before starting any task)

### Tab syntax
```mdx
<Tabs>
  <Tab title="SwiftUI">
    ```swift
    // SwiftUI code here
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    // UIKit code here
    ```
  </Tab>
</Tabs>
```
Tab labels are always exactly `SwiftUI` and `UIKit`. No other variants.

### Note syntax for UIKit callouts
```mdx
<Note>
  **UIKit:** Explanation of the non-code difference UIKit users need to know.
</Note>
```
Only add a `<Note>` when there is a load-bearing behavioral or architectural difference — not as a rote disclaimer on every tab.

### When NOT to use tabs
If the Swift code is identical in both frameworks (e.g., `ZeroSettle.shared.configure(...)`, `ZeroSettle.shared.fetchProducts()`), use a single code block — no tabs.

---

## File Map

| File | Status | Task |
|------|--------|------|
| `.worktrees/uikit-docs-phase1/docs.json` | Modify | 1 |
| `.worktrees/uikit-docs-phase1/iap/quickstart.mdx` | Rebuild | 2 |
| `.worktrees/uikit-docs-phase1/iap/checkout-errors.mdx` | Create (new) | 3 |
| `.worktrees/uikit-docs-phase1/iap/promotions.mdx` | Create (new) | 4 |
| `.worktrees/uikit-docs-phase1/iap/payment-sheet.mdx` | Rebuild | 5 |
| `.worktrees/uikit-docs-phase1/iap/preloading.mdx` | Update | 6 |
| `.worktrees/uikit-docs-phase1/iap/universal-links.mdx` | Trim + tabs | 7 |

**Task order matters:** Complete Tasks 3 and 4 before Task 5 (payment-sheet rebuild references the new extracted articles). Task 6 can start after Task 5 (absorbs a section from payment-sheet). Tasks 1, 2, 7 are independent and can be done in any order.

---

## Task 1: `docs.json` nav updates + quickstart frontmatter rename

**Files:**
- Modify: `docs.json`
- Modify: `iap/quickstart.mdx` (frontmatter only)

- [ ] **Step 1: Rename quickstart title in frontmatter**

Open `iap/quickstart.mdx`. Change line 2:
```
title: 'Quickstart (Swift)'
```
to:
```
title: 'Quickstart (iOS)'
```
Also change line 3:
```
description: 'Get started with ZeroSettle In-App Purchase for Swift'
```
to:
```
description: 'Get started with ZeroSettle In-App Purchase on iOS'
```

- [ ] **Step 2: Add new articles to `docs.json` Checkout group**

In `docs.json`, find the Checkout group. It currently reads:
```json
{
  "group": "Checkout",
  "icon": "cart-shopping",
  "pages": [
    "iap/payment-sheet",
    "iap/preloading",
    "iap/switch-and-save",
    "iap/offer-tip-view",
    "iap/migration-manager",
    "iap/offer-manager",
    "iap/universal-links"
  ]
}
```

Replace with:
```json
{
  "group": "Checkout",
  "icon": "cart-shopping",
  "pages": [
    "iap/payment-sheet",
    "iap/preloading",
    "iap/checkout-errors",
    "iap/promotions",
    "iap/switch-and-save",
    "iap/offer-tip-view",
    "iap/migration-manager",
    "iap/offer-manager",
    "iap/universal-links"
  ]
}
```

- [ ] **Step 3: Verify**

```bash
# Check JSON is valid
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
python3 -c "import json; json.load(open('docs.json')); print('valid')"
```
Expected: `valid`

```bash
# Confirm new entries are present
grep -n "checkout-errors\|promotions" docs.json
```
Expected: two lines showing the new entries.

- [ ] **Step 4: Commit**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
git add docs.json iap/quickstart.mdx
git commit -m "docs(nav): rename quickstart to iOS, add checkout-errors and promotions to nav"
```

---

## Task 2: Rebuild `iap/quickstart.mdx`

**Files:**
- Modify: `iap/quickstart.mdx`

The current file is 285 lines titled "Quickstart (Swift)" with a SwiftUI-only narrative. Goal: rebuild as "Quickstart (iOS)" with `<Tabs>` on every framework-specific snippet. Target ≤350 lines. Prose stays SwiftUI-first; UIKit differences get `<Note>` callouts.

**Key SDK facts for UIKit:**
- `ZeroSettle.shared.configure(.init(publishableKey:))` — identical, call from `SceneDelegate.scene(_:willConnectTo:)`
- `ZeroSettle.shared.bootstrap(userId:)` — identical API, call via `Task { }` from SceneDelegate
- Universal link handler: three SceneDelegate methods (see Step 3 below)
- Checkout presentation: `CheckoutSheet.present(from: self, product: product, userId: currentUser.id) { result in ... }`
- Preloader host: `UIHostingController(rootView: Color.clear.zeroSettleHandler())` as 1pt child view

- [ ] **Step 1: Rewrite the file**

Replace the entire content of `iap/quickstart.mdx` with:

```mdx
---
title: 'Quickstart (iOS)'
description: 'Get started with ZeroSettle In-App Purchase on iOS'
icon: 'rocket'
---

## What You'll Build

In this quickstart, you'll integrate ZeroSettle into your iOS app and make your first test purchase. By the end, your app will:

- Fetch your product catalog from ZeroSettle
- Display products with web checkout pricing
- Present a payment sheet with Apple Pay and card support
- Verify the purchase via entitlements

**Time to complete:** ~10 minutes (assumes you've completed [Account Setup](/iap/account-setup) and [Installation](/iap/installation))

<Note>
  Requires **iOS 17.0+**, **Swift 5.9+**, and **Xcode 15.0+**. See [Installation](/iap/installation) if you haven't added the package yet.
</Note>

<Tip>
  Using Android? See the [Android Quickstart](/iap/quickstart-android). Using Flutter? See the [Flutter Quickstart](/iap/quickstart-flutter).
</Tip>

<Tip>
  **Want to see a complete integration?** The [JustOne habit tracker](/iap/sample-app) is a production app that demonstrates every pattern in this quickstart.
</Tip>

## Prerequisites

<Check>A ZeroSettle account with at least one product created ([Account Setup](/iap/account-setup))</Check>
<Check>ZeroSettleKit added to your Xcode project ([Installation](/iap/installation))</Check>
<Check>Your sandbox API key (`zs_pk_test_...`) from the dashboard</Check>
<Check>Xcode 15.0+ with an iOS 17.0+ target</Check>

<Steps>
<Step title="Configure the SDK">

Configure the SDK early in your app lifecycle. `configure()` is synchronous — call it before anything else. Then call `bootstrap()` to fetch products and restore entitlements.

<Tabs>
  <Tab title="SwiftUI">
    ```swift
    import ZeroSettleKit

    @main
    struct YourApp: App {
        var body: some Scene {
            WindowGroup {
                ContentView()
                    .task {
                        ZeroSettle.shared.configure(.init(
                            publishableKey: "zs_pk_test_..."
                        ))
                        try? await ZeroSettle.shared.bootstrap(userId: Auth.currentUser.id)
                    }
                    .zeroSettleHandler()
            }
        }
    }
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    import UIKit
    import ZeroSettleKit

    class SceneDelegate: UIResponder, UIWindowSceneDelegate {
        var window: UIWindow?

        func scene(
            _ scene: UIScene,
            willConnectTo session: UISceneSession,
            options connectionOptions: UIScene.ConnectionOptions
        ) {
            ZeroSettle.shared.configure(.init(
                publishableKey: "zs_pk_test_..."
            ))
            Task {
                try? await ZeroSettle.shared.bootstrap(userId: Auth.currentUser.id)
            }
        }
    }
    ```
  </Tab>
</Tabs>

`bootstrap()` automatically fetches your product catalog, restores entitlements, and optionally pre-creates PaymentIntents for all products.

**Configuration options**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `publishableKey` | Required | Your publishable key from the dashboard |
| `syncStoreKitTransactions` | `true` | Set to `false` if using RevenueCat |
| `preloadCheckout` | `false` | Pre-create PaymentIntents on bootstrap for instant checkout. See [Preloading](/iap/preloading) |
| `maxPreloadedWebViews` | `nil` | Cap WebView pre-rendering pool size. See [Preloading](/iap/preloading) |

</Step>
<Step title="Install the universal link handler">

ZeroSettle uses universal links to return checkout results to your app when the user pays in Safari or an in-app browser. You must install a handler so these callbacks reach the SDK.

<Tabs>
  <Tab title="SwiftUI">
    ```swift
    // Add .zeroSettleHandler() to your root view (already shown in Step 1).
    // It handles onContinueUserActivity and onOpenURL automatically.
    ContentView()
        .zeroSettleHandler()
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    // SceneDelegate.swift — add all three methods.
    class SceneDelegate: UIResponder, UIWindowSceneDelegate {

        // Cold-start: app launched directly from a universal link tap
        func scene(
            _ scene: UIScene,
            willConnectTo session: UISceneSession,
            options connectionOptions: UIScene.ConnectionOptions
        ) {
            // ... configure() and bootstrap() calls from Step 1 ...

            if let activity = connectionOptions.userActivities.first(where: {
                $0.activityType == NSUserActivityTypeBrowsingWeb
            }), let url = activity.webpageURL {
                ZeroSettle.shared.handleUniversalLink(url)
            }
        }

        // Warm-start: app already running when universal link is tapped
        func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
            guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
                  let url = userActivity.webpageURL else { return }
            // Returns false if not a ZeroSettle URL — chain to your own router if needed
            ZeroSettle.shared.handleUniversalLink(url)
        }

        // Safety net for edge cases
        func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
            for ctx in URLContexts {
                ZeroSettle.shared.handleUniversalLink(ctx.url)
            }
        }
    }
    ```
  </Tab>
</Tabs>

Make sure you've also added the Associated Domains entitlement to your app target:
- **Sandbox:** `applinks:api.zerosettle.io?mode=developer`
- **Production:** `applinks:api.zerosettle.io`

See [Universal Links](/iap/universal-links) for step-by-step setup.

</Step>
<Step title="Fetch and display products">

`fetchProducts()` returns your product catalog. Products are also cached in `ZeroSettle.shared.products` after `bootstrap()`.

```swift
// Framework-neutral — identical in SwiftUI and UIKit
let catalog = try await ZeroSettle.shared.fetchProducts(userId: currentUser.id)

for product in catalog.products {
    print("\(product.displayName): \(product.webPrice.formatted)")
}
```

</Step>
<Step title="Present the payment sheet">

Present a checkout sheet with Apple Pay and card support. The checkout mode (embedded webview, in-app browser, or Safari) is controlled by your dashboard configuration — the SDK chooses automatically.

<Tabs>
  <Tab title="SwiftUI">
    ```swift
    struct PaywallView: View {
        @State private var selectedProduct: Product?
        let product: Product

        var body: some View {
            Button("Subscribe — \(product.webPrice.formatted)") {
                selectedProduct = product
            }
            .checkoutSheet(
                item: $selectedProduct,
                userId: currentUser.id
            ) { result in
                switch result {
                case .success(let transaction):
                    print("Purchased: \(transaction.productId)")
                case .failure(let error):
                    if ZeroSettleError.isCancellation(error) { return }
                    print("Error: \(error)")
                }
            }
        }
    }
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    class PaywallViewController: UIViewController {
        let product: Product

        @objc func handleBuyTap() {
            CheckoutSheet.present(
                from: self,
                product: product,
                userId: currentUser.id
            ) { result in
                switch result {
                case .success(let transaction):
                    print("Purchased: \(transaction.productId)")
                case .failure(let error):
                    if ZeroSettleError.isCancellation(error) { return }
                    print("Error: \(error)")
                }
            }
        }
    }
    ```

    `CheckoutSheet.present(from:)` uses `UIWindowScene` lookup internally — it does not need to be called from a specific view controller hierarchy.
  </Tab>
</Tabs>

See [Checkout Sheet](/iap/payment-sheet) for the full guide, including custom headers, non-dismissible sheets, and preloading.

</Step>
<Step title="Listen for purchase events">

Use the delegate to respond to checkout events from anywhere in your app:

```swift
// Framework-neutral — identical in SwiftUI and UIKit
class PurchaseManager: ZeroSettleDelegate {
    init() {
        ZeroSettle.shared.delegate = self
    }

    func zeroSettleCheckoutDidComplete(transaction: CheckoutTransaction) {
        unlockProduct(transaction.productId)
    }

    func zeroSettleCheckoutDidCancel(productId: String) {
        // User cancelled — no action needed
    }

    func zeroSettleCheckoutDidFail(productId: String, error: Error) {
        showError(error)
    }

    func zeroSettleEntitlementsDidUpdate(_ entitlements: [Entitlement]) {
        updateUI(entitlements)
    }
}
```

</Step>
<Step title="Check entitlements">

Restore and check entitlements on app launch:

```swift
// Framework-neutral — identical in SwiftUI and UIKit
let entitlements = try await ZeroSettle.shared.restoreEntitlements(
    userId: currentUser.id
)

let hasPremium = entitlements.contains {
    $0.productId == "premium_monthly" && $0.isActive
}
```

</Step>
</Steps>

<Tip>
  **First purchase?** Use test card `4242 4242 4242 4242` with any future expiry and CVC. See [Testing & Sandbox](/iap/testing) for more test cards.
</Tip>

## Complete Example

<Tabs>
  <Tab title="SwiftUI">
    ```swift
    import SwiftUI
    import ZeroSettleKit

    @main
    struct MyApp: App {
        var body: some Scene {
            WindowGroup {
                ContentView()
                    .task {
                        ZeroSettle.shared.configure(.init(
                            publishableKey: "zs_pk_test_..."
                        ))
                        try? await ZeroSettle.shared.bootstrap(userId: Auth.currentUser.id)
                    }
                    .zeroSettleHandler()
            }
        }
    }

    struct ContentView: View {
        @ObservedObject var iap = ZeroSettle.shared
        @State private var selectedProduct: Product?

        var body: some View {
            VStack {
                ForEach(iap.products) { product in
                    Button("\(product.displayName) — \(product.webPrice.formatted)") {
                        selectedProduct = product
                    }
                }
            }
            .checkoutSheet(
                item: $selectedProduct,
                userId: Auth.currentUser.id
            ) { result in
                switch result {
                case .success(let transaction):
                    print("Purchased \(transaction.productId)")
                case .failure(let error):
                    if ZeroSettleError.isCancellation(error) { return }
                    print("Error: \(error)")
                }
            }
        }
    }
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    import UIKit
    import SwiftUI
    import ZeroSettleKit

    class SceneDelegate: UIResponder, UIWindowSceneDelegate {
        var window: UIWindow?

        func scene(
            _ scene: UIScene,
            willConnectTo session: UISceneSession,
            options connectionOptions: UIScene.ConnectionOptions
        ) {
            ZeroSettle.shared.configure(.init(publishableKey: "zs_pk_test_..."))
            Task { try? await ZeroSettle.shared.bootstrap(userId: Auth.currentUser.id) }

            if let activity = connectionOptions.userActivities.first(where: {
                $0.activityType == NSUserActivityTypeBrowsingWeb
            }), let url = activity.webpageURL {
                ZeroSettle.shared.handleUniversalLink(url)
            }
        }

        func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
            guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
                  let url = userActivity.webpageURL else { return }
            ZeroSettle.shared.handleUniversalLink(url)
        }

        func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
            for ctx in URLContexts { ZeroSettle.shared.handleUniversalLink(ctx.url) }
        }
    }

    class PaywallViewController: UIViewController {
        private let product: Product

        init(product: Product) {
            self.product = product
            super.init(nibName: nil, bundle: nil)
        }

        required init?(coder: NSCoder) { fatalError() }

        override func viewDidLoad() {
            super.viewDidLoad()

            let buyButton = UIButton(type: .system)
            buyButton.setTitle("Subscribe — \(product.webPrice.formatted)", for: .normal)
            buyButton.addTarget(self, action: #selector(handleBuyTap), for: .touchUpInside)
            // Add buyButton to view and set up constraints here
        }

        @objc private func handleBuyTap() {
            CheckoutSheet.present(
                from: self,
                product: product,
                userId: Auth.currentUser.id
            ) { result in
                switch result {
                case .success(let transaction):
                    print("Purchased \(transaction.productId)")
                case .failure(let error):
                    if ZeroSettleError.isCancellation(error) { return }
                    print("Error: \(error)")
                }
            }
        }
    }
    ```
  </Tab>
</Tabs>

## Next Steps

<CardGroup cols={2}>
  <Card title="Checkout Sheet" icon="credit-card" href="/iap/payment-sheet">
    Custom headers, preloading, non-dismissible mode
  </Card>
  <Card title="Universal Links" icon="link" href="/iap/universal-links">
    Full setup guide for Safari checkout callbacks
  </Card>
  <Card title="User Identity" icon="user" href="/iap/user-identity">
    How to identify users across sessions
  </Card>
  <Card title="Best Practices" icon="star" href="/iap/best-practices">
    Recommendations for a reliable integration
  </Card>
</CardGroup>
```

- [ ] **Step 2: Verify structure**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
# Check all <Tabs> have closing </Tabs>
python3 -c "
content = open('iap/quickstart.mdx').read()
opens = content.count('<Tabs>')
closes = content.count('</Tabs>')
print(f'<Tabs> opens: {opens}, closes: {closes}')
assert opens == closes, 'Mismatched Tabs!'
print('OK')
"
```
Expected: `<Tabs> opens: 4, closes: 4` and `OK`.

```bash
# Confirm both tab labels appear
grep -c '"SwiftUI"' iap/quickstart.mdx
grep -c '"UIKit"' iap/quickstart.mdx
```
Expected: both commands return `4` (one per Tabs group).

- [ ] **Step 3: Commit**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
git add iap/quickstart.mdx
git commit -m "docs(quickstart): rebuild as Quickstart (iOS) with SwiftUI/UIKit tabs"
```

---

## Task 3: Create `iap/checkout-errors.mdx`

**Files:**
- Create: `iap/checkout-errors.mdx`

This article is extracted from the "Error Handling" section of the current `payment-sheet.mdx` (lines ~984–1128). The key change: the current doc uses `PaymentSheetError` (an internal type) for the iOS section. We promote `ZeroSettleError` as the canonical catch type (as the inline SDK doc says: "Prefer catching `ZeroSettleError` instead for a unified error type").

- [ ] **Step 1: Create the file**

Create `iap/checkout-errors.mdx` with this content:

```mdx
---
title: 'Checkout Errors'
description: 'Error types returned by ZeroSettle checkout and how to handle them'
icon: 'triangle-exclamation'
---

When a checkout attempt fails, the result handler or `try/catch` block receives an error. On iOS, catch `ZeroSettleError` — it's the unified error type across all SDK surfaces.

## Catching Errors

<Tabs>
  <Tab title="SwiftUI">
    ```swift
    .checkoutSheet(item: $selectedProduct, userId: currentUser.id) { result in
        switch result {
        case .success(let transaction):
            unlockContent(transaction.productId)

        case .failure(let error):
            // Cancellations are not errors — user changed their mind
            if ZeroSettleError.isCancellation(error) { return }

            if let zsError = error as? ZeroSettleError {
                switch zsError {
                case .checkoutFailed(let reason):
                    showError("Payment failed: \(reason)")
                case .userIdRequired:
                    showError("Please sign in to purchase.")
                case .apiError(let detail):
                    showError("Could not load checkout: \(detail.message)")
                default:
                    showError(error.localizedDescription)
                }
            }
        }
    }
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    CheckoutSheet.present(from: self, product: product, userId: currentUser.id) { result in
        switch result {
        case .success(let transaction):
            self.unlockContent(transaction.productId)

        case .failure(let error):
            if ZeroSettleError.isCancellation(error) { return }

            if let zsError = error as? ZeroSettleError {
                switch zsError {
                case .checkoutFailed(let reason):
                    self.showError("Payment failed: \(reason)")
                case .userIdRequired:
                    self.showError("Please sign in to purchase.")
                case .apiError(let detail):
                    self.showError("Could not load checkout: \(detail.message)")
                default:
                    self.showError(error.localizedDescription)
                }
            }
        }
    }
    ```
  </Tab>
</Tabs>

`ZeroSettleError.isCancellation(_:)` returns `true` for user-initiated cancellations regardless of which layer threw — `ZeroSettleError.cancelled`, Swift's `CancellationError`, StoreKit's `.userCancelled`, or `PaymentSheetError.cancelled`. Always filter cancellations before showing an error UI.

## iOS Error Reference (`ZeroSettleError`)

| Error | Cause | Recommended Action |
|-------|-------|--------------------|
| `.cancelled` | User dismissed the sheet | No action — use `isCancellation(_:)` to filter |
| `.checkoutFailed(reason:)` | Payment processing failed | Show error to user; inspect `reason` for detail |
| `.userIdRequired(productId:)` | Subscription/non-consumable purchased without a `userId` | Require sign-in before presenting checkout |
| `.apiError(APIErrorDetail)` | Network or server error creating PaymentIntent | Show retry prompt |
| `.notConfigured` | `configure()` was not called | Call `configure()` before any SDK use |
| `.invalidPublishableKey` | Key format invalid | Check key from dashboard |
| `.productNotFound(String)` | Product ID not in catalog | Verify product ID matches dashboard |
| `.transactionVerificationFailed(String)` | Server verification failed after payment | Log and contact support; don't unlock content |
| `.webCheckoutDisabledForJurisdiction` | Dashboard config disables web checkout in this region | Gracefully fall back or hide the button |
| `.purchasePending` | Ask to Buy pending parental approval | Show "Purchase pending approval" UI |
| `.restoreEntitlementsFailed` | Partial restore failure | Check `partialEntitlements` for recovered items |

## Android Error Reference (`ZeroSettleError`)

| Error | Description |
|-------|-------------|
| `ZeroSettleError.Cancelled` | User dismissed — no action needed |
| `ZeroSettleError.NotConfigured` | Call `configure()` first |
| `ZeroSettleError.CheckoutFailed(reason)` | Inspect `reason` (`CheckoutFailure`) for detail |
| `ZeroSettleError.UserIdRequired(productId)` | Require sign-in before checkout |
| `ZeroSettleError.TransactionVerificationFailed(detail)` | Server verification failed |

### Android `CheckoutFailure` reasons

| Reason | Description |
|--------|-------------|
| `CheckoutFailure.NetworkUnavailable` | No network connection |
| `CheckoutFailure.StripeError(code, message)` | Card declined or Stripe error |
| `CheckoutFailure.ServerError(statusCode, message)` | Server returned non-2xx |
| `CheckoutFailure.ProductNotFound` | Product not in catalog |
| `CheckoutFailure.MerchantNotOnboarded` | Stripe onboarding incomplete |
| `CheckoutFailure.Other(message)` | Unclassified error |

<Note>
  `.cancelled` / `ZeroSettleError.Cancelled` is not a real error — it means the user changed their mind. Do not show an error message for cancellations.
</Note>
```

- [ ] **Step 2: Verify**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
python3 -c "
content = open('iap/checkout-errors.mdx').read()
opens = content.count('<Tabs>')
closes = content.count('</Tabs>')
print(f'Tabs opens: {opens}, closes: {closes}')
assert opens == closes
print('OK')
"
```
Expected: `Tabs opens: 1, closes: 1` and `OK`.

- [ ] **Step 3: Commit**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
git add iap/checkout-errors.mdx
git commit -m "docs(checkout-errors): extract error handling into dedicated article"
```

---

## Task 4: Create `iap/promotions.mdx`

**Files:**
- Create: `iap/promotions.mdx`

Extracted from the "Promotions" section of `payment-sheet.mdx` (lines ~1128–1251). Content is mostly multi-platform, so tabs are light. Add the `stripeCustomerId` parameter table which currently lives buried at the bottom of payment-sheet.

- [ ] **Step 1: Create the file**

Create `iap/promotions.mdx` with this content:

```mdx
---
title: 'Promotions & Free Trials'
description: 'Display promotional pricing and free trial eligibility in your paywall'
icon: 'tag'
---

If a product has an active promotion, the checkout sheet automatically shows the promotional price. Read promotion metadata from the `Product` object to update your paywall copy.

## Reading Promotion Data

```swift
// Framework-neutral — identical in SwiftUI and UIKit
if let promo = product.promotion {
    print("\(promo.displayName): \(promo.promotionalPrice.formatted)")
    // e.g. "Launch Sale: $4.99"
}
```

Promotion types:
- **Percent off** — e.g., 50% off the regular price
- **Fixed amount** — e.g., $5 off
- **Free trial** — e.g., 7 days free

## Free Trials

Free trial configuration is **server-side** — configure trial length per product in the [ZeroSettle dashboard](https://dashboard.zerosettle.io). The SDK reads trial data from your product catalog automatically.

Two fields on `Product` reflect trial status:

| Field | Type | Description |
|-------|------|-------------|
| `isTrialEligible` | `Bool` | Whether this user is eligible for a trial on this product (server-determined based on prior trial history) |
| `freeTrialDuration` | `String?` | Human-readable trial length (e.g., `"7 days"`) when configured |

Use these to update your paywall CTA copy:

```swift
// Framework-neutral — identical in SwiftUI and UIKit
if product.isTrialEligible, let duration = product.freeTrialDuration {
    buyButton.title = "Start \(duration) Free Trial"
} else {
    buyButton.title = "Subscribe — \(product.webPrice.formatted)"
}
```

No trial parameter is needed when presenting the checkout sheet — the server applies the trial automatically when the user is eligible.

<Note>
  As of ZeroSettleKit 1.1.5, the legacy `freeTrialDays:` parameter on `.checkoutSheet`, `CheckoutSheet.init`, `preload()`, `warmUp()`, and `warmUpAll()` is silently ignored. Configure trials in the dashboard instead.
</Note>

## Stripe Customer Attachment

To attach a checkout to an existing Stripe customer (for unified billing portal use cases), pass `stripeCustomerId`:

<Tabs>
  <Tab title="SwiftUI">
    ```swift
    .checkoutSheet(
        item: $selectedProduct,
        userId: currentUser.id,
        stripeCustomerId: "cus_abc123"
    ) { result in
        handleResult(result)
    }
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    CheckoutSheet.present(
        from: self,
        product: product,
        userId: currentUser.id,
        stripeCustomerId: "cus_abc123"
    ) { result in
        handleResult(result)
    }
    ```
  </Tab>
</Tabs>

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `stripeCustomerId` | `String?` | `nil` | Existing Stripe Customer ID (`cus_xxx`). Attaches the checkout to this customer for a consolidated [Stripe Billing Portal](https://docs.stripe.com/customer-management/integrate-customer-portal) view. |

## Testing Promotions

Create a test promotion in the ZeroSettle dashboard under your product settings. Promotions appear immediately in sandbox mode — no app release required.
```

- [ ] **Step 2: Verify**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
python3 -c "
content = open('iap/promotions.mdx').read()
opens = content.count('<Tabs>')
closes = content.count('</Tabs>')
print(f'Tabs opens: {opens}, closes: {closes}')
assert opens == closes
print('OK')
"
```
Expected: `Tabs opens: 1, closes: 1` and `OK`.

- [ ] **Step 3: Commit**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
git add iap/promotions.mdx
git commit -m "docs(promotions): extract promotions and free trials into dedicated article"
```

---

## Task 5: Rebuild `iap/payment-sheet.mdx`

**Files:**
- Modify: `iap/payment-sheet.mdx`

Current: 1260 lines, SwiftUI + UIKit + Android + Presentation API + Performance & Caching + Error Handling + Promotions.
Target: ~400 lines, iOS-only, SwiftUI/UIKit tabs, cross-links to extracted articles.

**What to keep:**
- Intro + screenshot (lines 1–14)
- SwiftUI presentation section (rebuild as tabs merging lines 15–345 SwiftUI + 345–435 UIKit)
- Presentation API section (12 lines — fold into Presentation section as an explanatory paragraph)
- "How It Works" (13 lines — keep as-is)
- Custom header (rebuild as tabs)
- Non-dismissible mode (rebuild as tabs)
- Direct view usage (rebuild as tabs, iOS-only)
- Next Steps (update to link checkout-errors and promotions)

**What to extract/remove:**
- Kotlin/Android and multi-platform `<CodeGroup>` tabs throughout — remove. Android devs use [Android Quickstart](/iap/quickstart-android).
- "Performance & Caching" section → replaced by link to `/iap/preloading`
- "Error Handling" section → replaced by link to `/iap/checkout-errors`
- "Promotions" section → replaced by link to `/iap/promotions`
- "Android" section (lines 752–971) → remove; unique content (if any) belongs in a future `payment-sheet-android.mdx`

- [ ] **Step 1: Rewrite the file**

Replace the entire contents of `iap/payment-sheet.mdx` with:

```mdx
---
title: 'Checkout Sheet'
description: 'Embedded checkout UI with Apple Pay and card support'
icon: 'credit-card'
---

`CheckoutSheet` is an embedded checkout experience that presents a native-feeling bottom sheet inside your app. Users can pay with Apple Pay or enter a card — without ever leaving your app. It automatically reads your dashboard configuration to choose the correct checkout mode (webview, in-app browser, or external Safari).

This is the recommended way to accept payments on iOS with ZeroSettle.

<Frame caption="ZeroSettle payment sheet with Apple Pay and card entry">
  <img src="/images/payment-sheet-ios.png" alt="Payment sheet showing Apple Pay button, product name, and card entry option" style={{ maxWidth: '350px' }} />
</Frame>

## Basic Usage

<Tabs>
  <Tab title="SwiftUI">
    Use the `.checkoutSheet()` view modifier on any view. Pass an optional `Product?` binding — the sheet presents when the binding is non-nil and resets to `nil` on dismiss. This mirrors SwiftUI's `.sheet(item:)` pattern.

    ```swift
    struct PaywallView: View {
        @State private var selectedProduct: Product?
        let product: Product

        var body: some View {
            Button("Subscribe — \(product.webPrice.formatted)") {
                selectedProduct = product
            }
            .checkoutSheet(item: $selectedProduct, userId: currentUser.id) { result in
                switch result {
                case .success(let transaction):
                    unlockContent(transaction.productId)
                case .failure(let error):
                    if ZeroSettleError.isCancellation(error) { return }
                    showError(error)
                }
            }
        }
    }
    ```

    The modifier works for both single-product paywalls (set `selectedProduct = product` on button tap) and multi-product stores (set it from a list selection).
  </Tab>
  <Tab title="UIKit">
    Call `CheckoutSheet.present(from:)` from any view controller. It uses `UIWindowScene` lookup internally and does not need to be called from a specific view controller hierarchy.

    ```swift
    class PaywallViewController: UIViewController {
        let product: Product

        @objc func handleBuyTap() {
            CheckoutSheet.present(
                from: self,
                product: product,
                userId: currentUser.id
            ) { result in
                switch result {
                case .success(let transaction):
                    self.unlockContent(transaction.productId)
                case .failure(let error):
                    if ZeroSettleError.isCancellation(error) { return }
                    self.showError(error)
                }
            }
        }
    }
    ```
  </Tab>
</Tabs>

## Custom Header

Add a custom header view above the payment sheet to show product details, branding, or promotional copy.

<Tabs>
  <Tab title="SwiftUI">
    ```swift
    .checkoutSheet(item: $selectedProduct, userId: currentUser.id, header: {
        VStack(spacing: 8) {
            Image("premium-icon")
                .resizable()
                .frame(width: 60, height: 60)
            Text(selectedProduct?.displayName ?? "")
                .font(.title2.bold())
            Text(selectedProduct?.productDescription ?? "")
                .font(.subheadline)
                .foregroundStyle(.secondary)
        }
        .padding()
    }) { result in
        handleResult(result)
    }
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    CheckoutSheet.present(
        from: self,
        product: product,
        userId: currentUser.id,
        header: {
            AnyView(
                VStack(spacing: 8) {
                    Image("premium-icon")
                        .resizable()
                        .frame(width: 60, height: 60)
                    Text(product.displayName)
                        .font(.title2.bold())
                }
                .padding()
            )
        }
    ) { result in
        handleResult(result)
    }
    ```
  </Tab>
</Tabs>

## Non-Dismissible Sheet

Pass `dismissible: false` to hide the close button and disable interactive dismiss.

<Tabs>
  <Tab title="SwiftUI">
    ```swift
    .checkoutSheet(item: $selectedProduct, userId: currentUser.id, dismissible: false) { result in
        handleResult(result)
    }
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    CheckoutSheet.present(
        from: self,
        product: product,
        userId: currentUser.id,
        dismissible: false
    ) { result in
        handleResult(result)
    }
    ```
  </Tab>
</Tabs>

## Direct View Usage (SwiftUI only)

Use `CheckoutSheet` as a standalone SwiftUI `View` inside a `.sheet()` — useful when you need full control over sheet presentation:

```swift
.sheet(item: $selectedProduct) { product in
    CheckoutSheet(
        product: product,
        userId: currentUser.id,
        header: { Text("Premium Access").font(.title.bold()).padding() }
    ) { result in
        selectedProduct = nil
        handleResult(result)
    }
}
```

<Note>
  **UIKit:** If you embed SwiftUI inside UIKit (e.g., with a `UIHostingController`), you can use the SwiftUI modifier path above. Otherwise, use `CheckoutSheet.present(from:)` as shown in "Basic Usage".
</Note>

## How It Works

The checkout sheet is backed by a `WKWebView` that loads a Stripe-hosted checkout page pre-configured for your product. The SDK:

1. Creates (or reuses a cached) Stripe `PaymentIntent` for the product
2. Loads the checkout URL in an off-screen `WKWebView` while you tap "Buy"
3. Presents the pre-warmed WebView as a bottom sheet the moment the user taps
4. Receives a JavaScript callback on payment completion
5. Verifies the transaction server-side and delivers the result to your handler

The checkout mode (embedded webview, in-app browser, or Safari) is determined by your dashboard configuration and the user's jurisdiction — the SDK selects it automatically.

## Preloading

Preloading eliminates checkout latency by pre-creating the PaymentIntent and pre-rendering the WebView before the user taps "Buy". When properly configured, checkout opens instantly with no spinner.

See [Preloading](/iap/preloading) for setup and configuration options.

## Error Handling

The result handler receives `Result<CheckoutTransaction, Error>`. Use `ZeroSettleError.isCancellation(_:)` to filter user-initiated cancellations before showing error UI.

See [Checkout Errors](/iap/checkout-errors) for the full error reference and handling patterns.

## Promotions & Free Trials

If a product has an active promotion, the sheet automatically displays the promotional price. Read `product.promotion` and `product.isTrialEligible` to update your paywall CTA copy.

See [Promotions & Free Trials](/iap/promotions) for details.

## Next Steps

<CardGroup cols={2}>
  <Card title="Preloading" icon="bolt" href="/iap/preloading">
    Eliminate checkout latency with pre-warmed WebViews
  </Card>
  <Card title="Checkout Errors" icon="triangle-exclamation" href="/iap/checkout-errors">
    Error reference and handling patterns
  </Card>
  <Card title="Universal Links" icon="link" href="/iap/universal-links">
    Required setup for Safari checkout callbacks
  </Card>
  <Card title="Promotions" icon="tag" href="/iap/promotions">
    Promotional pricing and free trial setup
  </Card>
</CardGroup>
```

- [ ] **Step 2: Verify structure and length**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
wc -l iap/payment-sheet.mdx
```
Expected: ≤420 lines.

```bash
python3 -c "
content = open('iap/payment-sheet.mdx').read()
opens = content.count('<Tabs>')
closes = content.count('</Tabs>')
print(f'Tabs opens: {opens}, closes: {closes}')
assert opens == closes
# Verify extracted sections are gone
assert 'Error Handling' not in content or 'Checkout Errors' in content
assert 'Performance & Caching' not in content
assert 'class AppCompatActivity' not in content, 'Android code still present'
print('OK')
"
```
Expected: counts match, `OK`.

- [ ] **Step 3: Commit**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
git add iap/payment-sheet.mdx
git commit -m "docs(payment-sheet): rebuild iOS-only with SwiftUI/UIKit tabs, extract error/promo/perf sections"
```

---

## Task 6: Update `iap/preloading.mdx`

**Files:**
- Modify: `iap/preloading.mdx`

Current: 196 lines covering PI caching and WebView pre-rendering. UIKit has one-liner mention. Goal: add the UIKit hosting pattern `<Note>` (the single most important UIKit callout in the whole docs), absorb the Performance & Memory table from the old payment-sheet, add UIKit tab to `CheckoutSheet.preload()` method, and add tabs to `warmUp` calls.

**What to add:**
1. After the "How It Works" section: a `<Note>` explaining UIKit apps need a `UIHostingController` host to get WebView pre-rendering working.
2. Tabs on `checkoutSheet(preload:)` (SwiftUI modifier) and `CheckoutSheet.preload()` (UIKit-accessible method).
3. The Performance & Memory table from old payment-sheet (absorbed here).

- [ ] **Step 1: Add UIKit hosting note after "How It Works → Layer 2" section**

Open `iap/preloading.mdx`. After the "Layer 2 — WebView Pre-Rendering" section (after the closing `</Note>` block around line 60), add:

```mdx
<Note>
  **UIKit:** WebView pre-rendering requires `WKWebView` instances to be in the app's view hierarchy. The `.zeroSettleHandler()` modifier handles this automatically in SwiftUI apps. In UIKit apps, add a 1pt invisible child view controller to your root view controller so WebKit can render off-screen:

  ```swift
  // In your root UIViewController.viewDidLoad()
  import SwiftUI
  import ZeroSettleKit

  let host = UIHostingController(rootView: Color.clear.zeroSettleHandler())
  addChild(host)
  host.view.frame = CGRect(x: 0, y: 0, width: 1, height: 1)
  host.view.isUserInteractionEnabled = false
  view.addSubview(host.view)
  host.didMove(toParent: self)
  ```

  Without this, the SDK still caches PaymentIntents (Layer 1) — checkout works fine — but the WebView won't pre-render. The first tap on "Buy" will show a brief spinner (~1–2s) while Stripe's payment element renders.
</Note>
```

- [ ] **Step 2: Add UIKit tab to the `checkoutSheet(preload:)` section**

Find the section "2. Declarative `.checkoutSheet(preload:)`" (around line 102). The current content is SwiftUI-only. Wrap the existing code block in SwiftUI tab and add a UIKit tab:

Replace the current code block in that section with:

```mdx
<Tabs>
  <Tab title="SwiftUI">
    ```swift
    // Preload all products
    .checkoutSheet(item: $selectedProduct, userId: currentUser.id, preload: .all) { result in
        handleResult(result)
    }

    // Preload specific products
    .checkoutSheet(
        item: $selectedProduct,
        userId: currentUser.id,
        preload: .specified(topProducts)
    ) { result in
        handleResult(result)
    }

    // Default: preloads only the bound product
    .checkoutSheet(item: $selectedProduct, userId: currentUser.id) { result in
        handleResult(result)
    }
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    // In UIKit, use CheckoutSheet.warmUp() or warmUpAll() for equivalent preloading.
    // The preload: parameter is specific to the .checkoutSheet() SwiftUI modifier.

    // Warm up a single product
    await CheckoutSheet.warmUp(productId: product.id, userId: currentUser.id)

    // Warm up all products
    await CheckoutSheet.warmUpAll(userId: currentUser.id)
    ```
  </Tab>
</Tabs>
```

- [ ] **Step 3: Add UIKit tab to `CheckoutSheet.preload()` section**

Find section "5. `CheckoutSheet.preload()`" (around line 152). The current code shows SwiftUI usage. Add a UIKit tab:

Replace the existing code block with:

```mdx
<Tabs>
  <Tab title="SwiftUI">
    ```swift
    let preloaded = await CheckoutSheet.preload(
        productId: product.id,
        userId: currentUser.id
    )

    .checkoutSheet(
        item: $selectedProduct,
        userId: currentUser.id,
        checkoutURL: preloaded?.checkoutURL,
        transactionId: preloaded?.transactionId
    ) { result in
        handleResult(result)
    }
    ```
  </Tab>
  <Tab title="UIKit">
    ```swift
    let preloaded = await CheckoutSheet.preload(
        productId: product.id,
        userId: currentUser.id
    )

    CheckoutSheet.present(
        from: self,
        product: product,
        userId: currentUser.id,
        checkoutURL: preloaded?.checkoutURL,
        transactionId: preloaded?.transactionId
    ) { result in
        handleResult(result)
    }
    ```
  </Tab>
</Tabs>
```

- [ ] **Step 4: Absorb Performance & Memory table from old payment-sheet**

The current `preloading.mdx` ends with a "Performance & Memory" section at line ~174. The old `payment-sheet.mdx` had a similar table. Verify the existing table in `preloading.mdx` covers:

```
| Resource | Cost | Notes |
| PI cache entry | ~1-2 KB | JSON metadata, negligible |
| Pre-rendered WebView | ~3-7 MB | Shares WKProcessPool |
| Network call (preload) | ~200-500ms | Eliminated at presentation time |
```

If it does, no change needed. If it's missing rows, add them.

- [ ] **Step 5: Verify**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
python3 -c "
content = open('iap/preloading.mdx').read()
opens = content.count('<Tabs>')
closes = content.count('</Tabs>')
print(f'Tabs opens: {opens}, closes: {closes}')
assert opens == closes
assert 'UIHostingController' in content, 'Missing UIKit hosting note'
assert 'UIKit' in content
print('OK')
"
```
Expected: `OK`.

- [ ] **Step 6: Commit**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
git add iap/preloading.mdx
git commit -m "docs(preloading): add UIKit hosting note and SwiftUI/UIKit tabs"
```

---

## Task 7: Update `iap/universal-links.mdx`

**Files:**
- Modify: `iap/universal-links.mdx`

Current: 574 lines. Has UIKit SceneDelegate and AppDelegate sections but they're sequential separate headings, not tabbed with SwiftUI. Step 4 ("Handle Checkout Results") is duplicated in quickstart and payment-sheet — remove and cross-link.

**Changes:**
1. Convert "Step 3: Handle the Callback" into tabs (SwiftUI + UIKit). Keep Android in the existing `<CodeGroup>` alongside.
2. Consolidate "UIKit (SceneDelegate)" and "UIKit (AppDelegate - iOS 12 and earlier)" into one UIKit tab with an AppDelegate callout inside.
3. Delete "Step 4: Handle Checkout Results" (~90 lines) — replace with 2-sentence cross-link.
4. Keep "Callback URL Format", "Testing Universal Links", "Fallback Handling", "Android Deep Links", "Troubleshooting" — untouched.

- [ ] **Step 1: Replace Step 3 content**

Find "## Step 3: Handle the Callback" (line ~133). Replace everything from that heading through the end of the two UIKit sub-sections (end of "### UIKit (AppDelegate - iOS 12 and earlier)" block, around line 241) with:

```mdx
## Step 3: Handle the Callback

<Tabs>
  <Tab title="SwiftUI">
    Add `.zeroSettleHandler()` to your root view. It handles `onContinueUserActivity` and `onOpenURL` automatically — no additional setup needed.

    ```swift
    @main
    struct YourApp: App {
        var body: some Scene {
            WindowGroup {
                ContentView()
                    .zeroSettleHandler()
            }
        }
    }
    ```
  </Tab>
  <Tab title="UIKit">
    Add three methods to your `SceneDelegate`. Each covers a different entry point — all three are needed for reliable delivery.

    ```swift
    class SceneDelegate: UIResponder, UIWindowSceneDelegate {

        // Cold-start: universal link tapped while app was not running
        func scene(
            _ scene: UIScene,
            willConnectTo session: UISceneSession,
            options connectionOptions: UIScene.ConnectionOptions
        ) {
            if let activity = connectionOptions.userActivities.first(where: {
                $0.activityType == NSUserActivityTypeBrowsingWeb
            }), let url = activity.webpageURL {
                ZeroSettle.shared.handleUniversalLink(url)
            }
        }

        // Warm-start: universal link tapped while app is running
        func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
            guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
                  let url = userActivity.webpageURL else { return }
            // Returns false if not a ZeroSettle URL — safe to chain your own router
            if !ZeroSettle.shared.handleUniversalLink(url) {
                myDeepLinkRouter.handle(url)
            }
        }

        // Safety net for edge cases
        func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
            for ctx in URLContexts {
                ZeroSettle.shared.handleUniversalLink(ctx.url)
            }
        }
    }
    ```

    <Note>
      **iOS 12 / AppDelegate apps:** If you don't have a `SceneDelegate`, implement `application(_:continue:restorationHandler:)` in your `AppDelegate` instead:

      ```swift
      func application(
          _ application: UIApplication,
          continue userActivity: NSUserActivity,
          restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void
      ) -> Bool {
          guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
                let url = userActivity.webpageURL else { return false }
          return ZeroSettle.shared.handleUniversalLink(url)
      }
      ```
    </Note>
  </Tab>
</Tabs>

For Android and React Native, see the existing `<CodeGroup>` entries in the original Step 3 section (keep those unchanged below the tabs).
```

After the closing `</Tabs>` block, add the Android and React Native handling under a plain label — copy this content exactly (it comes from the current file's Step 3 `<CodeGroup>`):

```mdx
**Android / React Native / Flutter**

<CodeGroup>
```kotlin Kotlin
class MainActivity : ComponentActivity() {
    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        intent?.data?.let { uri ->
            ZeroSettle.handleDeepLink(uri)
        }
    }

    override fun onResume() {
        super.onResume()
        ZeroSettle.onResume()
    }
}
```

```tsx React Native
import { ZeroSettle } from 'react-native-zerosettle-kit';
import { Linking } from 'react-native';

Linking.addEventListener('url', ({ url }) => {
  ZeroSettle.handleUniversalLink(url);
});
```

```dart Flutter
// Flutter handles deep links automatically via the native plugin.
await ZeroSettle.instance.handleUniversalLink(url);
```
</CodeGroup>
```

- [ ] **Step 2: Replace Step 4 with a cross-link**

Find "## Step 4: Handle Checkout Results" (around line 243). Replace the entire section (through the closing `</CodeGroup>` at line ~330) with:

```mdx
## Step 4: Handle Checkout Results

When the SDK receives the callback URL, it verifies the transaction and calls your `ZeroSettleDelegate` methods (or resolves the `checkoutSheet` result handler directly). See [Checkout Sheet](/iap/payment-sheet) for the complete result-handling patterns.
```

- [ ] **Step 3: Verify length and structure**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
wc -l iap/universal-links.mdx
```
Expected: ≤420 lines (down from 574).

```bash
python3 -c "
content = open('iap/universal-links.mdx').read()
opens = content.count('<Tabs>')
closes = content.count('</Tabs>')
print(f'Tabs opens: {opens}, closes: {closes}')
assert opens == closes
assert 'UIWindowSceneDelegate' in content, 'UIKit SceneDelegate code missing'
assert 'willConnectTo' in content, 'Cold-start handler missing'
print('OK')
"
```
Expected: `OK`.

- [ ] **Step 4: Commit**

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1
git add iap/universal-links.mdx
git commit -m "docs(universal-links): convert Step 3 to SwiftUI/UIKit tabs, trim Step 4 duplicate"
```

---

## Spec Self-Review Checklist

After completing all tasks, run this verification across all modified files:

```bash
cd /Users/ryanelliott/dev/docs/.worktrees/uikit-docs-phase1

# 1. All Tabs components balanced
for f in iap/quickstart.mdx iap/payment-sheet.mdx iap/preloading.mdx iap/universal-links.mdx iap/checkout-errors.mdx iap/promotions.mdx; do
  python3 -c "
content = open('$f').read()
o = content.count('<Tabs>')
c = content.count('</Tabs>')
status = 'OK' if o == c else f'MISMATCH: {o} opens, {c} closes'
print(f'$f: {status}')
"
done

# 2. All new articles referenced in docs.json
grep -c "checkout-errors\|promotions" docs.json

# 3. No broken internal links to old sections
grep -rn "payment-sheet#error-handling\|payment-sheet#promotions\|payment-sheet#performance" iap/ || echo "No stale anchors"

# 4. UIKit appears in each rebuilt article
for f in iap/quickstart.mdx iap/payment-sheet.mdx iap/preloading.mdx iap/universal-links.mdx; do
  count=$(grep -c "UIKit" $f)
  echo "$f: $count UIKit references"
done
```

Expected:
- All files: `OK` (balanced Tabs)
- `docs.json` grep: `2`
- No stale anchors
- Each file: ≥2 UIKit references
