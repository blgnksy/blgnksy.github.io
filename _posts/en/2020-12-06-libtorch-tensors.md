---
layout: post
comments: true
title: C++ Deep Learning-2 PyTorch C++ API LibTorch Tensor Operations
lang: en
header:
  teaser: "https://pytorch.org/assets/images/logo-dark.svg"
tags: [Deep Learning, PyTorch, LibTorch, PyTorch C++ API, Machine Learning C++, Deep Learning C++, Machine Learning, Tensors, Tensor Operations]
---
In this article of the series, I will explain how tensors are created, accessed, and modified in *LibTorch*. If you haven't read the introductory article where I explained what *LibTorch* is and what can be done, I recommend starting from [that article](https://blgnksy.github.io/2020/12/03/libtorch-config.html). At this point, remember that this article could have been much longer, but it has been filtered to provide all possible simplicity but sufficient information, and the main reference is the documentation itself.

The ATen tensor library is the library that *PyTorch* uses in the background for tensor operations and is written in accordance with C++14 standards. Tensor types are resolved dynamically. As a result, regardless of the data type it holds or whether it is a CPU/GPU tensor, a single tensor interface meets us. You can examine the interface of the tensor class from [the documentation](https://pytorch.org/cppdocs/api/classat_1_1_tensor.html#exhale-class-classat-1-1-tensor).

There are hundreds of functions that operate on tensors. You can reach the [list of these functions](https://pytorch.org/cppdocs/api/namespace_at.html#functions) using the link. Regarding function naming, I want to draw your attention to the point that functions ending with the `_` character make changes on the tensor, that is, the tensor is passed as a left-side reference (lvalue reference) in C++ when this function is called. Now let's start using this class:



# 1. Tensor Creation:

## 1.1 Using Factory Functions

These functions work as in *Factory Design Patterns* and ultimately return `torch::Tensor`. In fact, I used one of them in the first [article](https://blgnksy.github.io/2020/12/03/libtorch-config.html): the `torch::rand()` function returns a tensor according to the shape it takes as an argument. These functions are:

- [arange](https://pytorch.org/docs/stable/torch.html#torch.arange): Returns a tensor of sequential integers,
- [empty](https://pytorch.org/docs/stable/torch.html#torch.empty): Uninitialized,
- [eye](https://pytorch.org/docs/stable/torch.html#torch.eye): Returns an identity matrix,
- [full](https://pytorch.org/docs/stable/torch.html#torch.full): Returns a tensor filled with a single value,
- [linspace](https://pytorch.org/docs/stable/torch.html#torch.linspace): Returns a tensor with values linearly spaced in some interval,
- [logspace](https://pytorch.org/docs/stable/torch.html#torch.logspace): Returns a tensor with values logarithmically spaced in some interval,
- [ones](https://pytorch.org/docs/stable/torch.html#torch.ones): Returns a tensor filled with all ones,
- [rand](https://pytorch.org/docs/stable/torch.html#torch.rand): Returns a tensor filled with values drawn from a uniform distribution on `[0, 1)`.
- [randint](https://pytorch.org/docs/stable/torch.html#torch.randint): Returns a tensor with integers randomly drawn from an interval,
- [randn](https://pytorch.org/docs/stable/torch.html#torch.randn): Returns a tensor filled with values drawn from a unit normal distribution,
- [randperm](https://pytorch.org/docs/stable/torch.html#torch.randperm): Returns a tensor filled with a random permutation of integers in some interval,
- [zeros](https://pytorch.org/docs/stable/torch.html#torch.zeros): Returns a tensor filled with all zeros.

The links give connections to the *Python* documentation. The functions, parameters, and named arguments in the C++ API are the same. Note that named arguments can be defined, accessed, and modified via the `torch::TensorOptions` object. I will cover this in the `torch:rand()` function shortly, and it will be valid for other functions as well.

Now let's take a closer look at useful factory functions (Note: I will examine the first function in detail, I recommend using the documentation for the others. Because APIs are subject to rapid change and it is tedious to keep them synchronized.):

### 1.1.1. `torch::rand()`

This function produces random floating-point numbers in the range `[0,1)`. Let's look at the function's prototype: `torch::rand(*size, *, out=None, dtype=None, layout=torch.strided, device=None, requires_grad=False) → Tensor`.

#### Parameters

- **size** (int): The parameter that determines the shape of the tensor. Takes integer values.

#### Named Arguments- **out** ([*Tensor*](https://pytorch.org/docs/stable/tensors.html#torch.Tensor)*,* *optional*) – The output tensor.
- **dtype** ([`torch.dtype`](https://pytorch.org/docs/stable/tensor_attributes.html#torch.torch.dtype), optional) – The tensor data type.
- **layout** ([`torch.layout`](https://pytorch.org/docs/stable/tensor_attributes.html#torch.torch.layout), optional) – The tensor's memory storage method (`dense`/`strided`). This option is planned to be removed in future versions.
- **device** ([`torch.device`](https://pytorch.org/docs/stable/tensor_attributes.html#torch.torch.device), optional) – Which device the tensor will be stored on (CPU/GPU).
- **requires_grad** ([*bool*](https://docs.python.org/3/library/functions.html#bool)*,* *optional*) – Whether the returned tensor will be subject to automatic gradient computation.

#### Usage:

The simplest usage is to give the shape and use the tensor.

```c++
torch::Tensor randTensor = torch::rand(/*size:*/{2, 3});
```

As you remember from the first article, this usage returned a CPU tensor with shape `2, 3`. If we print this tensor to the standard output stream, we get the following output (remember that the floating-point numbers in the outputs will be random):

```shell
0.6147  0.6752  0.8963
0.5627  0.4836  0.5589
[ CPUFloatType{2,3} ]
```

For the use of named arguments, we first need to define these options within a `torch::TensorOptions` object. **Reminding that these options are used in the same way in other factory functions** let's look at the values they can take:

- `dtype`: `kUInt8`, `kInt8`, `kInt16`, `kInt32`, `kInt64`, `kFloat32` and `kFloat64`,
-  `layout`: `kStrided` and `kSparse`,
-  `device`: `kCPU` or `kCUDA` (takes device index if you have multiple GPUs),
-  `requires_grad`: `true` or `false`.

Now let's create a tensor using the options. To obtain a tensor with 32-bit floating-point numbers in `strided` memory layout, on GPU 0, that will be included in automatic gradient, we can use the following code block:

```c++
auto options = torch::TensorOptions()
            .dtype(torch::kFloat32)
            .layout(torch::kStrided)
            .device(torch::kCUDA, 0)
            .requires_grad(true);

torch::Tensor randTensor = torch::rand(/*size:*/{2, 3}, options);
```

You can also use one or several of these options directly as functions. These functions return `torch::TensorOptions` object as expected:

```c++
torch::Tensor randTensor = torch::rand(/*size:*/{2, 3}, torch::TensorOptions().dtype(torch::kFloat32));
/* or
torch::Tensor randTensor = torch::rand({2, 3}, torch::dtype(torch::kFloat32));
torch::Tensor randTensor = torch::rand({2, 3}, torch::dtype(torch::kFloat32).device(torch::kCUDA, 0));
*/
```

### 1.1.2. `torch::randint()`

This function returns a tensor of integers drawn uniformly from a given interval. Let's look at its usage immediately:

```c++
auto intTensor = torch::randint(/*low:*/1, /*high:*/9, /*size:*/{3});
```

With the above definition, we create a tensor that will be a vector with 3 elements between 1 and 9. If we look at the output of this tensor:

```shell
 6
 1
 1
[ CPUFloatType{3} ]
```

Let's create a 3D tensor (you can think of it as a tensor carrying typical image data):

```c++
auto intTensor = torch::randint(/*low:*/1, /*high:*/9, /*size:*/{1920, 1080, 3});
```

### 1.1.3. torch::ones()/torch::zeros()

As their names suggest, these functions allow you to create tensors consisting of ones or zeros. Their usage is similar to other functions again. For example:

```c++
auto onesTensor = torch::ones(/*size:*/{5, 10});
auto zerosTensor = torch::zeros(/*size:*/{1, 5, 10});
```

### 1.1.4. `torch::from_blob()`

Mostly, we read the data that will create the tensor from another source and transfer it to this tensor. To do this, there is a useful function that takes the data as `void*` and returns the tensor.:

```c++
float data[] = {1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0 8.0, 9.0, 10.0};
auto blobData = torch::from_blob(data, /*size:*/{2, 5});
/*
  1   2   3   4   5
  6   7   8   9  10
[ CPUFloatType{2,5} ]
*/
```

**This function does not take ownership of the data sent to its argument. However, when the tensor object's life ends, it also deletes the original object from memory.** Again, if you wish, you can use tensor options.
```c++
auto blobDataD = torch::from_blob(data, {1, 10}, torch::requires_grad(False));
```

### 1.1.5. `torch::tensor` function

This function, which directly uses the constructor functions of the class, also allows creating tensors.

```C++
auto tensorInit = torch::tensor({1.0, 2.0, 4.0, 2.0, 3.0, 5.0});
```

Leaving other factory functions to the documentation, now let's examine the member functions of the `torch::Tensor` class.



# 2. `torch::Tensor` Class Member Functions

Now we have our tensor and we want to get information about it/modify it. Here, let's look at the most important member functions provided by the `torch::Tensor` class:

```c++
// Let's create a 1D tensor
auto tensorInit = torch::tensor({1.0, 2.0, 4.0, 2.0, 3.0, 5.0, 1.0, 2.0, 4.0, 2.0, 3.0, 5.0});
// Convert to 2D tensor
tensorInit = tensorInit.reshape({2,6});

// The dim() class member function returns how many dimensions the tensor has. In our example, 2:
auto tDims = tensorInit.dim();

// The dtype() class member function returns the data type of the tensor. In our example, float:
auto tDtype = tensorInit.dtype();

// The sizes() class member function returns the shape of the data held by the tensor. In our example, [2, 6]:
auto f = tensorInit.sizes();
```



# 3. Tensor Element Access, Indexing and Modification

There are multiple methods for accessing tensor elements. First, using one of the `torch::Tensor` class member functions `torch::Tensor.data_ptr()` function, we can access all data. Alternatively, it is possible to use the `data()` function and access one of its elements as in *Python*.

```c++
auto tensorInit = torch::tensor({1.0, 2.0, 4.0, 2.0, 3.0, 5.0, 11.0, 12.0, 14.0, 12.0, 13.0, 15.0});
tensorInit = tensorInit.reshape({2,6});

// Returns a pointer of type void*
auto pDataVoid  = tensorInit.data_ptr();
// Assuming the data type is float, we can convert and access elements using the pointer.
auto pDataFloat = static_cast<float*>(pDataVoid);
std::cout << pDataFloat[7] << "\n"; // In two dimensions [1][1], in one dimension 7th element, i.e. 12

// Alternatively
// We can also access the data directly at index 1,0 using the data function.
std::cout << tensorInit.data()[1][1] <<"\n" ; // in the example 12
```

Another alternative and recommended method offered by the *LibTorch* library for data access is the use of `accessor`. Here, separate `accessor` must be used for CPU and GPU. First, let's use this operation for a CPU tensor, then for a GPU tensor:

 ```c++
auto tensorInit = torch::tensor({1.0, 2.0, 4.0, 2.0, 3.0, 5.0, 11.0, 12.0, 14.0, 12.0, 13.0, 15.0});
tensorInit = tensorInit.reshape({2,6});

auto tensorInitAccessor = tensorInit.accessor<float, 2>();
for (int i = 0; i < tensorInitAccessor.size(0); i++)
    for (int j = 0; j < tensorInitAccessor.size(1); j++) {
        std::cout << "Data in position " << i << "-" << j << ": " << tensorInitAccessor[i][j] << "\n";
    }
 ```

The point to be careful about in this usage is that we need to send the data type `float` and the dimension `2` to the template parameters of the `accessor` object. So, specialization will be required for different data types and dimensions. So I can't say it shortens things a lot. The documentation claims that it provides faster access, but since I don't think pointer usage is slow, I wanted to test accessing all elements in a large tensor:

```c++
#include <iostream>
#include <torch/torch.h>
#include <chrono>

int main() {
    using namespace std::chrono;
    
    const int HEIGHT = 1920, WIDTH = 1080, CH = 300;
    long double sum = 0;
    
    auto randBigTensor = torch::rand({HEIGHT, WIDTH, CH});
    
    auto start = high_resolution_clock::now();
    auto tensorInitAccessor = randBigTensor.accessor<float, 3>();
    for (int i = 0; i < tensorInitAccessor.size(0); i++)
        for (int j = 0; j < tensorInitAccessor.size(1); j++)
            for (int k = 0; k < tensorInitAccessor.size(2); k++) {
                sum += (tensorInitAccessor[i][j][k]) / 1000;
            }
    auto end = high_resolution_clock::now();
    duration<double> time_span = duration_cast<duration<double>>(end - start);
    std::cout << "It took me " << time_span.count() << " seconds (Using accesscor). Sum = " << sum << "\n";

    sum = 0;
    start = high_resolution_clock::now();
    auto pDataVoid = randBigTensor.data_ptr();
    auto pDataFloat = static_cast<float *>(pDataVoid);
    for (int i = 0; i < (HEIGHT * WIDTH * CH); i++)
        sum += (pDataFloat[i]) / 1000;
    end = high_resolution_clock::now();
    time_span = duration_cast<duration<double>>(end - start);
    std::cout << "It took me " << time_span.count() << " seconds (Using data_ptr). Sum = " << sum << "\n";
    
    return 0;
}
```

The results were actually as I expected. The fastest was to take the data pointer and access the data. But I think it would be more logical to test this with different sizes, even ideally with tensor sizes you will use in your application.

```shell
It took me 18.573 seconds (Using accesscor). Sum = 311033
It took me 4.24126 seconds (Using data_ptr). Sum = 311033
```

Sometimes we may want to access certain elements of this data. In this case, using the Indexing API we are familiar with from *Python* will be easier. Both reading and writing operations are possible in this API:

```c++
auto randTensor = torch::rand({100, 100, 3});

// Element at position 1,0,5
std::cout << randTensor.index({1,0,5}); 

// Using the Slice function, it takes the data in dimension 0 starting from index 1 up to 10 in steps of 2, and shows the elements corresponding to index 0 in the other two axes. Note that the torch::indexing namespace is added here.
using namespace torch::indexing;
std::cout << randTensor.index({Slice(/*start_idx:*/1, /*stop_idx:*/10, /*step:*/2), 0, 0})}); 

// Assign value to element at position 1,0,5
randTensor.index({1,0,5}) = 0.05
```

For comparison of Indexing API usage for *Python* and *C++*, click on the [link](https://pytorch.org/cppdocs/notes/tensor_indexing.html).

# 4. Conversion Operations

Before ending this article, finally, let's look at the conversion operations of tensors. Here, it is possible to convert the tensor options we determined during initial creation.

```c++
auto sourceTensor = torch::randn({2, 3}, torch::kFloat16);

// Data type conversion
auto floatTensor32 = sourceTensor.to(torch::kFloat32);

// Conversion according to device type
auto gpuTensor = floatTensor32.to(torch::kCUDA);
```

Yes, we have come to the end of the article. In the next article of the series, we will transition to models, and when the article is ready, I will add the link under this article.

*Other articles in the series:*

1. [C++ Deep Learning-1 PyTorch C++ API LibTorch Introduction](https://blgnksy.github.io/2020/12/03/libtorch-config.html)

3. [C++ Deep Learning-3 PyTorch C++ API LibTorch Running Models](https://blgnksy.github.io/2020/12/13/libtorch-inference.html)
