---
layout: post
comments: true
title: C++ Deep Learning-1 PyTorch C++ API LibTorch Introduction
lang: en
header:
  teaser: "https://pytorch.org/assets/images/logo-dark.svg"
tags: [Deep Learning, PyTorch, LibTorch, PyTorch C++ API, Machine Learning C++, Deep Learning C++, Machine Learning]
---

Many of us have encountered various discussions in different forums about which is the best language for Machine Learning or its popular subfield Deep Learning (and usually the best ML library discussions are added to it). If you haven't encountered it yet, don't get too excited, you will soon. In my opinion, the most accurate answer to this question can be *"it depends on the situation"*. Also, Elon Musk's tweet on February 2, 2020, was a good indicator of this.

![](/assets/img/libtorch-intro/elon_musk.png)

At this point, instead of making more noise on this topic, I will move on to what you can find in this article. The sections of the article:

# 1. Introduction

Especially when you want to use models with low latency/near real-time in the production phase, you encounter a few problems. I think the most critical problems are the speed/latency issues experienced in *Python* (I don't mean that the language has a different claim) and problems arising from multi-process/thread usage (I mean the issues restricted by Global Interpreter Lock in object access). At this point, you may want to use some conveniences provided by a language like *C++*. I plan to write articles explaining how libraries like *ONNX*, *TensorRT* can be included in this regard. But in this article, I will talk about the *LibTorch* library provided by *PyTorch* to developers, which has a not too steep learning curve. This introductory article will be followed by separate articles explaining tensor operations, performing inference, and creating/training a model from scratch in C++.

The *LibTorch* library is the *C++ API* of *PyTorch* that entered our lives with *PyTorch 1.0* version. It is used by Facebook for both research and production. Almost all features used on the *Python* side are also available in the *C++ API*. However, although a bit behind, I generally observe that the features used in *PyTorch* are added to the *C++ API* with a 1-version difference. In its own documentation, it emphasizes that we should evaluate this feature as *"beta"* and that *PyTorch*'s *Python* interface is more stable. However, I have been using it for a long time and I would like to state that I have not experienced any significant problems.

So what does *LibTorch* offer us:

- An interface to define machine learning models (equivalent to `torch.nn.Module` in Python),
- A standard library for the most common modules (convolution, recurrent networks, batch normalization, etc.),
- Optimization API (SGD, Adam, etc.),
- Data sets and data pipelines that allow parallel processing on CPU cores,
- Ability to automatically parallelize models on GPU (`torch.nn.parallel.DataParallel`),
- Ability to easily use *C++* models in Python,
- Use of TorchScript JIT compiler,
- ATen (basic tensor interface) and Autograd API (API that automatically calculates gradients on the computational graph).

You can reach the list and definitions of the components it offers from [its documentation](https://pytorch.org/cppdocs/frontend.html).

# 2. Usage Scenarios

So what can be the usage scenarios of *LibTorch* in *C++*?

- In my opinion, first of all, in the production phase, a model that is completely developed, trained, and saved in *Python* can be read and inference can be performed directly. Meanwhile, you can use advantages like multi-process/thread usage, low latency. I will cover its use for direct inference in the 3rd article of this series. Of course, you will be able to find different comparisons in that article.
- The model can be created/trained from scratch in *C++*. In the 4th article of the series, I will write and train the model from scratch in *C++*.
- You can write your own extensions on PyTorch.

# 3. [Installation](#installation)

There are two options for installation:

 1. Download the compiled library:

    1. Click on the [link](https://pytorch.org/get-started/locally/).

    2. First click on the Stable link, then on the operating system link you use, then on the LibTorch link, then on the C++/Java link, and finally on the CUDA version link for GPU or None link if you will only use CPU, and download the file link that comes and copy its contents to any directory you want.

         ![Download the compiled library](/assets/img/libtorch-intro/start_locally.png)

 2. Compile from [source code](https://github.com/pytorch/pytorch#from-source): It can be a bit troublesome (actually quite troublesome). But personally, this is my preference. Clone the GitHub repo and follow the necessary steps (I skip this section so it doesn't get too long, but I can dedicate an article to this in the future).

# 4. `Hello LibTorch`

What I will explain in this article series will include using [Clion](https://www.jetbrains.com/clion/) as the development editor and the `cmake` tool on Linux operating system, but you can adapt the same procedures to your own operating system and compilation tool. Let's start and first create a file named `CMakeLists.txt` and add the following content:

```cmake
cmake_minimum_required(VERSION 3.17) # You can use your current cmake installation as long as the cmake version is at least 3.0
project(libtorchHelloWorld)

set(CMAKE_CXX_STANDARD 14)

find_package(Torch 1.7.0 REQUIRED) # Currently the available version is 1.7.0, correct this if you use a different version
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")

add_executable(libtorchHelloWorld main.cpp) 
target_link_libraries(libtorchHelloWorld ${TORCH_LIBRARIES})
```

Now let's add our source codes in `main.cpp`:

```c++
#include <iostream>
#include <torch/torch.h>

int main() {
    torch::Tensor randTensor = torch::rand({2, 3});
    
    std::cout << "Hello LibTorch\n" << "Torch Tensor: " << randTensor << "\n";
}
```

And let's create a directory to contain the compilation outputs and start compiling in it (`-DCMAKE_PREFIX_PATH=/absolute/path/libtorch` the path you specify should be the directory where you copied the content from the [installation](#installation) section):

```shell
$ mkdir build
$ cd build
$ cmake -DCMAKE_PREFIX_PATH=/absolute/path/libtorch ..
$ cmake --build . --config Debug
```

Now let's run our compiled file and fulfill the Hello World classic:

```shell
$ ./libtorchHelloWorld
Hello Libtorch
Torch Tensor:  0.6147  0.6752  0.8963
 0.5627  0.4836  0.5589
[ CPUFloatType{2,3} ]
```

Now let's take a closer look at the API by looking at this short application. First, the `torch/torch.h` header file contains the `torch/all.h` header file that includes all other header files of *LibTorch*:

```c++
#pragma once

#include <torch/all.h>

#ifdef TORCH_API_INCLUDE_EXTENSION_H
#include <torch/extension.h>

#endif // defined(TORCH_API_INCLUDE_EXTENSION_H)
```

If we examine the `torch/all.h` header file, we see that we have all the necessary header files. At this point, we have seen that we do not need any other header file besides the `torch/torch.h` header file:

```c++
#pragma once

#include <torch/cuda.h>
#include <torch/data.h>
#include <torch/enum.h>
#include <torch/jit.h>
#include <torch/nn.h>
#include <torch/optim.h>
#include <torch/serialize.h>
#include <torch/types.h>
#include <torch/utils.h>
#include <torch/autograd.h>
```

Actually, as a tip, if I say that when using *PyTorch*, you mostly write valid code for the *C++ API* by changing the `.` operator you use to `::`, I don't think it would be too misleading. Therefore, the variable named `randTensor` we defined in the `main` function is an instance of the `torch.tensor` class we are familiar with from the *Python API*. I will examine this class in more detail later.

```c++
	torch::Tensor randTensor = torch::rand({2, 3});
```

With the above code, we obtain a `tensor` filled with random values in the shape of `2, 3` by using one of the different overloads of the `rand` function defined in the `torch` namespace.

```c++
	std::cout << "Hello LibTorch\n" << "Torch Tensor: " << randTensor << "\n";
```

In the above line of code, when `randTensor` is the right operand of the `<<` operator, it calls the `print` function in the `at` namespace (this namespace contains the ATen library, which is the basic tensor library used by LibTorch and PyTorch) and writes the tensor data, data type, and shape to the standard output. The `torch::Tensor` class provides a lot of things just like in the *Python API*. But they will be in the next article. We have reached the end of this article. As I mentioned in the introduction, this article will be followed by a series of articles. After writing the new sections, I will add their links under this article.

*Other articles in the series:*

2. [C++ Deep Learning-2 PyTorch C++ API LibTorch Tensor Operations](https://blgnksy.github.io/2020/12/06/libtorch-tensors.html)

3. [C++ Deep Learning-3 PyTorch C++ API LibTorch Running Models](https://blgnksy.github.io/2020/12/13/libtorch-inference.html)