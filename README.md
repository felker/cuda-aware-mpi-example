# cuda-aware-mpi-example
Adapted NVIDIA code example for ALCF Polaris and Cray wrapper compilers

Original source: https://github.com/NVIDIA-developer-blog/code-samples/tree/master/posts/cuda-aware-mpi-example/src

Key takeaways:
- `-cuda` getting accidentally passed to `nvcc` (meant to be passed only to `nvc++`) tells `nvcc` to compile `.cu` input files to `.cu.cpp.ii` output files (something that needs to be compiled), not an object file. hence the unrecognized format when trying to link
- 

## General procedure for Cray MPICH wrapper compilers + `nvcc`

- https://centers.hpc.mil/users/advancedTopics/Build_CUDA_src_with_MPI_when_mpih_not_found.html
> The Cray MPI environment is hidden from the user in general. Most platforms provide MPI-specific compiler wrappers such as mpif90 and mpicxx which adds in the proper include directories for `mpi.h`, etc. The Cray wrappers `ftn`,`cc`, and `CC` do this same thing but they apply to all builds regardless if the source needs MPI headers.

> Some CUDA source (`.cu`) files need MPI library calls and the mpi.h header but the CUDA compiler (nvcc) does not have knowledge of the include path to mpi.h. The the proper path to `<mpi dir>/include/mpi.h` can be tricky to find, especially with many MPI versions.

> The Cray MPI module (cray-mpich) provides the environmental variable `$MPICH_DIR` which points to the base of MPI distribution of whichever module version is loaded. Users can add `-I$(MPICH_DIR)/include` to the `nvcc` build line.

This (somewhat old) resource also recommends `-I\`mpicc --showme:incdirs\`` for OpenMPI platforms



NVIDIA itself is still not touting this capability, so neither are we. I expect this by the end of the summer to work well enough for adventures users to try it out

this is very much a codesign effort right now, where NVIDIA is using Kokkos as a driver for compiler development

I also would think that NVIDIA will do proper announcements then, Kokkos working will probably be part of that announcement as an example of some production stuff using nvhpc


you can use nvc++ as the host compiler for nvcc
just add --ccbin nvc++ to your cmake_cxx_flags
that should be working and has been working for a while

it should, I think we test that for CI even
that is a distinct capability from using nvc++ as the CUDA compiler
without nvcc
i.e. when use nvc++ as the host compiler for nvcc, the CUDA parsing of nvc++ is not turned on

basically you have to turn it on with something like -cuda -gpu cc_70 or so as flags

if you don't have those it will just be a C++ CPU compiler
