---
layout: post
title:  "🚫🐍: Using Python without using Python - Part 2"
date:   2024-09-20 17:53:16 -0700
tags: Python C/C++
categories: Programming
math: true
---
Part 2, better late than never!  This is a follow up to "Using Python without using Python - Part 1" in which we look at a few more methods for using Python as a wrapper/orchestrator to integrate code written in other languages.

## CFFI: a ctypes alternative
Continuing where we left off, we explore [libffi](https://github.com/libffi/libffi) via the [CFFI](https://cffi.readthedocs.io/en/latest/index.html) Python library for calling C code using a foreign function interface.

I find the CFFI documentation relatively difficult to follow, but the purported benefits of CFFI are the ability to avoid learning new syntaxes and APIs for interacting between C and Python, interacting directly with C (note: **not** C++) libraries by essentially copy-pasting their header files into Python.  Anecdotally, this all proved true in this example, barring the fact that strings in Python 3 need to be explicitly encoded/decoded to/from bytes as we will see below.

As before, we start by installing a Python library (again assuming use of a `conda` environment):

{% highlight bash %}
$ conda install cffi
{% endhighlight %}

In this case, since CFFI supports C but not C++, the header and source files look a little different than in the previous post, mostly because of the need to manage C strings:
{% highlight c %}
/* cffi_example.h */

char* greet(const char* greeting, const char* name);
{% endhighlight %}

{% highlight c %}
/* cffi_example.c */

#include "cffi_example.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char* greet(const char* greeting, const char* name) {
    /* This *seems* to be the right buffer size, including null terminator. */
    const int output_size = strlen(greeting) + strlen(name) + 3;
    char* output = (char*) malloc(output_size);
    sprintf(output, "%s, %s", greeting, name);
    return output;
}
{% endhighlight %}

