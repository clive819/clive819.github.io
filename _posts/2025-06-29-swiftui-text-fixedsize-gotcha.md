---
title: 'SwiftUI Text `fixedSize` Gotcha'
author: 'Clive Liu'
layout: post
tags: [Swift, SwiftUI]
---

The `fixedSize` modifier in SwiftUI forces a view to adopt its ideal size, preventing it from being compressed or truncated.

For example, you can use it to make sure a long text isn’t cut off:

```swift
Text(aVeryLongString)
    .fixedSize(horizontal: false, vertical: true)
```

However, there’s a subtle gotcha: the `lineLimit` modifier takes precedence over `fixedSize`. If a `lineLimit` is applied anywhere in the view hierarchy and affects the text view, it will still truncate the text — even if `fixedSize` is used.

To avoid this, make sure no unintended `lineLimit` is affecting your text. A simple fix is to explicitly set `.lineLimit(nil)` on the `Text` view alongside `fixedSize`:

```swift
Text(aVeryLongString)
    .lineLimit(nil)
    .fixedSize(horizontal: false, vertical: true)
```

This ensures the text can grow vertically to fit its content as expected.
