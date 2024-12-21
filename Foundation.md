# Foundation

There are two(ish) `Foundation` implementations, and this is a major source of confusion for the
module's types and their capabilities.

<details><summary><em>The implementations, and the state of how they are layered, built, and distributed is a
work in progress.</em></summary>

<br/>Here are a few resources to help you understand the current state, and some of the history:

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
*  [Talk at SwiftServerConf](https://www.youtube.com/watch?v=EUKSZiOaWKk) describing the migration
   plan in a more visual way.

</details>

Currently, there's two distributions of Foundation, each with a set of documentation:
*  On Apple platforms, there is
   [`Foundation.framework`](https://developer.apple.com/documentation/foundation). This is built
   into the operating systems.
*  The open source [`Foundation`](https://swiftinit.org/docs/swift/foundation). This implementation
   is shipped with the open source Swift toolchains for Linux and Windows. This distribution, and
   its generated documentation, is composed from two separate open source projects:
   *  The "umbrella" framework,
      [swift-corelibs-foundation](https://github.com/swiftlang/swift-corelibs-foundation), which
      serves as a compatibility layer.
   *  The lower-level [swift-foundation](https://github.com/swiftlang/swift-foundation), which is
      intended to back most of the API in `Foundation` in the future. The
      [`FoundationEssentials`](https://swiftinit.org/docs/swift/foundationessentials) and
      [`FoundationInternationalization`](https://swiftinit.org/docs/swift/foundationinternationalization)
      modules in the open source distribution are built from here. _These same sources already back
      many `Foundation.framework` APIs in recent releases of the Apple operating systems. But that
      distribution also contains many closed source additions, so the authoritative documentation is
      still listed above._

There is a lot in common between these sets of docs, but there are also differences and omissions.
These differences can be surprising and hard to trace, especially when trying to produce functioning
cross-platform code that potentially depends on more than one implementation (based on the build).

## Differences

### URLSession

#### File system

*  **On Apple platforms**, `URLSession` can be used to read files using methods such as
[`data(from:delegate:)`](https://developer.apple.com/documentation/foundation/urlsession/3767353-data)
or
[`bytes(from:delegate)`](https://developer.apple.com/documentation/foundation/urlsession/3767351-bytes)
when the `URL` is a file URL (using the `file://` scheme).

*  **In the open source Foundation**, `URLSession` has these same methods but will throw an error
when using a file URL (using the `file://` scheme). The error description is:
`Domain=NSURLErrorDomain Code=-1002`. The only way to figure out what this error means is to read
the sources. The error is an
[`.unsupportedURL`](https://github.com/swiftlang/swift-corelibs-foundation/blob/25d044f2c4ceb635d9f714f588673fd7a29790c1/Sources/Foundation/NSError.swift#L748)
error. Unfortunately, this isn't descriptive enough to know that the `file://` scheme specifically
is unsupported. On deeper inspection of the sources, you'll find that there is no `URLProtocol`
implementation for file URLs [registered in the
`URLSession`](https://github.com/swiftlang/swift-corelibs-foundation/blob/25d044f2c4ceb635d9f714f588673fd7a29790c1/Sources/FoundationNetworking/URLSession/URLSession.swift#L197-L203)
class.

Note: On Apple platforms, `URLSession` is available by importing `Foundation`. In the open source
Foundation, it is available by importing
[`FoundationNetworking`](https://swiftinit.org/docs/swift/foundationinternationalization). **TODO**
Verify this. I believe you cannot import `FoundationNetworking` on Apple, and you won't be able to
use `URLSession` by importing `Foundation` on Linux. Once I have verified, include a snippet that
uses compiler checks to do the right import in both places.

Using the open source Foundation, there are a few alternatives for reading from the file system:
*  [`Data.init(contentsOf:options:)`](https://swiftinit.org/docs/swift/foundationessentials/data.init(contentsof:options:))
   with a file URL can synchronously read the entire contents of a file into memory. This is
   cross-platform compatible.
   *  For asynchronous usage, this call can done in a block on the global `DispatchQueue`. And in
      order to use Swift's `async`/`await`, that can be wrapped in `withUnsafeThrowingContinuation`.
*  `FileHandle.bytes` and its property `.lines`. **TODO:** Verify this works and then document it.
