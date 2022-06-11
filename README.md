# cuda-aware-mpi-example
Adapted NVIDIA code example for ALCF Polaris and Cray wrapper compilers

Original source: https://github.com/NVIDIA-developer-blog/code-samples/tree/master/posts/cuda-aware-mpi-example/src

Key takeaways:
- `-cuda` was getting accidentally passed to `nvcc` (meant to be passed only to `nvc++`); the flag works completely differently for both compilers. It tells `nvcc` to compile `.cu` input files to `.cu.cpp.ii` output files (something that needs to be compiled), not an object file. Hence the "file format not recognized" error when trying to link
- For `nvc++`, the `-cuda` flag:
> Enable CUDA; please refer to `-gpu` for target-specific options. 

See below for more details.

- Use `MPICFLAGS= $(shell CC --cray-print-opts=cflags)` to get the `mpi.h` include path, etc. flags to be passed to `nvcc` at compile (not link) time for CUDA source code files that `#inlcude <mpi.h>`
- Link with `CC` or `cc` 

See my offline note, `nvcc_vs_nvc_vs_nvc++_etc_and_kokkos.txt` for more details.

## General procedure for Cray MPICH wrapper compilers + `nvcc`

- https://centers.hpc.mil/users/advancedTopics/Build_CUDA_src_with_MPI_when_mpih_not_found.html
> The Cray MPI environment is hidden from the user in general. Most platforms provide MPI-specific compiler wrappers such as mpif90 and mpicxx which adds in the proper include directories for `mpi.h`, etc. The Cray wrappers `ftn`,`cc`, and `CC` do this same thing but they apply to all builds regardless if the source needs MPI headers.

> Some CUDA source (`.cu`) files need MPI library calls and the `mpi.h` header but the CUDA compiler (nvcc) does not have knowledge of the include path to `mpi.h`. The the proper path to `<mpi dir>/include/mpi.h` can be tricky to find, especially with many MPI versions.

> The Cray MPI module (cray-mpich) provides the environmental variable `$MPICH_DIR` which points to the base of MPI distribution of whichever module version is loaded. Users can add `-I$(MPICH_DIR)/include` to the `nvcc` build line.

This (somewhat old) resource also recommends `-I\`mpicc --showme:incdirs\`` for OpenMPI platforms

# `nvc++ -cuda` and related flags

> Usage
> The following command-line requests that CUDA interoperability be enabled and CUDA Fortran syntax be recognized and processed in all Fortran files.
```
$ nvfortran -cuda myprog.f
```

> `-gpu`
> Used in combination with the `-stdpar`, `-acc` and `-cuda` flags to specify options for GPU code generation. The following suboptions may be used following an equals sign ("="), with multiple sub-options separated by commas:

> Specify details of GPU code generation including compute capability, CUDA version and more.

> `-acc`
> Enable OpenACC directives.

> `-stdpar`
> Enable ISO C++17 Parallel Algorithms behavior; please refer to `-gpu` for target-specific options

(KGF: although not mentioned above, ISO Fortran 2008 also supports a StdPar, `do concurrent`)

> Enable parallelization/offload of Standard C++ and Fortran parallel constructs; default is `-stdpar=gpu`.

Bryce Adelstein Lelbach's GTC 2021 talk promises NVC++ programming model interoperability as "coming soon":
```
nvc++ -stdpar -cuda -acc -mp
```
all in the same translation unit. ABI compatibility with each other and NVCC. 
No more `__host__`, `__device__` annotations, `__CUDA_ARCH_`. Execution space inference


- `nvc++ -stdpar=gpu` supported since NVHPC 20.7
- `nvc++ -stdpar=multicore` supported since NVHPC 21.3

- `nvfortran` 20.11 : standard **array** intrinsics
- `nvc++` and `nvfortran` 21.3 offer limited support for OpenMP Target offload for GPUs
- [ ] Parallel range (C++2x) vs. parallel sequence (C++17) algorithms?
