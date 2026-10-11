# The back chain in code

From the checkout example (Address, Payment, Review in a module; Cart in the old GetX app). The code compiles and the tests pass on Flutter 3.44.3 with go_router 16.3.0 and 18.0.2 and get 4.7.3.

## 1 + 2: the module decides

```dart
/// What Back did inside the module.
enum BackResolution { handled, notHandled }

final class CheckoutModule {
  // ...create(), routes, dispose(): see the module-ports-and-adapters skill.

  /// The module's own navigator. The host passes it to the router it creates.
  final GlobalKey<NavigatorState> navigatorKey =
      GlobalKey<NavigatorState>(debugLabel: 'checkout');

  /// Back, forwarded by the host: step back inside the module first; at the
  /// first step the module decides what Back means.
  BackResolution requestBack() {
    final navigator = navigatorKey.currentState;
    if (navigator != null && navigator.canPop()) {
      navigator.pop();
      return BackResolution.handled;
    }
    if (_flow.draft.isEmpty) return BackResolution.notHandled;
    _flow.discardDraft(); // this module's policy: Back at the start cancels
    unawaited(_host.cancelled());
    return BackResolution.handled;
  }
}
```

The policy at the first step is a product decision. Cancelling, asking "discard your changes?", or keeping the draft for later are all valid; what matters is that the module names the decision instead of the host guessing it from the current path.

## 3: the host goes last

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

The same shape works with any Navigator 1.0 host; replace `Get.back()` with that host's pop.

## The on-screen Back at the module root

The step template shows Back where the inner navigator cannot pop, and sends it through the same chain:

```dart
final canPopInModule = Navigator.of(context).canPop();
// ...
AppBar(
  title: Text(title),
  leading: canPopInModule
      ? null // the AppBar shows its own back arrow for inner steps
      : BackButton(
          onPressed: () => Navigator.of(context, rootNavigator: true).maybePop(),
        ),
)
```

`maybePop` on the root navigator reaches the host page, whose `PopScope` runs `requestBack()`.

## Compose: the same decision in `onExit`

In a composed shell there is no host page. The root router pops the module's first page like any other page. Keep the first-step decision in one module function and call it from the module subtree's `onExit` (return `true` to leave, `false` to stay):

```dart
GoRoute(
  path: '/checkout',
  onExit: (context, state) => decideBackAtStart(), // FutureOr<bool>
  // builder, redirect and routes as before
)
```

go_router calls `onExit` when the route leaves its history: after a pop, and also after a `go` to a location outside the subtree. So the decision returns `true` once the journey has finished (for example after "order placed"); it only questions a Back from an unfinished journey.

## The bug this prevents

A multi-step flow moved between sibling routes with `go()`. Its inner stack always held one page. The user pressed Back in the middle of the flow; the inner router had nothing to pop; the host treated that as "leave" and closed the flow. Users landed on the home screen, or on Android the app went to the background, halfway through. Steps 2 and 3 of the skill fix both halves: a real stack, and a module decision before any host fallback.

## Tests

```dart
void main() {
  tearDown(Get.reset);

  Future<void> openCheckout(WidgetTester tester) async {
    await tester.pumpWidget(const LegacyApp());
    await tester.tap(find.text('Checkout'));
    await tester.pumpAndSettle();
    expect(find.text('Address'), findsWidgets);
  }

  testWidgets('Back pops a module step before it leaves the module', (tester) async {
    await openCheckout(tester);
    await tester.tap(find.text('Continue'));
    await tester.pumpAndSettle();
    expect(find.text('Payment'), findsWidgets);

    await tester.binding.handlePopRoute(); // system Back
    await tester.pumpAndSettle();
    expect(find.text('Address'), findsWidgets); // still inside the module
  });

  testWidgets('Back at the module root with a draft cancels through the exit port',
      (tester) async {
    await openCheckout(tester);
    await tester.tap(find.text('Continue'));
    await tester.pumpAndSettle();
    await tester.binding.handlePopRoute(); // Payment -> Address
    await tester.pumpAndSettle();

    await tester.binding.handlePopRoute(); // root: module cancels, host goes back
    await tester.pumpAndSettle();
    expect(find.text('Cart'), findsOneWidget);
  });

  testWidgets('Back at the module root with nothing to lose falls back to the host',
      (tester) async {
    await openCheckout(tester);
    await tester.binding.handlePopRoute();
    await tester.pumpAndSettle();
    expect(find.text('Cart'), findsOneWidget);
  });

  testWidgets('the on-screen Back at the module root uses the same chain', (tester) async {
    await openCheckout(tester);
    await tester.tap(find.byType(BackButton));
    await tester.pumpAndSettle();
    expect(find.text('Cart'), findsOneWidget);
  });
}
```

`tester.binding.handlePopRoute()` is the system Back (Android button or gesture end). Add the same cases for any other Back source the app supports.
