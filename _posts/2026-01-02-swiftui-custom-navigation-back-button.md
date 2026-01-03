---
title: 'SwiftUI Custom Navigation Back Button'
author: 'Clive Liu'
layout: post
tags: [Swift, SwiftUI]
---

SwiftUI's default `NavigationStack` and `NavigationView` provide a system back button, but customizing its appearance requires replacing it entirely. Unlike UIKit's `navigationItem.leftBarButtonItem`, there's no direct API to style the default back button. This guide shows how to implement a custom back button with full control over appearance and behavior while preserving the native swipe-to-dismiss gesture users expect.

## Solution: Custom Back Button with Toolbar

The cleanest approach is to replace the default back button with a custom `ToolbarItem` using the `.topBarLeading` placement:

```swift
.toolbar {
    ToolbarItem(placement: .topBarLeading) {
        Button {
            // dismiss logic
        } label: {
            // custom icon
        }
    }
}
.navigationBarBackButtonHidden()
```

## Preserving the Swipe-to-Dismiss Gesture

When you hide the navigation bar back button with `.navigationBarBackButtonHidden()`, SwiftUI also disables the built-in swipe-to-pop gesture (swiping from the left edge). This gesture is valuable for accessibility and user experience, so you'll want to restore it:

```swift
extension UINavigationController: @retroactive UIGestureRecognizerDelegate {

    override open func viewDidLoad() {
        super.viewDidLoad()
        self.interactivePopGestureRecognizer?.delegate = self
        self.interactiveContentPopGestureRecognizer?.delegate = self
    }

    public func gestureRecognizerShouldBegin(_: UIGestureRecognizer) -> Bool {
        // Prevent gesture on root view to avoid crashes/freezes
        // The gesture should only work when there are views to pop
        return self.viewControllers.count > 1
    }
}
```

## Important Notes

- **Add the UINavigationController extension once** - It only needs to exist in your project, not in every view
- **Swipe gesture won't trigger custom back button logic** - The swipe gesture directly pops the view controller. If you need custom logic on all navigations (including swipe), you can simply add `.onDisappear { // custom logic }` to your SwiftUI view to execute it.
