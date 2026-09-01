# MATC

MATC is a small interpreter for numerical expressions and matrix arithmetic. It is the expression language used in [Elmer FEM](https://github.com/ElmerCSC/elmerfem) solver input files, where it evaluates expressions either as the command file is read or while the solver runs.

This repository packages MATC on its own, as a library and a command line interpreter, for use outside Elmer and so that it can be developed and released independently.

## Provenance

MATC is part of Elmer, written at CSC - IT Center for Science Ltd., Finland, and licensed under the LGPL 2.1. This repository was extracted from [ElmerCSC/elmerfem](https://github.com/ElmerCSC/elmerfem) with `git filter-repo`, so the commit history of the `matc/` directory is preserved rather than starting from a squashed import. It is not an official CSC distribution.

## Install

Download a binary for your platform from the [releases page](https://github.com/AltElmer/matc/releases). The release binaries are self-contained: on Linux and Windows the C runtime is linked in, so there is nothing to install alongside them.

There is no single binary that runs everywhere. Native code is built per operating system and per CPU architecture, so the releases carry one archive for each of the combinations listed there.

## Usage

`matc` reads expressions from standard input and writes results to standard output. It does not take expressions as command line arguments.

```
$ matc
1 + 2
         3
quit
```

Because it reads standard input, it also works in a pipeline, which is the usual way to script it:

```
$ printf '2^10\nquit\n' | matc
  1.02e+03
```

Note the default output format: `2^10` prints as `1.02e+03`, not `1024`. `format(n)` sets the number of digits, so `format(12)` before the same expression prints `1024`.

Assignments are silent; an expression on its own prints its value.

```
2^10
  1.02e+03

x = 3
y = vector(0, 10, 1)      # 0 1 2 ... 10, as start, end, increment
A = zeros(2, 3)
size(A)
         2         3

det(eye(3))
         1

sin(pi/4)
     0.707
```

Functions are introduced with `function`, and return by assigning to an underscore followed by the function name:

```
function sq(t)
{
  _sq = t^2 + 1;
}

sq(4)
        17
```

`help` on its own lists the built-in functions. `help("name")`, with the quotes, describes one:

```
help("vector")
r=vector(start,end,inc)
Return vector of values going from start to end by inc.
```

The bundled reference in [`doc/`](doc/) documents the language in full: operators, control flow, the `import` and `export` statements for function scope, and the file, matrix and plotting builtins.

## Building

Requires CMake 3.12 or newer and a C compiler. GCC, Clang, MSVC, clang-cl and mingw-w64 are all built and tested in CI, on Linux, macOS and Windows, x86_64 and arm64.

Two things worth knowing if you port it further:

The code predates C23 and relies on `()` in a function pointer type meaning unspecified arguments rather than none, so it is compiled as C99. Compilers defaulting to C23, such as GCC 15 and newer, reject it otherwise.

It uses exactly two POSIX functions, `unlink()` and `isatty()`. On MSVC these come from `<io.h>` rather than `<unistd.h>`, which is the only source change needed for that compiler. `libm` is linked only where it exists as a separate library, detected rather than assumed from the compiler name.

```
cmake -S . -B build
cmake --build build
```

That produces the shared library `libmatc` and the `matc` executable in `build/src`.

For a self-contained executable, as shipped in the releases:

```
cmake -S . -B build -DMATC_STATIC_CLI=ON
cmake --build build
```

`MATC_STATIC_CLI` builds the library static and links the runtime into the executable where the platform allows it. macOS does not ship static system libraries, so a macOS build links `libSystem` regardless; that is normal and the binary is still portable across macOS versions.

## Using it inside Elmer

Elmer consumes this repository as a submodule, so its own build is unchanged: the directory is added with `ADD_SUBDIRECTORY(matc)` exactly as before, and the `matc` target it produces is the same shared library. A clone of Elmer needs `--recurse-submodules`, or `git submodule update --init`, for the directory to be populated.

The CMake in this repository works both ways. When it is the top level project it supplies its own `project()` and install layout; when it is added as a subdirectory it takes those from the parent, and nothing about the Elmer build changes.

## Licence

LGPL 2.1. See [`LGPL-2.1`](LGPL-2.1) and [`COPYING`](GPL-2). Copyright CSC - IT Center for Science Ltd., Finland.
