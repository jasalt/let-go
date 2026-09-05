---
status: planning
last-verified: 2026-09-05
human-verified:
---

# babashka.fs support

## Finding

`babashka/fs` can be supported by let-go, but the current upstream source is
not loadable as-is. This is a host-compatibility port, not a dependency
resolution problem.

A probe against upstream `babashka/fs` at `293cc23` demonstrates the first
failure:

```text
go run . -source-paths /tmp/bbfs/src \\
  -e '(require (quote [babashka.fs :as fs])) (println (fs/exists? "."))'

unable to load namespace clojure.java.io
```

The reason is that `fs.cljc` has `:clj`, `:cljs`, and `:default` branches, but
not `:lg`. let-go selects `:default`; that branch imports `clojure.java.io`
and uses `java.io`, `java.nio.file`, `java.nio.file.attribute`,
`java.nio.channels`, and `java.util.zip`. let-go intentionally has no JVM or
JAR loader, so adding Maven resolution would not make those classes available.

## What already exists in let-go

The runtime already covers the common scripting substrate:

| Need                                  | Existing surface                                          |
|---------------------------------------|-----------------------------------------------------------|
| Current/temp directory, listing, stat | `os/cwd`, `os/temp-dir`, `os/ls`, `os/stat`               |
| Recursive removal and rename          | `os/delete-tree`, `os/rename`                             |
| Text file I/O                         | `io/slurp`, `io/spit`, `io/read-lines`, `io/write-lines`  |
| Process search path                   | `os/getenv`, `io` and core path/string functions          |
| Source loading                        | `.lg`, `.cljc`, and `.clj` search through `-source-paths` |
| Host Java-shape compatibility         | small, deliberately bounded shims in `host_jvm_*`         |

The missing pieces are precisely the host effects that make `babashka.fs`
useful: file modes and times, symlink identity/targets, byte-file I/O,
copy/move, temporary files, recursive visitor traversal, glob matching, and
zip/gzip streams. The existing `os` API cannot be composed into all of these
without losing `nofollow-links`, error, or return-value semantics.

## Compatibility assessment

| Upstream API group                                                                                   | Assessment                               | Notes                                                                                        |
|------------------------------------------------------------------------------------------------------|------------------------------------------|----------------------------------------------------------------------------------------------|
| `path`, `file`, `file-name`, `parent`, `normalize`, `absolutize`, `relativize`, `root`, `components` | feasible                                 | Return strings on let-go rather than JVM `Path` values.                                      |
| `exists?`, `directory?`, `regular-file?`, `absolute?`, `relative?`, `hidden?`                        | feasible                                 | Needs a host stat primitive for no-follow behavior.                                          |
| `list-dir`, `list-dirs`, `glob`, `match`, `which`, `exec-paths`                                      | feasible                                 | `list-dir` callbacks and glob semantics need a native implementation.                        |
| `create-*`, `delete*`, `copy`, `copy-tree`, `move`, `touch`                                          | feasible                                 | Implement natively with explicit option handling.                                            |
| `read/write-all-*`, `slurp`, `spit`, `update-file`                                                   | feasible                                 | Byte arrays and charset options need a defined let-go representation.                        |
| POSIX permissions and file attributes/times                                                          | feasible with documented differences     | Use Go file metadata; `FileTime`, `Instant`, and arbitrary attribute providers do not exist. |
| symlinks and `same-file?`                                                                            | feasible on native builds                | Must return clear unsupported errors on WASM/TinyGo.                                         |
| `walk-file-tree`                                                                                     | feasible with a let-go callback contract | Do not expose Java `FileVisitor` objects.                                                    |
| `zip`, `unzip`, `gzip`, `gunzip`                                                                     | feasible natively                        | `archive/zip` and `compress/gzip` are already suitable Go dependencies.                      |
| `with-temp-dir`                                                                                      | feasible                                 | Must be a let-go macro, with cleanup in `finally`.                                           |
| Java `File`, `Path`, `FileTime`, `Instant`, `InputStream` interop                                    | not a compatibility target               | A string/native-value API is the appropriate let-go contract.                                |

The important semantic boundary is that “supports babashka.fs” should mean the
filesystem API works for let-go values, not that JVM code receiving a returned
`java.nio.file.Path` works. Code which explicitly calls Java methods on these
objects remains outside the target.

## Could `gojava` provide the shim?

[`gloathub/gojava`](https://github.com/gloathub/gojava) would help with a
portion of the implementation. Its `file` package covers much of `java.io.File`
(path construction, predicates, listing, basic mutation, and timestamps), and
its stream, charset, and instant packages cover several supporting types. It
is also a plain Go module with no JVM or JNI dependency, which fits let-go's
runtime model.

It is not a drop-in solution for upstream `babashka.fs`:

- `babashka.fs` is primarily built on `java.nio.file.Files`, `Path`,
  `FileSystems`, `FileVisitor`, and file-attribute classes; gojava currently
  does not provide those `java.nio.file` packages.
- gojava's API is Go-idiomatic (`file.GetName(f)`), while let-go's source uses
  JVM-shaped forms (`(.getName f)` and `Files/readAllBytes`). A bridge would
  still be needed to expose gojava values and methods to the VM.
- The `file` package intentionally collapses some Java behavior into boolean
  results or millisecond timestamps. That is not enough by itself to preserve
  babashka.fs's exceptions, `nofollow-links`, nanosecond file times, and
  attribute-map contracts.
- It does not cover the archive and recursive visitor layer needed by the full
  API.

Therefore gojava is a useful backend or reference for a let-go host shim, and
could reduce duplicated work for `File`, streams, `Charset`, and `Instant`. It
would not avoid the native `babashka.fs.host` layer or make Pomegranate
necessary. Depending on its API stability, adding it as a dependency may be
less attractive than using the standard library directly for this focused
filesystem implementation.

## Recommended implementation

Follow the useful part of Jolt's approach—ship a host implementation behind a
stable namespace—but do not reproduce its Java shim wholesale:

1. Add a native `babashka.fs.host` namespace for the filesystem operations that
   need Go or platform-specific behavior.
2. Add an embedded `babashka.fs` `.lg` veneer for path/string helpers, option
   normalization, `with-temp-dir`, and the public API shape.
3. Return strings for paths and ordinary let-go maps/vectors for metadata.
4. Preserve upstream names and arities where practical; document deliberate
   differences for Java-only values and unsupported targets.
5. Add a native integration suite covering ordinary files, directories,
   symlinks, modes, times, globbing, copy/move, and archive round trips.
6. Keep external source precedence explicit. If a project supplies its own
   `babashka.fs`, it must either opt into the built-in implementation or use a
   future `:lg` branch; silently replacing project code would be surprising.

Pomegranate is not useful for this plan: it resolves Maven dependencies for
JVM Clojure, while the upstream library's source has no non-JVM dependency that
let-go can execute. A future upstream `:lg` branch could make source loading
possible, but it would still need the native operations above.

## Decision

**Supported in principle; not supported by this branch yet.** The smallest
credible first milestone is the non-archive scripting subset (path predicates,
listing, creation/deletion, copy/move, symlinks, text/byte I/O, and globbing),
followed by archive and attribute APIs. A Java interop shim is technically
possible but is the larger and less idiomatic implementation for let-go.
