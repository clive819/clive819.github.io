---
title: 'Lost Hours Integrating SwiftData'
author: 'Clive Liu'
layout: post
tags: [Swift, SwiftData, LostHours]
---

## Debug vs. Release: When Code Only Crashes in Release

```swift
extension ModelContext {
    func fetch<Model: PersistentModel & Identifiable<UUID>>(id: UUID) throws -> Model? {
        let predicate = #Predicate<Model> { model in
            model.id == id
        }
        return try self.fetch(FetchDescriptor(predicate: predicate)).first
    }
}
```

This helper runs perfectly in debug builds. However, in release builds, the fetch function crashes immediately.

### Crash Stack Trace

```
Exception Type:  EXC_BREAKPOINT (SIGTRAP)
Exception Codes: 0x0000000000000001, 0x0000000191fad8c0
Termination Reason: SIGNAL 5 Trace/BPT trap: 5
Terminating Process: exc handler [90190]

Triggered by Thread:  0

Thread 0 name:
Thread 0 Crashed:
0   libswiftCore.dylib            	0x0000000191fad8c0 _assertionFailure(_:_:file:line:flags:) + 172 (AssertCommon.swift:171)
1   SwiftData                     	0x00000001a914b3b4 static PersistentModel.graph_keyPathToString(keypath:) + 1452 (DataUtilities.swift:0)
2   SwiftData                     	0x00000001a917e420 Schema.KeyPathCache.validateAndCache(keypath:on:) + 3016 (Schema.swift:331)
3   SwiftData                     	0x00000001a909fce0 static PersistentModel.keyPathToString(keypath:) + 292 (DataUtilities.swift:36)
4   SwiftData                     	0x00000001a918585c closure #1 in static PersistentModel.fetchDescriptorKeyPathString(for:) + 44 (FetchDescriptor.swift:86)
5   SwiftData                     	0x00000001a90b052c static PersistentModel.fetchDescriptorKeyPathString(for:) + 76 (FetchDescriptor.swift:84)
6   SwiftData                     	0x00000001a918b2b8 closure #1 in closure #1 in PredicateExpressions.KeyPath.convert(state:) + 520 (FetchDescriptor.swift:521)
7   SwiftData                     	0x00000001a918a494 closure #1 in PredicateExpressions.KeyPath.convert(state:) + 400 (<compiler-generated>:0)
8   SwiftData                     	0x00000001a918e870 PredicateExpressions.KeyPath.convert(state:) + 88
9   SwiftData                     	0x00000001a90aff88 protocol witness for ConvertibleExpression.convert(state:) in conformance PredicateExpressions.KeyPath<A, B> + 36 (<compiler-generated>:0)
10  SwiftData                     	0x00000001a91875d8 closure #1 in PredicateExpression.convertToExpressionOrPredicate(state:) + 748 (FetchDescriptor.swift:265)
11  SwiftData                     	0x00000001a90afb08 PredicateExpression.convertToExpressionOrPredicate(state:) + 88 (FetchDescriptor.swift:261)
12  SwiftData                     	0x00000001a90afef8 PredicateExpression.convertToExpression(state:) + 24 (FetchDescriptor.swift:284)
13  SwiftData                     	0x00000001a918e0c0 closure #1 in PredicateExpressions.Equal.convert(state:) + 368
14  SwiftData                     	0x00000001a90afbc0 PredicateExpressions.Equal.convert(state:) + 116
15  SwiftData                     	0x00000001a90b00c8 protocol witness for ConvertibleExpression.convert(state:) in conformance PredicateExpressions.Equal<A, B> + 80 (<compiler-generated>:0)
16  SwiftData                     	0x00000001a91875d8 closure #1 in PredicateExpression.convertToExpressionOrPredicate(state:) + 748 (FetchDescriptor.swift:265)
17  SwiftData                     	0x00000001a90afb08 PredicateExpression.convertToExpressionOrPredicate(state:) + 88 (FetchDescriptor.swift:261)
18  SwiftData                     	0x00000001a90afc28 PredicateExpression.convertToPredicate(state:) + 28 (FetchDescriptor.swift:291)
19  SwiftData                     	0x00000001a9185e8c closure #1 in nsFetchRequest<A>(for:in:) + 1296 (FetchDescriptor.swift:101)
20  SwiftData                     	0x00000001a90af744 nsFetchRequest<A>(for:in:) + 88 (FetchDescriptor.swift:96)
21  SwiftData                     	0x00000001a90abee8 closure #1 in DefaultStore.fetch<A>(_:) + 232 (DefaultStore.swift:431)
22  SwiftData                     	0x00000001a90aec8c partial apply for closure #1 in DefaultStore.fetch<A>(_:) + 24 (<compiler-generated>:0)
23  SwiftData                     	0x00000001a910d560 closure #2 in closure #1 in DefaultStore.performAndWaitOnContext<A>(for:_:) + 276 (DefaultStore.swift:401)
24  SwiftData                     	0x00000001a91195c0 partial apply for closure #2 in closure #1 in DefaultStore.performAndWaitOnContext<A>(for:_:) + 32 (<compiler-generated>:0)
25  CoreData                      	0x000000019732d1f8 thunk for @callee_guaranteed () -> (@out A, @error @owned Error) + 28 (<compiler-generated>:0)
26  CoreData                      	0x000000019732d1d4 partial apply for thunk for @callee_guaranteed () -> (@out A, @error @owned Error) + 24 (<compiler-generated>:0)
27  CoreData                      	0x00000001973862a4 closure #1 in closure #1 in NSManagedObjectContext._rethrowsHelper_performAndWait<A>(fn:execute:rescue:) + 192 (NSManagedObjectContext.swift:143)
28  CoreData                      	0x0000000197383d08 thunk for @callee_guaranteed @Sendable () -> () + 28 (<compiler-generated>:0)
29  CoreData                      	0x0000000197383d30 thunk for @escaping @callee_guaranteed @Sendable () -> () + 28 (<compiler-generated>:0)
30  CoreData                      	0x0000000197308a4c developerSubmittedBlockToNSManagedObjectContextPerform + 228 (NSManagedObjectContext.m:3981)
31  libdispatch.dylib             	0x00000001cd2067ec _dispatch_client_callout + 16 (client_callout.mm:85)
32  libdispatch.dylib             	0x00000001cd1fc8c0 _dispatch_lane_barrier_sync_invoke_and_complete + 56 (queue.c:1128)
33  CoreData                      	0x0000000197432400 -[NSManagedObjectContext performBlockAndWait:] + 308 (NSManagedObjectContext.m:4105)
34  CoreData                      	0x000000019732cffc NSManagedObjectContext.performAndWait<A>(_:) + 544 (NSManagedObjectContext.swift:209)
35  SwiftData                     	0x00000001a910c700 closure #1 in DefaultStore.performAndWaitOnContext<A>(for:_:) + 496 (DefaultStore.swift:390)
36  SwiftData                     	0x00000001a90aec3c DefaultStore.performAndWaitOnContext<A>(for:_:) + 100 (DefaultStore.swift:382)
37  SwiftData                     	0x00000001a90ae990 DefaultStore.fetch<A>(_:) + 132 (DefaultStore.swift:430)
38  SwiftData                     	0x00000001a90ae9b8 protocol witness for DataStore.fetch<A>(_:) in conformance DefaultStore + 16 (<compiler-generated>:0)
39  SwiftData                     	0x00000001a90adc84 asDataStore #1 <A><A1>(_:) in closure #1 in ModelContext.fetch<A>(_:) + 2856 (ModelContext.swift:2840)
40  SwiftData                     	0x00000001a90afd54 partial apply for closure #1 in ModelContext.fetch<A>(_:) + 100 (<compiler-generated>:0)
41  SwiftData                     	0x00000001a90af0fc closure #1 in ModelContext.enumerateFetchableStores<A>(_:_:) + 208 (ModelContext.swift:2658)
42  SwiftData                     	0x00000001a90af3b0 specialized ModelContext.enumerateFetchableStores<A>(_:_:) + 236 (ModelContext.swift:2653)
43  SwiftData                     	0x00000001a90aef1c ModelContext.fetch<A>(_:) + 144 (ModelContext.swift:2786)
44  SwiftData                     	0x00000001a90af024 dispatch thunk of ModelContext.fetch<A>(_:) + 56
```

