# cuda-aware-mpi-example
Adapted NVIDIA code example for ALCF Polaris and Cray wrapper compilers

Original source: https://github.com/NVIDIA-developer-blog/code-samples/tree/master/posts/cuda-aware-mpi-example/src

Key takeaways:
- `-cuda` was getting accidentally passed to `nvcc` (meant to be passed only to `nvc++`); the flag works completely differently for both compilers. It tells `nvcc` to compile `.cu` input files to `.cu.cpp.ii` output files (something that needs to be compiled), not an object file. Hence the "file format not recognized" error when trying to link
- For `nvc++`, the `-cuda` flag might not be needed for this application; might only be useful for CUDA Fortran targets at this time? Help documentation is very sparse:
```
> nvc++ -help -cuda
         Enable CUDA; please refer to -gpu for target-specific options. 
```
See final section below for more details.

### Solution

- Use `MPICFLAGS= $(shell CC --cray-print-opts=cflags)` to get the `mpi.h` include path, etc. flags to be passed to `nvcc` at compile (not link) time for CUDA source code files that `#inlcude <mpi.h>`
- Link with `CC` or `cc` 

See my offline note, `nvcc_vs_nvc_vs_nvc++_etc_and_kokkos.txt` for more details.

### TODO
- [ ] Describe the differences between parallel range (C++2x) vs. parallel sequence (C++17) algorithms?
- [ ] Still unclear; does this application need `nvc++ -cuda` flag at link-time for interoperability with the `nvcc`-compiled `device.o`? Try swapping the commented-out line:

```
#CUDALDFLAGS=-L${CUDA_INSTALL_PATH}/lib64 -lcudart
CUDALDFLAGS=-L${CUDA_INSTALL_PATH}/lib64 -lcudart -cuda
```

- [ ] Summarize discussion from late summer 2022 in AMReX repo about this:


- https://github.com/AMReX-Codes/amrex/pull/2282
- https://github.com/AMReX-Codes/amrex/issues/2284
- https://github.com/AMReX-Codes/amrex/pull/2066

## General procedure for Cray MPICH wrapper compilers + `nvcc`
https://centers.hpc.mil/users/advancedTopics/Build_CUDA_src_with_MPI_when_mpih_not_found.html
> The Cray MPI environment is hidden from the user in general. Most platforms provide MPI-specific compiler wrappers such as mpif90 and mpicxx which adds in the proper include directories for `mpi.h`, etc. The Cray wrappers `ftn`,`cc`, and `CC` do this same thing but they apply to all builds regardless if the source needs MPI headers.

> Some CUDA source (`.cu`) files need MPI library calls and the `mpi.h` header but the CUDA compiler (nvcc) does not have knowledge of the include path to `mpi.h`. The the proper path to `<mpi dir>/include/mpi.h` can be tricky to find, especially with many MPI versions.

> The Cray MPI module (cray-mpich) provides the environmental variable `$MPICH_DIR` which points to the base of MPI distribution of whichever module version is loaded. Users can add `-I$(MPICH_DIR)/include` to the `nvcc` build line.

This (somewhat old) resource also recommends ``-I`mpicc --showme:incdirs` `` for OpenMPI platforms

## `nvc++ -cuda` and related flags
Latest version of the NVIDIA HPC SDK as of writing is 22.5. 

### GTC 2021 video 

Bryce Adelstein Lelbach's GTC 2021 talk promises NVC++ programming model interoperability as "coming soon":
```
nvc++ -stdpar -cuda -acc -mp
```
all in the same translation unit. ABI compatibility with each other and NVCC. 
No more `__host__`, `__device__` annotations, `__CUDA_ARCH_`. Execution space inference

#### Summary of progress / new features
First release of NVIDIA HPC SDK was 20.7.
- `nvc++ -stdpar=gpu` supported since NVHPC 20.7
- `nvc++ -stdpar=multicore` supported since NVHPC 21.3
- `nvfortran` 20.11 : standard **array** intrinsics
- `nvc++` and `nvfortran` 21.3 offer limited support for OpenMP Target offload for GPUs

