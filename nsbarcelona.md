# Swift Package Manager

## NSBarcelona, April 2016

### Boris Bügling - @NeoNacho

![20%, original, inline](images/contentful.png)

![](images/package.gif)

<!--- use Next theme, white -->

---

## CocoaPods

![original](images/cocoapods.jpg)

---

## Contentful

![original](images/contentful-bg.png)

---

# Agenda

- What is swiftpm?
- Making our own package
- How does it work?
- Comparison with related tools

---

# What is swiftpm?

---

## Development Snapshot March 24, 2016

```swift
$ swift --version
Apple Swift version 3.0-dev (LLVM b010debd0e, Clang 3e4d01d89b, Swift 7182c58cb2)
```

---

```swift
import PackageDescription

let package = Package(
    name: "Hello",
    dependencies: [
        .Package(url: "ssh://git@example.com/Greeter.git", 
          versions: Version(1,0,0)..<Version(2,0,0)),
    ]
)
```

---

```bash
$ swift build
Compiling Swift Module 'Clock' (2 sources)
Linking Library:  .build/debug/Clock.a
```

---

# What does it do?

- Compiles and links Swift packages
- Resolves, fetches and builds their dependencies
- Runs tests

---

# Current state

- Currently builds dynamic/static libraries or binaries
- Supported platforms are OS X and Ubuntu Linux
- Only builds Swift code, no C/C++/Objective-C/...
(but there's an accepted proposal [SE-0038](https://github.com/apple/swift-evolution/blob/master/proposals/0038-swiftpm-c-language-targets.md))

---

## swift build

```bash
OVERVIEW: Build sources into binary products

USAGE: swift build [options]

MODES:
  --configuration <value>        Build with configuration (debug|release) [-c]
  --clean[=<mode>]               Delete artefacts (build|dist) [-k]
  --init <mode>                  Creates a new Swift package (executable|library)
  --fetch                        Fetch package dependencies
  --generate-xcodeproj [<path>]  Generates an Xcode project for this package [-X]

OPTIONS:
  --chdir <value>    Change working directory before any other operation [-C]
  -v[v]              Increase verbosity of informational output
  -Xcc <flag>        Pass flag through to all C compiler instantiations
  -Xlinker <flag>    Pass flag through to all linker instantiations
  -Xswiftc <flag>    Pass flag through to all Swift compiler instantiations
```

---

## swift test

```bash
OVERVIEW: Build and run tests

USAGE: swift test [options]

OPTIONS:
  TestModule.TestCase         Run a test case subclass
  TestModule.TestCase/test1   Run a specific test method
```

---

![250%](images/gh-repo.png)

---

# Making our own package

---

# 🕕

A small library for parsing and writing ISO8601 date strings.

<https://github.com/neonichu/Clock/tree/swift-3.0>

---

```
Sources
└── Clock
    ├── ISO8601Parser.swift
    └── ISO8601Writer.swift

1 directory, 2 files
```

---

```bash
$ touch Package.swift
$ swift build
```

---

## Tests?

---

```
Tests
├── Clock
│   ├── Localtime.swift
│   ├── String.swift
│   ├── TMStruct.swift
│   └── UTC.swift
└── LinuxMain.swift

1 directory, 5 files
```

---

## XCTest

```swift
import XCTest
@testable import Clock

class DateToStringConversionTests: XCTestCase {
    func testConvertTMStructToAnISO8601GMTString() {
        let actual = tm_struct(year: 1971, month: 2, day: 3, hour: 9, minute: 16, second: 6)

        XCTAssertEqual(actual.toISO8601GMTString(), "1971-02-03T09:16:06Z")
    }
}
```

---

## Linux

```swift
#if os(Linux)
extension DateToStringConversionTests {
    static var allTests : [(String, 
      DateToStringConversionTests -> () throws -> Void)] {
        return [
            ("testConvertTMStructToAnISO8601GMTString", 
             testConvertTMStructToAnISO8601GMTString)
        ]
    }
}
#endif
```

---

## LinuxMain.swift

```swift
import XCTest
@testable import ClockTestSuite

XCTMain([
  testCase(TMStructTests.allTests),
  testCase(DateToStringConversionTests.allTests),
])
```

---

## Side-note about Linux

---

Foundation is incomplete and sometimes different from OS X:

```swift
#if os(Linux)
      let index = p.startIndex.distanceTo(p.startIndex.successor())
      path = NSString(string: p).substringFromIndex(index)
#else
      path = p.substringFromIndex(p.startIndex.successor())
#endif
```

---

Some things in the standard library might not be available:

```objectivec
#if _runtime(_ObjC)
// Excluded due to use of dynamic casting and Builtin.autorelease, neither
// of which correctly work without the ObjC Runtime right now.
// See rdar://problem/18801510
[...]
public func getVaList(args: [CVarArgType]) -> CVaListPointer {
```

---

OS X libc and Glibc can differ:

```swift
let flags = GLOB_TILDE | GLOB_BRACE | GLOB_MARK
    if system_glob(cPattern, flags, nil, &gt) == 0 {
#if os(Linux)
      let matchc = gt.gl_pathc
#else
      let matchc = gt.gl_matchc
#endif
```

---

And other random fun:

```
./.build/debug/spectre-build
/usr/bin/ld: .build/debug/Clock.a(ISO8601Parser.swift.o): 
undefined reference to symbol '_swift_FORCE_LOAD_$_swiftGlibc'
/home/travis/.swiftenv/versions/swift-2.2-SNAPSHOT-2015-12-22-a/
usr/lib/swift/linux/libswiftGlibc.so: error adding symbols: DSO 
missing from command line
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

---

## Running the tests

```bash
$ swift test
Test Suite 'All tests' started at 2016-04-06 17:06:00.900
Test Suite 'Package.xctest' started at 2016-04-06 17:06:00.901
Test Suite 'DateToStringConversionTests' started
  at 2016-04-06 17:06:00.901
Test Case '-[ClockTestSuite.DateToStringConversionTests 
  testCanConvertNSDateToAnISO8601GMString]' started.
