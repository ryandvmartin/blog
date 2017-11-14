+++
tags = ["Julia", "Python"]
highlight = true
math = true
image = ""
date = "2017-11-13"
title = "Transitioning from Python to Julia"
summary = """
A few useful tips and tricks for transitioning from Python + Compiled language to Julia
"""
+++

## Two Languages
Python suffers from a two language problem. On one end the syntax, vast and mature library ecosystem, dynamic style and interactive workflows that Python offers provide a flexible and productive prototyping environment for new ideas. However, when moving past initial ideas and beginning to apply those ideas to datasets of meaningful size, *pythonistas* often look to things like [Cython, Numba, F2PY or C++]({{< relref "/post/fast_subroutines.md" >}}) to gain additional performance. In my research group we spend much of our time developing and implementing ideas using Python (or Matlab), and once those ideas are demonstrated, often a follow-up Fortran implementation is generated so that statically compiled executables implementing those ideas can reach as wide an audience as possible. Thus, to be successful we are required to work with some high-level and dynamic language and also some secondary language that can speed up the implementation and be compiled to a static executable for distribution. This two language problem is a common `pain point` for numerical computing in Python.

## The Motivation
[Julia](https://julialang.org/) claims to offer a fundamentally new paradigm targeting this problem. This language is tailored to numerical computing by offering a dynamic programming environment and static type definitions with JIT-compilation to within 1-2x the performance compiled languages (Fortran, C). Thus it is easy to see just why Julia is so appealing: fast development times with dynamic and interactive programming; seemingly painless code optimization using static typing and an LLVM compiler that gets speeds in the ballpark of compiled languages; and as a final perk to the language, a glimmer of hope that static compilation may one day be a standard part of the language.

```julia
# dynamic containers
list = Any[]
# iteration
for item in items
    push!(list, item)
end
```

## Really Switching?
One of the larger hurdles for actually switching from Python is the fact that I have developed a large suite of utilities that I depend on in my scripting and research workflows. One of those components is F2PY, which I use to write fast-running custom loops. I am hoping that Julia fills the requirements of fast-running code going forward.. but one important point of concern is losing all the utilities that I have worked so hard on in the past. Luckily a package `PyCall.jl` allows Julia to interact directly with Python code. The syntax is a bit strange but a Python call like:

```python
import pygeostat as gs
import numpy as np

x = np.random.randn(100)
y = np.random.randn(100)

ax = gs.scatxval(x, y)
# and to modify properties, e.g.:
ax.set_ylim([-4, 4])
ax.set_xlim([-4, 4])
```

turns into this when called from Julia:

```julia
using PyCall
@pyimport pygeostat as gs

x = randn(100)  # Julia includes this by default
y = randn(100)

ax = gs.scatxval(x, y)
ax[:set_ylim]([-4, 4])
ax[:set_xlim]([-4, 4])
```

Note the similarity of the calls to the `scatxval()` function. A large number of standard Python types are interoperable. One potential concern is the use of `pd.DataFrames()` in Python versus the DataFrames in Julia.. but much of that concern can be alleviated by using Arrays which PyCall automates the type conversion back and forth between the Julia and Python objects. Infact, an entire Python-based workflow can be generated using existing scripts and tools developed in Python. Thus, any researcher who depends largely on Python can rest easy that their hard work will still aid them as they venture out to learn Julia.

## Quick Speed Demo
In my [previous post]({{< relref "/post/fast_subroutines.md" >}}) I demonstrated a number of methods to improve the speed of a naive triple loop implementation of a gradient calculation for a 3-dimensional array. A similar implementation using the same triple-loop strategy in Julia is shown below:

```julia
" calculate the forward-reverse diffs on the 3D array"
function gridgradients(a)
    nx, ny, nz = size(a)
    gx = zeros(a)
    gy = zeros(a)
    gz = zeros(a)
    for k in 1:nz
        for j in 1:ny
            for i in 1:nx
                is = max(1, i - 1)
                ie = min(nx, i + 1)
                js = max(1, j - 1)
                je = min(nx, j + 1)
                ks = max(1, k - 1)
                ke = min(nx, k + 1)
                gx[i, j, k] = (a[ie, j, k] - a[is, j, k]) / 2.0
                gy[i, j, k] = (a[i, je, k] - a[i, js, k]) / 2.0
                gz[i, j, k] = (a[i, j, ke] - a[i, j, ks]) / 2.0
            end
        end
    end
    return gx, gy, gz
end
```

Over 1000 iterations, this function averages $1.13ms$, which is ~1.5x the speed of Fortran or Cython, and almost 2x the speed of Numba with no additional decorations or work on my behalf. Note that the arrays are $1$ indexed, and the ordering of the fastest axis is familiar as it is the same as Fortran and the opposite of C and Python. The speedup here over pure Python is immense for code generated with similar effort.

Recalling one of the optimizations that we chose in Cython:

```python
@cython.boundscheck(False)
@cython.wraparound(False)
```

A similar optimization is immediately available in the Julia code with the `@inbounds` macro, which simply disables bounds-checks:

```julia
>?@inbounds
@inbounds(blk)

Eliminates array bounds checking within expressions.
```

```julia
" calculate the forward-reverse diffs on the 3D array"
function gridgradients_inbounds(a)
    nx, ny, nz = size(a)
    gx = zeros(a)
    gy = zeros(a)
    gz = zeros(a)
    @inbounds for k in 1:nz
        @inbounds for j in 1:ny
            @inbounds for i in 1:nx
                is = max(1, i - 1)
                ie = min(nx, i + 1)
                js = max(1, j - 1)
                je = min(nx, j + 1)
                ks = max(1, k - 1)
                ke = min(nx, k + 1)
                gx[i, j, k] = (a[ie, j, k] - a[is, j, k]) / 2.0
                gy[i, j, k] = (a[i, je, k] - a[i, js, k]) / 2.0
                gz[i, j, k] = (a[i, j, ke] - a[i, j, ks]) / 2.0
            end
        end
    end
    return gx, gy, gz
end
```

And this modified code runs in $0.67ms$ averaged over 1000 iterations. In other words, we are faster than both Cython and Fortran, and very close to Numba, and the syntax and complexity of the code hasn't changed dramatically. The other touted benefit to the user (which I am still trying to wrap my head around as I learn more about Julia) is that all user defined types are statically compiled, so if `a` was a matrix of:

```julia
type MyFloat <: Real
    xi::Float64
    yi::Float64
end
```

and certain methods are properly overloaded to make `+`, `-`, `min` and `max` work correctly, the code using that custom type *would be as fast as* that using the native types from the loops above.

## Conclusions
I am very new to Julia. The type system is familiar since Fortran had a form of multiple dispatch by defining function interfaces. Static typing is familiar, plotting is available using existing Python libraries. Static compiling to exe remains to be seen. Going forward I expect to carry out all work in Julia in hopes of finally dropping Fortran as my speed crutch.
