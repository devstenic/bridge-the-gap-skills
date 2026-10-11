# Embed and compose

Two ways for a host to show the same module. The module does not change between them; only the host side does.

| | Embed | Compose |
|---|---|---|
| When | The shell still runs on the old navigator (GetX, `Navigator.pushNamed`, an older router) | The shell runs on one root GoRouter |
| Host side | One old page per module entry; the module's own GoRouter inside it | The module's subtree joins the root router's routes |
| URL | Two routers write the address bar (transitional) | The root router owns the URL, deep links and browser history |
| Back | The host page forwards Back to the module | The root router pops pages; the module's subtree decides at its root |
| Delete condition | Remove when the shell runs on go_router | Target state |

All code below is the checkout example: Address, Payment and Review are a new module; Cart and Order placed stay in the old GetX app. It compiles and its tests pass on Flutter 3.44.3 with go_router 16.3.0 and 18.0.2 and get 4.7.3.

## The module's route subtree

One subtree carries the steps and the module-wide guard. Nesting the steps makes `go('/checkout/payment/review')` build the stack Address, Payment, Review, so Back returns one step at a time.

```dart
GoRoute checkoutRoutes({
  required CheckoutFlow flow,
  required CartPort cart,
  required CheckoutHostPort host,
  required ModuleChrome chrome,
}) {
  return GoRoute(
    path: '/checkout',
    // Cold entry to a later step without its data restarts the flow.
    redirect: (context, state) {
      final atStart = state.uri.path == '/checkout';
      if (!atStart && flow.draft.address == null) return '/checkout';
      return null;
    },
    builder: (context, state) => AddressStep(flow: flow, chrome: chrome),
    routes: [
      GoRoute(
        path: 'payment',
        builder: (context, state) => PaymentStep(flow: flow, chrome: chrome),
        routes: [
          GoRoute(
            path: 'review',
            builder: (context, state) =>
                ReviewStep(flow: flow, cart: cart, host: host, chrome: chrome),
          ),
        ],
      ),
    ],
  );
}
```

The guard sits on the subtree, so no host can mount the routes without it. The handle exposes the subtree as `routes` (see the module-ports-and-adapters skill for the full handle).

## Written once: EmbeddedRouter

Lives in the shared routing package. Owns one GoRouter for the lifetime of the page that shows it.

```dart
class EmbeddedRouter extends StatefulWidget {
  const EmbeddedRouter({super.key, required this.createRouter});

  final GoRouter Function() createRouter;

  @override
  State<EmbeddedRouter> createState() => _EmbeddedRouterState();
}

class _EmbeddedRouterState extends State<EmbeddedRouter> {
  late final GoRouter _router = widget.createRouter();

  @override
  void dispose() {
    _router.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Router.withConfig(config: _router);
}
```

## Written once: ModuleHost

Also in the shared routing package. Owns one module instance per visit. Two owners that end together: `ModuleHost` owns the module, `EmbeddedRouter` owns the router built from it.

```dart
class ModuleHost<M extends Object> extends StatefulWidget {
  const ModuleHost({
    super.key,
    required this.create,
    required this.dispose,
    required this.builder,
  });

  final M Function() create;
  final Future<void> Function(M module) dispose;
  final Widget Function(BuildContext context, M module) builder;

  @override
  State<ModuleHost<M>> createState() => _ModuleHostState<M>();
}

class _ModuleHostState<M extends Object> extends State<ModuleHost<M>> {
  late final M _module = widget.create();

  @override
  void dispose() {
    // State.dispose cannot await. If cleanup must finish before the next
    // visit, await it from the host's route-removal hook instead.
    unawaited(widget.dispose(_module));
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => widget.builder(context, _module);
}
```

## Embed: the host page and its GetPage

The host page has no State. It forwards Back (back-navigation-chain skill) and shows the module's routes through `EmbeddedRouter`.

```dart
class CheckoutHostPage extends StatelessWidget {
  const CheckoutHostPage({super.key, required this.checkout});

  final CheckoutModule checkout;

  @override
  Widget build(BuildContext context) {
    return PopScope<Object?>(
      canPop: false, // GetX never pops this page by itself
      onPopInvokedWithResult: (didPop, _) {
        if (didPop) return;
        if (checkout.requestBack() == BackResolution.handled) return; // 1 + 2
        Get.back<void>(); // 3: host fallback
      },
      child: EmbeddedRouter(
        createRouter: () => GoRouter(
          initialLocation: '/checkout',
          navigatorKey: checkout.navigatorKey,
          routes: [checkout.routes],
        ),
      ),
    );
  }
}
```

One `GetPage` per module entry. This is the only place the old app knows about the module:

```dart
GetPage(
  name: '/checkout',
  page: () => ModuleHost<CheckoutModule>(
    create: () => CheckoutModule.create(
      cart: GetxCartPort(Get.put(LegacyCartController()), LegacyOrderRepository()),
      host: GetxCheckoutHost(),
    ),
    dispose: (module) => module.dispose(),
    builder: (context, checkout) => CheckoutHostPage(checkout: checkout),
  ),
),
```

Each module that the old app shows needs one such page (a few lines). `EmbeddedRouter` and `ModuleHost` are written once. Inside the module, navigation is plain go_router.

## Compose: the target

The same subtree joins the root router. No host page, no `EmbeddedRouter`, no Back forwarding: the root router pops pages, and the module decides at its root with `onExit` (back-navigation-chain skill).

```dart
GoRouter buildAppRouter(CheckoutModule checkout) => GoRouter(
      initialLocation: '/cart',
      routes: [
        GoRoute(path: '/cart', builder: (context, state) => const CartPage()),
        checkout.routes,
        GoRoute(path: '/orders/placed', builder: (context, state) => const OrderPlacedPage()),
      ],
    );
```

## The browser URL

Facts, checked in the Flutter and go_router sources (Flutter 3.44.3 and 3.47.6 stable; go_router 16.3.0 and 18.0.2):

- The root `Navigator` of a `MaterialApp` without a router reports route changes to the engine (`reportsRouteUpdateToEngine: true`) in single-entry history mode. A GetX app built with `GetMaterialApp(getPages: ...)` is such an app.
- A GoRouter mounted with `Router.withConfig` also reports its location: its route information provider calls `SystemNavigator.selectMultiEntryHistory()` and `SystemNavigator.routeInformationUpdated(...)`.
- So an embedded module on the web means two routers writing one address bar. `routerNeglect: true` on the inner GoRouter makes it replace the current history entry instead of adding new ones; the URL still changes.
- Real URLs per step, deep links and working browser Back and Forward come with compose, where one root router owns the URL.

## Why not switch GetX to its own Router mode

`GetMaterialApp.router` (get 4.7.x) builds a `GetDelegate` whose `Navigator` uses `pages` and has no `onGenerateRoute`. Classic `Get.toNamed`, `Get.offNamed` and `Get.offAllNamed` call `pushNamed` on that navigator, which needs `onGenerateRoute`. Every named-route call would have to move to `Get.rootDelegate` at once. That is a big-bang rewrite of the old app, and it still would not be go_router.