[...]
Test Suite 'All tests' passed at 2016-04-06 17:06:00.911.
Executed 12 tests, with 0 failures (0 unexpected)
  in 0.008 (0.011) seconds
```

---

## Swift versions

```bash
$ cat .swift-version 
swift-2.2-SNAPSHOT-2015-12-22-a
```

- Is read by either `chswift` or `swiftenv`

---

## Travis CI

```haml
os:
- linux
- osx
language: generic
sudo: required
dist: trusty
osx_image: xcode7.2
install:
- curl -sL https://gist.github.com/kylef/
5c0475ff02b7c7671d2a/raw/
621ef9b29bbb852fdfd2e10ed147b321d792c1e4/swiftenv-install.sh | bash
script:
- . ~/.swiftenv/init
```

---

# How does it work?

---

## Modules inside SwiftPackageManager

```swift
Target(
    /** “Swifty” POSIX functions from libc */
    name: "POSIX",
    dependencies: ["libc"]),
Target(
    /** Abstractions for common operations */
    name: "Utility",
    dependencies: ["POSIX"]),
Target(
    /** Base types for the package-engine */
    name: "PackageType",
    dependencies: ["PackageDescription", "Utility"]),
Target(
    name: "ManifestParser",
    dependencies: ["PackageDescription", "PackageType"]),
Target(
    /** Turns Packages into Modules & Products */
    name: "Transmute",
    dependencies: ["PackageDescription", "PackageType"]),
Target(
    /** Fetches Packages and their dependencies */
    name: "Get",
    dependencies: ["PackageDescription", "PackageType"]),
```

---

## Modules inside SwiftPackageManager

```swift
Target(
    /** Builds Modules and Products */
    name: "Build",
    dependencies: ["PackageType"]),
Target(
    /** Common components of both executables */
    name: "Multitool",
    dependencies: ["PackageType"]),
Target(
    /** Generates Xcode projects */
    name: "Xcodeproj",
    dependencies: ["PackageType"]),
Target(
    /** The main executable provided by SwiftPM */
    name: "swift-build",
    dependencies: ["ManifestParser", "Get", "Transmute", "Build", "Multitool", "Xcodeproj"]),
