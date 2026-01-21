# Tiny Infini Train

A tiny version of [Infinitensor]() for taining.

One of the most knotty parts is the start point. Althought the assignment instructs all the needs, task, test dependencies explicitly, it is easy to be lost for a project of dozens of files and thousands of lines.  

However, Chatgpt is a good coach. 

## Task 1

Plus, the assignment for train is much better than Tensor for beginners, as it is so clear and the first task has a *template* - the task is to complete the missing part in `elementwise.cc`, while there are many similiar parts done as examples.

For the implementation, nothing is special, just copy and paste from one of others and edit the neccessary part. 

However, for better understanding, it is a pretty good example of how to understand a file in the project.

### Task 1 Overview

One should look into the task first.

```cpp
#include "infini_train/include/autograd/elementwise.h"

#include "infini_train/include/dispatcher.h"
#include "infini_train/include/tensor.h"
#include <optional>

namespace infini_train::autograd {
std::vector<std::shared_ptr<Tensor>> Neg::Forward(const std::vector<std::shared_ptr<Tensor>> &input_tensors) {
    // =================================== 作业 ===================================
    // TODO：通过Dispatcher获取设备专属kernel，对输入张量进行取反操作
    // NOTES: 依赖test_dispatcher，Neg kernel实现已给出
    // =================================== 作业 ===================================
}
```

First we look into the signatures. `std::vector<std::shared_ptr<Tensor>>` means this is a vector of shared ptrs for Tensor, and the argument is the same with just const reference so the argument be borrowed and remains not changed.  

Comparing to other operators, for instance, the `Reciprocal`:  

```cpp
std::vector<std::shared_ptr<Tensor>> Reciprocal::Forward(const std::vector<std::shared_ptr<Tensor>> &input_tensors) {
    CHECK_EQ(input_tensors.size(), 1);
    const auto &input = input_tensors[0];

    auto device = input->GetDevice().Type();
    auto kernel = Dispatcher::Instance().GetKernel({device, "ReciprocalForward"});
    return {kernel.Call<std::shared_ptr<Tensor>>(input)};
}

void Reciprocal::SetupContext(const std::vector<std::shared_ptr<Tensor>> &input_tensors,
                              const std::vector<std::shared_ptr<Tensor>> &) {
    const auto &input = input_tensors[0];
    saved_tensors_ = {input};
}
```

The implementation is clear, we first check the dimension of the argument. The *reciprocal* should recieve one argument only (while the *add* actually takes two) and so is the *neg* so we can just following the same steps. Then, we can get the device and the kernel and using the kernel call to return the value. Nothing is special.  

### Class `Tensor`
The next step is to look what is a `Tensor`. Just ctrl click, it redirect me to `tensor.h`.  

```cpp
class Tensor : public std::enable_shared_from_this<Tensor> {
public:
    // constructors
    Tensor() = default;

    Tensor(const std::vector<int64_t> &dims, DataType dtype, Device device);
    Tensor(const std::vector<int64_t> &dims, DataType dtype) : Tensor(dims, dtype, Device(DeviceType::kCPU, 0)) {}
    Tensor(const Tensor &tensor, size_t offset, const std::vector<int64_t> &dims);

    // member function prototypes.
    Device GetDevice() const;
    ...

    // operator overloading
    std::shared_ptr<Tensor> Equals(float scalar);

    // distribution
    std::shared_ptr<Tensor> Uniform(float from = 0.0f, float to = 1.0f,
                                    std::optional<std::mt19937> generator = std::nullopt);
    ...

    friend std::shared_ptr<Tensor> operator==(const std::shared_ptr<Tensor> &t, float scalar);
    ...

    void SaveAsNpy(const std::string &path) const;
    ...

private:
    std::shared_ptr<TensorBuffer> buffer_;
    size_t offset_ = 0;
    std::vector<int64_t> dims_;
    size_t num_elements_ = 0;
    DataType dtype_;

    // autograd related
public:
...
private:
...
};
```

The structure is quite clear. It is divided into 2 pieces, while the first half is about it self and the second part is about the autograd related function. 

#### Signature
First we look the signature:  
`class Tensor : public std::enable_shared_from_this<Tensor>`

The class is inherent from `std::enable_shared_from_this<Tensor>`. What does it do? 

> Chatgpt said:
>
> 那么 `std::enable_shared_from_this<T>` 是啥？
> 
> 它是 C++ 标准库提供的一个小工具类，作用是：
> 👉 让一个对象 **在自己内部安全地获取 `std::shared_ptr` 指针指向自己**。
>
> ```cpp
> class Tensor {
> public:
>     std::shared_ptr<Tensor> GetSelf() {
>         return std::shared_ptr<Tensor>(this);  // ⚠️ 错误用法！
>     }
> };
> ```
> 
> 这样写会 **创建一个新的 shared\_ptr**，导致引用计数错乱（甚至可能 double free）。
>
> 如果 `Tensor` 继承了 `std::enable_shared_from_this<Tensor>`：
> 
> ```cpp
> class Tensor : public std::enable_shared_from_this<Tensor> {
> public:
>     std::shared_ptr<Tensor> GetSelf() {
>         return shared_from_this(); // ✅ 正确，返回管理自己的 shared_ptr
>     }
> };
> ```
> 
> 现在 `shared_from_this()` 会返回一个和外部相同控制块的 `shared_ptr`，不会重复管理对象。
> 


#### Constructors

For the first part, there are 4 constructors. It is called *overloading*.

```cpp
Tensor() = default; // default

Tensor(const std::vector<int64_t> &dims, DataType dtype, Device device); // the most completed version
Tensor(const std::vector<int64_t> &dims, DataType dtype) : Tensor(dims, dtype, Device(DeviceType::kCPU, 0)) {} // delegating constructor, transferring the construction to the most completed version
Tensor(const Tensor &tensor, size_t offset, const std::vector<int64_t> &dims); // ???
```

The first 3 versions are easily reading, while the last one seems difficult to understand. The one might infer it is constructing a `Tensor` from an array rather than vector. But how can we know the exact implementation? It seems that there is no real codes!

That's right! It is the same thing for other functions as well. Hopefully, chatgpt helps:

### How to Read the Signature of Arugument with no name?

> 首先，这里的参数什么意思？我需要只读左值引用，这个没问题。但是为什么第二个参数没有名字？
> 
> ```cpp
> void Reciprocal::SetupContext(const std::vector<std::shared_ptr<Tensor>> &input_tensors,
>                               const std::vector<std::shared_ptr<Tensor>> &)
> ```
> 
> ChatGPT said:
> 
> **在 C++ 里，这是一种「参数占位」的技巧：不能去掉参数，因为函数签名必须匹配接口。不想用它，就不给名字，避免编译器警告“未使用变量”。**

That's it! To sync the interfaces with others, there must be 2 argument tough the second one is never used. To avoid the warning of unused variables, we give the type only without name. It is called **placeholder parameter**.

