# DEMAGIS: a DEnse MAtrix Generator with gIve Spectrum

**DEMAGIS** provides a dense matrix generator which is able to generate large-scale symmetric (Hermitian) matrices with given spectral distributions. The objective of DEMAGIS is to provide large-scale test suite for the benchmark and evaluation of spectrum-based iterative eigensolvers.

**DEMAGIS** is written in C++, which has two building options: 

- **shared-memory build**: this configuration can be used when only one computing node is available. 

- **MPI+threads build**: this configuration is for homogeneous clusters. This implementation is extremely useful when the matrix is too large to be generated on a single computing node, either constrained by the memory requirement or time-to-generation.

In order to generate random matrices with given spectral distribution, the first step is to construct a diagonal matrix with entries on the diagonal being exactly the prescribed eigenvalues, and all other entries being zeros. Then a dense matrix with given spectra can be generated by as `A=Q^TDQ`, in which `Q` is an orthogonal matrix, and `Q^T` is its transpose matrix. If we want to generate a complex matrix, then `A` is generated as `A=Q^*DQ`, in which `Q` is an unitary matrix, and `Q*` is its conjugate transpose matrix. The orthogonal (unitary) matrix `Q` can be built by the Q factor matrix of a QR factorization on a square matrix of size `n*n` whose entries are randomly generated with respecting a Gaussian distribution.

- For the **shared-memory build**, it is implemented based on routines of QR factorization and Apply Q provided by LAPACK.

- For the **MPI+threads build**, it is implemented based on routines of QR factorization and Apply Q provided by ScaLAPACK.



## Build

### Dependencies

In addition to a recent `C++` compiler, DEMAGIS's external dependencies are [CMake](https://cmake.org), [BLAS](http://www.netlib.org/blas/), [LAPACK](https://www.netlib.org/lapack/), [ScaLAPACK](https://www.netlib.org/scalapack/) , and [MPI](https://www.mcs.anl.gov/research/projects/mpi/) . Please make sure that these dependencies are available on your computing platforms before the compile of DEMAGIS. To enhance the usability of the ready-to-use examples, it is also necessary to install the [Boost](https://www.boost.org/) library.

### Compile

1. Clone this repository from GitHub:

   ```bash
   git clone xxxxxxxx
   ```

2. Compile it with `CMake`

   ```bash
   cd DEMAGIS 
   mkdir build
   cd build
   cmake ..
   make -j
   ```

   

### Usage

- shared-memory build: 

  - Using built-in spectral distribution: **Uniform Distribution** and **Geometric Distribution**

    ```c++
    // include the header of shared-memory build
    #include "matGen_lapack.hpp"
    // include the header myDist.hpp since we want to use built-in spectral distribution
    #include "myDist.hpp"
    // generate matrix A of type double, using built in Uniform Distribution
    // n: rank of sqaure dense matrix to be generated
    // mean and stddev: Mean value and Standard deviation value of Normal Distribution for the randomness
    // myUniformDist<double>: function which generates eigenvalues following a uniform distribution
    // n, eps, dmax: parameters of myUniformDist<double>
    double *A = matGen_lapack<double>(n, mean, stddev, myUniformDist<double>, n, eps, dmax);
    ```

    

  - Using user self-designed spectral distribution through `C++` `Lambda`.

    ```c++
    // include the header of shared-memory build
    #include "matGen_lapack.hpp"
    // generate matrix A of type double, using built in Uniform Distribution
    // n: rank of sqaure dense matrix to be generated
    // mean and stddev: Mean value and Standard deviation value of Normal Distribution for the randomness
    // generate the spectral distribution through a C++ Lambda
    double *A = matGen_lapack<double>(n, mean, stddev, 
                                      [](std::size_t n, int x){
                                        double *eigenv = new double[n];
                                        for(auto k = 0; k < n; k++){
                                          eigenv[k] = k * (x + 1);
                                        }
                                        return eigenv;
                                      }, 
                                      n, 1);
    ```

    

  - Save the generated matrix `A` into local binary for re-using as follows:

    ```c++
    #include "io.hpp"
    wrtMatIntoBinary<double>(A, path_out, n * n);
    ```

