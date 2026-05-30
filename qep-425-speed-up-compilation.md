# QGIS Enhancement: Speed up compilation

**Date** YYYY/MM/DD

**Author** David Koňařík (@dvdkon)

**Contact** dvdkon at konarici dot cz

**Version** QGIS 4.4

# Summary

A clean build of QGIS currently takes over an hour on a reasonable developer's
machine, with incremental builds touching often-used headers taking almost the
same amount of time. This slows down development and strains our CI
infrastructure.

As QGIS developers, we should take steps to refactor the codebase and build
system to speed up both clean and incremental builds.

**WIP**: This document is in its current state is meant for discussion on the
specific proposed changes.

## Proposed Solution

- **Split `qgis.h` into multiple files**:  
  Changing currently `qgis.h` triggers a near-complete rebuild, since it is
  (transitively) included in pretty much every compilation unit. I propose
  splitting it into multiple files of no more than 1000 lines by category.

  Other headers (especially those containing templated code) may also benefit
  from being split into multiple files.

- **Automate checking for unused headers:**  
  Every included header in a `.cpp` file slows down its compilation, and an
  include in a header can slow down compilation of hundreds of dependent files.
  Unused includes should be avoided by a `clang-tidy` check.

- **Automate maintenance of forward-declarations:**  
  Forward declarations can greatly speed up compilation by avoiding including
  headers, but are annoying (and error-prone) to maintain. I propose building a
  script that replaces includes by a forward-decl when possible, and removes
  stale forward-decls when not.

- **Adopt "d-pointer" pattern for select classes:**  
  The
  [d-pointer](https://wiki.qt.io/D-Pointer)/[pImpl](https://en.cppreference.com/cpp/language/pimpl)
  pattern hides private members of a class into heap-allocated opaque struct.
  This is often done for ABI-compatibility, but it can improve build times as
  well (e.g. when complex templated types are moved from the header).

- **Move method bodies from headers to `.cpp` files:**  
  A method body in a header is not just code that has to be parsed by each
  including compilation unit, but more importantly code whose change will cause
  various files to be rebuilt, even when the interface is unchanged.

- **Move away from expensive STL templated types:**  
  Sadly some useful types from the C++ STL (like `std::unique_ptr`) are
  expensive to instantiate (see clang's `-ftime-trace`). The ideal solution
  would be to change that, but from the QGIS side we can use less-expensive
  library (Qt) alternatives or make our own less-generic replacements.

- **Fix build system not to trigger rebuild on git operations:**  
  Since git operations change a file's modified time, this can cause a rebuild
  even when the file in question wasn't changed (or was changed back by a
  subsequent operation). Patching Ninja to keep hashes of input files may help.

- **Use extern explicit template instantiations:**  
  The "normal" way to use templates is to keep them in headers and let the
  compiler instantiate them whenever they are needed. This makes headers larger
  and makes the compiler do duplicate work (for each compilation unit).

  Maybe not as applicable for QGIS (since we don't have many templated
  classes).

- **Strip comments from headers before build:**  
  Now this is maybe a bit too far-fetched, but it *could* help with both
  parsing speed and fewer rebuilds.

- **Write down recommended practices for developers:**  
  Like using `mold`, which compiler is faster,
  `-DENABLE_LOCAL_BUILD_SHORTCUTS=ON`, ...

Other people's efforts on different codebases:
- https://vitaut.net/posts/2024/faster-cpp-compile-times/
- https://www.reddit.com/r/cpp/comments/hj66pd/comment/fwl8bdp/

### Affected Files

Most C++ source files in QGIS.

## Risks

The proposed changes in codebase organisation may prove to be burdensome to
maintain, though this should be alleviated through automation.

## Performance Implications

The proposals should only improve compile time, and with the exception of using
the "d-pointer" pattern, shouldn't impact runtime at all.

## Backwards Compatibility

Not applicable, Python API should stay the same.
