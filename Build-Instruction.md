At the moment of this writing, Assembly3 only works with a forked FreeCAD
[branch](https://github.com/realthunder/FreeCAD/tree/LinkStage3). You need to
first checkout this branch and [build](https://github.com/realthunder/FreeCAD/tree/LinkStage3#compiling)
it yourself.

After that, checkout this repository directly inside the `Ext/freecad/`
directory of your FreeCAD installation or build directory. Be sure to name the
directory as **asm3**. The Assembly3 workbench supports multiple constraint
solver backends. Currently, there are two backends available, `SolveSpace` and
`SymPy + SciPy`, both of which have external dependency. The current focus is
to get SolveSpace backend fully working first, with SymPy + SciPy serving as
a reference implementation for future exploration. All backends are optional.
But, you'll need at least one installed to be able to do constraint based
assembling, unless you are fine with manually movement, which is actually
doable because Assembly3 provides a powerful mouse dragger.

# SolveSpace

[SolveSpace](http://solvespace.com/) is by itself a standalone CAD software
with excellent assembly support. IMO, it has the opposite design principle of
FreeCAD, which is big, modular, and fully extensible. SolveSpace, on the other
hand  is lean and compact, and does extremely well for what it offers. But, you
most likely will find something you want that's missing, and have to seek out
other software for help. The constraint solver of SolveSpace is available as
a small library for integration by third party software, which gives us the
opportunity to bring the best from both worlds. 

There is no official python binding of SolveSpace at the moment. Besides, some
small modification is required to bring out the SolveSpace assembly
functionality into the solver library. You can find my fork at `asm3/slvs`
subdirectory. To checkout, 

```
cd asm3
git submodule update --init slvs
```

If you are using Ubuntu 16.04 or Windows 64-bit, then you can check out the
pre-built python binding at `asm3/py_slvs` subdirectory.

```
cd asm3
git submodule update --init py_slvs
```

## Build for Ubuntu

To build for Ubuntu, run

```
apt-get install libpng12-dev libjson-c-dev libfreetype6-dev \
                libfontconfig1-dev libgtkmm-2.4-dev libpangomm-1.4-dev \
                libgl-dev libglu-dev libglew-dev libspnav-dev cmake
```

Make sure to checkout one of the necessary sub module before building.

```
cd asm3/slvs
git submodule update --init extlib/libdxfrw 
```

To build the python binding only

```
cd asm3/slvs
mkdir build
cd build
cmake -DBUILD_PYTHON=1 ..
make _slvs
```

After compilation is done, copy `slvs.py` and `_slvs.so` from
`asm3/slvs/build/src/swig/python/CMakeFiles` to `asm3/py_slvs`. Overwrite
existing files if you've checked out the `py_slvs` sub module. If not, then be
sure to create an empty file named `__init__.py` at `asm3/py_slvs`.

## Cross Compile for Windows

To build for Windows 64-bit, you have two options. This section shows how to
cross compile for Windows on Ubuntu

```
apt-get install cmake mingw-w64
cd asm3/slvs
git submodule update --init --recursive
mkdir build_mingw
cd build_mingw
cmake -DBUILD_PYTHON=1 -DCMAKE_TOOLCHAIN_FILE=../cmake/Toolchain-mingw64.cmake ..
make _slvs
```
After finish, copy `slvs.py` and `_slvs.pyd` from
`asm3/slvs/build/src/swig/python/CMakeFiles` to `asm3/py_slvs`. Overwrite
existing files if you've checked out the `py_slvs` sub module. If not, then be
sure to create an empty file named `__init__.py` at `asm3/py_slvs`.

## Build on Windows

To build on Windows, you should use Visual Studio 2013, the same one FreeCAD
uses. Install CMake and Python. If you are building the 64-bit version, make
sure you install the Python 64-bit version. I have only tested the build with
Python 2.7.14 64-bit. You probably can use the python lib included in FreeCAD
libpack by adding the libpack path to `PATH` environment variable. But it
doesn't work for me somehow. CMake only found the debug version python lib in
the libpack.

Download and extract the latest [swig](http://www.swig.org/download.html) to
some where, and add the path to `PATH` environment variable. I haven't tested
to build with the old version swig that's bundled with FreeCAD libpack.

Be sure to checkout all the submodules of slvs before building. None of them
is actually used, but is still needed to satisfy CMake dependency checking,

```
cd asm3/slvs
git submodule update --init --recursive
```

Run CMake-gui, select a build directory. Add a `BOOL` type entry named
`BUILD_PYTHON`, and set it to `true`. Then click `configure` and select Visual
Studio 2013 Win64, which is what FreeCAD used. If done without error, click
`generate`. 

Finally, open the `solvespace.sln` file in the build directory. You only need to
build two projects, first `slvs_static_excp`, and then `_slvs`. Once finished,
copy the output at the following location to `asm/py_slvs`

```
asm/slvs/<your_build_directory>/src/swig/python/slvs.py
asm/slvs/<your_build_directory>/src/swig/python/Release/_slvs.pyd
```

If you want to build the Debug version, either download Python debug libraries,
or put FreeCAD libpack directory in `PATH` environment variable before
configuring CMake, so that CMake can find the debug version Python library.
Once built, you must rename `_slvs.pyd` to `_slvs_d.pyd` before copying to
`asm/py_slvs`


# SymPy + SciPy

The other constraint solver backend uses [SymPy](http://www.sympy.org/) and
[SciPy](https://www.scipy.org/). They are mostly Python based, with some native
acceleration in certain critical parts. The backend implementation models after
SolveSpace's solver design, that is, symbolic algebraic + non-linear least
square minimization. It can be considered as a python implementation of the
SolveSpace's solver. 

SciPy offers a dozen of different [minimization](https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.minimize.html) 
algorithms, but most of which cannot compete with SolveSpace performance wise.
The following list shows some non-formal testing result using default
parameters with the sample [assembly](#create-a-super-assembly-with-external-link-array) 
described later

| Algorithm | Time |
| --------- | ---- |
| SolveSpace (as reference) | 0.006s |
| Nelder-Mead | Not converge |
| Powell | 7.8s |
| CG | 10+37s <sup>[1](#f1)</sup> |
| BFGS | 10+0.8 <sup>[1](#f1)</sup> |
| Newton-CG | 10+61+0.5s <sup>[2](#f2),</sup><sup>[3](#f3)</sup> |
| L-BFGS-B | 10+1.5s <sup>[1](#f1),</sup><sup>[3](#f3)</sup> |
| TNC | 10+0.8s <sup>[1](#f1)</sup> |
| COBYLA | 0.2s <sup>[3](#f3)</sup> |
| SLSQP | 10+0.3 <sup>[1](#f1),</sup><sup>[3](#f3)</sup> |
| dogleg | 10+61+?s <sup>[2](#f2)</sup> Failed to solve, linalg error |
| trust-ncg | 10+61+1.5s <sup>[2](#f2)</sup> |

<b name="f1">[1]</b> Including Jacobian matrix calculation (10s in this test
case), which is implemented using sympy lambdify with numpy.

<b name="f2">[2]</b> Including Hessian matrix calculation (61s in this test
case), in addition to Jacobian matrix.

<b name="f3">[3]</b> The obtained solution contains small gaps in some of 
the coincidence constrained points. Incorrect use of the algorithm?

The reasons for writing this backend are,

* SolveSpace is under GPL, which is incompatible with FreeCAD's LGPL, 
* To gain more insight of the solver system, and easy experimentation with new
  ideas due to its python based nature,
* For future extension, physics based simulation, maybe?

You'll need to install SymPy and SciPy for your platform. 

```
pip install --upgrade sympy scipy
```

