+++
tags = ["F2PY", "Fortran", "Cython", "Numba"]
highlight = true
math = false
image = ""
date = "2017-02-07T21:00:01-07:00"
title = "Methods to write fast subroutines for Python - minimal computer science required"
summary = """
Speeding up research code in Python with Numpy, Numba, Cython and F2PY
"""

+++

---
>Note: I am not a Python or programming expert. There are probably better ways to optimize subroutines, but that is not the point of this post!. 

---

# Introduction 
Python is an impressive programming language for a beginning programmer/scientist. Syntax is easy to learn, the language is expressive (in the sense that it reads like it is doing things), and there are immense guides out there on the web to learn nearly any aspect of Python you can think of. I do not profess to be an expert of Python, by any means. I am self taught and know enough to make things happen for my research. I know (generally and definitely not enough) about the standard libraries, and I know enough about what *I want* to happen in my code that I can find libraries meaningful for my research, i.e. numpy matrix manipulations, sparse linear algebra and sklearn machine learning algorithms. 

However, as with any research project, you are on the cutting edge and sometimes functions you require are not implemented in standard packages, or you require access to specific portions of the functions to insert new ideas. Our research group is historically invested in Fortran, so naturally, new code is written in Fortran to leverage the vast library of existing codes. Many libraries can be used within Fortran, however, its commonly easier to prototype and test outside of Fortran.

# So where does Python fit in? 

Well, in the past few years Python scripting has become popular in our traditionally BASH dominated research group. There are so many reasons for choosing Python as a scripting language, and those are immediately clear if you've ever tried to script executables within the BASH universe while having to write insanely cryptic sed and awk one-liners, like this one: 

>awk '{ if (\$8 ~ \"isad\") {print \$2, \$5, \"SE\"} else {print \$2, \$5, \"ANT\"} }' $LOG_FILE '|' grep -v '127.0.0.1' '|' tr '\[' '\0' '|' tr [:lower:] [:upper:] '|' $SORT_CMD $SORT_KEY $SORT_REV

to do something with the data that comes out of the executables. Of course if you know BASH, then awesome. But for those new to programming, scripting, etc, this kind of thing is difficult to wrap your head around. The other nice part about Python is that it is not only an incredibly useful scripting language, but more and more it can be used for 'real' number crunching instead of offloading intense numerical workflows to standalone Fortran executable. The main issue with that is the loss of speed since Python is an interpreted language.

# Methods for *FAST* computations in Python