Target(
    /** Runs package tests */
    name: "swift-test",
    dependencies: ["Multitool"]),
```

---

# Dependencies

- Can be local or remote Git repositories
- Need to be tagged
- Will be fetched to `./Packages/MyPackage-0.0.1`

---

# Git tagging

- `Package.swift` only supports tagged dependencies
- Don't forget to push your tags to GitHub

---

# Rough build process

- `PackageDescription` generates TOML
- `Transmute` parses the TOML and can generate YAML
- Dependencies are fetched by `Get`
- YAML is used as input to `llbuild`
- `Build` calls out to `llbuild`

---

# llbuild

---

```ruby
$ cat .build/debug.yaml
client:
  name: swift-build
tools: {}
targets:
  test: [<Clock.testsuite.module>, <Package.test>]
  default: [<Clock.module>]
```

---

```ruby
commands: 
  .../.build/debug/Clock.build:
    tool: mkdir
    outputs: [.../.build/debug/Clock.build]

  <Clock.module>:
    tool: swift-compiler
    executable: .../bin/swiftc
    module-name: Clock
    module-output-path: .../.build/debug/Clock.swiftmodule
    inputs: []
    outputs: [<Clock.module>, .../.build/debug/Clock.build/ISO8601Parser.swift.o,
      .../.build/debug/Clock.build/ISO8601Writer.swift.o]
    import-paths: .../.build/debug
    temps-path: .../.build/debug/Clock.build
    objects: [.../.build/debug/Clock.build/ISO8601Parser.swift.o,
      .../.build/debug/Clock.build/ISO8601Writer.swift.o]
    other-args: ["-j8", "-Onone", "-g", 
      "-D", SWIFT_PACKAGE, "-enable-testing", "-F", ".../Developer/Library/Frameworks",
      "-target", "x86_64-apple-macosx10.10", "-sdk", ".../Developer/SDKs/MacOSX10.11.sdk"]
    sources: [.../Sources/Clock/ISO8601Parser.swift, .../Sources/Clock/ISO8601Writer.swift]
    is-library: true
```

---

```ruby
  .../.build/debug/ClockTestSuite.build:
    tool: mkdir
    outputs: [.../.build/debug/ClockTestSuite.build]

  <Clock.testsuite.module>:
    tool: swift-compiler
    executable: /Library/Developer/Toolchains/swift-DEVELOPMENT-SNAPSHOT-2016-03-24-a.xctoolchain/usr/bin/swiftc
    module-name: ClockTestSuite
    module-output-path: .../.build/debug/ClockTestSuite.swiftmodule
    inputs: [<Clock.module>]
    outputs: [<Clock.testsuite.module>, 
      .../.build/debug/ClockTestSuite.build/Localtime.swift.o,
      .../.build/debug/ClockTestSuite.build/String.swift.o,
      .../.build/debug/ClockTestSuite.build/TMStruct.swift.o,
      .../.build/debug/ClockTestSuite.build/UTC.swift.o]
    import-paths: .../.build/debug
    temps-path: .../.build/debug/ClockTestSuite.build
    objects: [.../.build/debug/ClockTestSuite.build/Localtime.swift.o,
      .../.build/debug/ClockTestSuite.build/String.swift.o,
      .../.build/debug/ClockTestSuite.build/TMStruct.swift.o,
      .../.build/debug/ClockTestSuite.build/UTC.swift.o]
    other-args: ["-j8", "-Onone", "-g",
      "-D", SWIFT_PACKAGE, "-enable-testing", "-F", ".../Developer/Library/Frameworks",
      "-target", "x86_64-apple-macosx10.10", "-sdk", ".../Developer/SDKs/MacOSX10.11.sdk"]
    sources: [.../Tests/Clock/Localtime.swift, .../Tests/Clock/String.swift,
      .../Tests/Clock/TMStruct.swift, .../Tests/Clock/UTC.swift]
    is-library: true
