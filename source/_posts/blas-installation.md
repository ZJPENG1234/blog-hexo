---
title: BLAS Installation
---

`BLAS`(Basic Linear Algebra Subprograms) has many implementations, mainly `ATLAS`, `GotoBLAS2`, `OpenBLAS`, `MKL`, `Eigen3`. This note is about how I **build & install & link** them.

### ATLAS
-----------
**Info**

 - Page: [http://math-atlas.sourceforge.net/](http://math-atlas.sourceforge.net/)
 - Version: 3.10.3 (2016-07-28)
 - Source Code: [https://sourceforge.net/projects/math-atlas/files/](https://sourceforge.net/projects/math-atlas/files/)
 - Installation Doc: ./INSTALL.txt

**Build Before**

Make sure CPU THROTTLING is turned off. See INSTALL.txt

The following code is how ATLAS checks whether CPU THROTTLING is turned off
```
// ./CONFIG/src/backend/archinfo_linux.c Line 530-572
/*
 * If cpufreq directory doesn't exist, guess no throttling.  If
 * cpufreq exists, and cur Mhz < max, throttling is enabled,
 * throttling also enabled if governer is not "performance", and min freq
 * is less than max
 */
```
So in BIOS, I set fixed 3.2 Mhz for all CPUs.

**Build**

```bash
$ mkdir my_build_dir ; cd my_build_dir
$ /path/to/ATLAS/configure [flags]
$ make              ! tune and compile library
$ make check        ! perform sanity tests
$ make ptcheck      ! checks of threaded code for multiprocessor systems
$ make time         ! provide performance summary as % of clock rate
```

**Install**
```bash
$ make install --prefix=/path/to/install
```
or
```bash
$ make install DESTDIR=/path/to/install
```

**How to use?**

Assume ATLAS is installed in <ATLAS_ROOT>
```bash
$ gcc test_blas.c -I<ATLAS_ROOT>/include -L<ATLAS_ROOT>/lib -lcblas -latlas -o test_blas.out
```

### GotoBLAS2
-----------
**Info**

 - Page: [https://www.tacc.utexas.edu/research-development/tacc-software/gotoblas2](https://www.tacc.utexas.edu/research-development/tacc-software/gotoblas2)
 - Version: 1.13
 - Source Code: [https://www.tacc.utexas.edu/documents/1084364/1087496/GotoBLAS2-1.13.tar.gz](https://www.tacc.utexas.edu/documents/1084364/1087496/GotoBLAS2-1.13.tar.gz)
 - Installation Doc: 

**Bug Fix:**
1. `./driver/others/dynamic.c Line 183-187` should be the following:
```c
#ifdef ARCH_X86
	if (gotoblas == NULL) gotoblas = &gotoblas_KATMAI;
#else
	if (gotoblas == NULL) gotoblas = &gotoblas_PRESCOTT;
#endif
```

2. `./f_check Line 266` should be the following:
```perl
($flags =~ /^\-l\w+/) 
```

**Build**

```bash
$ make
```
If `make` fails, try `make DYNAMIC_ARCH=1`

**Install**

Just copy all files into destination path.

**How to use?**

Assume GotoBLAS2 is installed in <GotoBLAS2_ROOT>
```bash
$ gcc ./test_blas.c -o ./test_blas.out -DEXPRECISION -m128bit-long-double -Wall -m64 -DF_INTERFACE_GFORT -fPIC  -DDYNAMIC_ARCH -DMAX_CPU_NUMBER=4  -w  -I<GotoBLAS2_ROOT> -L<GotoBLAS2_ROOT> -lgoto2 -lm -lgfortran -lquadmath -lc
```


### OpenBLAS
----------
> OpenBLAS is an optimized BLAS library based on GotoBLAS2 1.13 BSD version.

**Info**

 - Page: [https://www.openblas.net/](https://www.openblas.net/)
 - Github: [https://github.com/xianyi/OpenBLAS](https://github.com/xianyi/OpenBLAS)

**Build & Install**

[https://github.com/xianyi/OpenBLAS/wiki/Installation-Guide](https://github.com/xianyi/OpenBLAS/wiki/Installation-Guide)
```bash
$ make
$ make install --prefix=/path/to/install
```

**How to use?**
Assume OpenBLAS is installed in <OPEN_ROOT>
```bash
gcc ./test_blas.c -o ./test_blas.out -I<OPEN_ROOT> -L<OPEN_ROOT> -lopenblas
```

### MKL
---------
> Intel® Math Kernel Library (Intel® MKL) optimizes code with minimal effort for future generations of Intel® processors. It is compatible with your choice of compilers, languages, operating systems, and linking and threading models.

MKL has overall documents, so everything will be easy.

**Info**

 - Page: [https://software.intel.com/en-us/mkl](https://software.intel.com/en-us/mkl)
 - Doc: [https://software.intel.com/en-us/mkl/documentation](https://software.intel.com/en-us/mkl/documentation)


**Build & Install**

After download, it's compiled, so don't need build and install step.

**How to use?**

You can set environment variables, see [here](https://software.intel.com/en-us/mkl-linux-developer-guide-linking-on-intel-64-architecture-systems): *Getting Started*.

Also you can link directly, like [this](https://software.intel.com/en-us/mkl-linux-developer-guide-linking-on-intel-64-architecture-systems): *Linking Your Application with the Intel® Math Kernel Library*.



### Eigen3
----------
> Eigen is a C++ template library for linear algebra: matrices, vectors, numerical solvers, and related algorithms.

**Info**

 - Page: [http://eigen.tuxfamily.org](http://eigen.tuxfamily.org)
 - Github: [https://github.com/eigenteam/eigen-git-mirror](https://github.com/eigenteam/eigen-git-mirror)

**Build & Install**
```bash
$ mkdir build && cd build
$ cmake .. --DCMAKE_INSTALL_PREFIX=/path/to/install
$ make install
```

**How to use?**
Assume Eigen3 is installed in <EIGEN3_ROOT>
```bash
$ gcc -I<EIGEN3_ROOT> ./test_eigen.cpp -o ./test_eigen.out
```

### Appendix
---------
```c
// test_blas.c

#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <sys/time.h>

#ifndef USING_MKL
    #include "cblas.h"
#else
    #include "mkl_cblas.h"
#endif

#define MAX (int)(1e5)
typedef double T;   //S, D, C, Z

T *gen_matrix(int m,  int n, float sparsity)
{
    int i;
    T *mtx = (T *)malloc(sizeof(T) * m * n);
    for(i = 0; i < m*n; i++)
    {
        if(rand() < sparsity * RAND_MAX)
            mtx[i] = 0;
        else
            mtx[i] = rand() % MAX;
    }
    return mtx;
}

int main()
{

    double alhpa = 2.0, beta = 2.0;
    int m = 500, n = 500, k = 500;
    T *A = gen_matrix(m, k, 0);
    T *B = gen_matrix(k, n, 0);
    T *C = gen_matrix(m, n, 0);
    
    print_matrix(A, m, k);
    print_matrix(B, k, n);
    print_matrix(C, m, n);
    
    struct timeval start, end;
    gettimeofday(&start, 0);
    
    cblas_dgemm(CblasRowMajor, CblasNoTrans, CblasNoTrans, 
        m, n, k, alhpa, A, k, B, n, beta, C, n);
        
    gettimeofday(&end, 0);
      
    printf(
            "\nRoutine: C <- alpha*op(A)op(b) + beta*C\n"
            "  M = %d, K = %d, N = %d\n"
            "  Wall time is %ld us\n", 
            m, k, n, (end.tv_sec-start.tv_sec)*1000000 + (end.tv_usec - start.tv_usec));
    free(A);
    free(B);
    free(C);
    return 0;
}

```