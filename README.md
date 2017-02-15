Title: QETSc
Date: 2017-01-03 01:54
Category: Readme
# SIESTA-QETSc
Siesta-QETSc is a branch of Siesta (see README_SIESTA) distinguished by
a state-of-the-art parallel sparse eigensolver, which significantly improves the
performance of the code for large calculations i.e. more than a few hundred atoms
or a few thousand basis functions.

For more information on the eigensolver you can check the following papers:

1. Zhang, H.; Smith, B.; Sternberg, M.; Zapol, P. SIPs: Shift-and-Invert Parallel Spectral Transformations. ACM Trans. Math. Softw. 2007, 33, 9–es.
2. Campos, C.; Román, J. E. Strategies for Spectrum Slicing Based on Restarted Lanczos Methods. Numer. Algorithms 2012, 60, 279–295.
3. Keçeli, M.; Zhang, H.; Zapol, P.; Dixon, D. A.; Wagner, A. F. Shift-and-Invert Parallel Spectral Transformation Eigensolver: Massively Parallel Performance for Density-Functional Based Tight-Binding. J. Comput. Chem. 2016, 37, 448–459.

# How to install?

Siesta-QETSc uses the spectrum slicing eigensolver in SLEPc package.
SLEPc is an extension of PETSc. Hence, we need to install PETSc initially.
PETSc also manages the installation of other libraries that are required for
the eigensolver, i.e. MUMPS, METIS, ScaLAPACK etc.


## PETSc installation:
```
git clone https://bitbucket.org/petsc/petsc
cd petsc
export PETSC_DIR=$PWD
export PETSC_ARCH=arch-gcc
git checkout c616f458c34eeb4fc62c # Temporary solution to avoid recent changes in PETSc
./configure  --download-mpich --download-fblaslapack --download-scalapack --download-mumps --download-metis --download-ptscotch --download-scalapack --with-shared-libraries=0 --with-debugging=0
make all test
```
For more information on PETSc installation check https://www.mcs.anl.gov/petsc/documentation/installation.html
For better performance use vendor specific libraries, especially for blas and lapack.
However, we do not recommend MKL, since we encountered some problems with older versions. (More info needed)

## SLEPc installation:
```
git clone https://bitbucket.org/keceli/slepcfork
cd slepcfork
export SLEPC_DIR=$PWD
./configure
make all test
```
## Siesta installation:
```
git clone https://bitbucket.org/keceli/siesta-qetsc.git
cd siesta-qetsc
mkdir $PETSC_ARCH
cd $PETSC_ARCH
sh ../Src/obj_setup.sh
../Src/configure --enable-mpi CC=${PETSC_DIR}/${PETSC_ARCH}/bin/mpicc FC=${PETSC_DIR}/${PETSC_ARCH}/bin/mpif90 
```
Edit `arch.make` file to change LIBS and FFLAGS as:
```
LIBS="${SLEPC_EPS_LIB} $(SCALAPACK_LIBS) $(BLACS_LIBS) $(LAPACK_LIBS) $(BLAS_LIBS) $(NETCDF_LIBS)" 
FFLAGS=-g -I${PETSC_DIR}/include -I${PETSC_DIR}/${PETSC_ARCH}/include -I${SLEPC_DIR}/${PETSC_ARCH}/include -I${SLEPC_DIR}/include
```
Check arch.make file to make sure LIBS includes ${SLEPC_EPS_LIB}.

# How to run?

Create input file as described in Siesta manual, or see examples in Tests or Examples folder
Currently, you can only use INPUT_DEBUG as your input file due to a problem with fdf and petsc.
Once you have INPUT_DEBUG file ready, you can simply use
```
mpiexec -n 2 siesta
```
to run siesta without specifying the input file, since siesta will read INPUT_DEBUG
as the default input file.

## How to use QETSc solver?

