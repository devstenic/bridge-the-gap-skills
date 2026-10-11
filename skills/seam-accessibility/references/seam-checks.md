# Seam checks in code

From the checkout example: Address, Payment and Review are a module shown inside an old GetX app. The code compiles and its tests pass on Flutter 3.44.3 with go_router 16.3.0 and 18.0.2 and get 4.7.3.

## Open a screen: name the page

A design-system widget that every module template uses:

```dart
class PageAnnouncer extends StatefulWidget {
  const PageAnnouncer({super.key, required this.title, required this.child});

  final String title;
  final Widget child;

  @override
  State<PageAnnouncer> createState() => _PageAnnouncerState();
}

class _PageAnnouncerState extends State<PageAnnouncer> {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (!mounted || !kIsWeb || !MediaQuery.supportsAnnounceOf(context)) return;
      SemanticsService.sendAnnouncement(
        View.of(context),
        widget.title,
        Directionality.of(context),
      );
    });
  }

  @override
  Widget build(BuildContext context) {
    return Semantics(
      container: true,
      explicitChildNodes: true,
      namesRoute: true,
      label: widget.title,
      child: widget.child,
    );
  }
}
```

- `namesRoute` gives the page its name; on mobile the platform announces it.
- On the web a route change inside the app fires no browser event, so the page sends one polite announcement.
- `SemanticsService.announce` is deprecated in current Flutter; `sendAnnouncement` takes the view. Flutter's documentation asks to prefer implicit mechanisms like `Semantics`, and notes that Android has deprecated announcement events.

When the old app announces pages from a navigator observer (for example "Address, page loaded" on every push), that observer sees only the old navigator. Add the module's host page to the observer's quiet list, so the user hears the module's page names and not the host page's route name as well.

## Type: keep focus when the frame or keyboard changes

The shape that drops focus, in a shared frame widget (fragments; `keyboardInset` and `barHeight` come from the frame):

```dart
// The tree changes shape when the keyboard opens: the page is rebuilt from
// scratch, the focused field loses focus and the keyboard closes again.
if (keyboardInset > 0) return child;
return Padding(padding: EdgeInsets.only(bottom: barHeight), child: child);
```

The shape that keeps it: one tree, different values.

```dart
return Padding(
  padding: EdgeInsets.only(bottom: keyboardInset > 0 ? 0 : barHeight),
  child: child,
);
```

This bug appears when a new frame (a new navigation bar, a new shell) wraps old screens: the screens did not change, the frame did. Test it once in the frame and every screen inside is covered.

## Go back: a visible Back at the module root

The step template shows Back where the module's navigator cannot pop, and sends it through the back chain (see the back-navigation-chain skill):

```dart
class StepTemplate extends StatelessWidget {
  const StepTemplate({
    super.key,
    required this.title,
    required this.chrome,
    required this.child,
  });

  final String title;
  final ModuleChrome chrome;
  final Widget child;

  @override
  Widget build(BuildContext context) {
    final canPopInModule = Navigator.of(context).canPop();
    return Scaffold(
      appBar: chrome.appBar?.call(context, title) ??
          AppBar(
            title: Text(title),
            leading: canPopInModule
                ? null
                : BackButton(
                    onPressed: () =>
                        Navigator.of(context, rootNavigator: true).maybePop(),
                  ),
          ),
      body: SafeArea(
        child: PageAnnouncer(
          title: title,
          child: Padding(padding: const EdgeInsets.all(16), child: child),
        ),
      ),
      bottomNavigationBar: chrome.bottomNav?.call(context),
    );
  }
}
```

The template owns the frame slots, the safe area, the page name and Back. A screen inside it owns only its content.

## Get the result: the host announces

The module reports an outcome through its exit port and is then disposed. The host shows the next screen, so the host announces what happened (for example "Order placed") and the next screen starts with its title. Name outcomes as facts, so the host can phrase the announcement.

## One design system for both hosts

```dart
/// One token source for both hosts.
@immutable
final class AppTokens extends ThemeExtension<AppTokens> {
  const AppTokens({required this.accent, required this.minTapSize});

  final Color accent;
  final double minTapSize;

  static const light = AppTokens(accent: Color(0xFF0B57D0), minTapSize: 48);

  @override
  AppTokens copyWith({Color? accent, double? minTapSize}) => AppTokens(
        accent: accent ?? this.accent,
        minTapSize: minTapSize ?? this.minTapSize,
      );

  @override
  AppTokens lerp(AppTokens? other, double t) => other == null
      ? this
      : AppTokens(
          accent: Color.lerp(accent, other.accent, t)!,
          minTapSize: lerpDouble(minTapSize, other.minTapSize, t)!,
        );
}

/// The same theme object for GetMaterialApp (old host) and MaterialApp.router (new host).
final appTheme = ThemeData(extensions: const [AppTokens.light]);
```

Read tokens with `Theme.of(context).extension<AppTokens>()!`. An old screen can move onto tokens before its logic is migrated.

Copies of design-system widgets inside a module drift: the shared widget gets an accessibility fix, the copy does not. When a copy is unavoidable during a migration, write its owner and removal condition next to it, and improve the shared widget instead of the copy.
