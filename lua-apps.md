---
layout: page
title: Applications 
order: 2 
---
<p class="message">
Porting your existing Torch applications to MALT-2 is very easy. In this section, we describe existing Torch
apps that we've ported to run on MALT-2. We also describe how to port your existing code to MALT-2 to run 
over multiple GPUs.
</p>

### Existing Torch apps ported to MALT-2 

* Linear regression using Torch ([lr.malt](https://github.com/malt2/malt2.tutorials/tree/master/lr.malt))
* Imagenet over GPUs with Torch ([fb.resnet.torch](https://github.com/malt2/malt2.tutorials/tree/master/fb.resnet.malt))
* Action Recognition (art) (coming soon)
* OpenNMT (coming soon)

### Parallelizing your existing Torch app 

This section describes how to parallelize your existing app to support MALT-2.
MALT-2 torch package overloads the optim package. It provides functions for
distributed optimization as well as additional helper functions to split/permute
data correctly on each replica.

#### Install MALT-2 

* Install MALT-2 for Torch/MiLDE as described [here](../guide)

#### Include the distoptim package
Simple add the dstoptim package in your training file.
{% highlight Lua %}
require 'dstoptim'
{% endhighlight %}

Use the distributed SGD training procedure when calling optim. 
E.g. replace 'sgd' or 'adam' with 'dstsgd' in your code as:

{% highlight Lua %}
optim.dstsgd (optimeval, w, optimparams)
{% endhighlight %}

No other changes are required to communicate or average the parameters.
The optimizer in dstsgd takes care of communicating and averaging the
parallel models. However, one may require to permute/split the data on
each machine differently to ensure that each replica is not processing
the same inputs.

#### Controlling other distributed SGD options

Other options to modify SGD and malt-2 behavior can be controlled by
passing options (similar to passing options to the optimizer). As an example,
one may add the following lines to their code:

{% highlight Lua %}
optimparams.cuda = true
optimparams.transport = 'gpu' 
{% endhighlight %}

where optimparams is the optimization parameters passed using the optim api.
The other possible options for parameters with default values first are below:

{% highlight Lua %}
optimparams.cuda      = true | false
optimparams.transport = 'gpu' | 'mpi' 
optimparams.debug     = false | true
optimparams.cb_size   =    5  | any number greater than 0
{% endhighlight %}


#### Permute/Split the input data 
In order to make sure each parallel replica processes random split, 
MALT-2 provides function to accomplish this in your existing code.
E.g. Instead of using `torch.randperm (tensor)` use `optim.randperm(tensor)`.
This function has the same semantics as `torch.randperm()` but uses different
manual seed on each replicas to ensure that each replica do not do the same work.

Additionally, MALT-2 also provides `optim.nProc()` that returns the number of 
concurrent replicas. This is useful to split the data across replicas. See the
fb.resnet.lua for examples of how this is accomplished (see train.lua).

