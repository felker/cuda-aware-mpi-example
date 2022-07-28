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

**Need to try this on Polaris, not ThetaGPU. Need MPI-wrapped `nvc++`**

- [ ] Summarize discussion from late summer 2022 in AMReX repo about this:
- https://github.com/AMReX-Codes/amrex/pull/2282
- https://github.com/AMReX-Codes/amrex/issues/2284
- https://github.com/AMReX-Codes/amrex/pull/2066

## Instructions (and pre- vs. post- AT changes)
Needed to change the Makefile in d9b670fc71862ad7ded3468b7761713e1bc37f33 

See extensive notes about changes from Tcl -> Lua and `PrgEnv-nvidia` to `PrgEnv-nvhpc` (and cudatoolkit module behavior changes) in: https://github.com/felker/athenak-scaling

```
qsub -A datascience -q workq -I -l select=2:ncpus=64:ngpus=4,walltime=00:30:00 -j oe -S /bin/bash
cd ~/cuda-aware-mpi-example/src
make clean; make
cd ../bin
export MPICH_GPU_SUPPORT_ENABLED=1
mpiexec -np 8 --ppn 4 ./jacobi_cuda_aware_mpi -t 4 2 -d 128 128
```

Using defualt modules:
```
Currently Loaded Modules:
  1) craype-x86-rome          5) nvhpc/21.9          9) cray-pmi/6.1.2       13) PrgEnv-nvhpc/8.3.3
  2) libfabric/1.11.0.4.125   6) craype/2.7.15      10) cray-pmi-lib/6.0.17  14) craype-accel-nvidia80
  3) craype-network-ofi       7) cray-dsmml/0.2.2   11) cray-pals/1.1.7
  4) perftools-base/22.05.0   8) cray-mpich/8.1.16  12) cray-libpals/1.1.7
```

### GPUDirect newly broken as of 2022-07-27
1 node works fine:
```
> mpiexec -np 4 --ppn 4 ./jacobi_cuda_aware_mpi -t 2 2 -d 128 128
Topology size: 2 x 2
Local domain size (current node): 128 x 128
Global domain size (all nodes): 256 x 256
Starting Jacobi run with 4 processes using "NVIDIA A100-SXM4-40GB" GPUs (ECC enabled: 4 / 4):
Iteration: 0 - Residue: 0.249995
Iteration: 100 - Residue: 0.002388
Iteration: 200 - Residue: 0.001195
Iteration: 300 - Residue: 0.000795
Iteration: 400 - Residue: 0.000594
Iteration: 500 - Residue: 0.000474
Iteration: 600 - Residue: 0.000393
Iteration: 700 - Residue: 0.000336
Iteration: 800 - Residue: 0.000293
Iteration: 900 - Residue: 0.000260
Stopped after 1000 iterations with residue 0.000233
Total Jacobi run time: 3.1890 sec.
Average per-process communication time: 1.3001 sec.
Measured lattice updates: 20.23 MLU/s (total), 5.06 MLU/s (per process)
Measured FLOPS: 101.15 MFLOPS (total), 25.29 MFLOPS (per process)
Measured device bandwidth: 1.29 GB/s (total), 323.69 MB/s (per process)
```

vs. multiple nodes:
```
> mpiexec -np 8 --ppn 4 ./jacobi_cuda_aware_mpi -t 4 2 -d 128 128
...
aborting job:
Fatal error in PMPI_Sendrecv: Other MPI error, error stack:
PMPI_Sendrecv(249)........: MPI_Sendrecv(sbuf=0x1537d3200418, scount=128, MPI_DOUBLE, dest=0, stag=0, rbuf=0x1537d
3200008, rcount=128, MPI_DOUBLE, src=0, rtag=0, comm=0x84000002, status=0x7ffcadf8d490) failed
MPID_Isend(584)...........:
MPIDI_isend_unsafe(136)...:
MPIDI_OFI_send_normal(352): OFI tagged senddata failed (ofi_send.h:352:MPIDI_OFI_send_normal:Bad address)
x3202c0s19b1n0.hsn.cm.polaris.alcf.anl.gov: rank 4 exited with code 255
```

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