You have to set SolutionMethod to qetsc in INPUT_DEBUG, i.e.:
```
SolutionMethod  qetsc
```
Run with the following runtime options
```
mpiexec -np 2 `siesta` -options_file options.txt
```
`options.txt` file contains command line options that can be set at run time for SIESTA-QETSc runs.
Here is a sample `options.txt` file with explanations on their usage.
You can also fine options.txt file and sample siesta input files in `SIESTA_DIRECTORY/Tests/qetsc`
directory.

``` bash
# SLEPc eigensolver options
# -eps_interval a,b: where a and b are real numbers setting the global interval [a,b] for the eigenvalue search.
-eps_interval -3.0,1.0
# -eps_krylovschur_partitions n: where n is an interger setting the number of bins (slices) for dividing the global interval
# n should evenly divide the number of processors. This number has an impact on the performance.
# In general it can be set equal to the number of processors. i.e. for `mpirun -np 16 siesta -options_file options.txt`
-eps_krylovschur_partitions 16

# PETSc options
-mat_type mpisbaij # use a sym. matrix, reducing memory footprint
-log_view # Prints out a very useful profile log
-memory_view # Prints out memory usage for petsc operations. (On some systems, it doesn't work)
-mat_mumps_icntl_7 5 # Sets Metis for ordering rows/columns (Serial)

# QETSc options

# Below are the options set as default, so the user does not need to change any
#-ioptbin 7 # Sets the binning algorithm, use 0 for uniform slicing, 7 is the best nonuniform slicing
#-ioptdensity 0 # Sets the method for density matrix calculation
#-ioptinertia 1 # Perform inertia calculation until eigenvalues starts to converge
#-ioptwrite 1 # Write eigenvalues in log file, use 2 to write matrices to disk
#-roptbuffer 0.1 # Buffer value for end points of the interval
#-roptdiff 0.02 # Convergence threshold for eigenvalue, to stop inertia calculations
```

## Troubleshooting

1. If there are missing eigenvalues, i.e SIESTA-QETSc reports:
`Not enough eigenvalues`

You can try any or all of the following:
    1. Increase global interval for eigenvalue range,
-eps_interval -15,5
    2. Increase roptbuffer
-roptbuffer 0.5
    3. Increase ioptinertia
-ioptinertia 100
    4. Decrease roptdiff
-roptdiff 0.0001

2. If you get an error from MUMPS during factorization, i.e. 
`Error reported by MUMPS in numerical factorization phase: INFOG(1)=-9, `

You can try any or all of the following.

    1. Decrease number of bins (slices, partitions) using  -eps_krylovschur_partitions
    2. Change to parallel symbolic factorization
```
-mat_mumps_icntl_28 2
-mat_mumps_icntl_29 1 # ptscotch
#-mat_mumps_icntl_29 2 # parmetis
```
    3. Set available memory per core in MB:
`-mat_mumps_icntl_23 2500`
If none of these help, the problem might be too big for the machine you use, either change the problem or the computer.


## More configure options from Siesta manual:

* -DMPI_TIMING: to obtain the accounting of MPI communication times in parallel executions
* -DGRID_SP: to the DEFS variable in arch.make to use single-precision for all the grid
magnitudes, including the orbitals array and charge densities and potentials. This will
cause some numerical dierences and will have a negligible eect on memory consumption,
since the orbitals array is the main user of memory on the grid, and it is single-precision
by default. This setting will recover the default behavior of previous versions of Siesta.
* -DGRID_DP: to the DEFS variable in arch.make to use double-precision for all the grid
magnitudes, including the orbitals array. This will significantly increase the memory used
for large problems, with negligible diferences in accuracy.
* -DBROYDEN_DP: to the DEFS variable in arch.make to use double-precision arrays for the
Broyden historical data sets. (Remember that the Broyden mixing for SCF convergence
acceleration is an experimental feature.)
* -DON_DP: to the DEFS variable in arch.make to use double-precision for all the arrays
in the O(N) routines.
