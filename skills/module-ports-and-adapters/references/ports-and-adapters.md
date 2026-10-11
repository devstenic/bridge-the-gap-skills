# Ports and adapters in code

The checkout example. Address, Payment and Review are a new module (`packages/modules/checkout`); the cart, the order API, Cart and Order placed stay in the old GetX app (`lib/app`). The code compiles and its tests pass on Flutter 3.44.3 with go_router 16.3.0 and 18.0.2 and get 4.7.3.

```text
my_app/
├── lib/app/                    old app · GetX
│   └── adapters/               old meets new, only here
└── packages/
    ├── modules/checkout/       new module
    │   ├── lib/checkout.dart   public facade
    │   ├── lib/src/            private: routes, flow, steps
    │   └── test/architecture/  import rules
    └── common/                 shared: core, network, routing, design system
```

## Types that cross the boundary

They belong to the module.

```dart
final class OrderId {
  const OrderId(this.value);
  final String value;
}

final class CartLine {
  const CartLine({required this.name, required this.quantity});
  final String name;
  final int quantity;
}

final class Cart {
  const Cart(this.lines);
  final List<CartLine> lines;
  bool get isEmpty => lines.isEmpty;
}

final class OrderDraft {
  const OrderDraft({this.address, this.paymentMethod});
  final String? address;
  final String? paymentMethod;

  bool get isEmpty => address == null && paymentMethod == null;

  OrderDraft copyWith({String? address, String? paymentMethod}) => OrderDraft(
        address: address ?? this.address,
        paymentMethod: paymentMethod ?? this.paymentMethod,
      );
}
```

## The three ports

```dart
/// Port 1 - data. What the module needs, in the module's own words.
abstract interface class CartPort {
  Future<Cart> currentCart();
  Future<OrderId> placeOrder(OrderDraft draft);
}

/// Port 2 - exit. The module says what happened; the host decides where to go.
abstract interface class CheckoutHostPort {
  Future<void> orderPlaced(OrderId id);
  Future<void> cancelled();
  Future<void> signInRequired();
}

/// Port 3 - frame. The host lends its frame as builders; the module never
/// imports host widgets.
final class ModuleChrome {
  const ModuleChrome({this.appBar, this.bottomNav});

  final PreferredSizeWidget Function(BuildContext context, String title)? appBar;
  final WidgetBuilder? bottomNav;
}
```

Outcomes name facts. `orderPlaced` is a fact; `goToOrdersScreen` would be a host decision smuggled into the module. When the host later moves to a go_router shell, it maps the same outcomes to `router.go(...)` and the module does not change.

## The handle

```dart
/// What Back did inside the module.
enum BackResolution { handled, notHandled }

/// The handle: everything the host may do with the module.
final class CheckoutModule {
  CheckoutModule._(this._cart, this._host, this.chrome);

  /// A fresh module for one visit: its own flow, routes and navigator key.
  factory CheckoutModule.create({
    required CartPort cart,
    required CheckoutHostPort host,
    ModuleChrome chrome = const ModuleChrome(),
  }) =>
      CheckoutModule._(cart, host, chrome);

  final CartPort _cart;
  final CheckoutHostPort _host;
  final ModuleChrome chrome;
  final CheckoutFlow _flow = CheckoutFlow();

  /// The module's own navigator. The host passes it to the router it creates.
  final GlobalKey<NavigatorState> navigatorKey =
      GlobalKey<NavigatorState>(debugLabel: 'checkout');

  /// The module's routes and their guard: one subtree for embed and compose.
  late final GoRoute routes =
      checkoutRoutes(flow: _flow, cart: _cart, host: _host, chrome: chrome);

  /// Back, forwarded by the host (see the back-navigation-chain skill).
  BackResolution requestBack() { /* ... */ }

  Future<void> dispose() async => _flow.dispose();
}
```

`CheckoutFlow` is the module-private owner of the journey (the draft, the step data). Screens can have their own Cubits; the router never owns journey state.

## The facade

```dart
/// lib/checkout.dart: the handle, the ports, the types that cross the boundary.
library;

export 'src/api/checkout_module.dart' show BackResolution, CheckoutModule;
export 'src/api/ports.dart' show CartPort, CheckoutHostPort, ModuleChrome;
export 'src/api/types.dart' show Cart, CartLine, OrderDraft, OrderId;
```

Not exported: the DI container, repositories, the flow, inner routes and step widgets. A host that needs one of them is showing a missing use case on the handle; widen the handle instead.

## Adapters in the host

```dart
/// Adapter: implements the module's data port with old code.
/// Delete when: the cart and orders live in their own module.
final class GetxCartPort implements CartPort {
  GetxCartPort(this._cart, this._orders);

  final LegacyCartController _cart; // old GetX controller
  final LegacyOrderRepository _orders; // old, tested API

  @override
  Future<Cart> currentCart() async =>
      Cart([for (final name in _cart.items) CartLine(name: name, quantity: 1)]);

  @override
  Future<OrderId> placeOrder(OrderDraft draft) async => OrderId(
        await _orders.submit(address: draft.address!, payment: draft.paymentMethod!),
      );
}

/// Adapter: the host decides where each outcome leads.
/// Delete when: the checkout entry runs in the go_router shell.
final class GetxCheckoutHost implements CheckoutHostPort {
  @override
  Future<void> orderPlaced(OrderId id) async {
    Get.offNamed<void>('/orders/placed', arguments: id.value);
  }

  @override
  Future<void> cancelled() async {
    Get.back<void>();
  }

  @override
  Future<void> signInRequired() async {
    Get.offAllNamed<void>('/login');
  }
}
```

The frame adapter passes the old app's own app bar and navigation as builders (`LegacyAppBar` and `LegacyBottomBar` stand for your old app's widgets):

```dart
CheckoutModule.create(
  cart: GetxCartPort(Get.find(), LegacyOrderRepository()),
  host: GetxCheckoutHost(),
  chrome: ModuleChrome(
    appBar: (context, title) => LegacyAppBar(title: title),
    bottomNav: (context) => const LegacyBottomBar(),
  ),
);
```

Later, a go_router shell supplies a new frame through the same port; the module does not change.

## Shapes to replace

| You see | Replace with |
|---|---|
| The module calls `Get.toNamed` or `Navigator.pushNamed` to leave | An outcome on the exit port; the host adapter navigates |
| The package exports its DI container or repositories | A use case on the handle |
| The host reads or writes the module's internal state | A typed input to `create` or a method on the handle |
| A dependency treated as optional because it may not be registered | A required parameter, or an explicit mode the host chooses |
| An adapter with no stated delete condition | The condition written next to it, and tracked |
| A port carrying old models | Module types, mapped in the adapter |
