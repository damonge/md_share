# Quick CCL developer primer

This is meant mostly to get you up to speed with where the vital organs of CCL are and how they communicate with each other.

## CCL developer's guide

First, let me list a few resources that should be useful:

We host a [developer's guide](https://ccl.readthedocs.io/en/latest/index.html#devguide) on readthedocs, which you may have seen. Here are the parts of it you should have a quick look at.
- Navigating the code [here](https://ccl.readthedocs.io/en/latest/source/navigating_the_code.html).
- [This](https://ccl.readthedocs.io/en/latest/source/navigating_the_code.html) explains how C and Python communicate with each other. I'll expand a bit further down.
- [This](https://ccl.readthedocs.io/en/latest/source/development_workflow.html) explains how to add new functionality at the C level and communicate it to Python. Although you're gonna be touching C, I'm not sure this will be the most useful information for you, since it's mostly focused on adding new functionality, whereas you'll be mostly modifying what's there.


## C and python

C and python mostly communicate with each other through the SWIG wrapper. This is a list of files with extension `.i` that live inside the `pyccl` folder. As a developer you **only** need to touch those, and the code you write in them is C. As part of the CCL installation, SWIG then takes care of creating a wrapper function for all of the functionality declared in those files. It will place all of those in a python module called `ccllib.py`, which we then use internally in pyccl.

The very typical thing you'll want to do for this project is:
a) Getting a number of numpy arrays from python.
b) Passing them to C as pointers.
c) Getting C to do something and, potentially, return some arrays back to C.

SWIG offers an easy way to transform between numpy arrays and C pointers. At the beginning, however, the syntax can be a bit bewildering. The best thing to do is to learn by example. See the following lines from [ccl_tracers.i](https://github.com/LSSTDESC/CCL/blob/master/pyccl/ccl_tracers.i):

- If you want to pass a given array from python to C, you first need to declare it as an `IN_ARRAY1` at the beginning of the `.i` file. See [this](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/pyccl/ccl_tracers.i#L10), and let's focus on the `z_n` (the corresponding C array/pointer) and `nz_n` (the number of elements of this array).
- Now, if you want to do something with this array. You need to create a function within the `.i` file that receives this `IN_ARRAY1` as an input argument, then calls some internal ccl C-level function, and then possibly outputs something else. See for instance [this function](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/pyccl/ccl_tracers.i#L26).
- Sometimes you'll also want to returns C-level pointers as numpy arrays. For this, the first step is declaring an `ARGOUT_ARRAY1` at the beginning of the `.i` file (see [here](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/pyccl/ccl_tracers.i#L23)). Then, you need to use this as an argument on whatever function should generate this array. See [this](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/pyccl/ccl_tracers.i#L34) for example.

Now, to see how we can make use of these new functions from Python, let's have a look at [tracers.py](https://github.com/LSSTDESC/CCL/blob/master/pyccl/tracers.py).

- In [this line](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/pyccl/tracers.py#L77) you can see how we call one of the new functions created by SWIG (the first function I linked above). Essentially, the corresponding function will be available from `ccllib.py` (which we import here as `lib`). You just need to call that function passing the corresponding numpy array. Note that, while at the C/SWIG level an array is actually two variables (the pointer and the number of elements), in Python the array is just a single variable that contains both pieces of information. SWIG makes the translation between the two more transparent.
- If you want to extract a numpy array from python, however, it's only slightly more complicated. See [this line]( https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/pyccl/tracers.py#L80) for example, where we call the second function I linked above. In this case, because this function generates an output array, we need to pass it the expected number of elements of the array (`nchi` in this case). Otherwise C wouldn't know how much memory to allocate.