### Compiler reference guide
The compiler **reference guide** (vs. [user guide](https://docs.nvidia.com/hpc-sdk/compilers/hpc-compilers-user-guide/index.htm) ) is similarly terse to above `-help` output, and makes it seem like `-cuda` is only used for the `nvfortran` compiler frontend:
- https://docs.nvidia.com/hpc-sdk/compilers/hpc-compilers-ref-guide/
- https://docs.nvidia.com/hpc-sdk/pdf/hpc22ref.pdf

> `-cuda`
> 
> Enable CUDA; please refer to -gpu for target-specific options.

> Usage
> >
> The following command-line requests that CUDA interoperability be enabled and CUDA Fortran syntax be recognized and processed in all Fortran files.
```
$ nvfortran -cuda myprog.f
```

But the user guide (see below) explicitly mentions OpenACC and OpenMP interoperatibiltiy with CUDA, so this is likely just an example / outdated documentation from when the CUDA Fortran extension was the only thing handled by the PGI compiler predecessor?

More compiler option references have multiple ambiguous and/or conflicting descriptions (e.g. in the summary section vs. the option section):

> `-gpu`
> 
> Used in combination with the `-stdpar`, `-acc` and `-cuda` flags to specify options for GPU code generation.
> 
> Specify details of GPU code generation including compute capability, CUDA version and more.

> `-acc`
> 
> Enable OpenACC directives. 
> 
> Enable OpenACC directives for GPUs (default) or multicore CPUs.

> `-stdpar`
> 
> Enable ISO C++17 Parallel Algorithms behavior; please refer to `-gpu` for target-specific options
>
> Enable parallelization/offload of Standard C++ and Fortran parallel constructs; default is -stdpar=gpu.

KGF: ISO Fortran 2008 also supports a StdPar, `do concurrent`.

### Manpage and compiler user guide
The manpage is the most explicit endorsement that `-cuda` enables CUDA C++ compilation with `nvc++`, not just interoperability. 

```
> man nvc++

If coinstalled with nvfortran, Fortran file suffixes are also recognized and compiled with the nvfortran
compilers; see nvfortran and the  NVIDIA User's Guide.  Other files are passed to the linker (if linking
is requested) with a warning message.
       
-cuda[=option[,option...]
Enable CUDA C++ or CUDA Fortran, and link with the CUDA runtime libraries.  -cuda is required
on the link line. If -cuda is used in compilation, it must also be used for linking.  For
target specific options, please refer to -gpu.  For linking additional CUDA libraries (e.g.
math libraries), please refer to -cudalib.  Use the following sub-options to tailor the
compilation of CUDA Fortran accelerator regions:
```

https://docs.nvidia.com/hpc-sdk/compilers/hpc-compilers-user-guide/index.html

They (`nvc++`, `nvc`, `nvfortran` compilers) work in conjunction with an assembler, linker, libraries and header files on your target system, and include a CUDA toolchain, libraries and header files for GPU computing.
                 
Rather, all of these details are implicit in the programming model and are managed by the NVIDIA HPC SDK Fortran, C⁠+⁠+ and C compilers. GPU programming with CUDA extensions gives you access to all NVIDIA GPU features and full control over data management and offloading of compute-intensive loops and kernels.
                 
The NVIDIA HPC Compilers support interoperability between with CUDA Unified Memory.
                 
The NVIDIA HPC compilers use components from NVIDIA's CUDA Toolkit to build programs for execution on an NVIDIA GPU. The NVIDIA HPC SDK puts the CUDA Toolkit components into an HPC SDK installation sub-directory; the HPC SDK typically bundles three versions of recently-released Toolkits.
                 
The HPC Compilers support interoperability of OpenMP and CUDA to the same extent they support CUDA interoperability with OpenACC.