Keeping in mind that we are at the cutting edge and need to develop fast subroutines for complicated numerical analyses, and also that our definition of ***fast*** is to execute numerical calculations in the least amount of time (as opposed to querying databases or other non-numerical tasks), what are the ways that we can do this within our chosen Python scripting universe? Anyone who has traveled down this road is likely familiar with the major options, and there are benefits and drawbacks to each of them. It depends on the application, complexity of the code, and the intended use of the code. I am focused on numerical computations in my research using linear algebra, matrix solvers, and machine learning techniques. Since my research group primarily produces the standalone Fortran executables, [F2PY](https://docs.scipy.org/doc/numpy-dev/f2py/) project is attractive for mixing Python and Fortran code. There are other methods to call Fortran code from Python, too, like [ctypes](https://docs.python.org/3.5/library/ctypes.html) or a library called [cffi](http://cffi.readthedocs.io/en/latest/). However, all of these methods require fairly polished and working Fortran code that can be compiled and runs with minimal and easy to trace errors. This may not be completely suitable for speeding up research code since tracking down and fixing errors is difficult, especially for the newcomer. Compiling the code can be a headache in itself, especially if you are working on windows (which I am). As a beginning programmer and researcher learning about designing algorithms I guarantee that the flexibility in debugging and testing out subroutines that Python provides will improve the development of your ideas.

A number of projects address the need for fast Python subroutines, like [Numpy](http://www.numpy.org/), [PyPy](http://pypy.org/),  [numba](http://numba.pydata.org/), [cython](http://cython.org/), and many many many others.... I'm sure. So, for the beginning researcher? Which methodology provides the most benefit with the least amount of investment? I will focus on the following mainly because they are the ones that I have tried: 

1. [F2PY]({{< ref "fast_subroutines.md#F2PY" >}})
2. [Numpy]({{< ref "fast_subroutines.md#numpy" >}})
3. [Numba]({{< ref "fast_subroutines.md#numba" >}})
4. [Cython]({{< ref "fast_subroutines.md#cython" >}})

Each of the above methods attacks the problem in different ways. F2PY lets you compile Fortran code into easily callable python modules. Numpy provides a vast library of numerical computation functions that might allow you to skip the lower level languages all together. Numba uses 'just in time' (JIT) compiling to 'compile' native python code. This can be advantageous when the code consists of a number of loops with relatively simple mathematical operations. And finally Cython allows you to write static-typed 'Python' that is compiled to C++ and taken to the extreme using Cython would allow you to write an entire project essentially in C++. 

# Example where fast code is needed {#example}
Lets consider an example of computing the gradients on a 3D grid using the forward-reverse difference at each cell. Lets start with un-vectorized Python code using loops (pretty much the worst case and something that should never be done in Python). We also assume that we don't have any knowledge about specialized libraries that have already implemented this feature. Lets also assume that writing out the code like this benefits our research so we can add in some ad-hoc calculations in the grid that *couldn't* be handled if we used an existing package to get the gradients:

```python
import numpy as np

def calc_gradients(grid):
    nx, ny, nz = np.shape(grid)  # assumes the grid is 3D
    gx = np.zeros((nx, ny, nz))
    gy = np.zeros((nx, ny, nz))
    gz = np.zeros((nx, ny, nz))
    # the dreaded triple loop, a naive python implementation
    for k in range(nz):
        for j in range(ny):
            for i in range(nx): 
                # limit the array indexes, remember about Python 0-indexing
                ist = np.maximum(0, i - 1)
                jst = np.maximum(0, j - 1)
                kst = np.maximum(0, k - 1)
                ifn = np.minimum(nx - 1, i + 1)
                jfn = np.minimum(ny - 1, j + 1)
                kfn = np.minimum(nz - 1, k + 1)
                # calculate the gradients
                gx[i, j, k] = ( grid[ist, j, k] - grid[ifn, j, k] ) / 2.0
                gy[i, j, k] = ( grid[i, jst, k] - grid[i, jfn, k] ) / 2.0
                gz[i, j, k] = ( grid[i, j, kst] - grid[i, j, kfn] ) / 2.0
    return gx, gy, gz
```

Python implementations are nice because we can easily change the code and rerun instantly using the REPL and the Jupyter notebook. BUT. Man. Its. Slow. Painfully slow. Too slow for us to compute the gradients for large grids. So, first, lets try Numba. 

# Numba {#numba}
[Numba](http://numba.pydata.org/) is just-in-time (JIT) compiler that at a very high level takes Python code and makes machine code that runs very fast. This method is most suitable to the case presented here with many loops and simple (to moderately advanced) mathematical operations. Recent versions of Numba support many Numpy functions (you can [check it out here](http://numba.pydata.org/)). This is the simplest way to speed up this type of triple loop happening in Python. A simple decorator `@numba.jit` placed at the start of the function: 

```python
import numba
import numpy as np

@numba.jit
def calc_gradients_jit(grid):
    nx, ny, nz = np.shape(grid)  # assumes the grid is 3D
    gx = np.zeros((nx, ny, nz))
    gy = np.zeros((nx, ny, nz))
    gz = np.zeros((nx, ny, nz))
    # the dreaded triple loop, a naive python implementation
    for k in range(nz):
        for j in range(ny):
            for i in range(nx): 
                # limit the array indexes, remember about Python 0-indexing
                ist = max(0, i - 1)
                jst = max(0, j - 1)
                kst = max(0, k - 1)
                ifn = min(nx - 1, i + 1)
                jfn = min(ny - 1, j + 1)
                kfn = min(nz - 1, k + 1)
                # calculate the gradients
                gx[i, j, k] = ( grid[ist, j, k] - grid[ifn, j, k] ) / 2.0
                gy[i, j, k] = ( grid[i, jst, k] - grid[i, jfn, k] ) / 2.0
                gz[i, j, k] = ( grid[i, j, kst] - grid[i, j, kfn] ) / 2.0
    return gx, gy, gz
```
And the function runs about 266 times faster. Unfortunately if things outside of simple mathematical operations are required, and those are unsupported within the set of functions converted from Numpy (or other libraries) to standard Python implementations specifically for Numba, then you're stuck implementing them on your own. So, this can't be used for everything. But in this simple case, it is the best!

# Cython {#cython}
[Cython](http://cython.org/) is a nice library that essentially lets you write C++ with familiar Python-like syntax. Speed comes from declaring static types with specialized code. Cython is essentially its own syntax although many things are similar to Python or C++ (although I have never used the latter). The Cython code is converted to C++ using all sorts of trickery that isn't important at this level of understanding, and the result is an extension module that can be imported and called from Python, and has all the speed of a compiled language. I think Cython can be quite flexible, but the most speed comes from being explicit and defining the types of all variables that are used in the code. So, our 3D gradient subroutine becomes (using the Cython magic cell): 

```python
%%cython
import numpy as np
cimport numpy as np
cimport cython

@cython.boundscheck(False)
def calc_gradients_cy(np.ndarray[np.double_t, ndim=3] grid):
    # explicitly define ***ALL*** variables types
    cdef int nx = grid.shape[0]
    cdef int ny = grid.shape[1]
    cdef int nz = grid.shape[2]
    cdef int i, j, k, ist, jst, kst, ifn, jfn, kfn
    cdef np.ndarray[np.double_t, ndim=3] gx = np.zeros((nx, ny, nz), dtype=np.double)
    cdef np.ndarray[np.double_t, ndim=3] gy = np.zeros((nx, ny, nz), dtype=np.double)
    cdef np.ndarray[np.double_t, ndim=3] gz = np.zeros((nx, ny, nz), dtype=np.double)

    # the dreaded triple loop, a naive python implementation
    for k in range(nz):
        for j in range(ny):
            for i in range(nx): 
                # limit the array indexes, remember about Python 0-indexing
                ist = max(0, i - 1)
                jst = max(0, j - 1)
                kst = max(0, k - 1)
                ifn = min(nx - 1, i + 1)
                jfn = min(ny - 1, j + 1)
                kfn = min(nz - 1, k + 1)
                # calculate the gradients
                gx[i, j, k] = ( grid[ist, j, k] - grid[ifn, j, k] ) / 2.0
                gy[i, j, k] = ( grid[i, jst, k] - grid[i, jfn, k] ) / 2.0
                gz[i, j, k] = ( grid[i, j, kst] - grid[i, j, kfn] ) / 2.0
    return gx, gy, gz
```


# F2PY {#F2PY}
[F2PY](https://docs.scipy.org/doc/numpy-dev/f2py/) is my favorite methodology to compile fast code, but this is also not where I would start if I am learning. Usage of F2PY assumes you know Fortran. Many people view Fortran as an ancient language not worth learning. However, I see Fortran as a numerical computing language that is easy to learn but difficult to master; importantly it allows the beginner make ***BLAZING*** fast, albeit simple, subroutines (Fortran LOVES loops) that perform whatever simplistic function you can think of. 

Fortran is a compiled language - static types and known loop sizes allow the compiler to optimize the code. Additionally, if you are a student, you have access to the [free ifort compiler](https://software.intel.com/en-us/qualify-for-free-software/student). The following code is pretty well as fast as possible (for my skill level). However, I have experience programming with Fortran codes, so I have all types statically defined (that trusty implicit none), the loops are in the correct order (I think), and I know about some of the intrinsic functions like `min()` and `max()` which keep all my indices in-bounds in the loops. 

```fortran 
subroutine calc_gradients(nx, ny, nz, grid, gx, gy, gz)
    implicit none
    integer, intent(in) :: nx, ny, nz
    real*8, intent(in) :: grid(nx, ny, nz)
    real*8, intent(out) :: gx(nx, ny, nz), gy(nx, ny, nz), gz(nx, ny, nz)
    integer :: i, j, k, ist, jst, kst, ifn, jfn, kfn
    ! triple loop
    do j = 1, nz
        do k = 1, ny
            do i = 1, nx
                ! ensure the indices do not go out of bounds
                ist = max(1, i - 1)
                jst = max(1, j - 1)
                kst = max(1, k - 1)
                ifn = min(nx, i + 1)
                jfn = min(ny, j + 1)
                kfn = min(nz, k + 1)
                ! calculate each gradient
                gx(i, j, k) = ( grid(ist, j, k) - grid(ifn, j, k) ) / 2.0D0
                gy(i, j, k) = ( grid(i, jst, k) - grid(i, jfn, k) ) / 2.0D0
                gz(i, j, k) = ( grid(i, j, kst) - grid(i, j, kfn) ) / 2.0D0
            enddo
        enddo
    enddo
end subroutine calc_gradients
```

The issue with linking Fortran to Python is with F2PY. Depending on the version of Python, Fortran compiler, and platform, this can either be a breeze, or a complete pain. If you are having some issues in windows, you can check out the [pygeostat documentation on Fortran in Python](http://www.ccgalberta.com/pygeostat/fortran.html) for some examples on common errors and some suggested solutions. Typically the fact of running windows is the single greatest hurdle to using F2PY for this workflow. 

# So, what about straight up Numpy? {#numpy}

Of course, for a task such as computing the gradients on the grid (this is a very common task), Numpy and its vast library of computational functions should probably be investigated. A quick google of `'numpy gradients'` returns the [numpy.gradient](https://docs.scipy.org/doc/numpy/reference/generated/numpy.gradient.html) function which can be used to quickly compute the gradients of the 3D grid. 

Other implementations using numpy could be considered to compute the gradients, for example using slices of the matrix to compute the forward-reverse diffs, ensuring the operations are vectorized rather than sequential. 

# Conclusions

There are a variety of methods to make subroutines run faster in Python. The best way to prototype new ideas is definitely to *NOT* write research code in Fortran, rather use some mixed Python-Speed methodology as outlined above to develop ideas quickly while writing readable and quickly modifiable code. 




