# Back on each platform

Checked in the Flutter sources (3.44.3 and the 3.47.6 stable branch) and go_router (16.3.0 and 18.0.2).

## iOS

- The edge swipe is enabled only when the route can be popped. A route whose `PopScope` has `canPop: false` reports `popDisposition == RoutePopDisposition.doNotPop`, and `popGestureEnabled` returns false (`widgets/routes.dart`).
- At the module root that means no swipe-back. The visible Back button is the only way out, so the template must always show it, labelled and reachable with a screen reader and a keyboard.
- Inner steps of the module live on the module's own navigator, so swiping between steps still works there.

## Android

- System Back (button or gesture) reaches the root navigator through `WidgetsApp.didPopRoute`, which calls `navigator.maybePop()`, which runs the `PopScope` of the top route. That is the chain's entry point in an embedded module.
- The `PopScope.canPop` documentation says that Android's predictive back feature does not animate when `canPop` is false. Expect no predictive-back preview at the module root.
- In a router-based app (`MaterialApp.router`), system Back goes through the router's back button dispatcher; go_router pops the deepest navigator that can pop.

## Web

- A `MaterialApp` without a router reports route changes to the browser (`reportsRouteUpdateToEngine: true`, single-entry history). A GetX app on `GetMaterialApp(getPages: ...)` is such an app.
- An embedded GoRouter reports its location too (`SystemNavigator.selectMultiEntryHistory()` and `routeInformationUpdated`). Two routers write one address bar, so browser Back and Forward are not a reliable part of the chain while the module is embedded.
- One root router that owns the URL (compose) gives each step a real URL, deep links, and browser history that matches the in-app stack.
