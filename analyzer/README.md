IKOS Analyzer
=============

This folder contains the implementation of the analyzer.

Introduction
------------

The IKOS Analyzer is an abstract interpretation-based static analyzer that aims at proving the absence of runtime errors in C and C++ programs. See [Checks](#checks) for the full list of available checks.

Installation
------------

IKOS Analyzer can be installed independently from the other components, but we recommend to build the analyzer from the root directory. To do so, follow the instructions in the root [README.md](../README.md).

### Dependencies

To build and run the analyzer, you will need the following dependencies:

* A C++ compiler that supports C++14 (gcc >= 4.9.2 or clang >= 3.4)
* CMake >= 2.8.12.2
* GMP >= 4.3.1
* Boost >= 1.55
* Python 2 >= 2.7.3 or Python 3 >= 3.3
* SQLite >= 3.6.20
* LLVM and Clang 4.0.x
* (Optional) APRON >= 0.9.10
* (Optional) Pygments
* IKOS Core
* IKOS AR
* IKOS LLVM Frontend

### Build and Install

To build and install the analyzer, run the following commands in the `analyzer` directory:

```
$ mkdir build
$ cd build
$ cmake \
    -DCMAKE_INSTALL_PREFIX=/path/to/analyzer-install-directory \
    -DLLVM_CONFIG_EXECUTABLE=/path/to/llvm/bin/llvm-config \
    -DCORE_ROOT=/path/to/core-install-directory \
    -DAR_ROOT=/path/to/ar-install-directory \
    -DFRONTEND_LLVM_ROOT=/path/to/frontend-llvm-install-directory \
    ..
$ make
$ make install
```

### Tests

To build and run the tests, simply type:

```
$ make check
```

### Documentation

To build the documentation, you will need [Doxygen](http://www.doxygen.org).

Then, simply type:

```
$ make doc
$ open doc/html/index.html
```

How to run IKOS
---------------

Suppose we want to analyze the following C program in a file, called *loop.c*:

```c
 1: #include <stdio.h>
 2: int a[10];
 3: int main(int argc, char *argv[]) {
 4:     size_t i = 0;
 5:     for (;i < 10; i++) {
 6:         a[i] = i;
 7:     }
 8:     a[i] = i;
 9:     printf("%i", a[i]);
10: }
```

To analyze this program with IKOS, simply run:

```
$ ikos loop.c
```

You shall see the following output. IKOS reports two occurrences of buffer overflow at line 8 and 9.

```
[*] Compiling loop.c
[*] Running ikos preprocessor
[*] Running ikos analyzer
[*] Translating LLVM bitcode to AR
[*] Running liveness analysis
[*] Running fixpoint profile analysis
[*] Running interprocedural value analysis
[*] Analyzing entry point: main
[*] Checking properties and writing results for entry point: main

# Time stats:
clang        : 0.037 sec
ikos-analyzer: 0.023 sec
ikos-pp      : 0.007 sec

# Summary:
Total number of checks                : 7
Total number of unreachable checks    : 0
Total number of safe checks           : 5
Total number of definite unsafe checks: 2
Total number of warnings              : 0

The program is definitely UNSAFE

# Results
loop.c: In function 'main':
loop.c:8:10: error: buffer overflow, trying to access index 10 of global variable 'a' of 10 elements
    a[i] = i;
         ^
loop.c: In function 'main':
loop.c:9:18: error: buffer overflow, trying to access index 10 of global variable 'a' of 10 elements
    printf("%i", a[i]);
                 ^
```

The `ikos` command takes a source file (`.c`, `.cpp`) or a LLVM bitcode file (`.bc`) as input, analyzes it to find runtime errors (also called undefined behaviors), creates a result database `output.db` in the current working directory and prints a report.

In the report, each line has one of the following status:

* **safe**: the statement is proven safe;
* **error**: the statement always results into an error;
* **unreachable**: the statement is never executed (dead code);
* **warning** may mean three things:
   1. the statement results into an error for some executions, or
   2. the static analyzer did not have enough information to conclude (check dependent on an external input, for instance), or
   3. the static analyzer was not powerful enough to prove the absence of errors;

By default, ikos shows warnings and errors directly in your terminal, like a compiler would do.

If the analysis report is too big, you shall use:
* `ikos-report output.db` to examine the report in your terminal
* `ikos-view output.db` to examine the report in a web interface

Running IKOS on a whole C/C++ project
-------------------------------------

To run IKOS on a large project, you will first need to compile the program into LLVM bitcode (`.bc`). The LLVM bitcode is a generic assembly language that can be used as an intermediate representation for most compiled language.

The easiest way to compile your project into a `.bc` file is to use the tool **Whole Program LLVM**: https://github.com/travitch/whole-program-llvm

You can install it easily using pip:

```
$ pip install wllvm
```

Then, you will need to set the following environment variable:

```
$ export LLVM_COMPILER=clang
```

Now, just use `wllvm` and `wllvm++` as your C and C++ compilers. These are wrappers that invoke the compiler as normal, and then, for each object file, create the equivalent LLVM bitcode file. This way, you can actually build your project as you usually do, and extract the final LLVM bitcode afterwards. It is also independent from any build system (autoconfig, cmake, Makefiles, ..).

For instance, using autoconf, you shall run:

```
$ CC=wllvm CXX=wllvm++ ./configure
$ make
```

Once everything is built, you can extract the LLVM bitcode file from a binary using the `extract-bc` command:

```
$ extract-bc prog
```

It will produce the LLVM bitcode file `prog.bc`. You can now analyze it using ikos:

```
$ ikos prog.bc
```

Analysis Options
----------------

This section describes the most relevant options of the analyzer.

### Checks

The list of available checks are:

* **buffer overflow analysis**, `-a=boa`: checks for buffer overflows and out-of-bound array accesses.
* **division by zero analysis**, `-a=dbz`: checks for integer divisions by zero.
* **null pointer analysis**, `-a=nullity`: checks for null pointer dereferences.
* **assertion prover**, `-a=prover`: prove user-defined properties, using `__ikos_assert(condition)`.
* **unaligned pointer analysis**, `-a=upav`: checks for unaligned pointer dereferences.
* **uninitialized variable analysis**, `-a=uva`: checks for read of uninitialized variables.
* **signed integer overflow analysis**, `-a=sio`: checks for signed integer overflows.
* **unsigned integer overflow analysis**, `-a=uio`: checks for unsigned integer overflows.
* **shift count analysis**, `-a=shc`: checks for invalid shifts, where the amount shifted is greater or equal to the bit-width of the left operand, or less than zero.
* **pointer overflow analysis**, `-a=poa`: checks for pointer arithmetic overflows.
* **pointer comparison analysis**, `-a=pcmp`: checks for pointer comparisons between pointers referring to different objects.
* **soundness analysis**, `-a=sound`: checks for instructions that could make the analysis unsound, i.e miss bugs.
* **function call analysis**, `-a=fca`: checks for function calls through function pointers of the wrong type.
* **dead code analysis**, `-a=dca`: checks for unreachable statements.
* **double free analysis**, `-a=dfa`: checks for double free, invalid free, use after free and use after return.

By default, all the checks are enabled except:

* **unaligned pointer analysis**, because it needs a congruence domain to generate meaningful results. See [Numerical abstract domains](#numerical-abstract-domains).
* **uninitialized variable analysis**, because it currently generates a lot of false positives.
* **unsigned integer overflow analysis**, because it is not an undefined behavior according to the C standard.
* **pointer overflow analysis**, because it is redundant with the buffer overflow analysis.

If you want to run specific checks, use the `-a` parameter:

```
$ ikos -a=boa,nullity test.c
```

Note that you can use the wildcard character `*`, `+` and `-`:

```
$ ikos -a='*,-sio' test.c
```

In this example, all the checks are enabled except signed integer overflow checks.

### Numerical abstract domains

IKOS is based on the theory of [Abstract Interpretation](https://www.di.ens.fr/~cousot/AI/IntroAbsInt.html). The analysis uses a numerical abstract domain internally to model integer variables.

The list of available numerical abstract domains are:

* `-d=interval`: The interval domain, see [CC77](https://www.di.ens.fr/~cousot/COUSOTpapers/publications.www/CousotCousot-POPL-77-ACM-p238--252-1977.pdf).
* `-d=congruence`: The congruence domain, see [Gra89](http://www.tandfonline.com/doi/abs/10.1080/00207168908803778).
* `-d=interval-congruence`: The reduced product of interval and congruence.
* `-d=dbm`: The Difference-Bound Matrices domain, see [PADO01](https://www-apr.lip6.fr/~mine/publi/article-mine-padoII.pdf).
* `-d=var-pack-dbm`: The Difference-Bound Matrices domain with variable packing, see [VMCAI16](https://seahorn.github.io/papers/vmcai16.pdf).
* `-d=var-pack-dbm-congruence`: The reduced product of DBM with variable packing and congruence.
* `-d=gauge`: The gauge domain, see [CAV12](https://ti.arc.nasa.gov/publications/4767/download/).
* `-d=gauge-interval-congruence`: The reduced product of gauge, interval and congruence.
* `-d=apron-interval`: The APRON interval domain, see [Box](http://apron.cri.ensmp.fr/library/0.9.10/apron/apron_21.html#SEC54).
* `-d=apron-octagon`: The APRON octagon domain, see [Oct](http://apron.cri.ensmp.fr/library/0.9.10/apron/oct_doc.html).
* `-d=apron-polka-polyhedra`: The APRON polka polyhedra domain, see [NewPolka](http://apron.cri.ensmp.fr/library/0.9.10/apron/apron_25.html#SEC58).
* `-d=apron-polka-linear-equalities`: The APRON polka linear equalities domain, see [NewPolka](http://apron.cri.ensmp.fr/library/0.9.10/apron/apron_25.html#SEC58).
* `-d=apron-ppl-polyhedra`: The APRON PPL polyhedra domain, see [PPL](http://apron.cri.ensmp.fr/library/0.9.10/apron/apron_29.html#SEC65).
* `-d=apron-ppl-linear-congruences`: The APRON PPL linear congruences domain, see [PPL](http://apron.cri.ensmp.fr/library/0.9.10/apron/apron_29.html#SEC65).
* `-d=apron-pkgrid-polyhedra-lin-cong`: The APRON Pkgrid polyhedra and linear congruences domain, see [Pkgrid](http://apron.cri.ensmp.fr/library/0.9.10/apron/apron_33.html#SEC69).
* `-d=var-pack-apron-octagon`: The APRON octagon domain with variable packing.
* `-d=var-pack-apron-polka-polyhedra`: The APRON Polka polyhedra domain with variable packing.
* `-d=var-pack-apron-polka-linear-equalities`: The APRON Polka linear equalities domain with variable packing.
* `-d=var-pack-apron-ppl-polyhedra`: The APRON PPL polyhedra domain with variable packing.
* `-d=var-pack-apron-ppl-linear-congruences`: The APRON PPL linear congruences domain with variable packing.
* `-d=var-pack-apron-pkgrid-polyhedra-lin-cong`: The APRON Pkgrid polyhedra and linear congruences domain with variable packing.

By default, IKOS uses the fastest and least precise numerical domain, the **interval** domain. If you want to run the analysis with a specific domain, use the `-d` parameter:

```
$ ikos -d=var-pack-dbm test.c
```

For most users, we recommend to analyze your project with the fastest and least precise domain (i.e, interval) first, and then try slower but more precise domains until the analysis is too long for you. This is the best way to reach a low rate of false positives (i.e, warnings).

Here is a list of numerical domains, sorted from the fastest and least precise to the slowest and most precise:

* `-d=interval`
* `-d=gauge-interval-congruence`
* `-d=var-pack-dbm`
* `-d=var-pack-apron-octagon`
* `-d=var-pack-apron-ppl-polyhedra`
* `-d=dbm`
* `-d=apron-octagon`
* `-d=apron-ppl-polyhedra`

You should consider running different analyses in this specific order.

Please also note that:
* Floating point variables are safely ignored.
* In order to use the **APRON** abstract domain, you need to build IKOS with APRON first. See [APRON Support](#apron-support).

### Entry points

By default, IKOS assumes the entry point of the program is `main`. You can specify a list of entry points using the `--entry-points` parameter:

```
$ ikos --entry-points=foo,bar test.c
```

### Optimization level

The parameter `--opt` allows you to set the optimization level. Performing a set of LLVM transformations can improve both the precision of the subsequent analysis as well as the performance.

Unfortunately, it might also hide errors in your code. By default, optimizations are disabled.

### Inter-procedural vs Intra-procedural

An **inter-procedural** analysis analyzes a function considering its call stack while an **intra-procedural** analysis ignores it. The former produces more precise results than the latter but it is often much more expensive.

By default, IKOS performs an inter-procedural analysis. Use `--proc=intra` to perform an intra-procedural analysis.

### Degree of precision

Each analysis can be executed using one of the following levels of precision, presented from the coarsest (and cheapest) to the most precise (and most expensive):

* **reg**: models only integers.
* **ptr**: models integers and pointers.
* **mem**: models integers, pointers and memory contents.

By default, IKOS uses the precision `mem`. Provide `--prec {reg,ptr,mem}` if you want to use another level of precision.

### Hardware addresses

In C code for embedded systems, it is usual to read or write at specific addresses to communicate with the hardware. By default, IKOS treats memory accesses at specific addresses as errors.

You can provide the `--hardware-addresses` parameter to specify a range of valid memory addresses:

```
$ ikos --hardware-addresses=0x20-0x40 project.bc
```

During the analysis, IKOS will assume that memory accesses in the range `[0x20, 0x40]` (in bytes, inclusive) are safe.

### Other analysis options

* `--globals-init`: use the given strategy for initialization of global variables.
* `--no-init-globals`: disable global variable initialization for the given entry points.
* `--no-liveness`: disable the liveness analysis.
* `--no-pointer`: disable the pointer analysis.
* `--no-fixpoint-profiles`: disable the detection of widening hints.
* `--argc`: specify the value of `argc` for the analysis.
* `--no-libc`: do not use libc intrinsics. Useful for bare metal programming.

See `ikos --help` for more information.

Report Options
--------------

This section describes the most relevant report options supported by `ikos` and `ikos-report`.

### Format

You can specify the format of the report using the `--format` (or `-f`) parameter.

Available formats are:

* **text**: Text format, convenient for the terminal;
* **csv**: CSV format, convenient for spreadsheet import;
* **json**: JSON format, convenient for developers.
* **web**: Web interface, using ikos-view.
* **no**: Disable the report.

By default, if the report is small, it will be printed out using the text format.

We recommend to use [ikos-view](#ikos-view) to examine reports of large projects.

### File

By default, the report is generated on the standard output. You can write it into a file using `--report-file=/path/to/report`

### Status Filter

Use `--status-filter` to filter unwanted checks.

Possible values are: **error**, **warning**, **safe**, **unreachable**.

Note that you can use the wildcard character `*`, `+` and `-`.

### Analysis Filter

Use `--analyses-filter` to filter unwanted checks.

Possible values are described in [Checks](#checks).

Note that you can use the wildcard character `*`, `+` and `-`. For instance:

```
$ ikos-report --analyses-filter='*,-boa' output.db
```

This will generate a report with all the checks, except buffer overflows.

### Verbosity

Use `--report-verbosity [1-4]` to specify the verbosity. A verbosity of one will give you very short messages, where a verbosity of 4 will provide you with all the information the analyzer has.

#### Other report options

See `ikos-report --help` for more information.

APRON Support
-------------

[APRON](http://apron.cri.ensmp.fr/library/) is a C library for static analysis using Abstract Interpretation. It implements several complex abstract domains, such as the Polyhedra domain.

IKOS provides a wrapper for APRON, allowing you to use any APRON abstract domain in the analyzer.

To use APRON, first download, build and install it. Consider using the svn trunk. You will also need to build APRON with [Parma Polyhedra Library](http://bugseng.com/products/ppl/) enabled. Set `HAS_PPL = 1` and define `PPL_PREFIX` in your `Makefile.config`

Now, to build IKOS with APRON support, just provide the option `-DAPRON_ROOT=/path/to/apron-install` when running cmake. For instance:

```
cmake \
    -DCMAKE_INSTALL_PREFIX=/path/to/ikos-install \
    -DAPRON_ROOT=/path/to/apron-install \
    ..
```

See [Numerical abstract domains](#numerical-abstract-domains) for the list of numerical abstract domains.

IKOS-VIEW
---------

ikos-view provides a web interface to examine IKOS results. It is available directly in the analyzer.

The web interface shows the source code with syntax highlighting, and allows you to filter the warnings by checks.

To use ikos-view, first run the analyzer on your project to generate a result database `output.db`, then simply run:

```
$ ikos-view output.db
```

It will start a web server. You can then launch your favorite web browser and visit [http://localhost:8080](http://localhost:8080)

Note that if you want syntax highlighting, you will need to install [Pygments](http://pygments.org):

```
$ pip install --user pygments
```

Overview of the source code
---------------------------

The following illustrates the directory structure of this folder:

```
.
├── doc
│   └── doxygen
│       └── latex
├── include
│   └── ikos
│       └── analyzer
│           ├── analysis
│           │   ├── execution_engine
│           │   ├── pointer
│           │   └── value
│           ├── checker
│           ├── database
│           │   └── table
│           ├── json
│           ├── support
│           └── util
├── python
│   └── ikos
│       └── view
│           ├── static
│           │   ├── css
│           │   └── js
│           └── template
├── script
├── src
│   ├── analysis
│   │   ├── pointer
│   │   └── value
│   │       └── machine_int_domain
│   ├── checker
│   ├── database
│   │   └── table
│   ├── json
│   └── util
└── test
    └── regression
```

#### doc/

Contains Doxygen files.

#### include/

* [include/ikos/analyzer/intrinsic.h](include/ikos/analyzer/intrinsic.h) contains definition of IKOS intrinsics that can be used in analyzed source code.

##### include/ikos/analyzer/analysis

* [include/ikos/analyzer/analysis/call_context.hpp](include/ikos/analyzer/analysis/call_context.hpp) contains definition of a call context and the call context factory.

* [include/ikos/analyzer/analysis/context.hpp](include/ikos/analyzer/analysis/context.hpp) contains definition of the global context of the analyzer.

* [include/ikos/analyzer/analysis/literal.hpp](include/ikos/analyzer/analysis/literal.hpp) contains definition of the literal factory. It converts an AR operand to an AR-independent format.

* [include/ikos/analyzer/analysis/liveness.hpp](include/ikos/analyzer/analysis/liveness.hpp) contains definition of the liveness analysis. It computes the set of live and dead variables for all functions.

* [include/ikos/analyzer/analysis/memory_location.hpp](include/ikos/analyzer/analysis/memory_location.hpp) contains definition of symbolic memory locations (global, stack, heap-allocated, etc), and the memory location factory.

* [include/ikos/analyzer/analysis/option.hpp](include/ikos/analyzer/analysis/option.hpp) contains definition of analysis options.

* [include/ikos/analyzer/analysis/variable.hpp](include/ikos/analyzer/analysis/variable.hpp) contains definition of variables (local, global, etc), and the variable factory.

##### include/ikos/analyzer/analysis/execution_engine

* [include/ikos/analyzer/analysis/execution_engine/context_insensitive.hpp](include/ikos/analyzer/analysis/execution_engine/context_insensitive.hpp) contains definition of `ContextInsensitiveCallExecutionEngine`, a call execution engine for context-insensitive analyses.

* [include/ikos/analyzer/analysis/execution_engine/engine.hpp](include/ikos/analyzer/analysis/execution_engine/engine.hpp) contains definition of base classes for execution engines. It defines an API to execute AR statements.

* [include/ikos/analyzer/analysis/execution_engine/inliner.hpp](include/ikos/analyzer/analysis/execution_engine/inliner.hpp) contains definition of `InlineCallExecutionEngine`, a call execution engine performing dynamic inlining.

* [include/ikos/analyzer/analysis/execution_engine/numerical.hpp](include/ikos/analyzer/analysis/execution_engine/numerical.hpp) contains definition of `NumericalExecutionEngine`, the main execution engine of the analyzer. It executes AR statements on an abstract domain.

##### include/ikos/analyzer/analysis/pointer

* [include/ikos/analyzer/analysis/pointer/constraint.hpp](include/ikos/analyzer/analysis/pointer/constraint.hpp) contains definition of `PointerConstraintsGenerator`, a generator of pointer constraints given an AR function or global variable.

* [include/ikos/analyzer/analysis/pointer/function.hpp](include/ikos/analyzer/analysis/pointer/function.hpp) contains definition of a function pointer analysis.

* [include/ikos/analyzer/analysis/pointer/pointer.hpp](include/ikos/analyzer/analysis/pointer/pointer.hpp) contains definition of a pointer analysis.

##### include/ikos/analyzer/analysis/value

* [include/ikos/analyzer/analysis/value/abstract_domain.hpp](include/ikos/analyzer/analysis/value/abstract_domain.hpp) contains definition the abstract domain used during the value analysis.

* [include/ikos/analyzer/analysis/value/interprocedural.hpp](include/ikos/analyzer/analysis/value/interprocedural.hpp) contains definition the interprocedural value analysis.

* [include/ikos/analyzer/analysis/value/intraprocedural.hpp](include/ikos/analyzer/analysis/value/intraprocedural.hpp) contains definition the intraprocedural value analysis.

* [include/ikos/analyzer/analysis/value/machine_int_domain.hpp](include/ikos/analyzer/analysis/value/machine_int_domain.hpp) contains definition the machine integer abstract domain used during the value analysis.

##### include/ikos/analyzer/checker

Contains definition of the different checks on the code (buffer overflow, division by zero, etc.), given the result of an analysis.

##### include/ikos/analyzer/database/table

Contains definition of the different output database tables.

##### include/ikos/analyzer/json

Contains definition of a JSON library.

##### include/ikos/analyzer/support

Contains various helpers, e.g, assertions.

##### include/ikos/analyzer/util

Contains definition of utilities for the analyzer, e.g, logging, colors, timers, etc.

#### python/

* [python/ikos/analyzer.py](python/ikos/analyzer.py) contains implementation of the `ikos` command line tool.

* [python/ikos/report.py](python/ikos/report.py) contains implementation of the `ikos-report` command line tool.

* [python/ikos/settings.py.in](python/ikos/settings.py.in) contains implementation of the `ikos-config` command line tool.

* [python/ikos/view.py](python/ikos/view.py) contains implementation of the `ikos-view` command line tool.

##### python/ikos/analyzer/view

Contains the web resources for ikos-view. It includes HTML, CSS and JS code.

#### script

Contains python entry points for the command line tools.

#### src/

Contains implementation files, following the structure of `include/ikos/analyzer`.

* [src/ikos_analyzer.cpp](src/ikos_analyzer.cpp) contains the implementation of `ikos-analyzer`. This is the entry point for all analyses.