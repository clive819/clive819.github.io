---
title: 'How to protect your app from being reverse engineered?'
author: 'Clive Liu'
layout: post
tags: [Reverse Engineering, Swift]
---

Continuing from the previous article [Reverse engineering macOS app 101](../reverse-engineering-macos-app-101/), you might wonder: is there a way to prevent your application binary from being tampered with?

## Short answer

You can't.

Any logic that lacks server-side validation is vulnerable to being cracked.

## Can we do something at least?

There are indeed techniques that can make reverse engineering more difficult. For example, code obfuscation and function inlining are commonly used methods to increase the complexity of reverse engineering.

### Code obfuscation

Code obfuscation is the process of transforming source code or compiled code into a version that is difficult for humans (and sometimes automated tools) to understand, while maintaining its functionality. For example, this is your original Swift code:

```swift
struct User {
    let name: String
    func hasAccessToPremiumContent() -> Bool {
        return resultOfRESTAPICall()
    }
}
```

Using a tool like `SwiftShield`, a code obfuscator for Swift, the code can be transformed into something like this:

```swift
struct fjiovh4894bvic {
    let VNfhnfn3219d: String
    func cxncjnx8fh83FDJSDd() -> Bool {
        return vPAOSNdcbif372hFKF()
    }
}
```

The obfuscated code is significantly harder to read, increasing the difficulty for an attacker to locate and understand critical logic.

### Function inlining

Function inlining is another commonly used technique. It involves replacing a function call with the actual function body during compilation. For example, consider this code:

```swift
@inline(__always) func hasAccessToPremiumContent() -> Bool {
    // some complicated validation logic
}

func premiumContent() -> PremiumContent? {
    guard hasAccessToPremiumContent() else { return nil }
    ...
}

func anotherPremiumContent() -> PremiumContent? {
    guard hasAccessToPremiumContent() else { return nil }
    ...
}
```

The compiler will transform the code into something like this:

```swift
func premiumContent() -> PremiumContent? {
    guard <some complicated validation logic> else { return nil }
    ...
}

func anotherPremiumContent() -> PremiumContent? {
    guard <some complicated validation logic> else { return nil }
    ...
}
```

This makes it harder for attackers to tamper with the validation logic by simply modifying the `hasAccessToPremiumContent` function. Instead, they would need to locate and modify the validation logic at every point it is inlined.

> Please note that even with the `@inline(__always)` attribute, inlining is not guaranteed. The compiler ultimately decides whether to inline a function based on the context and optimization settings. By using this attribute, we are simply suggesting to the compiler that the function should be inlined whenever possible.
{: .prompt-info }

## Conclusion

There are other techniques I didn’t mention, but they all serve the same purpose: to increase the difficulty of reverse engineering. However, no matter how many layers of protection you add, a skilled attacker can still crack your app.

That’s why you should never place critical code or sensitive logic in your app’s client-side implementation. Instead, offload such logic to your server and always validate inputs from the app on the server side. This cannot be emphasized enough.

You’d be surprised how often even backend engineers—sometimes at large tech companies—overlook this fundamental principle and rely on app engineers to implement validation checks directly in the app. This approach is inherently insecure and should always be avoided.