### What’s Happening?

The crash arises from using a generic extension to perform a fetch with a predicate involving the model’s ID. While this works in debug mode, the release compiler’s optimizations likely break some runtime type information that SwiftData expects—triggering a fatal assertion in the data model layer.

### Solution

Avoid extending ModelContext with convenience APIs using generic predicates. Instead, manually create the fetch descriptor at call site.

## Relationship Issues: Crashes on iOS 17 but Not Newer Versions

```swift
@Model final class MyModel: Identifiable {
    @Attribute(.unique) private(set) var id: UUID
    private(set) var array: [MyOtherModel]
}

@Model final class MyOtherModel: Identifiable {
    @Attribute(.unique) private(set) var id: UUID
    @Relationship(inverse: nil) private(set) var source: MyModel?
}
```

Defining a one-way relationship with `@Relationship(inverse: nil)` works on iOS 18 and newer, but will crash immediately on iOS 17.

### Crash Stack

```
Exception Type:  EXC_BREAKPOINT (SIGTRAP)
Exception Codes: 0x0000000000000001, 0x000000018acbd8d8
Termination Reason: SIGNAL 5 Trace/BPT trap: 5
Terminating Process: exc handler [4989]

Triggered by Thread:  0

Thread 0 name:
Thread 0 Crashed:
0   libswiftCore.dylib            	0x000000018acbd8d8 _assertionFailure(_:_:file:line:flags:) + 264 (AssertCommon.swift:144)
1   SwiftData                     	0x000000022dc2c1c4 setInverse #1 <A>(_:) in Schema.init(_:version:) + 464 (Schema.swift:0)
2   SwiftData                     	0x000000022dc34450 specialized Schema.init(_:version:) + 1480 (Schema.swift:378)
3   SwiftData                     	0x000000022dbe1a3c ModelContainer.__allocating_init(for:migrationPlan:configurations:) + 96 (ModelContainer.swift:36)
```

### Solution

Fortunately, I don't really need to reference `MyModel` inside `MyOtherModel`, simply removing that property resolves the issue.
