# Accessibility tests at the seam

Test the new screen where old meets new: inside the old shell, with the old theme, at twice the text size. These tests come from the checkout example and pass on Flutter 3.44.3 with go_router 16.3.0 and 18.0.2 and get 4.7.3. `LegacyApp` is the old GetX app; tapping "Checkout" opens the module.

```dart
void main() {
  tearDown(Get.reset);

  Future<void> openCheckout(WidgetTester tester) async {
    await tester.pumpWidget(const LegacyApp());
    await tester.tap(find.text('Checkout'));
    await tester.pumpAndSettle();
  }

  testWidgets('address step inside the old shell at 2x text meets the guidelines',
      (tester) async {
    final semantics = tester.ensureSemantics();
    tester.platformDispatcher.textScaleFactorTestValue = 2.0;
    addTearDown(tester.platformDispatcher.clearTextScaleFactorTestValue);

    await openCheckout(tester);

    await expectLater(tester, meetsGuideline(textContrastGuideline));
    await expectLater(tester, meetsGuideline(labeledTapTargetGuideline));
    await expectLater(tester, meetsGuideline(androidTapTargetGuideline));
    await expectLater(tester, meetsGuideline(iOSTapTargetGuideline));
    semantics.dispose();
  });

  testWidgets('each module page is named for screen readers', (tester) async {
    final semantics = tester.ensureSemantics();
    await openCheckout(tester);
    expect(
      tester.getSemantics(find.byType(PageAnnouncer)),
      isSemantics(label: 'Address', namesRoute: true),
    );
    semantics.dispose();
  });

  testWidgets('the focused field keeps focus when the keyboard opens', (tester) async {
    await openCheckout(tester);
    await tester.tap(find.byKey(const Key('street')));
    await tester.pump();
    final before = tester.state<EditableTextState>(find.byType(EditableText));
    expect(before.widget.focusNode.hasFocus, isTrue);

    tester.view.viewInsets = const FakeViewPadding(bottom: 300); // keyboard opens
    addTearDown(tester.view.resetViewInsets);
    await tester.pumpAndSettle();

    final after = tester.state<EditableTextState>(find.byType(EditableText));
    expect(identical(before, after), isTrue, reason: 'the field was rebuilt from scratch');
    expect(after.widget.focusNode.hasFocus, isTrue);
  });
}
```

Notes:

- Set the text scale with `tester.platformDispatcher.textScaleFactorTestValue`; the app then builds its own `MediaQuery` with it.
- `isSemantics` replaced `containsSemantics` in current Flutter (deprecated after 3.40). On older Flutter use `containsSemantics` with the same arguments.
- Add a semantics snapshot for each template (for example a golden of `tester.getSemantics(...)` output, or a list of labels and flags), so a refactor that drops a label shows up in review.
- Guideline checks cover contrast, labels and tap targets. They do not prove reading order, focus order or what a screen reader says. Keep a real TalkBack and VoiceOver pass on the release checklist.
