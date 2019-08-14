# Instructions for Petascale Institute on Blue Waters

Author: [Matt Norman](https://mrnorman.github.io)

## Getting Started
First, log in with:

```
ssh -Y [username]@bwbay.ncsa.illinois.edu
```

Then clone this repo

```
git clone git@github.com:mrnorman/miniWeather.git
cd miniWeather
```

Choose either the `c` or `fortran` directories to work with, depending on your language preference.
All documentation is in [documentation/miniWeather_documentation.pdf](https://github.com/mrnorman/miniWeather/blob/master/documentation/miniWeather_documentation.pdf).

This code can be used to learn three different parallel programming aspects:
1. MPI task-level parallelism
2. OpenMP loop-level parallelism
3. OpenACC accelerator parallelism

For this particular context, we are focusing on the OpenACC parallelism, so the goal is to add your own OpenACC directives to `miniWeather_mpi.F90`. Given the extremely short time frame, though, for this exercise, it might be more appropriate to simply look at how `miniWeather_mpi.F90` and `miniWeather_mpi_openacc.F90` differ, where the OpenACC directives have already been introduced. There is some information in the Petascale Institute presentation as well as the documentation.pdf file in this repo regarding how to add OpenACC directives.

## Compiling the Code

To compile the code, you first need to create the correct environment with:

```
module swap PrgEnv-cray PrgEnv-pgi
module swap pgi pgi/18.7.0
module load cudatoolkit cray-parallel-netcdf
```

Use the `Makefile.bw` to compile the code:

```
cd miniWeather/[c | fortran]
make -f Makefile.bw
```

## Running the Code and Viewing the Results

To start an interactive 1-node job for an hour that contains a K20x GPU, use:

```
qsub -I -A [projectid] -lwalltime=01:00:00,nodes=1:ppn=16:xk
```

Setup the correct environment:

```
module swap PrgEnv-cray PrgEnv-pgi
module swap pgi pgi/18.7.0
module load cudatoolkit cray-parallel-netcdf ncview
```

Run the code:

```
aprun -n 1 ./miniweather_mpi_openacc
```

OR

```
aprun -n 1 ./miniweather_mpi
```

View the results

```
ncview output.nc
```

**IMPORTANT!**: Do not run `./miniweather*` without `aprun`, as this will run it either on the login node or a service node. Those nodes are shared resources with limited resources, and you risk keeping others from compiling their code or even potentially killing other peoples' jobs if you cause an OOM on a service node. So please, be a good neighbor, and always use `aprun` for anything computationally intensive.

## Playing with the code

### Changing the Problem Size

Find the `BEGIN USER-CONFIGURABLE PARAMETERS` in the source file, and change the number of grid points and the data specification according to the options in `documentation.pdf` in this repo. One interesting thing you can do is see how the OpenACC GPU efficiency changes as you change the problem size in terms of number of grid cells. The smaller the workload becomes, the worse the GPU performance should get because you begin to have too few threads to keep the device busy.

### Profiling

You can profile the code with:

```
aprun -n 1 nvprof -o prof.nvvp ./miniWeather_mpi_openacc
```

and view the profile with

```
nvvp prof.nvvp
```

You will need X11 forwarding for this, though. To view a reduced text-only profile, use:

```
aprun -n 1 nvprof --print-gpu-summary ./miniWeather_mpi_openacc
```

### Using Managed Memory

The PGI compiler allows you to use Managed Memory instead of explicit data statements via the `-ta=nvidia,managed` flag. Try editing the Makefile to use Managed Memory, and see how the performance of the code changes. To do this, change the `ACCFLAGS` in `Makefile.bw` to:

```
ACCFLAGS := -ta=tesla,pinned,cc35,managed,ptxinfo -Minfo=accel 
```

### Debugging with PGI's PCAST

The PGI compiler also has a neat tool called PCAST, which automatically compares variables from redundatly executed CPU and GPU versions of the OpenACC code every time you move data from GPU memory to CPU memory (e.g., `update host(...)` or `copyout(...)` or the end of a block that has `copy(...)`). You can also control the size of the absolute or relative differences that will trigger terminal output. Change your `ACCFLAGS` in `Makefile.bw` to:

```
ACCFLAGS := -ta=tesla,pinned,cc35,autocompare,ptxinfo -Minfo=accel 
```

For more options regarding the PCAST tool, see: https://www.pgroup.com/blogs/posts/pcast.htm
