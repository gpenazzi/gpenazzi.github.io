---
layout: post
title:  "Speed up std::vector to numpy array conversion using array interface"
date:   2020-03-07
categories: [python]
tags: [numpy, python, c++, swig]
---

Sometimes ago I was looking for a solution to speed up the conversion from
different `std::vector` objects to `numpy.array`, probably an extemely common case. Scipy offers a [swig interface for numpy](https://docs.scipy.org/doc/numpy-1.13.0/reference/swig.interface-file.html), however it does not cover
the standard library.

On the other hand, conversion from standard library to
Python is handles natively in swig in the `stl.i` module, but vectors are
translated in regular iterable objects, which makes the conversion to
`numpy.array` painfully slow for large data (it will have to iterate on the whole object).

The simplest solution I could came up with is monkey patching the [array interface](https://docs.scipy.org/doc/numpy/reference/arrays.interface.html) dictionary on the swigged object directly in Python and I published a minimal
example as Gist. The monkey-patching itself looks like this:

{% gist 1f521296479a3b6cb590ada468f00234 vector_to_numpy.py %}

Running this gives on my laptop:

```
************************************************************
Array size:  100000
With stl.i default solution: 0.15158677101135254 s
With monkey-patched __array_interface__: 0.0002703666687011719 s
************************************************************
************************************************************
Array size:  1000000
With stl.i default solution: 1.1882779598236084 s
With monkey-patched __array_interface__: 0.0023055076599121094 s
************************************************************
************************************************************
Array size:  10000000
With stl.i default solution: 10.990112781524658 s
With monkey-patched __array_interface__: 0.012539863586425781 s
************************************************************
```

That's a speedup of about 50x, not bad. This is a very simple solution
and it seems to do the job, it might be desirable to add this directly
somewhere in the swig phase.