```

---

```ruby
  <Package.test>:
    tool: shell
    description: Linking .build/debug/Package.xctest/Contents/MacOS/Package
    inputs: [<Clock.module>, <Clock.testsuite.module>,
      .../.build/debug/ClockTestSuite.build/Localtime.swift.o,
      .../.build/debug/ClockTestSuite.build/String.swift.o,
      .../.build/debug/ClockTestSuite.build/TMStruct.swift.o,
      .../.build/debug/ClockTestSuite.build/UTC.swift.o]
    outputs: [<Package.test>, .../.build/debug/Package.xctest/Contents/MacOS/Package]
    args: [".../bin/swiftc", "-target", "x86_64-apple-macosx10.10",
      "-sdk", ".../Developer/SDKs/MacOSX10.11.sdk", "-g", "-L.../.build/debug",
      "-o", .../.build/debug/Package.xctest/Contents/MacOS/Package,
      "-Xlinker", "-bundle", "-F", ".../Developer/Library/Frameworks",
      .../.build/debug/ClockTestSuite.build/Localtime.swift.o,
      .../.build/debug/ClockTestSuite.build/String.swift.o,
      .../.build/debug/ClockTestSuite.build/TMStruct.swift.o,
      .../.build/debug/ClockTestSuite.build/UTC.swift.o,
      .../.build/debug/Clock.build/ISO8601Parser.swift.o, 
      .../.build/debug/Clock.build/ISO8601Writer.swift.o]
```

---

# Additional Package.swift syntax

---

# Targets

```swift
import PackageDescription

let package = Package(
    name: "Example",
    targets: [
        Target(
            name: "top",
            dependencies: [.Target(name: "bottom")]),
        Target(
            name: "bottom")
    ]
)
```

---

# Exclusion

```swift
let package = Package(
    name: "Example",
    exclude: ["tools", "docs", "Sources/libA/images"]
)
```

---

# Test dependencies

```swift
import PackageDescription

let package = Package(
    name: "Hello",
    testDependencies: [
        .Package(url: "ssh://git@example.com/Tester.git",
          versions: Version(1,0,0)..<Version(2,0,0)),
    ]
)
```

---

# Customizing builds

```swift
import PackageDescription

var package = Package()

#if os(Linux)
let target = Target(name: "LinuxSources/foo")
package.targets.append(target)
#endif
```

---

# Integrate system libraries

- Empty `Package.swift`
- `module.modulemap`:

```c
module curl [system] {
    header "/usr/include/curl/curl.h"
    link "curl"
    export *
}
```

---

```swift
let package = Package(
    name: "example",
    dependencies: [
        .Package(url: "https://github.com/neonichu/curl",
          majorVersion: 1)
    ]
)
```

---

# Comparison with related tools

---

# CocoaPods

- Centralized discovery
- Xcode integration
- Support for other languages
- Additional metadata in the Manifest
- Does not cover the build process

---

# Carthage

- No manifest for packages
- Xcode integration
- Support for other languages

---

# Support all three

- `Package.swift`
- `.podspec`
- `.xcodeproj` / `.xcworkspace`

---

# 😢

---

# CocoaPods

- `chocolat-cli` converts `Package.swift` to a JSON Podspec

---

```swift
public func parse_package(packagePath: String) throws -> PackageDescription.Package {
  // FIXME: We depend on `chswift` installation and use here
  let toolchainPath = PathKit.Path(POSIX.getenv("CHSWIFT_TOOLCHAIN") ?? "")
  libc.setenv("SPM_INSTALL_PATH", toolchainPath.parent().description, 1)
  print_if("Using libPath \(Resources.runtimeLibPath)", false)

  let package = (try Manifest(path: packagePath)).package
  print_if("Converting package \(package.name) at \(packagePath)", false)

  return package
}
```

---

> You should think of it as an alpha code base that hasn't had a release yet. Yes, it is useful for doing some things [...]
-- Daniel Dunbar

---

## References

- <https://swift.org>
- <https://github.com/apple/swift-package-manager>
- <https://github.com/apple/llbuild>
- <https://github.com/neonichu/chocolat>
- <https://github.com/neonichu/chswift>
- <https://github.com/neonichu/freedom>
- <https://github.com/kylef/swiftenv>

---

# Thank you!

![](images/thanks.gif)

---

@NeoNacho

boris@contentful.com

http://buegling.com/talks

![](images/contentful-bg.png)