- MPI+threads build

  - Setup 2D grid of MPI, and ScaLAPACK context 

    ```c++
    // ScaLAPACK context which stores the information of 2D MPI grid and matrix distribution
    int ictxt;
    int val;
    blacs_get( &ictxt, &i_zero, &val );
    // create a dim0*dim1 2D MPI grid with column major
    blacs_gridinit( &ictxt, 'C', &dim0, &dim1 );
    ```

  - Generating matrix either with built-in or user-provided spectral distribution

    ```C++
    // include required headers
    #include "matGen_scalapack.hpp"
    #include "myDist.hpp"
    // ictxt: ScaLAPACK context created in previous step
    // N: global size of matrix to be generted
    // mbsize and nbsize: ScaLAPACK block size in the first/second dimension of 2D MPI cartesian grid
    // mean and stddev: Mean value and Standard deviation value of Normal Distribution for the randomness
    // myUniformDist<double>: function which generates eigenvalues following a uniform distribution
    // n, eps, dmax: parameters of myUniformDist<double>
    A = matGen_scalapack<double>(ictxt, N, mbsize, nbsize, 
                                 mean, stddev, myUniformDist<double>, N, eps, dmax);
    ```

  - Save the generated matrix `A` into local binary with parallel IO for re-using as follows:

    ```c++
    #include "io_mpi.hpp"
    wrtMatIntoBinaryMPI<double>(ictxt, A, path_out, N, mbsize, nbsize);
    ```

    

> **WARNING**: Due to the simplication of implementation of parallel IO, N should be divisible by mbsize and nbsize, and N/mbsize and N / nbsize should be divisible by dim0 and dim1, respectively.



## Examples

Two examples are provided, which helps user get familiar with DEMAGIS, one for shared-memory build and one for MPI+threads build.

- Arguments for the example of shared-memory build

```bash
Artificial Matrices MT: Options:
  -h [ --help ]                        show the help
  --N arg (=10)                        number of row and column of matrices to
                                       be generated.
  --dmax arg (=21)                     A scalar which scales the generated
                                       eigenvalues, this makes the maximum
                                       absolute eigenvalue is abs(dmax).
  --epsilon arg (=0.10000000000000001) This value is epsilon.
  --myDist arg (=0)                    Specifies my externel setup distribution
                                       for generating eigenvalues:
                                        0: Uniform eigenspectrum lambda_k =
                                       dmax * (epsilon + k * (1 - epsilon) / n
                                       for k = 0, ..., n-1)
                                        1: Geometric eigenspectrum lambda_k
                                       =lambda_k = epsilon^[(n - k) / n] for k
                                       = 0, ..., n-1)
                                       2: 1-2-1 matrix
                                       3: Wilkinson matrix

  --mean arg (=0.5)                    Mean value of Normal distribution for
                                       the randomness.
  --stddev arg (=1)                    Standard deviation value of Normal
                                       distribution for the randomness.
```

- Arguments for the example of MPI+threads build

```bash
Artificial Matrices with ScaLAPACK: Options:
  -h [ --help ]                        show the helpAttention, for the current
                                       implementation of parallel IO, please
                                       make sure N/mbsize/dim0 == 0 and
                                       N/bbsize/dim1 == 0
  --N arg (=10)                        number of row and column of matrices to
                                       be generated.
  --dim0 arg (=1)                      first dimension of 2D MPI cartesian
                                       grid.
  --dim1 arg (=1)                      second dimension of 2D MPI cartesian
                                       grid.
  --mbsize arg (=5)                    ScaLAPACK block size in the first
                                       dimension of 2D MPI cartesian grid.
  --nbsize arg (=5)                    ScaLAPACK block size in the second
                                       dimension of 2D MPI cartesian grid.
  --dmax arg (=1)                      A scalar which scales the generated
                                       eigenvalues, this makes the maximum
                                       absolute eigenvalue is abs(dmax).
  --epsilon arg (=0.10000000000000001) This value is epsilon.
  --myDist arg (=0)                    Specifies my externel setup distribution
                                       for generating eigenvalues:
                                        0: Uniform eigenspectrum lambda_k =
                                       dmax * (epsilon + k * (1 - epsilon) / n
                                       for k = 0, ..., n-1)
                                        1: Geometric eigenspectrum lambda_k
                                       =lambda_k = epsilon^[(n - k) / n] for k
                                       = 0, ..., n-1)

  --mean arg (=0.5)                    Mean value of Normal distribution for
                                       the randomness.
  --stddev arg (=1)                    Standard deviation value of Normal
                                       distribution for the randomness.
```

## Copyright and License

MIT License

