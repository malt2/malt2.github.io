---
layout: page
title: Guide
order: 3 
---
<p class="message">
This page describes how to quickly get started with MALT-2. MALT-2
 parallelizes Torch over multiple CPUs and GPUs.
</p>


## Building MALT with Torch

### Requirements

* [Torch](http://torch.ch)
* [MPI (OpenMPI or MPICH)](https://www.open-mpi.org/)
* [Boost (1.54 or higher)](http://www.boost.org/) 


### Setup

### Install Torch, MPI, Boost and CUDA (if using GPU).

* Checkout the latest version of MALT-2 from [github](https://github.com/malt-2)

{% highlight Bash %}
git clone https://github.com/malt2/malt2.git --recursive
{% endhighlight %}

### Setup the environment variables

### Source your torch/cuda/MKL environment:
on some machines, you might need things something like (MKL is optional):

{% highlight Bash %}
source [torch-dir]/install/bin/torch-activate
source /opt/intel/mkl/bin/intel64/mklvars.sh intel64
{% endhighlight %}

If using modules, you can try:

{% highlight Bash %}
module install icc cuda80 luajit
{% endhighlight %}

### To build everything including dstorm, orm and torch, just type from the top-level directory:

{% highlight Bash %}
make
{% endhighlight %}

This command builds the distributed shared memory component (dstorm), the shared memory transport hook (orm)
and the luarocks for torch hooks and distributed optimization.

### Component-wise build

To build componenet-wise (not required if using make above):

#### Build the dstorm directory, run:

{% highlight Bash %}
cd dstorm
./mkit.sh GPU test
{% endhighlight %}

You should get a `SUCCESS` as the output. Check the log files to ensure the build is successful.

The general format is:
{% highlight Bash %}
./mkit.sh <type> 
{% endhighlight %}

where TYPE is: 
          or MPI (liborm  + mpi)
          or GPU (liborm + mpi + gpu)
A side effect is to create ../dstorm-env.{mk|cmake} environment files, so lua capabilities
can match the libdstorm compile options.

#### Build the orm


{% highlight Bash %}
cd orm
./mkorm.sh GPU
{% endhighlight %}

#### Building Torch packages. With Torch environment setup, install the malt-2 and dstoptim (distributed optimization packages)

{% highlight Bash %}
cd dstorm/src/torch
rm -rf build && VERBOSE=7 luarocks make malt-2-scm-1.rockspec >& mk.log && echo YAY #build and install the malt-2 package
cd dstoptim
rm -rf build && VERBOSE=7 luarocks make dstoptim-scm-1.rockspec >&mk.log && echo YAY # build the dstoptim package
{% endhighlight %}


### Test

* A very basic test is to run th and then try, by hand,

{% highlight Lua %}
require "malt2"
{% endhighlight %}

### Run a quick test.


* With MPI, then you'll need to run via mpirun, perhaps something like:

{% highlight Bash %}
mpirun -np 2 `which th` `pwd -P`/test.lua mpi 2>&1 | tee test-mpi.log
{% endhighlight %}

* if GPU,

{% highlight Bash %}
mpirun -np 2 `which th` `pwd -P`/test.lua gpu 2>&1 | tee test-GPU-gpu.log
{% endhighlight %}

* NEW: a `WITH_GPU` compile can also run with MPI transport

{% highlight Bash %}
mpirun -np 2 `which th` `pwd -P`/test.lua mpi 2>&1 | tee test-GPU-mpi.log
{% endhighlight %}

default transport is set to the "highest" built into libdstorm2: GPU > MPI  > SHM
{% highlight Bash %}
mpirun -np 2 `which th` `pwd -P`/test.lua 2>&1 | tee test-best.log
{% endhighlight %}

### Running over multiple GPUs.
* MPI only sees the hostname. By default, on every host, MPI jobs enumerate the
GPUs and start running the processes. The only way to change this and run on
other GPUs in a round-robin fashion is to change this enumeration for every
rank using `CUDA_VISIBLE_DEVICES`. An example script is in `redirect.sh` file
in the top-level directory.

* To run:
{% highlight Bash %}
mpirun -np 2 ./redirect.sh `which th` `pwd`/test.lua
{% endhighlight %}

This script assigns available GPUs in a round-robin fashion. Since MPI requires
visibility of all other GPUs to correctly access shared memory, this script only
changes the enumeration order and does not restrict visibility.

### Running applications.
* Check out [here](../lua-apps) to see how to run Torch applications. 

