---
title: 'Hiding SwiftUI View'
author: 'Clive Liu'
layout: post
tags: [Swift, SwiftUI]
---

When building user interfaces, it’s common to show or hide views based on state. In SwiftUI, there are several options:

1. A simple `if-else` statement
2. The `.hidden()` modifier
3. The `.opacity(_:)` modifier

## `if-else`

```swift
if condition {
    MyView()
}

if condition {
    MyView()
} else {
    MyOtherView()
}
```

Notes:
- The view’s structural identity changes when the condition toggles, which resets any internal state.
- View transitions are used instead of standard property animations.
- Since views are inserted or removed from the hierarchy, layout may shift unexpectedly.

Use this approach when view state preservation or layout consistency are not required.

## `.hidden()` Modifier

If you need to preserve layout, you can do this:

```swift
MyView()
    .hidden()
```

However, the `.hidden()` modifier **doesn't accept a boolean**, so you can’t directly control visibility conditionally—an odd limitation.

One workaround is a custom modifier, for example:

```swift
extension View {

    func hidden(_ hide: Bool) -> some View {
        self.modifier(HiddenModifier(hide: hide))
    }
}

fileprivate struct HiddenModifier: ViewModifier {

    let hide: Bool

    func body(content: Content) -> some View {
        content
            .hidden()
            .overlay {
                if !hide {
                    content
                }
            }
    }
}
```

## `.opacity(_:)` Modifier

But a simpler solution is to toggle opacity:

```swift
extension View {

    func hidden(_ hide: Bool) -> some View {
        self.opacity(hide ? 0 : 1)
    }
}
```

This can feel unconventional, and you might worry about accessibility. However, per WWDC24 [Catch up on accessibility in SwiftUI](https://developer.apple.com/videos/play/wwdc2024/10073/?time=470), when a view is visually hidden (for example, when its opacity is zero), SwiftUI will automatically hide the element from accessibility technologies such as VoiceOver.

Therefore, this approach is safe for conditional hiding.
