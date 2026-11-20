# What is considered a security issue in the Swift project?

We define "security-sensitive" to mean that a discovered bug or vulnerability warrants coordinated, private disclosure — that is, it should be reported through a repository's GitHub private vulnerability reporting feature rather than filed as a public issue, and will be embargoed until a fix is generally available.

**Guiding principle.** Swift aims to guarantee two things:

1. **Safe Swift is memory safe.** A program written in the safe subset of Swift cannot do any of the following:
   * Corrupt memory.
   * Violate type safety.
   * Violate the isolation guarantees of the concurrency model.

   "Safe subset" means the program uses no `unsafe`, `@unchecked`, or `Unsafe*` constructs, and runs on the Swift runtime.
2. **SwiftPM enforces the protections it offers.**
   * Where SwiftPM applies a sandbox, code that escapes the sandbox is security-sensitive.
   * If SwiftPM accepts dependency content that failed an integrity check, that is security-sensitive.
   * If SwiftPM skips an integrity check it was configured to perform, that is security-sensitive.

A bug is security-sensitive when a user who has done nothing unsafe loses one of these guarantees. "Nothing unsafe" means:
* No unsafe code.
* No deliberate compilation of untrusted source.
* No bypassed integrity check.

The Swift project spans a compiler, a language runtime, a standard library, package tooling, and IDE/editor infrastructure, each with its own threat model and its own untrusted-input surface. Not everything that could theoretically be misused is a security issue. This section scopes that out per component so that reporters and maintainers agree on what belongs in a private report versus a public issue.

### Security-sensitive

* The Swift runtime, for programs built from valid, type-checked Swift source that contains no use of `unsafe`/`@unchecked`/`Unsafe*` APIs: metadata handling, existential containers, witness tables, dynamic casting (`as?`/`as!`), and reference counting/exclusivity enforcement. Memory corruption reachable here without any unsafe code in the program is in scope.
* Swift’s on-crash backtracer, `swift-backtrace`. Any exploit that causes the backtracer or its runtime components to execute an arbitrary process or write to an attacker-specified file during on-crash backtracing of a non-adversarial program is in scope, including cases where the non-adversarial program has itself been compromised at runtime by an attacker.
* The Swift Concurrency runtime: task lifecycle, actor isolation enforcement, and executor scheduling. A data race that the language's isolation model is supposed to prevent statically or dynamically, but which instead corrupts memory at runtime, is in scope.
* Standard library safe APIs (`Array`, `String`, `Dictionary`, `Set`, and similar), when used without `Unsafe*` types: a bounds-check bypass, use-after-free, or buffer overflow reachable through safe API surface is in scope.
* Swift Package Manager (SwiftPM) supply-chain surfaces: dependency resolution, registry and signature verification, and the manifest execution sandbox. A sandbox escape during manifest evaluation, or successful dependency resolution despite a checksum/signature mismatch, is in scope, since resolving untrusted package graphs is an explicit, supported use case.
* Code generation: most miscompilations are not security-sensitive, but a miscompilation with a clear path to making the produced binary significantly easier to exploit (for example, silently dropping a bounds or overflow check that safe Swift code relies on) is in scope.

### Not security-sensitive

* The compiler frontend, SIL optimizer, and IRGen when fed malicious source. A crash, hang, or even arbitrary code execution while compiling an untrusted `.swift` file is out of scope. Compiling untrusted source has never been a hardened use case, and typically also invokes other unhardened tooling such as build systems and SwiftPM manifests.
* `unsafe` APIs used incorrectly, such as `UnsafePointer`, `UnsafeMutableRawPointer`, `Unmanaged`, and `@unchecked Sendable`. These carry the same contract as C: misuse is undefined behavior, not a Swift vulnerability, unless documentation explicitly promises hardening against that misuse.
* SourceKit, SourceKit-LSP, and swift-driver operating on untrusted source or command-line input, for the same reason as the compiler frontend.
* Sanitizer- and debug-only tooling (for example, ASan/TSan/UBSan integration and debug-only assertions), since these are not meant to be included in production binaries.
* Developer tooling such as `swift-inspect` used to examine an adversarial target process, which is treated like the compiler frontend, unless a specific API is documented as safe for that use.

Both lists are expected to change over time. If you are not sure whether something is in scope, err toward reporting it privately anyway — a maintainer can downgrade a private report to a public issue, but not the reverse, and the outcome of your report may be used to update this document.
