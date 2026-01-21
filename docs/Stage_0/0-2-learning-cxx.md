# learning-cxx

## Declaration

- 总结：解读复杂声明的三步法  
    - 从变量名开始。  
    - 先看右边再看左边，括号优先。  
    - 遇到类型修饰符向外扩展直到到达基本类型。  

### What is `const int* x` and `int* const x`?
- `const int* x`: a pointer pointing to a `const int` var  
    - `* x`: x is a pointer  
    - `const int`: pointing to `const int`  
- `int* const x`: a `const` pointer pointing to an `int` var  
    - `const x`: `x` is `const`  
    - `int*`: the const `x` is a pointer and pointing to `int`

## Parameter passing method

| 方式                                       | 语法示例                                                   | 特点                          | 适用场景                                   |
| ---------------------------------------- | ------------------------------------------------------ | --------------------------- | -------------------------------------- |
| **值传递** (Pass by Value)                  | `void foo(int x)`                                      | **拷贝**一份参数，函数内部修改不影响外部      | 小型数据类型（`int`, `char`, 小型结构体等）；保证外部数据安全 |
| **指针传递** (Pass by Pointer)               | `void foo(int* p)`                                     | 传递地址，可修改外部数据；可能传入 `nullptr` | 需要可选（可为空）修改参数，或数组、动态分配对象               |
| **引用传递** (Pass by Reference)             | `void foo(int& x)`                                     | 直接操作外部变量，**不能为 null**       | 必须修改外部数据且保证非空                          |
| **const 引用传递** (Pass by const Reference) | `void foo(const std::string& s)`                       | 避免拷贝，保证只读                   | 大对象（`std::string`, `vector`等）且不需要修改    |
| **右值引用传递** (Pass by Rvalue Reference)    | `void foo(std::string&& s)`                            | 绑定临时对象，可**移动语义**            | 需要接管临时对象所有权，避免拷贝                       |
| **通用引用** (Forwarding Reference)          | `template <typename T> void foo(T&& t)`                | 可同时绑定左值和右值                  | 模板中实现完美转发（`std::forward`）              |
| **数组传递**                                 | `void foo(int arr[])` 或 `void foo(int* arr, size_t n)` | 实际是指针传递                     | 处理连续内存数据                               |
| **初始化列表传递**                              | `void foo(std::initializer_list<int> list)`            | 支持 `{}` 列表语法                | 配合容器初始化参数                              |


### What is the diff btw *lvalue reference* and *pointer*?

| 特性    | **左值引用** (`T&`)    | **指针** (`T*`)    |
| ----- | ------------------ | ---------------- |
| 使用    | 直接当作变量用 `r = 5;`   | 需要解引用 `*p = 5;`  |
| 可为空   | ❌（必须绑定有效对象）        | ✅（可以 `nullptr`）  |
| 可重新指向 | ❌（引用一旦绑定就不能换）      | ✅（可以改变指向的地址）     |
| 存储开销  | 编译器实现类似常量指针（底层有地址） | 就是一个指针变量         |

### *lvalue reference* vs *rvalue reference*

| 类型         | 能绑定左值?            | 能绑定右值? |
| ---------- | ----------------- | ------ |
| `T&`       | ✅                 | ❌      |
| `const T&` | ✅                 | ✅      |
| `T&&`      | ❌（除非 `std::move`） | ✅      |


So the *lvalue reference* if for `var`, as a more decent way for pointers, while the *rvalue reference* is to avoid passing by value of large mem vars.  

更精确的总结:  
    - T&：绑定左值，语法像值传递，语义是“引用同一个对象”  
    - T&&：绑定右值（临时对象），允许“偷”资源（移动）  
    - const T&：左值和右值都能绑定（只读），常用于高效只读传参  
    - T&& 在模板：可能是右值引用，也可能是通用引用（取决于类型推导）  

## `static`
3 levels:  
1. Function level:  
    ```c++  
    void foo() {
        static int a = 0;
        return a++;
    }
    ```

    If calling the above function multiple times:  
    - 1st: ~~init `a=0`~~ (init when ~~compiling~~ still in runtime, but at the program starting, befor first calling 静态初始化), return `0` and increase `a` to 1.  
    - 2nd: static var will not init again, a is set to 1 after the first call, so this time it will return `1` and increase `a` to `2`.  
    - 3nd: return `2` and increase to `3`.   
    - ...  
    
    The initialisation will be processed when compiling as a is known in compilation. If the `a` in `foo()` is defined by the param of `foo()`, saying `void foo(int v) {static int a = v; return a++;}`, then the static var `a` is not known in compilation and thus it must be initialised in the runtime, i.e. the first time calling.       
2. File Level: 
    In the `a.cpp` file:   
    ```c++
    static int counter = 0; //var
    static int helper(int, int, int); // func
    ``` 
    and in `b.cpp` file:   
    ```c++
    static int counter = 0; //var
    static int helper(int, int, int); // func
    ```  
    The `counter` and `helper` in `a.cpp` and `b.cpp` are not the same - this will compiled successfully.  
    If there is no `static`, there would be a linker error *multiple definition of `counter`*, since all vars are `external` by default (saying they are visable across files), while static will prevent the visibility.  
3. static class level:  
    ```c++
    #include <iostream>
    class MyClass {
    public:
        static int shared_value; // 声明（类内）
    };

    int MyClass::shared_value = 0; // must have this line. If no def outside class, will raise linked error. 

    int main() {
        MyClass a, b;
        a.shared_value = 42; // changed for all classes  
        std::cout << b.shared_value << "\n"; // 输出 42
    }
    ```

## `constexpr`
The idea is simple: the result of a `constexpr` function must be known at compile time. **However**  
> 运行期调用的本质  
> constexpr 只是告诉编译器 如果参数是编译期常量就可以在编译期计算  
> 如果参数不是常量，编译器会把它当作普通函数执行 → 完全合法  


[A Deep research of `constexpr` by `.S` file and GDB](../9_Extra/01-constexpr.md)

## Pure functions 纯函数
- Always return same result  
- no side effect  
    - No modifying global/external state  
    - No modifying params  
    - No syscall e.g. I/O, exceptions, interrupts...  
