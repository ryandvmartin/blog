+++
tags = ["Fortran", "Python", "Julia"]
highlight = true
math = true
image = ""
date = "2018-03-12"
title = "Calling Fortran DLL's from Python and Julia"
summary = "Calling those legacy DLL's from our fun new languages :-)"

+++

The standard Python library contains a package called ``ctypes`` which provides interfaces to foreign functions that are found in shared libraries. With the Intel compiler, a Fortran function or subroutine can be exported to dll by ensuring that the `!DEC$` is present to define the exported symbol:


```fortran
! Note: this only works with the Intel Compiler
subroutine sqr(val)
    !DEC$ ATTRIBUTES DLLEXPORT, ALIAS:'sqr' :: sqr
    integer, intent(inout) :: val
    val = val ** 2
end subroutine sqr
```

If the above subroutine is contained in the ``example.f90`` file, it is compiled with :

    $ ifort /dll example.f90

which produces the shared library ``example.dll``. Inspecting this object with dumpbin:

    $ dumpbin /exports example.dll
    1    0 00001000 sqr

The function can be imported to Python with the ``ctypes`` module and called with the following Python code:

```python
import ctypes as ct

fortlib = ct.CDLL('example.dll')
val = 100
val = ct.pointer(ct.c_int(val)) # setup the pointer to the correct data structure
_ = fortlib.sqr(val)            # call the function
print(val[0])                   # index the memory address setup above
```

It is immediately clear that compiling the shared library in this way is easier (when compared to F2PY), and calling the Fortran from Python is much difficult. The advantages are that this is a C-independent method to build and link the Fortran extension, and the same dll called from Python here could also be called from other languages (e.g. Julia).

# Maintaining GNU - Intel compatibility
The method to call specific functions should be compiler independent. A method to maintain a compatible Python calling sequence for the dll's is required since one set of Python code should be able to call the Fortran function regardless of how it was compiled. This can be done by using the ``iso_c_binding`` module:

```fortran
subroutine sqr2(val) BIND(C, NAME='sqr2')
    use iso_c_binding
    !DEC$ ATTRIBUTES DLLEXPORT :: sqr2
    integer, intent(inout) :: val
    val = val ** 2
end subroutine sqr2
```

The ``iso_c_binding`` in this case overwrites the name mangling for each compiler (gfortran and ifort), and ensures the call sequence from Python is constant. In this case the `!DEC$` statement is ignored by gfortran and the function is available in the exported dll. Notably, the `ALIAS` is dropped from the `!DEC$` export for Intel Fortran since the name in the binding overwrites this setting.

# More Complicated Data Types
The main concern for calling Fortran functions in the dll's from Python is how to setup the data structures. Consider the following function which take a 2-dimensional array of integers and computes some arbitrary value from the inputs by replacing the values in the array:

```fortran
module example
    use iso_c_binding
    implicit none
contains
    subroutine sqr_2d_arr(nd, val) BIND(C, NAME='sqr_2d_arr')
        !DEC$ ATTRIBUTES DLLEXPORT :: sqr_2d_arr
        integer, intent(in) :: nd
        integer, intent(inout) :: val(nd, nd)
        integer :: i, j
        do j = 1, nd
        do i = 1, nd
            val(i, j) = (val(i, j) + val(j, i)) ** 2
        enddo
        enddo
    end subroutine sqr_2d_arr
end module example
```

The module is compiled with:

    $ ifort /dll example.f90

or:

    $ gfortran -shared example.f90 -o example.dll

and can be called from Python with:

```python
import ctypes as ct
import numpy as np

# import the dll
fortlib = ct.CDLL('example.dll')

# setup the data
N = 10
nd = ct.pointer( ct.c_int(N) )          # setup the pointer
pyarr = np.arange(0, N, dtype=int) * 5  # setup the N-long
for i in range(1, N):                   # concatenate columns until it is N x N
    pyarr = np.c_[pyarr, np.arange(0, N, dtype=int) * 5]

# call the function by passing the ctypes pointer using the numpy function:
_ = fortlib.sqr_2d_arr(nd, np.ctypeslib.as_ctypes(pyarr))

print(pyarr)
```

# Important Considerations
In the above Python code some things are notable:

1. All data is passed by pointer reference to the Fortran dll, the data is modified in place if the variable is `intent(inout)`, or replaced if the variable is `intent(out)`. All memory is controlled on the Python side of things
2. The memory must be allocated in some form on the Python side using ``np.zeros(size, dtype=type)`` (or similar) even if the variable is `intent(out)`
3. **The types of all data initialized on the Python size must match those being called in the Fortran module**
    a. This is very important. For example, use ``int`` for ``integer``, ``float`` for ``real*8``, etc. Python ``float`` is double precision by default. This may not be a correct list, and is definitely not exhaustive, but it has worked with the limited testing that has been done with this method of calling Fortran
    b. the function ``ctypes.c_pointer()`` can be used to create the necessary references to non-array elements, be sure to use ``ctypes.c_int()`` and ``ctypes.c_double()`` (and others) as required for the needed parameter
    c. initializing arrays on the python end is strange. For example, say a Fortran subroutine is defined with an `integer, intent(out)` array with size `nd`, to initialize this array on the python side, you can use::

        import ctypes as ct
        intarray = (ct.c_int * nd)()

      and to give the array values for input::

        import ctypes as ct
        pyarray = np.zeros(nd, dtype=np.int)
        valarray = (ct.c_int * nd)(*pyarray)

4. Numpy provides some convenient methods ``np.ctypeslib.as_ctypes()`` to pass initialized arrays with the correct types to the Fortran functions (as demonstrated above)

# Mixed Language Debugging
The use of dll's and ``ctypes`` library for calling the Fortran code provides some exciting opportunities to speed up code development while leveraging the plotting and data management capabilities of pygeostat with the GSLIB Fortran geostatistical library. For example, to debug a Fortran dll which is present in a Visual Studio project, compiled with debug flags, generally follow these steps::


Visual Studio 2015 Community is currently free and has recently developed a set of Python tools that will utilize Python distributions found on the machine. Combined with the Intel Compiler, projects containing Fortran code compiled to dll, and called VIA the Python libraries and tools presented above, can be debugged after it is called from Python!

# What about Julia
Julia provides first class support for calling C libraries using the `ccall` function. The example above requires no modification on the fortran side:

```fortran
module example
    use iso_c_binding
    implicit none
contains
    subroutine sqr_2d_arr(nd, val) BIND(C, NAME='sqr_2d_arr')
        !DEC$ ATTRIBUTES DLLEXPORT :: sqr_2d_arr
        integer, intent(in) :: nd
        integer, intent(inout) :: val(nd, nd)
        integer :: i, j
        do j = 1, nd
        do i = 1, nd
            val(i, j) = (val(i, j) + val(j, i)) ** 2
        enddo
        enddo
    end subroutine sqr_2d_arr
end module example
```

The module is compiled with:

    $ ifort /dll example.f90

or:

    $ gfortran -shared example.f90 -o example.dll

But now the call in Julia is vastly simplified:

```julia
nd = Int(100)
arr = zeros(Int64, (nd, nd));
ccall((:sqr_2d_arr, "./example.dll"), Void, (Ref{Int64}, Ref{Matrix{Int64}}), nd, arr)
```

# UPDATE 04-05-2018

Through some suffering I have learned of the joys of string interop between C and Fortran. I wrote some Fortran to facilitate the conversions. Maybe this will help someone somewhere someday...

<script src="https://gist.github.com/ryandvmartin/a3b9bb0c485d2ae58143d71f55a8d9e4.js"></script>

