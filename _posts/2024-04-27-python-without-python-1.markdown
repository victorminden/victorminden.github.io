---
layout: post
title:  "🚫🐍: Using Python without using Python - Part 1"
date:   2024-04-29 07:53:16 -0700
tags: Python C/C++
categories: Programming
math: true
---

Python is one of the most-used programming languages in the world.

By some measure that I didn't bother to look up, the Github blog has Python holding steady as the [second-most popular programming language as of 2023](https://github.blog/2023-11-08-the-state-of-open-source-and-ai/)  (first-most: JavaScript 😿).

Python is the *lingua franca* ("French language") of data science, machine learning, and technical phone screens, beloved by students, researchers, and engineers alike for its relatively light syntax and extremely large collection of high quality libraries.

Most computers today come with a Python interpreter pre-installed or just-a-click-away (I'm looking at you, [Windows](https://devblogs.microsoft.com/python/python-in-the-windows-10-may-2019-update/)), and for those that don't there is a simple, standardized method for [setting up a new Python interpreter from scratch](https://www.python.org/downloads/) (actually, [two](https://docs.conda.io/projects/conda/en/stable/)), and [a number of standard tools](https://xkcd.com/927/) for Python package and environment management ([a](https://docs.conda.io/projects/conda/en/stable/),[b](https://docs.python.org/3/library/venv.html),[c](https://python-poetry.org/)).

Ultimately, though, popularity is not quality and "widely-used" does not mean "suitable for all problems".
In my career I have seen Python applied in many questionable ways (*e.g.*, [pytest](https://docs.pytest.org/) as a universal build orchestrator), and in many places where its nimble, dynamic, fast-and-loose nature quickly became more of a liability than a selling point.

In this post, I'm going to play with a few different methods of using Python "on the outside" to talk to some other language "on the inside", enabling spending most of your time writing "not-Python" while still ultimately incorporating your code as part of a Python library or application.  I won't comment on *why* or *when* you might want to do this except to say that there are some good standard reasons (compile-time type safety and code optimization, true multithreaded parallelism, *etc.*) and that you can do some [really fun but probably ill-advised things](https://p403n1x87.github.io/running-c-unit-tests-with-pytest.html).

In everything that follows I will assume we are using CPython since I have never used PyPy or Jython or any other flavor that might exist.  In particular, I will be working with a `conda` environment:

{% highlight bash %}
conda create -n py311 python=3.11
conda activate py311
{% endhighlight %}

Further, the example we will use for interop will be relatively contrived:
we would like a function to concatenate two strings, separated by a comma and single space, and return the result.

## pybind11: writing C++ and liking it
The header-only [pybind11](https://pybind11.readthedocs.io/en/latest/) library is maybe the most popular approach to Python<->C++ interoperability.

Since I'm already using `conda`, I'll just install `pybind11` using the command
{% highlight bash %}
$ conda install -c conda-forge pybind11
{% endhighlight %}

With a slight variation on the basic example provided by the `pybind11` docs, we can write a simple C++ file, `pybind_example.cxx`, whose filename *must match the name of the Python module being defined with `PYBIND11_MODULE`*:

{% highlight C++ %}
#include <pybind11/pybind11.h>
#include <string>

namespace py = pybind11;
using namespace pybind11::literals;

std::string greet(const std::string& greeting, const std::string& name) {
    return greeting + ", " + name;
}

PYBIND11_MODULE(pybind_example, m) {
    m.doc() = "An example module for pybind11";
    m.def("greet", &greet, "greeting"_a = "Hello", "name"_a = "World");
}
{% endhighlight %}

To build this Python module, we will use `setuptools` with a minimal `setup.py` file:
{% highlight python %}
from pybind11.setup_helpers import Pybind11Extension
from setuptools import setup

setup(
    name="pybind_example",
    ext_modules=[
        Pybind11Extension("pybind_example",["pybind_example.cxx"]),
    ],
)
{% endhighlight %}

We build and install this locally into the current directory with
{% highlight bash %}
$ python -m pip install .
{% endhighlight %}
and that's it, we're done.

{% highlight bash %}
$ python -c "from pybind_example import greet; print(greet())"
# Prints: "Hello, World"
$ python -c "from pybind_example import greet; print(greet('Howdy', 'stranger'))"
# Prints: "Howdy, stranger"
{% endhighlight %}

One of the benefits of `pybind11` is just how little boilerplate is required to get started, though admittedly there is not much complicated going on in this example.

## Vanilla extensions: C with some really strange types
Because CPython is implemented in C, it is in theory straight-forward to add new Python modules "just" by writing C and gluing it to Python.  The Python documentation refers to these as [extension modules](https://docs.python.org/3/extending/extending.html).
In practice, however, writing extension modules directly in this fashion can be quite a bit more verbose than using `pybind11`.

In particular, writing the C extension module requires manually specifying directly how your function should be used in terms of the CPython runtime types.  In our example here, this is mostly mechanical, but it can get cumbersome in practice.  The upshot of this verbosity is that extension modules written directly in this fashion have full control of how data ownership and reference counting works between the C/C++ code and the Python code, which becomes necessary for more complex applications, especially when performance is a factor.
The file `extension_example.cxx` is:
{% highlight c++ %}
#include <Python.h>
#include <string>

std::string greet(const std::string& greeting, const std::string& name) {
    return greeting + ", " + name;
}

static PyObject* method_greet(PyObject* self, PyObject* args) {
    const char* greeting;
    const char* name;

    if (!PyArg_ParseTuple(args, "ss", &greeting, &name))
        return NULL;

    const std::string result = greet(greeting, name);
    return PyBytes_FromString(result.c_str());
}

static PyMethodDef Methods[] = {
    {"greet",  method_greet, METH_VARARGS, "Greets the user."},
    {NULL, NULL, 0, NULL} // Sentinel
};

static struct PyModuleDef module = {
    PyModuleDef_HEAD_INIT,
    "extension_example",
    "My example C/C++ extension module",
    -1,
    Methods
};

PyMODINIT_FUNC PyInit_extension_example(void) {
    return PyModule_Create(&module); // Maybe null
}
{% endhighlight %}

As before, we will use `setuptools` to install the extension module locally by defining an appropriate `setup.py`:
{% highlight python %}
from setuptools import Extension, setup

setup(
    name="extension_example",
    ext_modules=[
        Extension("extension_example", ["extension_example.cxx"]),
    ],
)
{% endhighlight %}
and running
{% highlight bash %}
$ python -m pip install .
{% endhighlight %}

The result can be used as in the `pybind11` case:
{% highlight bash %}
$ python -c "from extension_example import greet; print(greet('Howdy', 'stranger'))"
# Prints: "Howdy, stranger"
{% endhighlight %}

## Cython: "[We have] outlived most other [...] static compilers for the Python language"
I am sure I have used [cython](https://cython.org/) in the past, but it is a bit of an older methodology in my mind and not something I've ever set up and used on purpose.

We once again begin by install Cython into the current `conda` environment and write our C++ function ("cython_example.hpp"):
{% highlight bash %}
$ conda install Cython
{% endhighlight %}

{% highlight c++ %}
#include <string>

std::string greet(
    const std::string& greeting = "Hello",
    const std::string& name = "World") {
    return greeting + ", " + name;
}
{% endhighlight %}

New to Cython, we write a "wrapper.pxd" file containing our C++ code interface and a "cython_example.pyx" file containing our Python wrapper (surprisingly, each with working syntax highlighting!), mixing in some custom Cython syntax to define our Python wrapper for our C++ function:
{% highlight cython %}
# wrapper.pxd
from libcpp.string cimport string

cdef extern from "cython_example.hpp":
    string greet(const string& greeting, const string& name)
{% endhighlight %}

{% highlight cython %}
# cython_example.pyx
# distutils: language=c++
# cython: c_string_type=unicode, c_string_encoding=utf8

cimport wrapper

def greet(greeting, name):
    return wrapper.greet(greeting, name)
{% endhighlight %}

Finally, we define our `setuptools` `setup.py`, install, and use our function:
{% highlight python %}
from setuptools import setup

from Cython.Build import cythonize

setup(name="cython_example", ext_modules=cythonize("cython_example.pyx"))
{% endhighlight %}

{% highlight bash %}
python -m pip install .
{% endhighlight %}

{% highlight bash %}
$ python -c "from cython_example import greet; print(greet('Howdy', 'stranger'))"
# Prints: "Howdy, stranger"
{% endhighlight %}

## SWIG: the Simplified Wrapper and Interface Generator

Much more general than Python, SWIG is an interesting tool for generating wrappers for all sorts of languages (*e.g.*, `go`, OCaml, *etc.*).
It is pretty old and involves a few more pieces working together than the other methods I've played with here,
but it is still very much in use.

To begin, we will install `swig` and `pipx` packages into our `conda` environment so that we can build our SWIG wrappers completely from Python.
Obviously this is not the preferred method across other languages, but it is pretty convenient for Python:
{% highlight bash %}
$ conda install swig pipx
{% endhighlight %}

We separate our C++ code into the header file and implementation file,
with nothing out of the ordinary here:
{% highlight c++ %}
// swig_example.hpp
#include <string>

std::string greet(
    const std::string& greeting = "Hello",
    const std::string& name = "World"
);
{% endhighlight %}

{% highlight c++ %}
// swig_example.cxx
#include "swig_example.hpp"

std::string greet(
    const std::string& greeting = "Hello",
    const std::string& name = "World") {
    return greeting + ", " + name;
}
{% endhighlight %}

New to SWIG, now, is the interface file, `swig_example.i`, which defines the code to be wrapped.
We do nothing more here than include the header with the declaration of our `greet` function
and redeclare `greet` as part of the interface.

{% highlight c++ %}
// swig_example.i
%module swig_example
%include "std_string.i"

%{
#define SWIG_FILE_WITH_INIT
#include "swig_example.hpp"
%}

std::string greet(const std::string& greeting, const std::string& name);
{% endhighlight %}

Finally, to build the Python module using our SWIG wrapper, we
again define a `setup.py`:

{% highlight python %}
# setup.py
from distutils.core import setup, Extension

setup(name = 'swig_example',
       ext_modules = [
        Extension(
           '_swig_example',
           sources=[
               'swig_example_wrap.cxx',
               'swig_example.cxx',
            ])],
       py_modules = ["swig_example"],
       )
{% endhighlight %}

Whereas previously we just used `pip` to install the module to the local directory,
with SWIG I found it necessary to explicitly install the extensions with `--inplace`,
(as suggested by SWIG docs), compiling the dynamic library into the local directory directly
such that the Python module can load it at runtime without having to worry about paths.
However, the basics are the same.

{% highlight bash %}
$ pipx run swig -c++ -python swig_example.i
$ python setup.py build_ext --inplace
{% endhighlight %}

Finally, we have our greeter:
{% highlight bash %}
$ python -c "from swig_example import greet; print(greet('Howdy', 'stranger'))"
# Prints: "Howdy, stranger"
{% endhighlight %}

## cppyy: a challenger approaches!
A relatively new method for Python<->C++ interop is [cppyy](https://cppyy.readthedocs.io/en/latest/index.html) (pronounced "cppyy"), by which I mean I had never heard of it before starting to write this.  Based on the `Cling` C++ interpreter, `cppyy` differs from some of the other options here in that it can compile your C++ code at runtime, which can lead to additional opportunities for performance.

We begin by installing `cppyy` with

{% highlight bash %}
$ conda install -c conda-forge cppyy
{% endhighlight %}

and then write a minimal `cppyy_example.hpp`, which knows nothing about Python:

{% highlight C++ %}
#include <string>

std::string greet(
    const std::string& greeting = "Hello",
    const std::string& name = "World") {
    return greeting + ", " + name;
}
{% endhighlight %}

Compiling and using this code in Python is as easy as:
{% highlight python %}
>>> import cppyy
>>> cppyy.include("cppyy_example.hpp")
True
>>> from cppyy.gbl import greet
>>> greet()
b'Hello, World'
>>> greet('Howdy', 'stranger')
b'Howdy, stranger'
{% endhighlight %}

### Parting Thoughts

That's a lot of syntax for one day, so let's stop here for now and follow up in another post.

My initial impressions, looking at all of this together, are that `pybind11` is really nice,
`cppyy` seems interesting for toy use cases, and everything else gets a but more involved.
Not to mention, this code barely does anything!  Things get much more complicated as we venture
into wrapping objects, memory management / reference counting, *etc.*, and I imagine that is where
some of the more complicated options here start to become necessary rather than just verbose.

Next time, I aim to look at `CFFI`, `PyO3`, `gRPC`, and `Mojo 🔥`.
