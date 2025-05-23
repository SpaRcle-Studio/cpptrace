# Changelog

- [Changelog](#changelog)
- [v0.5.2](#v052)
- [v0.5.1](#v051)
- [v0.5.0](#v050)
- [v0.4.1](#v041)
- [v0.4.0](#v040)
- [v0.3.1](#v031)
- [v0.3.0](#v030)
- [v0.2.1](#v021)
- [v0.2.0](#v020)
- [v0.1.1](#v011)
- [v0.1](#v01)

# v0.5.2

Fixes:
- Fixed bug with resolution of inlined calls

Other changes:
- Improved internal string formatting
- Improved internal error handling

# v0.5.1

Fixes:
- Fix MSVC warning treated as error for 32-bit windows
- Fix MSVC issue with min/max macros
- Fix potential null dereference issue identified by eyalgolan1337

# v0.5.0

New:
- Traces with source code snippets with `cpptrace::stacktrace::print_with_snippets`
- Added `cpptrace::get_snippet` utility
- Added `cpptrace::can_signal_safe_unwind` utility
- Added `stacktrace_frame::get_object_info`

Changes:
- The library is now compiled with position-independent code by default

Fixes:
- Fixed issue with `_dl_find_object` implementation

Misc:
- Various refactoring, cleanup, and improvements

# v0.4.1

Changes:
- Renamed `stacktrace_frame.address` -> `stacktrace_frame.raw_address`
- Added `stacktrace_frame.object_address`
- Fixed segfault due to an edge case with dwarf file table indices
- For the libdwarf back-end: At least show object frame information if resolution fails
- Extremely small performance improvements
- Small documentation updates
- Small fix for conan
- Updated cmake to not FetchContent zstd when using CPPTRACE_USE_EXTERNAL_LIBDWARF
- CI improvements
  - Test the default configuration first before doing the exhaustive and slow matrix of all configurations.
  - Cleanup of duplicated prerequisite installation code
  - Cleanup of built and test python scripts

# v0.4.0

What's new:
- Cpptrace now has a C API! 🎉
- Cpptrace is now able to parse macOS debug maps and resolve stack traces without dSYM files

Most notable improvements:
- Updated cpptrace exception objects to generate traces at the callsite for improved consistency with trace output. As
  part of this cpptrace exception objects have had their constructors updated.
- Improved dwarf back-end robustness
  - Fallback to the compilation-unit cache or walking compilation-units if aranges lookup fails
- Eliminated reliance on a CMake-generated export header
- Added a configuration to control resolution of inlined function calls
- Made architecture selection in Mach-O universal binaries

Other improvements:
- Improved documentation for installation and usage
- Generally improved README content and organization
- Fixed an MSVC workaround producing dozens of warnings
- Better handle compiler color diagnostic arguments if compiler families differ
- Improvements for handling libdwarf's header placement
- Fixed issue with libunwind resolution
- `-Werror` is now used in CI
- More library configurations are now tested in CI
- Updated to libdwarf 9
- Updated the library's CMake to acquire zstd through FetchContent
- Fixed minor issue with stacktrace printing always trying to enable virtual terminal processing, even when not actually
  printing to the terminal.

# v0.3.1

Tiny patch:
- Fix `CPPTRACE_EXPORT` annotations
- Add workaround for [msvc bug][msvc bug] affecting msvc 19.38.

[msvc bug]: https://developercommunity.visualstudio.com/t/MSVC-1938331290-preview-fails-to-comp/10505565

# v0.3.0

Interface Changes:
- Overhauled the API for traced `cpptrace::exception` objects
- Added `cpptrace::isatty` utility
- Added specialized `std::terminate` handler and `cpptrace::register_terminate_handler` utility
- Added `cpptrace::frame_ptr` as an alias for the appropriate type capable of representing an instruction pointer
- Added signal-safe tracing support and a guide for [how to trace safely](signal-safe-tracing.md)
- Added `cpptrace::nullable<T>` utility for better indicating when line / column information is not present
- Added `CPPTRACE_FORCE_NO_INLINE` utility macro to cpptrace.hpp
- Added `CPPTRACE_WRAP` and `CPPTRACE_WRAP_BLOCK` utilities to catch non-`cpptrace::exception`s and rethrow wrapped in a
  traced exception.
- Updated `cpptrace::stacktrace::to_string` to take a `bool color` parameter
- Eliminated uses of `std::uint_least32_t` in favor of other types
- Updated `object_frame` data member names

Other changes:
- Added object resolution with `_dl_find_object` which is much faster than `dladdr`
- Added column support for dwarf
- Added inlined call resolution
  - Added `cpptrace::stacktrace_frame::is_inline`
- Added libunwind as a back-end
- Unbundled libdwarf
- Increased hard max frame count, used by some back-end requiring fixed buffers, from 100 to 200
- Improved libgcc unwind backend
- Improved trace output when information is missing
- Added a lookup table for faster dwarf line information lookup

News:
- The library is now on conan and vcpkg

Minor changes:
- Assorted bug fixes
- Various code quality improvements
- CI improvements
- Documentation improvements
- CMake improvements
- Internal refactoring

# v0.2.1

Patches:
- Fixed uintptr_t implicit conversion issue for msvc
- Better handling for PIC and static linkage in CMake
- Added gcc 5 support
- Various warning fixes
- Added stackwalk64 support for 32-bit x86 mingw/clang and architecture detection
- Added check for stackwalk64 support and CaptureStackBacktrace as a fallback
- Various cmake cleanup and changes to use cpptrace through package managers
- Added sonarlint and implemented some sonarlint fixes

# v0.2.0

Key changes:
- Added libdwarf as a back-end so cpptrace doesn't have to rely on addr2line or libbacktrace
- Overhauled library's public-facing interface to make the library more useful
  - Added `raw_trace` interface
  - Added `object_trace` interface
  - Added `stacktrace` interface
  - Updated `generate_trace` to return a `stacktrace` rather than a vector of frames
  - Added `generate_trace` counterparts for raw and object traces
  - Added `generate_trace` overloads with max_depth
  - Added interface for internal demangling utility
  - Added cache mode configuration
  - Added option to absorb internal trace exceptions (by default it absorbs)
  - Added `cpptrace::exception`, which automatically generates and stores a stacktrace when thrown
  - Added `exception_with_message`
  - Added traced analogs for stdexcept errors: `logic_error`, `domain_error`, `invalid_argument`, `length_error`,
    `out_of_range`, `runtime_error`, `range_error`, `overflow_error`, and `underflow_error`.

Other changes:
- Bundled libdwarf with cpptrace so the library can essentially be self-contained and not have to rely on libraries that
  might not already be on a system
- Added StackWalk64 as an unwinding back-end on windows
- Added system for multiple symbol back-ends to be used, mainly for more complete stack traces on mingw
- Fixed sporadic line number reporting errors due to not adjusting the program counter from the unwinder
- Improved addr2line/atos invocation back-end on macos
- Lots of error handling improvements
- Performance improvements
- Updated default back-ends for most systems
- Removed full tracing backends
- Cleaned up library cmake
- Lots of internal cleanup and refactoring
- Improved library usage instructions in README

# v0.1.1

Fixed:
- Handle errors when object files don't exist or can't be opened for reading
- Handle paths with spaces when using addr2line on windows

# v0.1

Initial release of the library 🎉
