# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

OpenBLAS is an optimized BLAS (Basic Linear Algebra Subprograms) library based on GotoBLAS2. It provides high-performance implementations of BLAS routines and includes LAPACK. The library supports numerous CPU architectures with hand-tuned assembly kernels.

## Build Commands

### Basic Build (Make)

```bash
make                          # Auto-detect CPU and build
make TARGET=HASWELL           # Build for specific CPU target
make -j$(nproc)               # Parallel build

make DEBUG=1                  # Debug build
make USE_OPENMP=1             # Build with OpenMP threading
make NO_LAPACK=1              # Build without LAPACK
make DYNAMIC_ARCH=1           # Multi-target library with runtime CPU detection
```

### CMake Build

```bash
mkdir build && cd build
cmake ..
cmake --build .
```

### Installation

```bash
make PREFIX=/path/to/install install
```

### Cross-Compilation

```bash
make CC=aarch64-linux-gnu-gcc FC=aarch64-linux-gnu-gfortran HOSTCC=gcc TARGET=ARMV8
```

## Testing

```bash
make -C test        # BLAS Fortran tests (requires Fortran compiler)
make -C ctest       # CBLAS tests
make -C utest       # OpenBLAS regression tests
make lapack-test    # LAPACK tests
make blas-test      # Netlib BLAS tests
```

## Key Build Variables

Set in `Makefile.rule` or on command line:

- `TARGET` - CPU target (see TargetList.txt for full list)
- `DYNAMIC_ARCH=1` - Build for multiple CPUs with runtime selection
- `USE_THREAD=0/1` - Disable/enable threading
- `USE_OPENMP=1` - Use OpenMP instead of pthreads
- `NUM_THREADS=N` - Maximum thread count
- `INTERFACE64=1` - 64-bit integer interface (ILP64)
- `NO_CBLAS=1` - Skip CBLAS interface
- `NO_LAPACK=1` - Skip LAPACK
- `BINARY=32/64` - Build bitness

## Architecture

### Source Layout

```
interface/          - BLAS/CBLAS API entry points
driver/level2/      - Level 2 BLAS implementations (matrix-vector)
driver/level3/      - Level 3 BLAS implementations (matrix-matrix)
driver/others/      - Memory management, threading, CPU detection
kernel/             - Optimized assembly kernels per architecture
  kernel/x86_64/    - x86-64 kernels
  kernel/arm64/     - ARM64 kernels
  kernel/power/     - POWER kernels
  kernel/generic/   - Portable C fallback kernels
lapack/             - Optimized LAPACK routines
lapack-netlib/      - Reference LAPACK from Netlib
```

### Call Flow (e.g., DGEMM)

```
interface/gemm.c → driver/level3/level3.c → kernel/<arch>/dgemm_kernel_*.S
```

### Kernel Selection

CPU-specific kernels are defined in `kernel/<arch>/KERNEL.<CPU>` files. Example from `kernel/x86_64/KERNEL.HASWELL`:
```
DGEMMKERNEL = dgemm_kernel_4x8_haswell.S
```

### Build System Flow

1. `Makefile.prebuild` runs first - builds and runs `getarch` for CPU detection
2. Generates `Makefile.conf` and `config.h`
3. Main build compiles interface, drivers, kernels, and LAPACK
4. For `DYNAMIC_ARCH=1`, kernels are built for each target in `DYNAMIC_CORE`

## Adding CPU Support

1. Add CPUID handling in `cpuid_<arch>.c` for `getarch` detection
2. Clone/create `kernel/<arch>/KERNEL.<CPU>` defining kernel files
3. Add parameter section in `param.h` with `GEMM_UNROLL` values
4. For `DYNAMIC_ARCH`, update `driver/others/dynamic.c` or `dynamic_<arch>.c`
5. Add to `TargetList.txt`

## Runtime Environment Variables

- `OPENBLAS_NUM_THREADS` - Set thread count
- `OPENBLAS_CORETYPE` - Force CPU kernel selection (with DYNAMIC_ARCH)
- `OPENBLAS_MAIN_FREE=1` - Disable CPU affinity

## Key Files

- `Makefile.rule` - All configurable build options with descriptions
- `Makefile.system` - Core build system logic
- `TargetList.txt` - Supported CPU targets
- `param.h` - CPU-specific tuning parameters (block sizes, unroll factors)
