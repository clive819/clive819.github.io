---
title: 'Lost Hours Integrating Firebase Realtime Database'
author: 'Clive Liu'
layout: post
tags: [Swift, Python, Firebase, LostHours]
---

Integrating Firebase Realtime Database with Swift’s `Codable` protocol should be straightforward, but a few non-obvious behaviors can lead to hours of frustrating debugging. Here are three key pitfalls to watch out for.

### Dictionary Keys Must Be Strings

- [FirebaseDataEncoder.swift#L1290](https://github.com/firebase/firebase-ios-sdk/blob/ea3f1293fbb32ec41ee2abd6ad363c1dffa6d23d/FirebaseSharedSwift/Sources/third_party/FirebaseDataEncoder/FirebaseDataEncoder.swift#L1290)

Firebase Realtime Database requires that all dictionary keys be `String` types. If you attempt to encode a model with a dictionary using a non-string key, like `Int`, the operation will fail.

For example, this `Codable` struct will cause an encoding error:

```swift
// This will fail to encode.
struct MyModel: Codable {
    let foo: [Int: String]
}
```

The seemingly simple fix is to convert the `Int` keys to `String`s. However, this introduces a subtle and confusing decoding issue.

```swift
// This will encode, but may decode incorrectly.
struct MyModel: Codable {
    let foo: [String: String]
}
```

The problem lies in how Firebase handles arrays. It stores them as dictionaries with integer keys represented as strings (e.g., "0", "1", "2"). Because of this, when the SDK's decoder encounters a dictionary with sequential, string-based integer keys, it interprets the data as an `Array`, not a `Dictionary`. This can cause your `[String: String]` property to be unexpectedly decoded as an `[String]`. 

- [FChildrenNode.m#L202-L215](https://github.com/firebase/firebase-ios-sdk/blob/ea3f1293fbb32ec41ee2abd6ad363c1dffa6d23d/FirebaseDatabase/Sources/Snapshot/FChildrenNode.m#L202-L215)

### Empty Collections Are Omitted

Another quirk is that Firebase does not store empty arrays or dictionaries. When you save a model, any properties that are empty collections are simply omitted from the database. Consequently, when you fetch the data, those fields will be missing, causing the default `Codable` decoding to fail.

Consider this model:

```swift
struct MyModel: Codable {
    let bar: [String]
    let foo: [String: String]
}
```

If you save an instance where `bar` or `foo` are empty, they won't exist in the fetched snapshot. To prevent decoding from failing, you must implement a custom `init(from:)` and use `decodeIfPresent`, providing a default empty value for any missing collections.

```swift
struct MyModel: Codable {
    let bar: [String]
    let foo: [String: String]

    // Manually handle potentially omitted empty collections.
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.bar = try container.decodeIfPresent([String].self, forKey: .bar) ?? []
        self.foo = try container.decodeIfPresent([String: String].self, forKey: .foo) ?? [:]
    }
}
```

### Integer Serialization from Python

If you're writing data to the Realtime Database from a backend service, such as a Python Cloud Function, be aware of how integers are serialized. A standard `Int` in a Swift `Codable` struct expects a simple numeric value.

Your Swift model might expect this JSON structure:
```json
{
    "foo": 123
}
```

However, some Python admin SDK will instead store the integer as an object, including type metadata. This results in an incompatible structure that your Swift decoder cannot parse.

```json
{
    "foo": {
        "@type": "type.googleapis.com/google.protobuf.Int64Value",
        "value": "123"
    }
}
```

This mismatch will cause decoding to fail, requiring you to either adjust your backend serialization logic or write a custom decoder on the client to handle this specific object format. 🙄🙄🙄
