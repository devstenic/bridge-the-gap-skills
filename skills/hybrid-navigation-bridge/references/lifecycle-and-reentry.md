# Lifecycle and re-entry

## The router cache trap

A common first version of embedding keeps routers in a global map, one per start location, and reuses them on every visit:

```dart
// before: one global map, alive for the whole app session
static final _routers = <String, GoRouter>{};

Widget checkoutPage() {
  final router = _routers.putIfAbsent('/checkout', buildRouter);
  return Router.withConfig(config: router); // reused on every visit
}
```

It looks like a cheap way to keep state. It turns the router into the owner of the journey, and it fails only on the second visit, so the first test passes. Typical symptoms:

- **Dead screen on re-entry.** Open the module, go back, open it again: buttons do nothing. The second visit got the old router in its old state.
- **Duplicate GlobalKey.** The old navigator animates between pages, so for a moment two pages are alive and both show the same router instance. Its navigator key is mounted twice.
- **Stuck on a stale page.** The cached router still points at a page whose condition has expired (a lockout, an error). Re-entry lands there again, sometimes in a loop.

Each patch (clear the cache here, add a key there) treats one symptom. The fix is ownership: a fresh module and a fresh router per visit, disposed on leave (`ModuleHost` + `EmbeddedRouter`, see [embed-and-compose.md](embed-and-compose.md)).

## Rules

- **Start, resume or focus.** Every entry resolves to one of them explicitly. Start creates new journey state. Resume restores state from its owner (not from an old router) and derives the step from it. Focus (or reject) applies when the same journey is already on screen.
- **One mount per journey.** A journey is never shown by two routers at once; the same GoRouter instance is never mounted twice.
- **Leaving decides nothing by itself.** Completed, cancelled and suspended are decisions of the journey. Disposing a widget or a router clears presentation only.
- **Late results stay in their visit.** An async call started by a visit that has ended must not navigate through a new visit's router. Check that the visit is still current before reporting an outcome.
- **Awaited cleanup.** `State.dispose` cannot await. If module cleanup must finish before the next visit (for example it cancels server work), run it from the host's route-removal hook and await it there.

## Tests

Use real routers: the old navigator and the module's GoRouter. These tests come from the checkout example and pass on Flutter 3.44.3 with go_router 16.3.0 and 18.0.2 and get 4.7.3. `LegacyApp` is the old GetX app (`GetMaterialApp(initialRoute: '/cart', getPages: ...)`); its optional `onDispose` callback reports each module the `ModuleHost` disposes.

```dart
void main() {
  tearDown(Get.reset);

  testWidgets('leaving disposes the module and re-entry starts fresh', (tester) async {
    final disposed = <CheckoutModule>[];
    await tester.pumpWidget(LegacyApp(onDispose: disposed.add));

    await tester.tap(find.text('Checkout'));
    await tester.pumpAndSettle();
    await tester.tap(find.text('Continue'));
    await tester.pumpAndSettle();
    expect(find.text('Payment'), findsWidgets);

    Get.back<void>(); // the host leaves the module
    await tester.pumpAndSettle();
    expect(disposed, hasLength(1));

    await tester.tap(find.text('Checkout'));
    await tester.pumpAndSettle();
    expect(find.text('Address'), findsWidgets); // a fresh router starts at the first step
    expect(find.text('Payment'), findsNothing);
  });

  testWidgets('cold entry to a later step restarts the flow', (tester) async {
    final module = CheckoutModule.create(cart: FakeCart(), host: FakeHost());
    final router = GoRouter(
      initialLocation: '/checkout/payment/review',
      navigatorKey: module.navigatorKey,
      routes: [module.routes],
    );
    addTearDown(router.dispose);

    await tester.pumpWidget(MaterialApp.router(routerConfig: router));
    await tester.pumpAndSettle();
    expect(find.text('Address'), findsWidgets);
    expect(find.text('Review'), findsNothing);
  });
}
```

`FakeCart` and `FakeHost` are minimal implementations of the module's ports. Test a guard's decision table as plain unit tests; keep the real-router tests for wiring: which route the guard sits on, redirect order, the stack, re-entry.
