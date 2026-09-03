# Z3's CMake build system

[CMake](https://cmake.org/) is a "meta build system" that reads a description of
the project written in the `CMakeLists.txt` files and emits a build system for
that project of your choice using one of CMake's "generators". This allows CMake
to support many different platforms and build tools. You can run `cmake --help`
to see the list of supported "generators" on your platform. Example generators
include "Unix Makefiles", "Ninja", and "Visual Studio 17 2022".

## Using Z3 in a CMake project

Z3's installed CMake package is consumable with CMake 3.5 or newer. Find the
package and link to its imported target:

```cmake
find_package(Z3 5.1 CONFIG REQUIRED)
target_link_libraries(your_target PRIVATE z3::libz3)
```

The `z3::libz3` target provides the correct headers, language requirements, and
link dependencies. If Z3 is installed in a nonstandard prefix, point CMake at
that prefix when configuring your project:

```sh
cmake -S . -B build -DCMAKE_PREFIX_PATH=/path/to/z3/prefix
```

As a narrower alternative, `Z3_DIR` may name the directory containing
`Z3Config.cmake`, typically `<prefix>/lib/cmake/z3`.

### Selecting static or shared libz3

When both variants are installed, select one with a package component:

```cmake
find_package(Z3 5.1 CONFIG REQUIRED COMPONENTS static) # or shared
target_link_libraries(your_target PRIVATE z3::libz3)
```

Without a component, the package follows the consumer's `BUILD_SHARED_LIBS`
setting when possible and otherwise prefers the shared variant. Projects that
cannot use components may set `Z3_SHARED_LIBS` before `find_package` as a strict
package-specific preference.

A CMake directory can load only one variant because both variants intentionally
provide the same `z3::libz3` target. A later `find_package` call requesting the
other variant reports the package as not found instead of silently reusing the
wrong library.

## Alternative ways to consume Z3

### Optional FetchContent fallback

If your project may build Z3 from source when no suitable installed package is
available, use `FIND_PACKAGE_ARGS` with
[FetchContent](https://cmake.org/cmake/help/latest/module/FetchContent.html):

```cmake
set(Z3_MIN_VERSION 5.1)

include(FetchContent)
FetchContent_Declare(
  Z3
  GIT_REPOSITORY https://github.com/Z3Prover/z3
  GIT_TAG        z3-5.1.0
  FIND_PACKAGE_ARGS ${Z3_MIN_VERSION} CONFIG
)
FetchContent_MakeAvailable(Z3)

target_link_libraries(your_target PRIVATE z3::libz3)
```

`FetchContent_MakeAvailable` first tries the equivalent of
`find_package(Z3 5.1 CONFIG)` and fetches Z3 only if that fails. Both paths
provide the same `z3::libz3` target. `FIND_PACKAGE_ARGS` requires CMake 3.24;
falling back to this Z3 source tree requires CMake 3.30.

### Compatibility variables

Older consumers may use `Z3_LIBRARIES`, `Z3_C_INCLUDE_DIRS`, and
`Z3_CXX_INCLUDE_DIRS`, but new code should prefer `z3::libz3`:

```cmake
target_include_directories(your_target PRIVATE ${Z3_CXX_INCLUDE_DIRS})
target_link_libraries(your_target PRIVATE ${Z3_LIBRARIES})
```

## Building Z3

Building Z3 requires CMake 3.30 or newer. The shortest portable command-line
workflow is:

```sh
cmake -S . -B build
cmake --build build --parallel
```

For a single-configuration generator, Z3 defaults to `RelWithDebInfo`. Generator
selection, build types, compiler selection, and less common workflows are
described below.

### Choosing a generator

CMake selects a platform-appropriate generator by default. To choose another,
pass its name during the first configuration. For example, to use
[Ninja](https://ninja-build.org/):

```sh
cmake -S . -B build -G Ninja
cmake --build build --parallel
```

Run `cmake --help` to list the generators available on the current platform.
Visual Studio 2019 and newer can also open Z3's source directory directly as a
CMake project. Regardless of the interface or generator, use the repository
root—not `src`—as the source directory.

### Choosing a compiler or target architecture

Set the `CC` and `CXX` environment variables when first configuring a build
directory. For example:

```sh
CC=clang CXX=clang++ cmake -S . -B build
```

To change compilers later, configure a new build directory. The CMake build
detects the target architecture from the compiler and toolchain information. For
cross-compilation, use a CMake toolchain file that sets the usual variables,
such as `CMAKE_SYSTEM_PROCESSOR` and `CMAKE_CXX_COMPILER_TARGET`.

Compiler flags that change the ABI can be supplied through `CFLAGS` and
`CXXFLAGS` when appropriate. For example, a 32-bit GCC build on a system with
multilib support can be configured with:

```sh
CFLAGS=-m32 CXXFLAGS=-m32 cmake -S . -B build
```

These environment variables also take effect only when the build directory is
first configured.

### Interactive configuration

The `ccmake` and `cmake-gui` interfaces can configure a build directory and edit
its cache interactively. See CMake's
[User Interaction Guide](https://cmake.org/cmake/help/latest/guide/user-interaction/index.html)
for details.

### Cleaning a source tree used by the Python build

The Python build system creates generated files in the source tree. The CMake
build refuses to configure when it detects these files. If the source tree was
previously used by the Python build, preview the files that Git would clean:

```sh
git clean -nx src
```

After checking the list, remove them with:

```sh
git clean -fx src
```

## Build Types

The standard CMake build types are:

- Release
- Debug
- RelWithDebInfo
- MinSizeRel

For single-configuration generators (e.g. "Unix Makefiles" and "Ninja"), set the
build type by passing `-DCMAKE_BUILD_TYPE=<build_type>`. Z3 defaults to
`RelWithDebInfo` when it is the top-level project and no type was selected. It
also accepts custom build types; the Z3-specific `Z3DEBUG` and
`_EXTERNAL_RELEASE` definitions are applied only to the standard configurations
listed above.

For multi-configuration generators (e.g. Visual Studio) you don't set the build
type when invoking CMake and instead set the build type within Visual Studio
itself.

## MSVC Security Features

When building with Microsoft Visual C++ (MSVC), Z3 automatically enables several
security features by default:

### Control Flow Guard (CFG)

- **CMake Option**: `Z3_ENABLE_CFG` - Defaults to `ON` for MSVC builds
- **Compiler flag**: `/guard:cf` - Automatically enabled when `Z3_ENABLE_CFG=ON`
- **Linker flag**: `/GUARD:CF` - Automatically enabled when `Z3_ENABLE_CFG=ON`
- **Purpose**: Control Flow Guard analyzes control flow for indirect call
  targets at compile time and inserts runtime verification code to detect
  attempts to compromise your code by redirecting control flow to
  attacker-controlled locations
- **Note**: Automatically enables `/DYNAMICBASE` as required by `/GUARD:CF`

### Address Space Layout Randomization (ASLR)

- **Linker flag**: `/DYNAMICBASE` - Enabled when Control Flow Guard is active
- **Purpose**: Randomizes memory layout to make exploitation more difficult
- **Note**: Required for Control Flow Guard to function properly

### Incompatibilities

Control Flow Guard is incompatible with:

- `/ZI` (Edit and Continue debug information format)
- `/clr` (Common Language Runtime compilation)

When these incompatible options are detected, Control Flow Guard will be
automatically disabled with a warning message.

### Disabling Control Flow Guard

To disable Control Flow Guard, set the CMake option:

```bash
cmake -S . -B build -DZ3_ENABLE_CFG=OFF
```

## Useful options

The following useful options can be passed to CMake whilst configuring.

| Option                                  | Type     | Description                                                                                                                                                                                                                                                                                                                                                                           |
| --------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CMAKE_BUILD_TYPE`                      | `STRING` | The build type to use. Only relevant for single configuration generators (e.g. "Unix Makefile" and "Ninja").                                                                                                                                                                                                                                                                          |
| `CMAKE_INSTALL_BINDIR`                  | `STRING` | The path to install z3 binaries (relative to `CMAKE_INSTALL_PREFIX`), e.g. `bin`.                                                                                                                                                                                                                                                                                                     |
| `CMAKE_INSTALL_INCLUDEDIR`              | `STRING` | The path to install z3 include files (relative to `CMAKE_INSTALL_PREFIX`), e.g. `include`.                                                                                                                                                                                                                                                                                            |
| `CMAKE_INSTALL_LIBDIR`                  | `STRING` | The path to install z3 libraries (relative to `CMAKE_INSTALL_PREFIX`), e.g. `lib`.                                                                                                                                                                                                                                                                                                    |
| `CMAKE_INSTALL_PREFIX`                  | `STRING` | The install prefix to use (e.g. `/usr/local/`).                                                                                                                                                                                                                                                                                                                                       |
| `CMAKE_INSTALL_PKGCONFIGDIR`            | `STRING` | The path to install pkg-config files (relative to `CMAKE_INSTALL_PREFIX` by default).                                                                                                                                                                                                                                                                                                 |
| `CMAKE_INSTALL_PYTHON_PKG_DIR`          | `STRING` | The path to install the z3 python bindings. This can be relative (to `CMAKE_INSTALL_PREFIX`) or absolute.                                                                                                                                                                                                                                                                             |
| `CMAKE_INSTALL_Z3_CMAKE_PACKAGE_DIR`    | `STRING` | The path to install CMake package files (e.g. `/usr/lib/cmake/z3`).                                                                                                                                                                                                                                                                                                                   |
| `CMAKE_INSTALL_API_BINDINGS_DOC`        | `STRING` | The path to install documentation for API bindings.                                                                                                                                                                                                                                                                                                                                   |
| `Python3_EXECUTABLE`                    | `STRING` | The python executable to use during the build.                                                                                                                                                                                                                                                                                                                                        |
| `Z3_ENABLE_TRACING_FOR_NON_DEBUG`       | `BOOL`   | If set to `TRUE` enable tracing in non-debug builds, if set to `FALSE` disable tracing in non-debug builds. Note in debug builds tracing is always enabled.                                                                                                                                                                                                                           |
| `BUILD_SHARED_LIBS`                     | `BOOL`   | If set to `TRUE` (default) build libz3 as a shared library; otherwise build it as a static library. The historical `Z3_BUILD_LIBZ3_SHARED` spelling is still accepted for compatibility.                                                                                                                                                                                              |
| `Z3_BUILD_LIBZ3_MSVC_STATIC`            | `BOOL`   | If set to `TRUE`, use the static MSVC runtime library. This does not select the static libz3 variant; use `BUILD_SHARED_LIBS` for that.                                                                                                                                                                                                                                               |
| `Z3_BUILD_LIBZ3_CORE`                   | `BOOL`   | If set to `TRUE` (default) build the core libz3 library. If set to `FALSE`, skip building libz3 and find an installed, same-version shared Z3 CMake package instead. This mode is intended for building only the Python bindings; set `CMAKE_PREFIX_PATH` or `Z3_DIR` if necessary.                                                                                                   |
| `Z3_ENABLE_EXAMPLE_TARGETS`             | `BOOL`   | If set to `TRUE` add the build targets for building the API examples.                                                                                                                                                                                                                                                                                                                 |
| `Z3_USE_LIB_GMP`                        | `BOOL`   | If set to `TRUE` use the GNU multiple precision library. If set to `FALSE` use an internal implementation.                                                                                                                                                                                                                                                                            |
| `Z3_BUILD_PYTHON_BINDINGS`              | `BOOL`   | If set to `TRUE` then Z3's Python bindings will be built. When building libz3 in the same tree, this requires `BUILD_SHARED_LIBS=ON`. When `Z3_BUILD_LIBZ3_CORE` is `FALSE`, this builds only the Python bindings using an installed shared libz3.                                                                                                                                    |
| `Z3_INSTALL_PYTHON_BINDINGS`            | `BOOL`   | If set to `TRUE` and `Z3_BUILD_PYTHON_BINDINGS` is `TRUE` then running the `install` target will install Z3's Python bindings.                                                                                                                                                                                                                                                        |
| `Z3_BUILD_DOTNET_BINDINGS`              | `BOOL`   | If set to `TRUE` then Z3's .NET bindings will be built. Requires `BUILD_SHARED_LIBS=ON`.                                                                                                                                                                                                                                                                                              |
| `Z3_INSTALL_DOTNET_BINDINGS`            | `BOOL`   | If set to `TRUE` and `Z3_BUILD_DOTNET_BINDINGS` is `TRUE` then running the `install` target will install Z3's .NET bindings.                                                                                                                                                                                                                                                          |
| `Z3_BUILD_JAVA_BINDINGS`                | `BOOL`   | If set to `TRUE` then Z3's Java bindings will be built. Requires `BUILD_SHARED_LIBS=ON`.                                                                                                                                                                                                                                                                                              |
| `Z3_INSTALL_JAVA_BINDINGS`              | `BOOL`   | If set to `TRUE` and `Z3_BUILD_JAVA_BINDINGS` is `TRUE` then running the `install` target will install Z3's Java bindings.                                                                                                                                                                                                                                                            |
| `Z3_JAVA_JAR_INSTALLDIR`                | `STRING` | The path to directory to install the Z3 Java `.jar` file. This path should be relative to `CMAKE_INSTALL_PREFIX`.                                                                                                                                                                                                                                                                     |
| `Z3_JAVA_JNI_LIB_INSTALLDIR`            | `STRING` | The path to directory to install the Z3 Java JNI bridge library. This path should be relative to `CMAKE_INSTALL_PREFIX`.                                                                                                                                                                                                                                                              |
| `Z3_BUILD_GO_BINDINGS`                  | `BOOL`   | If set to `TRUE` then Z3's Go bindings will be built. Requires Go 1.20+ and `BUILD_SHARED_LIBS=ON`.                                                                                                                                                                                                                                                                                   |
| `Z3_BUILD_OCAML_BINDINGS`               | `BOOL`   | If set to `TRUE` then Z3's OCaml bindings will be built. Requires `BUILD_SHARED_LIBS=ON`.                                                                                                                                                                                                                                                                                             |
| `Z3_INSTALL_OCAML_BINDINGS`             | `BOOL`   | If set to `TRUE` and `Z3_BUILD_OCAML_BINDINGS` is `TRUE` then running the `install` target will install Z3's OCaml bindings.                                                                                                                                                                                                                                                          |
| `Z3_BUILD_JULIA_BINDINGS`               | `BOOL`   | If set to `TRUE` then Z3's Julia bindings will be built. Requires `BUILD_SHARED_LIBS=ON`.                                                                                                                                                                                                                                                                                             |
| `Z3_INSTALL_JULIA_BINDINGS`             | `BOOL`   | If set to `TRUE` and `Z3_BUILD_JULIA_BINDINGS` is `TRUE` then running the `install` target will install Z3's Julia bindings.                                                                                                                                                                                                                                                          |
| `Z3_INCLUDE_GIT_DESCRIBE`               | `BOOL`   | If set to `TRUE` and the source tree of Z3 is a git repository then the output of `git describe` will be included in the build.                                                                                                                                                                                                                                                       |
| `Z3_INCLUDE_GIT_HASH`                   | `BOOL`   | If set to `TRUE` and the source tree of Z3 is a git repository then the git hash will be included in the build.                                                                                                                                                                                                                                                                       |
| `Z3_BUILD_DOCUMENTATION`                | `BOOL`   | If set to `TRUE` then documentation for the API bindings can be built by invoking the `api_docs` target.                                                                                                                                                                                                                                                                              |
| `Z3_INSTALL_API_BINDINGS_DOCUMENTATION` | `BOOL`   | If set to `TRUE` and `Z3_BUILD_DOCUMENTATION` is `TRUE` then documentation for API bindings will be installed when running the `install` target.                                                                                                                                                                                                                                      |
| `Z3_ALWAYS_BUILD_DOCS`                  | `BOOL`   | If set to `TRUE` and `Z3_BUILD_DOCUMENTATION` is `TRUE` then documentation for API bindings will always be built. Disabling this is useful for faster incremental builds. The documentation can be manually built by invoking the `api_docs` target.                                                                                                                                  |
| `Z3_LINK_TIME_OPTIMIZATION`             | `BOOL`   | If set to `TRUE` link time optimization will be enabled.                                                                                                                                                                                                                                                                                                                              |
| `Z3_ENABLE_CFI`                         | `BOOL`   | If set to `TRUE` will enable Control Flow Integrity security checks. This is only supported by Clang and will fail on other compilers. This requires Z3_LINK_TIME_OPTIMIZATION to also be enabled.                                                                                                                                                                                    |
| `Z3_ENABLE_CFG`                         | `BOOL`   | If set to `TRUE` will enable Control Flow Guard security checks. This is only supported by MSVC and will fail on other compilers. This does not require link time optimization. Control Flow Guard is enabled by default for MSVC builds. Note: Control Flow Guard is incompatible with `/ZI` (Edit and Continue debug information) and `/clr` (Common Language Runtime compilation). |
| `Z3_API_LOG_SYNC`                       | `BOOL`   | If set to `TRUE` will enable experimental API log sync feature. This uses locking to allow concurrent API log access across multiple threads. This option is incompatible with `Z3_SINGLE_THREADED`.                                                                                                                                                                                  |
| `WARNINGS_AS_ERRORS`                    | `STRING` | If set to `ON` compiler warnings will be treated as errors. If set to `OFF` compiler warnings will not be treated as errors. If set to `SERIOUS_ONLY` a subset of compiler warnings will be treated as errors.                                                                                                                                                                        |
| `Z3_C_EXAMPLES_FORCE_CXX_LINKER`        | `BOOL`   | If set to `TRUE` the C API examples will request that the C++ linker is used rather than the C linker.                                                                                                                                                                                                                                                                                |
| `Z3_BUILD_EXECUTABLE`                   | `BOOL`   | If set to `TRUE` build the z3 executable. Defaults to `TRUE` when Z3 is the top-level project and `FALSE` when it is a subproject.                                                                                                                                                                                                                                                    |
| `Z3_BUILD_TEST_EXECUTABLES`             | `BOOL`   | If set to `TRUE` build the z3 test executables. Defaults to `TRUE` when Z3 is the top-level project and `FALSE` when it is a subproject.                                                                                                                                                                                                                                              |
| `Z3_SAVE_CLANG_OPTIMIZATION_RECORDS`    | `BOOL`   | If set to `TRUE` saves Clang optimization records by setting the compiler flag `-fsave-optimization-record`.                                                                                                                                                                                                                                                                          |
| `Z3_SINGLE_THREADED`                    | `BOOL`   | If set to `TRUE` compiles Z3 for single threaded mode.                                                                                                                                                                                                                                                                                                                                |
| `Z3_POLLING_TIMER`                      | `BOOL`   | If set to `TRUE` compiles Z3 to use polling based timer instead of requiring a thread. This is useful for wasm builds and avoids spawning threads that interfere with how WASM is run.                                                                                                                                                                                                |
| `Z3_ADDRESS_SANITIZE`                   | `BOOL`   | If set to `TRUE` compiles Z3 with address sanitization enabled.                                                                                                                                                                                                                                                                                                                       |

On the command line these can be passed to `cmake` using the `-D` option. In
`ccmake` and `cmake-gui` these can be set in the user interface.

For example:

```sh
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DZ3_ENABLE_TRACING_FOR_NON_DEBUG=FALSE
```

## Installing static and shared libz3

Each build directory produces one library variant, selected with CMake's
standard `BUILD_SHARED_LIBS` option. Static and shared builds can be installed
to the same prefix without overwriting one another. Configure both builds with
the same feature and dependency options (apart from `BUILD_SHARED_LIBS`),
because they share headers and package metadata such as `z3.pc`.

```sh
cmake -S . -B build-shared -DBUILD_SHARED_LIBS=ON
cmake --build build-shared
cmake --install build-shared --prefix /path/to/prefix

cmake -S . -B build-static -DBUILD_SHARED_LIBS=OFF
cmake --build build-static
cmake --install build-static --prefix /path/to/prefix
```

See [Selecting static or shared libz3](#selecting-static-or-shared-libz3) for
consumer-side selection.

### pkg-config

The installed `z3.pc` file is relocatable. With `--static`, pkg-config also
reports Z3's private system and GMP dependencies. As usual, `--static` does not
itself force the linker to select the archive when both library variants are in
its search path.

```sh
export PKG_CONFIG_PATH=/path/to/prefix/lib/pkgconfig
cc example.c $(pkg-config --cflags --libs z3)
cc example.c $(pkg-config --cflags --libs --static z3)
```

## Z3 API Bindings

Z3 exposes various language bindings for its API. Below are some notes on
building and/or installing these bindings when building Z3 with CMake.

### Python bindings

#### Building Python bindings with libz3

The default behavior when `Z3_BUILD_PYTHON_BINDINGS=ON` is to build both the
libz3 library and the Python bindings together:

```sh
cmake -S . -B build \
  -DBUILD_SHARED_LIBS=ON \
  -DZ3_BUILD_PYTHON_BINDINGS=ON
cmake --build build --parallel
```

#### Building only Python bindings (using pre-installed libz3)

For package managers like conda-forge that want to avoid rebuilding libz3 for
each Python version, you can build only the Python bindings by setting
`Z3_BUILD_LIBZ3_CORE=OFF`. This requires the shared CMake package for the exact
same Z3 version to already be installed:

```sh
# First, build and install libz3 (once)
cmake -S . -B build-libz3 \
  -DBUILD_SHARED_LIBS=ON \
  -DCMAKE_INSTALL_PREFIX=/path/to/prefix
cmake --build build-libz3 --parallel
cmake --install build-libz3

# Then, build Python bindings for each Python version without rebuilding libz3
cmake -S . -B build-py310 \
  -DZ3_BUILD_LIBZ3_CORE=OFF \
  -DZ3_BUILD_PYTHON_BINDINGS=ON \
  -DCMAKE_PREFIX_PATH=/path/to/prefix \
  -DCMAKE_INSTALL_PREFIX=/path/to/prefix \
  -DPython3_EXECUTABLE=/path/to/python3.10
cmake --build build-py310 --parallel
cmake --install build-py310
```

This approach significantly reduces build time when packaging for multiple
Python versions, as the expensive libz3 compilation happens only once.

### Java bindings

The CMake build uses the `FindJava` and `FindJNI` CMake modules to detect the
installation of Java. If CMake fails to find Java, set the `JAVA_HOME`
environment variable when invoking CMake. For example:

```sh
JAVA_HOME=/usr/lib/jvm/default cmake -S . -B build \
  -DBUILD_SHARED_LIBS=ON \
  -DZ3_BUILD_JAVA_BINDINGS=ON
cmake --build build --parallel
```

Note that the built `.jar` file is named `com.microsoft.z3-VERSION.jar` where
`VERSION` is the Z3 version. On non-Windows systems, a symbolic link named
`com.microsoft.z3.jar` is provided. This symbolic link is not created when
building under Windows.

### Go bindings

Go bindings can be built by setting `Z3_BUILD_GO_BINDINGS=ON`. The Go bindings
use CGO to wrap the Z3 C API, so you'll need:

- Go 1.20 or later installed on your system
- `BUILD_SHARED_LIBS=ON` (Go bindings require the shared library)

Example:

```sh
cmake -S . -B build \
  -DBUILD_SHARED_LIBS=ON \
  -DZ3_BUILD_GO_BINDINGS=ON
cmake --build build --parallel
```

If CMake detects a Go installation (via `go` executable in PATH), it will create
two optional targets:

- `go-bindings` - Builds the Go bindings
- `test-go-examples` - Runs the Go examples

Note that the Go bindings are installed as source files (not compiled) since Go
packages are typically distributed as source and compiled by the user's Go
toolchain.

To use the installed Go bindings, set the appropriate CGO flags:

```sh
export CGO_CFLAGS="-I/path/to/z3/include"
export CGO_LDFLAGS="-L/path/to/z3/lib -lz3"
export LD_LIBRARY_PATH="/path/to/z3/lib:$LD_LIBRARY_PATH"  # Linux/macOS
```

For detailed usage examples and API documentation, see `src/api/go/README.md`
and `examples/go/`.

## Developer/packager notes

These notes are for developers and packagers of Z3.

### Install/Uninstall

Install and uninstall operations are supported. Use `CMAKE_INSTALL_PREFIX` to
set the configured install prefix, or pass `--prefix` to `cmake --install`. To
control individual destinations, set the documented `CMAKE_INSTALL_*` variables.

To install, run:

```sh
cmake --install build
```

To uninstall, run:

```sh
cmake --build build --target uninstall
```

Note that `DESTDIR` is supported for
[staged installs](https://www.gnu.org/prep/standards/html_node/DESTDIR.html).

To install into a staging root:

```sh
DESTDIR=/full/path/to/staged cmake --install build
```

To uninstall from it:

```sh
DESTDIR=/full/path/to/staged cmake --build build --target uninstall
```

The uninstall target follows that build directory's install manifest. Static and
shared installations intentionally share headers and package metadata, so
uninstalling either one from a co-installation also removes those shared files;
reinstall the remaining variant to restore them.

### Examining invoked commands

To see exactly which commands the selected build tool invokes, use CMake's
generator-independent verbose option:

```sh
cmake --build build --verbose
```

### Additional targets

To see the list of targets, run:

```sh
cmake --build build --target help
```

There are a few special targets:

- `clean` all the built targets in the current directory and below
- `edit_cache` will invoke one of the CMake tools (depending on which is
  available) to let you change configuration options.
- `rebuild_cache` will reinvoke `cmake` for the project.
- `api_docs` will build the documentation for the API bindings.

### Setting build type specific flags

The build system supports both single- and multi-configuration generators.
`CMAKE_BUILD_TYPE` has no meaning for multi-configuration generators, so use
generator expressions to set configuration-specific compiler flags or
definitions.

For example:

```cmake
$<$<CONFIG:Debug>:Z3DEBUG>
```

If the build type at build time is `Debug` this evaluates to `Z3DEBUG` but
evaluates to nothing for all other configurations. You can see examples of this
in the `CMakeLists.txt` files.

### File-globbing

It is tempting to use file globbing in `CMakeLists.txt` to find source files. An
explicit source list describes the build manifest more clearly and reliably
causes CMake to regenerate when files are added or removed, so Z3 does not use
globs for target sources.

Long story short. Don't use file globbing.

### Serious warning flags

By default the `WARNINGS_AS_ERRORS` flag is set to `SERIOUS_ONLY` which means
some warnings will be treated as errors. These warnings are controlled by the
relevant `*_WARNINGS_AS_ERRORS` list defined in `cmake/compiler_warnings.cmake`.

Additional warnings should only be added here if the warnings have no false
positives.

### Building TPTP with CMake

Build instructions:

```sh
cmake -S . -B release -DBUILD_SHARED_LIBS=OFF \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build release --target z3_tptp5
cp release/examples/tptp_build_dir/z3_tptp5 ../bin/z3_tptp
```
