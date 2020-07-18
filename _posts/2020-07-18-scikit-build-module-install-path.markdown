---
layout: post
title:  "Scikit-build: setting correctly the module install destination"
date:   2020-07-18
categories: [python]
tags: [python, c++, fortran]
---

[scikit-builld](https://scikit-build.readthedocs.io) is a great package to
make life easier when it comes to compiling binary extensions for python modules.

It integrates [setuptools](https://setuptools.readthedocs.io/en/latest/) and [cmake](https://cmake.org), such that in principle adding an extension can be as
simple as specifying the location of the `CMakeLists.txt` file. However, there
is a catch which made me loose a lot of time.

This is a minimal example from the documentation, where an extension `_hello` is
installed in the package `hello`.

```
cmake_minimum_required(VERSION 3.11.0)
project(hello)
find_package(PythonExtensions REQUIRED)

add_library(_hello MODULE hello/_hello.cxx)
python_extension_module(_hello)
install(TARGETS _hello LIBRARY DESTINATION hello)
```

The install destination is the name of the package itself. In this example
the library will be installed is `site-packages\hello\lib\_hello.so`.

What if the install destination is not specified in `CMakeLists.txt`, or if
it is specified to something else than the package name? `scikit-build` will
then fallback to adding the library as data file instead of package file. Data files are installed in different locations (`~/.local/` for a user installation for example) and the library will not be added under the package folder.

In other words, as explained [here](https://github.com/scikit-build/scikit-build/issues/434), "Install destination of a module is expected to corresponds to the python package where it is expected to be imported from". Which is not stated clearly in the documentation, but rather passed as an assumption.

If the compiled library ends up being in some strange location out of the
package, it is probably because of this. If the `install` command is not present in the `CMakeLists.txt`, it is sufficient to define the argument `setup(cmake_install_dir=<package>)` in the `setup.py`.
