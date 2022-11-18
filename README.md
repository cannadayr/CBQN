# A [BQN](https://github.com/mlochbaum/BQN) implementation in C

[CBQN-specific documentation](docs/README.md) • [source code overview](src/README.md)

## Running

1. `make`
    - Third-party packages and other ways to run BQN are listed [here](https://mlochbaum.github.io/BQN/running.html)
    - `make CC=cc` if clang isn't installed
    - `make PIE=""` on ARM CPUs (incl. Android & M1)
    - `make FFI=0` if your system doesn't have libffi (see [macOS](#macos) for FFI support on macOS)
    - Use `gmake` on BSD
    - `make clean` if anything breaks and you want a clean build slate
    - Run `sudo make install` afterwards to install into `/usr/local/bin/bqn` (a `PREFIX=/some/path` argument will install to `/some/path/bin/bqn`); `sudo make uninstall` to uninstall
    - `make REPLXX=1` to enable replxx (syntax highlighting, completion)
2. `./BQN somefile.bqn` to execute a file, or `rlwrap ./BQN` for a REPL (or just `./BQN` if replxx is enabled)

## Configuration options

- Different build types:
    - `make o3` - `-O3`, the default build
    - `make o3n` - `-O3 -march=native`
    - `make o3g` - `-O3 -g`
    - `make debug` - unoptimized build with extra assertion checks (incl. `-g`)
    - `make debug1` - debug build without parallel compilation. Useful if everything errors, and you don't want error messages from multiple threads to be printed at the same time.
    - `make heapverify` - verify that refcounting is done correctly
    - `make o3n-singeli` - a Singeli build, currently only for x86-64 CPUs supporting AVX2 & BMI2
    - `make shared-o3` - produce the shared library `libcbqn.so`
    - `make c` - a build with no flags, for manual customizing
    - `make shared-c` - like `make c` but for a shared library
    - `make single-(o3|o3g|debug|c)` - compile everything as a single translation unit. Will take longer to compile & won't have incremental compilation, and isn't supported for many configurations.
    - `make emcc-o3` - build with Emscripten `emcc`
- Output executable/library location can be changed with `OUTPUT=output/path/file`.  
  For `emcc-o3`, that will be used as a directory to add the `BQN.js` and `BQN.wasm` files to.
- For any of the above (especially `make c`), you can add extra compiler flags with `f=...`, e.g.  
  `make f='-O3 -DSOME_MACRO=whatever -some_other_cc_flag' c` or `make debug f=-O2`.  
    - Linker flags can be added with `lf=...`, and flags for both with `CCFLAGS=...`; for replxx compilation, `REPLXX_FLAGS=...` will change the C++ flags.
    - If you want to use custom build types but your system doesn't have `shasum` or `sha256sum`, add `force_build_dir=build/obj/some_identifier`. That directory will be used to store incremental build object files.
  Macros that you may want to define are listed in `src/h.h`.  
- Adding `builddir=1` to the make argument list will give you the build directory of the current configuration. Adding `clean=1` will clean that directory.
- Tests can be run with `./BQN path/to/mlochbaum/BQN/test/this.bqn` (add `-noerr` if using `make heapverify`).
- Git submodules are used for Singeli, replxx, and bytecode. It's possible to override those by, respectively, linking/copying a local version to `build/singeliLocal`, `build/replxxLocal`, and `build/bytecodeLocal`.

### Precompiled bytecode

CBQN uses [the self-hosted BQN compiler](https://github.com/mlochbaum/BQN/blob/master/src/c.bqn) & some parts of [the runtime](https://github.com/mlochbaum/BQN/blob/master/src/r1.bqn), and therefore needs to be bootstrapped.  
By default, the CBQN will use [precompiled bytecode](https://github.com/dzaima/cbqnBytecode). In order to build everything from source, you need to:

1. get another BQN implementation; [dzaima/BQN](https://github.com/dzaima/BQN) is one that is completely implemented in Java (clone it & run `./build`).
2. clone [mlochbaum/BQN](https://github.com/mlochbaum/BQN).
3. From within CBQNs directory, run `mkdir -p build/bytecodeLocal/gen`
4. Run `said-other-bqn-impl ./build/genRuntime path/to/mlochbaum/BQN build/bytecodeLocal`  
   In the case of the Java impl, `java -jar path/to/dzaima/BQN/BQN.jar ./build/genRuntime ~/git/BQN build/bytecodeLocal`

## macOS

To use FFI in macOS, libffi must be installed, and manually added to `C_INCLUDE_PATH` and `LIBRARY_PATH`. In addition, the `NO_DYNAMIC_LIST=1` make argument is needed, so the full command might, depending on where libffi is installed, look like one of these:

```sh
C_INCLUDE_PATH=/opt/homebrew/opt/libffi/include:$C_INCLUDE_PATH LIBRARY_PATH=/opt/homebrew/opt/libffi/lib:$LIBRARY_PATH make PIE="" NO_DYNAMIC_LIST=1
C_INCLUDE_PATH=/usr/local/opt/libffi/include:$C_INCLUDE_PATH LIBRARY_PATH=/usr/local/opt/libffi/lib:$LIBRARY_PATH make PIE="" NO_DYNAMIC_LIST=1
```

Further configuration (different build type, compiler options, etc) can still be done by adding more make arguments.

## Requirements

CBQN requires either gcc or clang as the C compiler (though it defaults to `clang`, as optimizations are written based on whether or not clang needs them; add a `CC=cc` make arg to use the default system compiler), and, optionally libffi for `•FFI`, and C++ (C++11; defaults to `c++`, override with `CXX=your-c++`) for replxx.

While there aren't hard expectations of specific versions for any of those, nevertheless here are some configurations that CBQN is tested on by dzaima:

```
x86-64 (Linux):
  gcc 9.5; gcc 11.3; clang 10.0.0; clang 14.0.0
  libffi 3.4.2
  cpu microarchitecture: Haswell
  replxx: g++ 11.3.0
x86 (Linux):
  clang 14.0.0; known to break on gcc - https://gcc.gnu.org/bugzilla/show_bug.cgi?id=58416
  running on the above x86-64 system, compiled with CCFLAGS=-m32
AArch64 ARMv8-A (within Termux on Android):
  clang 15.0.4
  libffi 3.4.4 (structs were broken as of 3.4.3)
  replxx: clang++ 15.0.4
```
Additionally, CBQN is known to compile as-is on macOS (with [some extra options](#macOS) for FFI), but Windows requires [WinBQN](https://github.com/actalley/WinBQN) to properly set up Cygwin/Msys2.

## License

Any file without an explicit copyright message is copyright (c) 2021 dzaima, GNU GPLv3 - see LICENSE