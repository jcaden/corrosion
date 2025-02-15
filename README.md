# Corrosion
[![Build Status](https://github.com/corrosion-rs/corrosion/workflows/.github/workflows/test.yaml/badge.svg)](https://github.com/corrosion-rs/corrosion/actions?query=branch%3Amaster)

Corrosion, formerly known as cmake-cargo, is a tool for integrating Rust into an existing CMake
project. Corrosion is capable of importing executables, static libraries, and dynamic libraries
from a crate.

## Features
- Automatic Import of Executable, Static, and Shared Libraries from Rust Crate
- Easy Installation of Rust Executables
- Trivially Link Rust Executables to C++ Libraries in Tree
- Multi-Config Generator Support
- Simple Cross-Compilation

## Coming Soon
- Automatic Generation of Rust Bindings (via bindgen) and C/C++ Bindings (via cbindgen)
- Easy Install of Libraries

## Sample Usage
```cmake
cmake_minimum_required(VERSION 3.12)
project(MyCoolProject LANGUAGES CXX)

find_package(Corrosion REQUIRED)

corrosion_import_crate(MANIFEST_PATH rust-lib/Cargo.toml)

add_executable(cpp-exe main.cpp)
target_link_libraries(cpp-exe PUBLIC rust-lib)
```

# Documentation

## Table of Contents
- [Installation](#installation)
  - [Package Manager](#package-manager)
  - [CMake Install](#cmake-install)
  - [Fetch Content](#fetch-content)
  - [Subdirectory](#subdirectory)
- [Usage](#usage)
  - [Options](#options)
    - [Advanced](#advanced)
  - [Importing C-Style Libraries Written in Rust](#importing-c-style-libraries-written-in-rust)
    - [Generate Bindings to Rust Library Automatically](#generate-bindings-to-rust-library-automatically)
  - [Importing Libraries Written in C and C++ Into Rust](#importing-libraries-written-in-c-and-c++-into-rust)
  - [Cross Compiling](#cross-compiling)
    - [Windows-to-Windows](#windows-to-windows)
    - [Linux-to-Linux](#linux-to-linux)
    - [Android](#android)

## Installation
There are two fundamental installation methods that are supported by Corrosion - installation as a
CMake package or using it as a subdirectory in an existing CMake project. Corrosion strongly
recommends installing the package, either via a package manager or manually using cmake's
installation facilities.

Installation will pre-build all of Corrosion's native tooling, meaning that configuring any project
which uses Corrosion is much faster. Using Corrosion as a subdirectory will result in the native
tooling for Corrosion to be re-built every time you configure a new build directory, which could
be a non-trivial cost for some projects. It also may result in issues with large, complex projects
with many git submodules that each individually may use Corrosion. This can unnecessarily exacerbate
diamond dependency problems that wouldn't otherwise occur using an externally installed Corrosion.

### Package Manager

Coming soon...

### CMake Install
After using a package manager, the next recommended way to use Corrosion is to install it as a
package using CMake. This means you won't need to rebuild Corrosion's tooling every time you
generate a new build directory. Installation also solves the diamond dependency problem that often
comes with git submodules or other primitive dependency solutions.

First, download and install Corrosion:
```bash
git clone https://github.com/corrosion-rs/corrosion.git
# Optionally, specify -DCMAKE_INSTALL_PREFIX=<target-install-path>. You can install Corrosion anyway
cmake -Scorrosion -Bbuild -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
# This next step may require sudo or admin privileges if you're installing to a system location,
# which is the default.
cmake --install build --config Release
```

You'll want to ensure that the install directory is available in your `PATH` or `CMAKE_PREFIX_PATH`
environment variable. This is likely to already be the case by default on a Unix system, but on
Windows it will install to `C:\Program Files (x86)\Corrosion` by default, which will not be in your
`PATH` or `CMAKE_PREFIX_PATH` by default.

Once Corrosion is installed and you've ensured the package is avilable in your `PATH`, you
can use it from your own project like any other package from your CMakeLists.txt:
```cmake
find_package(Corrosion REQUIRED)
```

### FetchContent
If installation is difficult or not feasible in your environment, you can use the
[FetchContent](https://cmake.org/cmake/help/latest/module/FetchContent.html) module to include
Corrosion. This will download Corrosion and use it as if it were a subdirectory at configure time.

In your CMakeLists.txt:
```cmake
include(FetchContent)

FetchContent_Declare(
    Corrosion
    GIT_REPOSITORY https://github.com/corrosion-rs/corrosion.git
    GIT_TAG origin/master # Optionally specify a version tag or branch here
)

FetchContent_MakeAvailable(Corrosion)
```

### Subdirectory
Corrosion can also be used directly as a subdirectory. This solution may work well for small
projects, but it's discouraged for large projects with many dependencies, especially those which may
themselves use Corrosion. Either copy the Corrosion library into your source tree, being sure to
preserve the `LICENSE` file, or add this repository as a git submodule:
```bash
git submodule add https://github.com/corrosion-rs/corrosion.git
```

From there, using Corrosion is easy. In your CMakeLists.txt:
```cmake
add_subdirectory(path/to/corrosion)
```

## Usage
### Options
All of the following variables are evaluated automatically in most cases. In typical cases you
shouldn't need to alter any of these.

- `Rust_TOOLCHAIN:STRING` - Specify a named rustup toolchain to use. Changes to this variable
resets all other options. Default: If the first-found `rustc` is a `rustup` proxy, then the default
rustup toolchain (see `rustup show`) is used. Otherwise, the variable is unset by default.
- `Rust_ROOT:STRING` - CMake provided. Path to a Rust toolchain to use. This is an alternative if
you want to select a specific Rust toolchain, but it's not managed by rustup. Default: Nothing

#### Advanced
- `Rust_COMPILER:STRING` - Path to an actual `rustc`. If set to a `rustup` proxy, it will be
replaced by a path to an actual `rustc`. Default: The `rustc` in the first-found toolchain, either
from `rustup`, or from a toolchain available in the user's `PATH`.
- `Rust_CARGO:STRING` - Path to an actual `cargo`. Default: the `cargo` installed next to
`${Rust_COMPILER}`.
- `Rust_CARGO_TARGET:STRING` - The default target triple to build for. Alter for cross-compiling.
Default: On Visual Studio Generator, the matching triple for `CMAKE_VS_PLATFORM_NAME`. Otherwise,
the default target triple reported by `${Rust_COMPILER} --version --verbose`.

If you want to set environment variables during the invocation of `cargo build`, you can set the
`CORROSION_ENVIRONMENT_VARIABLES` property to `KEY=VALUE` pairs on the targets created by `corrosion_import_crate`.
The pairs can also be generator expressions, as they will be evaluated using $<GENEX_EVAL>.
For example this may be useful if the build scripts of crates look for environment variables.
This feature requires CMake >= 3.19.0.

#### Information provided by Corrosion

For your convenience, Corrosion sets a number of variables which contain information about the version of the rust 
toolchain. You can use the CMake version comparison operators 
(e.g. [`VERSION_GREATER_EQUAL`](https://cmake.org/cmake/help/latest/command/if.html#version-comparisons)) on the main
variable (e.g. `if(Rust_VERSION VERSION_GREATER_EQUAL "1.57.0")`), or you can inspect the major, minor and patch 
versions individually.
- `Rust_VERSION<_MAJOR|_MINOR|_PATCH>` - The version of rustc.
- `Rust_CARGO_VERSION<_MAJOR|_MINOR|_PATCH>` - The cargo version.
- `Rust_LLVM_VERSION<_MAJOR|_MINOR|_PATCH>` - The LLVM version used by rustc.
- `Rust_IS_NIGHTLY` - 1 if a nightly toolchain is used, otherwise 0. Useful for selecting an unstable feature for a 
  crate, that is only available on nightly toolchains.

### Cargo feature selection

Cargo crates sometimes offer features, which traditionally can be specified with `cargo build` on the
command line as opt-in. You can select the features to enable with the `FEATURES` argument when
calling `corrosion_import_crate`. The following example enables the "chocolate" and "vanilla"
features of the imported Cargo.toml that hypothetically provides ice cream:

```cmake
corrosion_import_crate(MANIFEST_PATH rust/Cargo.toml FEATURES chocolate vanilla)
```

That's equivalent to calling `cargo build --features chocolate vanilla`.

It is also possible to specify the features as a target list property on the CMake target created
by `corrosion_import_crate`. The property is called `CORROSION_FEATURES`.

Finally, regarding features, cargo offers the ability to turn off default features or enable all
features, with the `--all-features` and `--no-default-features` options. Corrosion offers corresponding
parameters for `corrosion_import_crate` called `ALL_FEATURES` and `NO_DEFAULT_FEATURES`. The corresponding
boolean target properties - which override any specified values with `corrosion_import_crate` are called
`CORROSION_ALL_FEATURES` and `CORROSION_NO_DEFAULT_FEATURES`.

#### RUSTFLAGS

Sometimes you may need to pass `RUSTFLAGS` to rustc, e.g. to enable some nightly option. Corrosion allows you to set
RUSTFLAGS on a per-crate basis:
```cmake
corrosion_add_target_rustflags(cmake_target rustflag [additional_rustflag1 ...])
```
`corrosion_add_target_rustflags()` can be called multiple times, each time appending the new rustflag to the list of 
flags.

### Developer/Maintainer Options
These options are not used in the course of normal Corrosion usage, but are used to configure how
Corrosion is built and installed. Only applies to Corrosion builds and subdirectory uses.

- `CORROSION_DEV_MODE:BOOL` - Indicates that Corrosion is being actively developed. Default: `OFF`
if Corrosion is a subdirectory, `ON` if it is the top-level project
- `CORROSION_BUILD_TESTS:BOOL` - Build the Corrosion tests. Default: `Off` if Corrosion is a
subdirectory, `ON` if it is the top-level project
- `CORROSION_GENERATOR_EXECUTABLE:STRING` - Specify a path to the corrosion-generator executable.
This is to support scenarios where it's easier to build corrosion-generator outside of the normal
bootstrap path, such as in the case of package managers that make it very easy to import Rust
crates for fully reproducible, offline builds.
- `CORROSION_INSTALL_EXECUTABLE:BOOL` - Controls whether corrosion-generator is installed with the
package. Default: `ON` with `CORROSION_GENERATOR_EXECUTABLE` unset, otherwise `OFF`

### Importing C-Style Libraries Written in Rust
Corrosion makes it completely trivial to import a crate into an existing CMake project. Consider
a project called [rust2cpp](test/rust2cpp) with the following file structure:
```
rust2cpp/
    rust/
        src/
            lib.rs
        Cargo.lock
        Cargo.toml
    CMakeLists.txt
    main.cpp
```

This project defines a simple Rust lib crate, like so, in [`rust2cpp/rust/Cargo.toml`](test/rust2cpp/rust/Cargo.toml):
```toml
[package]
name = "rust-lib"
version = "0.1.0"
authors = ["Andrew Gaspar <andrew.gaspar@outlook.com>"]
license = "MIT"
edition = "2018"

[dependencies]

[lib]
crate-type=["staticlib"]
```

In addition to `"staticlib"`, you can also use `"cdylib"`. In fact, you can define both with a
single crate and switch between which is used using the standard
[`BUILD_SHARED_LIBS`](https://cmake.org/cmake/help/latest/variable/BUILD_SHARED_LIBS.html) variable.

This crate defines a simple crate called `rust-lib`. Importing this crate into your
[CMakeLists.txt](test/rust2cpp/CMakeLists.txt) is trivial:
```cmake
# Note: you must have already included Corrosion for `corrosion_import_crate` to be available. See # the `Installation` section above.

corrosion_import_crate(MANIFEST_PATH rust/Cargo.toml)
```

Now that you've imported the crate into CMake, all of the executables, static libraries, and dynamic
libraries defined in the Rust can be directly referenced. So, merely define your C++ executable as
normal in CMake and add your crate's library using target_link_libraries:
```cmake
add_executable(cpp-exe main.cpp)
target_link_libraries(cpp-exe PUBLIC rust-lib)
```

That's it! You're now linking your Rust library to your C++ library.

#### Generate Bindings to Rust Library Automatically

Currently, you must manually declare bindings in your C or C++ program to the exported routines and
types in your Rust project. You can see boths sides of this in
[the Rust code](test/rust2cpp/rust/src/lib.rs) and in [the C++ code](test/rust2cpp/main.cpp).

Integration with [cbindgen](https://github.com/eqrion/cbindgen) is
planned for the future.

### Importing Libraries Written in C and C++ Into Rust
TODO

### Cross Compiling
Corrosion attempts to support cross-compiling as generally as possible, though not all
configurations are tested. Cross-compiling is explicitly supported in the following scenarios.

In all cases, you will need to install the standard library for the Rust target triple. When using
Rustup, you can use it to install the target standard library:

```bash
rustc target add <target-rust-triple>
```

If the target triple is automatically derived, Corrosion will print the target during configuration.
For example:

```
-- Rust Target: aarch64-linux-android
```

#### Windows-to-Windows
Corrosion supports cross-compiling between arbitrary Windows architectures using the Visual Studio
Generator. For example, to cross-compile for ARM64 from any platform, simply set the `-A`
architecture flag:

```bash
cmake -S. -Bbuild-arm64 -A ARM64
cmake --build build-arm64
```

#### Linux-to-Linux
In order to cross-compile on Linux, you will need to install a cross-compiler. For example, on
Ubuntu, to cross compile for 64-bit Little-Endian PowerPC Little-Endian, install
`g++-powerpc64le-linux-gnu` from apt-get:

```bash
sudo apt install g++-powerpc64le-linux-gnu
```

Currently, Corrosion does not automatically determine the target triple while cross-compiling on
Linux, so you'll need to specify a matching `Rust_CARGO_TARGET`.

```bash
cmake -S. -Bbuild-ppc64le -DRust_CARGO_TARGET=powerpc64le-unknown-linux-gnu -DCMAKE_CXX_COMPILER=powerpc64le-linux-gnu-g++
cmake --build build-ppc64le
```

#### Android

Cross-compiling for Android is supported on all platforms with the Makefile and Ninja generators,
and the Rust target triple will automatically be selected. The CMake
[cross-compiling instructions for Android](https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html#cross-compiling-for-android)
apply here. For example, to build for ARM64:

```cmake
cmake -S. -Bbuild-android-arm64 -GNinja -DCMAKE_SYSTEM_NAME=Android -DCMAKE_ANDROID_NDK=/path/to/android-ndk-rxxd -DCMAKE_ANDROID_ARCH_ABI=arm64-v8a
```

**Important note:** The Android SDK ships with CMake 3.10 at newest, which Android Studio will
prefer over any CMake you've installed locally. CMake 3.10 is insufficient for using Corrosion,
which requires a minimum of CMake 3.12. If you're using Android Studio to build your project,
follow the instructions in the Android Studio documentation for
[using a specific version of CMake](https://developer.android.com/studio/projects/install-ndk#vanilla_cmake).

#### Host Builds

By default, Corrosion will cross-compile the imported crates to the specified
target triple. If some of your crates are for example build tools, then you may
want those not to be cross-compiled but rather built as a host binary. This can
be done by setting the `CORROSION_USE_HOST_BUILD` boolean property to true on
the imported cmake target.

Note that this requires the use of CMake 3.19 or newer.