The above is more cumbersome than the equivalent C++, but seems to work (disclaimer: I have fortunately avoided string processing in C in my career, and my understanding is that there are many [footguns](https://www.ctfrecipes.com/pwn/stack-exploitation/stack-buffer-overflow/dangerous-functions/sprintf)).

The new part specific to CFFI here is below.  The code is relatively self-explanatory barring the fact that the `char*` arguments and the return need to be wrangled into the proper formats for Python to interpret them as strings.

{% highlight python %}
# cffi_example.py

# Compile the interface.
from cffi import FFI

ffi = FFI()
ffi.cdef("char* greet(const char* greeting, const char* name);")
ffi.set_source("pygreet", '#include "cffi_example.h"', sources=["cffi_example.c"])
ffi.compile()

# Later, use the interface.
from pygreet.lib import greet

print(ffi.string(greet("Salutations".encode(), "friend".encode())))
{% endhighlight %}

Executing the code directly will compile the interface (generating some files in the local directory), then import it and use it:
{% highlight bash %}
$ python cffi_example
# Prints: b'Salutations, friend'
{% endhighlight %}

As a note, I'm being lazy here but it is of course possible and advisable to compile the interface separately from the time of use.

## PyO3: something else
Thus far, we have looked only at using C/C++ from Python, but as all the cool kids will tell you C/C++ is a dead-end for systems programming and in the future we wil all be writing our software using [Rust](https://www.rust-lang.org/) or [Zig](https://ziglang.org/) or [the blockchain](https://soliditylang.org/).
Luckily, interfacing Python and Rust is maybe even easier than interfacing Python and C++, which we will explore here.  Interfacing Python and web3 is left as an exercise for the reader.

We previously tacitly assumed the availability of C/C++ compilers and relevant system libraries.  We will do the same here for Rust for use with [PyO3](https://github.com/PyO3/pyo3), Rust bindings for Python.  We follow the advice of the [PyO3 user guide](https://pyo3.rs/v0.22.3/), which recommends starting with the `maturin` Python library for building such bindings:

{% highlight bash %}
# Conda version is outdated as of 2024/09/20, easiest to use pip directly.
$ pip install maturin
{% endhighlight %}

More-or-less following the guide, we use `maturin` to create a new package for our Rust-based greeter:
{% highlight bash %}
$ mkdir pyo3_example && cd pyo3_example && maturin init
{% endhighlight %}

As part of the `init`, this creates a `src/lib.rs` in the (now) current working directory with an example function.  Actually, conveniently (by design) `init` sets up the rest of the relevant build/packaging files as well, but we won't touch those.  We slightly modify the generated Rust code to the following:
{% highlight rust %}
// lib.rs
use pyo3::prelude::*;

#[pyfunction]
fn greet(greeting: &str, name: &str) -> PyResult<String> {
    Ok((format!("{greeting}, {name}")).to_string())
}

#[pymodule]
fn pyo3_example(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(greet, m)?)?;
    Ok(())
}
{% endhighlight %}

If you're not familiar with Rust, the `greet` function above probably has some unfamiliar types that [warrant further reading](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html), but rest assured: it works.

{% highlight bash %}
$ maturin develop
$ python -c "import pyo3_example; print(pyo3_example.greet('Welcome', 'traveler'))"
# Prints "Welcome, traveler".
{% endhighlight %}


## gRPC: and now for something completely different

Everything we have looked at so far has centered around compiling libraries for use by Python that allow sending data from code written in Python to code written in another language but used as a library via the original Python interpreter.  A fundamentally different approach is a Remote Procedure Call (RPC), with [gRPC](https://grpc.io/) being the RPC framework I'm most familiar with.

With gRPC, rather than communicating with library written in another language as part of the same binary, the basic idea is to send data outside of the current "client" process to a "server" process that receives the data, performs some computation, and typically returns some result.  This post is not at all a full gRPC tutorial, but the highlights here are:
- The client and server can be written in separate programming languages (in fact, multiple clients in different languages can talk to the same server).
- The data sent back and forth has to be completely serialized to bytes when sent and then deserialized when received, which is much slower than passing around data "in-place" in memory.
- The complexity of sending data around is managed behind-the-scenes.

As an example here, we will spin up a simple Rust gRPC server and Python client for communicating with it.
Once again, we assume Rust is already installed.  For our gRPC Rust library we will use [tonic](https://github.com/hyperium/tonic), which requires the protocol buffer be installed (on Mac, with Homebrew: `brew install protobuf`).

To begin, we will set up a Rust project to run our server.  For simplicity, we will call it `greet-server` and run all successive commands within the `greet-server` directory:
{% highlight bash %}
$ cargo new --bin greet-server && cd greet-server
{% endhighlight %}

Before jumping into the Rust, we set up a `proto` subdirectory and a gRPC service definition that defines the method our service provides (`Greet`) as well as the required input (`GreetRequest`) and output (`GreetReply`):

{% highlight proto %}
// proto/greeter_service.proto

syntax = "proto3";
package greeter_service;

service GreeterService {
    rpc Greet (GreetRequest) returns (GreetReply);
}

message GreetRequest {
    string greeting = 1;
    string name = 2;
}

message GreetReply {
    string message = 1;
}
{% endhighlight %}
In a nutshell, the above proto file just says defines in a language-agnostic manner the input and output such that we can use the protocol buffer compiler to compile language-specific types for Rust (which we will use for our server) and Python (which we will use for our client).

With the proto defined as above, we now set up the Rust project such that it knows how to compile and use the Rust-specific types generated by the protocol buffer compiler:

{% highlight toml %}
# Cargo.toml

[package]
name = "greet-server"
version = "0.1.0"
edition = "2021"

[dependencies]
tonic = "0.12"
prost = "0.13"
tokio = { version = "1.40", features = ["macros", "rt-multi-thread"] }

[build-dependencies]
tonic-build = "0.12"
{% endhighlight %}

Here, `tokio` and `prost` are required additional dependencies to use `tonic` to build an asynchronous gRPC server based on the proto file we just defined.
To do this, we define a `build.rs` file in the current working directoy that indicates to tonic that we would like to compile our proto:

{% highlight rust %}
// build.rs

fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("proto/greeter_service.proto")?;
    Ok(())
}
{% endhighlight %}
From here, we can implement the main body of our service which consists of two pieces:
- We must define in Rust how we want to transform a `GreetRequest` into a `GreetResponse` when `Greet` is called.
- We must spin up a server to run our service.

This can be seen below:
{% highlight rust %}
// src/main.rs

use tonic::{transport::Server, Request, Response, Status};

use greeter_service::greeter_service_server::{GreeterService, GreeterServiceServer};
use greeter_service::{GreetReply, GreetRequest};

pub mod greeter_service {
    tonic::include_proto!("greeter_service");
}

#[derive(Debug, Default)]
pub struct GreeterServiceImpl {}

#[tonic::async_trait]
impl GreeterService for GreeterServiceImpl {
    async fn greet(
        &self,
        request: Request<GreetRequest>,
    ) -> Result<Response<GreetReply>, Status> {
        let greet_request = request.into_inner();
        let message =
            format!("{}, {}", greet_request.greeting, greet_request.name);
        let reply = greeter_service::GreetReply { message };

        Ok(Response::new(reply))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse()?;
    let greeter = GreeterServiceImpl::default();

    Server::builder()
        .add_service(GreeterServiceServer::new(greeter))
        .serve(addr)
        .await?;

    Ok(())
}
{% endhighlight %}

OK!  That finishes the implementation of the service itself, which is roughly analogous to what we have done previously for wrappers: we had to define the implementation of the function in the language that we want to implement it in.  It remains to actually invoke this function (via gRPC) using Python.
To accomplish this, we must compile the Python-language interface for the gRPC service that we defined previously:

{% highlight bash %}
$ conda install grpcio grpcio-tools
$ mkdir greet-client
$ python -m grpc_tools.protoc -Iproto --python_out=greet-client --pyi_out=greet-client --grpc_python_out=greet-client proto/greeter_service.proto
{% endhighlight %}
At this point, we can tell our Python script where to find the Rust server and invoke `Greet`:

{% highlight python %}
# greet-client/grpc_example.py

import grpc
import greeter_service_pb2
import greeter_service_pb2_grpc

def run():
    with grpc.insecure_channel("[::1]:50051") as channel:
        stub = greeter_service_pb2_grpc.GreeterServiceStub(channel)
        request = greeter_service_pb2.GreetRequest(
            greeting="Aloha",
            name="amigo",
        )
        response = stub.Greet(request)
        print(response.message)


if __name__ == "__main__":
    run()
{% endhighlight %}

To see this in action, we start up the Rust server that we previously implemented and, while it is running, run the Python script we just wrote:

{% highlight bash %}
$ cargo run
# (Run in a separate tab or put the previous command in the background.)
$ python greet-client/grpc_example.py
# Prints "Aloha, amigo"
{% endhighlight %}

Ta-da!

##  Mojo 🔥: deferred, for now

I originally said that we would be looking at  `Mojo 🔥` as part of this series of posts, but I have struggled to find anything worthwhile to say about it at this level of investigation.  This is not a slight against Mojo -- it just doesn't have much to offer for simple string processing helpers.

### Parting Thoughts

Compared to the previous post, we've seen a lot less new syntax this time around, which is nice -- SWIG, Cython, and C extensions just feel pretty cumbersome, whereas CFFI, PyO3, cppyy (previous post) hide a lot of this difficulty (and obviously gRPC is a completely different beast).  That said, we have not done any sort of discussion of computational efficiency or profiling here, and so it is not entirely fair to judge these tools based just on their syntax.  At the end of the day, it is generally easiest to minimize the "surface area" between languages when using bindings such as these, and so setting up the wrappers for passing information around is ideally a "one-and-done" operation and not something developers are doing every day.  As such, it makes a lot of sense to accept the pain of figuring out a gross domain specific language every once in a while if there are other benefits, and I don't know how these different tools compare on performance.

The most flexible but least performant solution here, in my experience, is gRPC, and I have not seen it employed in my career outside of my time at Alphabet.
To be fair, I've not seen Python seriously employed for any truly performance-critical pieces of code except e.g., to drop into CUDA via `cupy` (or `pytorch` or `tensorflow`).
From a usability standpoint, I've spent fair amounts of time debugging issues in memory management in
vanilla extensions and more time than I care to admit figuring out `ctypes` bindings (not covered in this series) for pre-compiled libraries, and view the complexity of those solutions as an important factor to consider.  That said, I don't think there's a universal recommendation that can be made for how to best interface Python with other languages without a careful understanding of what "the point" is.  I'm open to hearing opinions if anyone feels strongly!