# Foundation

There's more than one `Foundation` implementation, and this is a major source of confusion for the
module's types and their capabilities.

<details><summary>The various implementations, and the state of how they are layered, built, and distributed is a
work in progress.</summary>

Here are a few resources to help you understand the current state, and some of the history, of these details:
*  [2024 Swift-Foundation annual
   update](https://forums.swift.org/t/swift-foundation-2024-annual-update/75609) summarizes current
   state, and outlines focus areas for the upcoming year.
*  Swift 6.0: [swift-foundation implementation
   ships](https://forums.swift.org/t/swift-foundation-now-available/73530) as part of the
   implementation of `Foundation` in the toolchain on Linux and Windows.
*  [Initial migration plan](https://forums.swift.org/t/swift-foundation-2024-annual-update/75609)
   for swift-corelibs-foundation to transition into a compatibility layer, and push the unified
   cross-platform implementation into swift-foundation. There was a slight pivot to shipping
   swift-foundation in the toolchain distributions instead of using SwiftPM, described in [an
   update](https://forums.swift.org/t/migration-plan-swift-corelibs-foundation-to-swift-foundation/70943/23).
*  [Talk at SwiftServerConf](https://www.youtube.com/watch?v=EUKSZiOaWKk) describing the migration plan in a more visual way.

</details>

Currently, there's two prominent sets of documentation for Foundation:
*  Apple's documentation for [`Foundation.framework`](https://developer.apple.com/documentation/foundation). This is built into the operating system on Apple platforms.
*  swiftinit's documentation for the open source [`Foundation`](https://swiftinit.org/docs/swift/foundation), which
   is generated from the [swift-corelibs-foundation](https://github.com/swiftlang/swift-corelibs-foundation) source. This is shipped with the open source Swift toolchain for Linux and Windows.
   *  The `FoundationEssentials` and `FoundationInternationalization` modules, also documented here,
      are generated from the [swift-foundation](https://github.com/swiftlang/swift-foundation)
      sources.

There is a lot in common between these sets of docs, but there are also differences and omissions.
These differences can be surprising and hard to trace, especially when trying to produce functioning
cross-platform code that depends on more than once implementation (based on the build).

## Differences

### URLSession

#### File system

On Apple platforms, `URLSession` can be used to read files using methods such as
[`data(from:delegate:)`](https://developer.apple.com/documentation/foundation/urlsession/3767353-data)
or
[`bytes(from:delegate)`](https://developer.apple.com/documentation/foundation/urlsession/3767351-bytes)
when the `URL` is a file URL (using the `file://` scheme).

In the open source Foundation, `URLSession` has these same methods but will throw an error when
using a file URL (using the `file://` scheme). The error description is: `Domain=NSURLErrorDomain
Code=-1002`. The only way to figure out what this error means was to read the sources. This is an
[`.unsupportedURL`](https://github.com/swiftlang/swift-corelibs-foundation/blob/25d044f2c4ceb635d9f714f588673fd7a29790c1/Sources/Foundation/NSError.swift#L748)
error. Unfortunately, this isn't descriptive enough to know that the `file://` scheme specifically
is unsupported. On deeper inspection of the sources, you'll find that there is no `URLProtocol`
implementation for file URLs [registered in the
`URLSession`](https://github.com/swiftlang/swift-corelibs-foundation/blob/25d044f2c4ceb635d9f714f588673fd7a29790c1/Sources/FoundationNetworking/URLSession/URLSession.swift#L197-L203)
class.

Note: The open source Foundation implementation of `URLSession` is imported using the `import
FoundationNetworking` statement, instead of `import Foundation`. It is currently implemented in
`swift-corelibs-foundation`, not in `swift-foundation`.

When using the open source Foundation, there are a few alternatives for reading from the file
system.
*  `Data(contentsOf:)` with a file URL can synchronously read the entire contents of a file into memory. This is cross-platform compatible.
   *  For asynchronous usage, this call can done in a block on the global `DispatchQueue`. And in order to use Swift's `async`/`await`, that can be wrapped in `withUnsafeThrowingContinuation`.
*  `FileHandle.bytes` and its property `.lines`. **TODO:** Verify this works and then document it.
