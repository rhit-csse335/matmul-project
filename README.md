# Matrix multiply reference code

Matrix muliplication is a good first example of code optimization
for three reasons:

1.  It is ubiquitous
2.  It looks trivial
3.  The naive approach is orders of magnitude slower than the tuned version

Part of the point of this exercise is that it often makes sense to
build on existing high-performance libraries when they are available.

The rest of this README describes the reference code setup.

## Basic layout

The main files are:

* `README.md`: This file
* `Makefile`: The build rules
* `Makefile.in.*`: Platform-specific flag and library specs used in the Makefile
* `dgemm_*`: Different modules implementing the `square_dgemm` routine
* `fdgemm.f`: Reference Fortran `dgemm` from Netlib (c.f. `dgemm_f2c`)
* `matmul.c`: Driver script for testing and timing `square_dgemm` versions
* `plotter.py`: Python script for drawing performance plots
* `runner.sh`: Helper script for running the timer on the instructional nodes

You will probably mostly be looking at `Makefile.in` and `dgemm_*.c`. Note that "dgemm" stands for "**D**ouble Precision **GE**neral **M**atrix **M**ultiply".   

## Makefile system

By default, the `Makefile` uses `PLATFORM=gcc`. This is appropriate
for running on our cluster. If you want to run this project on other 
hardware, switch the `PLATFORM=` line. For example, to run on a mac,
I would do:

    make PLATFORM=mac

When using the default `gcc` build, the configuration is pulled from
`Makefile.in.gcc`. You may wish to read this file and see what 
settings (such as `OPTFLAGS`) are being used.

For those who aren't familiar with the Makefile system and would like an overview, please consult these two links: [tutorial](http://mrbook.org/blog/tutorials/make/) [more in-depth tutorial](http://www.cs.swarthmore.edu/~newhall/unixhelp/howto_makefiles.html) 

(Warning: only the `gcc` build system has been thoroughly tested.
I'll offer small amounts of extra credit for any improvements that
you'd like to send me.)

### Building on MacOS

Some of you use a Mac for development.  I have included `mac-gcc`
and `mac-clang` for you to use if you want, but you have to have
things installed right first.  Note that you are in no way obliged to
use these things; I provide it solely for your own edification.

By default, the `gcc` program in MacOS is not GCC at all; rather, it
is an alias for Clang. So assuming you have Homebrew installed, you can
build with `make PLATFORM=mac-clang` provided that you first run the
line

    brew install libomp

If you want to use the "real" GCC, make sure you do

    brew install gcc gfortran
    
and then you can build with `make PLATFORM=mac-gcc`.

### Notes on system BLAS

The `gcc` Makefile is configured to link against
OpenBLAS, a high-performance open-source BLAS library based on the Goto BLAS (as an aside, 
there is an excellent NYTimes [article](http://www.nytimes.com/2005/11/28/technology/writing-the-fastest-code-by-hand-for-fun-a-human-computer-keeps.html?mcubz=1) about the history behind Goto BLAS)

On OS X, the Makefile is configured to link against the Accelerate
framework with the `veclib` tag.

### Notes on mixed C-Fortran programming

The reference implementation includes three files for calling a
Fortran `dgemm` from the C driver:

* `dgemm_f2c.f`: An interface file for converting from C to Fortran conventions
* `dgemm_f2c_desc.c`: A stub file defining the global `dgemm_desc` variable
  used by the driver
* `fdgemm.f`: The Fortran `dgemm` routine taken from the Netlib reference BLAS

The "old-school" way of mixing Fortran and C involve figuring out how
types map between the two languages and how the compiler does name
mangling.  It's feasible, but a pain to maintain.  If you want, you
can find many examples of this approach to mixed language programming
online, including in the sources for old versions of this assignment.
But it has now been over a decade since the Fortran 2003 standard,
which includes explicit support for C-Fortran interoperability.  So
we're going this route, using the `iso_c_binding` module in
`dgemm_f2c.f` to wrap a Fortran implementation of `dgemm` from the
reference BLAS.

Apart from knowing how to create a "glue" layer with something like
`iso_c_binding`, one of the main things to understand if trying to mix
Fortran and C++ is that Fortran has its own set of required support
libraries and its own way of handling the `main` routine.  If you want
to link C with Fortran, the easiest way to do so is usually to compile
the individual modules with C or Fortran compilers as appropriate, then
use the Fortran compiler for linking everything together.

## Running the code

You'll submit a job using `qsub` and the provided job file:

```
qsub jobfile.pbs
```

Within this job file, you'll write commands that interact with the `make`
system. The starter code contains these commands:

```
make
make run
make plot
```

These are a great default. If you want to change them (to run a custom driver 
program, or to only run one of the dgemm implementations), keep reading.

### Build system details

The code writes a sequence of timings to a CSV (comma-separated value)
text file that can be loaded into a spreadsheet or array for further
processing.  By default, the name of the CSV file is based on the executable
name; for example, running

    ./matmul-blocked

with no arguments produces the output file `timing-blocked.csv`.  You can
also provide the file name as an argument, i.e.

    ./matmul-blocked timing-blocked.csv

To run all the timers, you will probably want to use

    make run

To clear up some of your workspace, use 

    make clean

which will remove executables and compute node logs (but not your code or benchmarks). To remove everything except for code, use

    make realclean

### Plotting results

You can produce timing plots by running

    make plot

The plotter assumes that all the relevant CSV files are already
in place.

You can also directly use the `plotter.py` script.  The `plotter.py`
script loads a batch of timings and turns them into a plot which is
saved to `timing.pdf`.  For example, running

    ./plotter.py basic blocked blas

or

    python plotter.py basic blocked blas

will compare the contents of `timing-basic.csv`, `timing-blocked.csv`,
and `timing-blas.csv`.


## Optimization Tips and Tricks

Please refer to these [notes](http://www.cs.cornell.edu/~bindel/class/cs5220-f11/notes/serial-tuning.pdf) to get started. The notes discuss blocking, buffering, SSE instructions, and auto-tuning, among other optimizations.
The [Roofline Paper](http://www.eecs.berkeley.edu/Pubs/TechRpts/2008/EECS-2008-134.pdf) is also worth looking at.

