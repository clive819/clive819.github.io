---
title: 'Understanding Swift’s OptionSet: A Swifty Way to Handle Bitmask Flags'
author: 'Clive Liu'
layout: post
tags: [Swift]
---

When writing Swift code, you may encounter situations where you need to represent a combination of options in a clean, efficient, and type-safe manner. This is where Swift’s `OptionSet` comes into play. It’s a powerful feature that enables you to define sets of options using bitmasking, while offering a Swift-friendly and expressive API.

`OptionSet` is extensively used throughout Apple’s SDKs. For example, consider `JSONEncoder.OutputFormatting`:

```swift
let jsonEncoder = JSONEncoder()
jsonEncoder.outputFormatting = .prettyPrinted
jsonEncoder.outputFormatting = [.prettyPrinted, .sortedKeys]
```

Or `.ignoresSafeArea` in SwiftUI:

```swift
MyView()
    .ignoresSafeArea(.all, edges: .all)
    .ignoresSafeArea([.container, .keyboard], edges: [.horizontal, .top])
```

These examples showcase how `OptionSet` allows you to combine multiple options in a concise and readable way.  

While `OptionSet` might resemble enums at first glance, it’s designed to work as a _set_—meaning you can use multiple values simultaneously. This makes it ideal for representing flags or configuration options.


## Defining Your Own OptionSet

To create a custom `OptionSet`, define a struct that conforms to the `OptionSet` protocol. Each case is a static constant with a unique raw value, typically a power of two. This ensures that each option can be represented by a single bit in the underlying integer. The `rawValue` must conform to `FixedWidthInteger` (e.g. `Int` or `UInt`), allowing bitwise operations under the hood.

```swift
struct Permissions: OptionSet {

    let rawValue: Int

    static let read  = Permissions(rawValue: 1 << 0)
    static let write = Permissions(rawValue: 1 << 1)

    static let all: Self = [.read, .write]
}
```

## Using OptionSet

### Combining Options

You can combine options using array-like syntax or set operations:

```swift
let readWrite: Permissions = [.read, .write]
// or
let readWriteAlternative = Permissions.read.union(.write)
```

### Checking for an Option

Use the contains method to check if a particular option is set:

```swift
if readWrite.contains(.read) {
    print("Read permission is granted.")
}
```

### Removing an Option

You can remove an option using subtracting:

```swift
let updatedPermissions = readWrite.subtracting(.write)
```

## Why Use OptionSet?

- **Efficiency**: Bitmasking is fast and memory-efficient. 
- **Type Safety**: `OptionSet` ensures only valid combinations are used. 
- **Readability**: Code is clearer and less error-prone than raw bitwise logic. 
- **Composability**: Options can be combined, checked, and manipulated with expressive syntax.

## Summary

Swift’s `OptionSet` protocol offers a clean and type-safe way to work with sets of configurable options—especially when multiple options can be active at once. It brings clarity, safety, and performance to bitmasking, making it a Swifty solution to a classic problem.
