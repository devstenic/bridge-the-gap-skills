# The architecture test

A rule only holds if a machine checks it. This test lives inside the module package (`test/architecture/`) and runs with `flutter test` from the package directory, so `lib/` is the module's own code. Plain Dart: read the files, check the imports.

```dart
import 'dart:io';

import 'package:flutter_test/flutter_test.dart';

/// Imports point in: the module never imports the old app, GetX or another
/// module.
void main() {
  final lib = Directory('lib');
  final directive = RegExp(r'''^\s*(?:import|export)\s+['"]([^'"]+)['"]''', multiLine: true);

  List<String> violations(bool Function(String uri) forbidden) => [
        for (final file in lib.listSync(recursive: true).whereType<File>())
          if (file.path.endsWith('.dart'))
            for (final match in directive.allMatches(file.readAsStringSync()))
              if (forbidden(match.group(1)!)) '${file.path}: ${match.group(1)}',
      ];

  test('no file reaches for GetX', () {
    expect(violations((uri) => uri.startsWith('package:get/')), isEmpty);
  });

  test('no file imports the old app', () {
    expect(violations((uri) => uri.startsWith('package:my_app/')), isEmpty);
  });

  test('no file imports another module', () {
    const siblings = ['package:auth/', 'package:orders/'];
    expect(violations((uri) => siblings.any(uri.startsWith)), isEmpty);
  });
}
```

Replace `my_app` with the host app's package name and list the sibling modules. A failing test prints every offending file and import.

## The pubspec check

An import test misses a dependency that is declared but not yet used. Check the manifest too:

```dart
bool dependsOn(String pubspec, String package) =>
    RegExp('^\\s+$package:', multiLine: true).hasMatch(pubspec);

test('the module does not depend on GetX or the old app', () {
  final pubspec = File('pubspec.yaml').readAsStringSync();
  expect(dependsOn(pubspec, 'get'), isFalse);
  expect(dependsOn(pubspec, 'my_app'), isFalse);
});
```

## Allowed and forbidden

| A module may import | A module never imports |
|---|---|
| Flutter, go_router, its state library (for example flutter_bloc) | GetX (`package:get`) |
| Shared packages: core, network, routing, design system | The old app (`package:my_app`) |
| Its own `src/` | Another module (`package:auth`, ...) |

Shared packages follow the same rule one level down: they never import a module or the old app. If a shared package still imports GetX, list it as debt with an owner; the test keeps the count from growing.

## In a monorepo

With a pub workspace (and melos for scripts), run the architecture tests of every module in CI, for example `melos exec --dir-exists=test/architecture -- flutter test test/architecture`. Every new module gets this test on day one.
