# A Deep research of `constexpr` by `.S` file and GDB

## The Test File
```c++
constexpr int square(int x) { return x*x; }

int main() {
    constexpr int a = square(5);  // 25 is calc at compile time
    int y = 10;
    int b = square(y);            // at runtime
}
```

## Are They Really Computed At the Time They Should Be?
Lets see if above is right.  
```bash
$ g++ -O2 -std=c++17 -S constexpr.cpp -o constexpr.S
$ cat ./constexpr.S | grep 25
        movl    $25, %esi
```

So it is clear that there is a literal put in `%esi`, where `%esi` is  

> what is %esi used for?
> 
> ChatGPT said:
> 
> Ah, %esi is one of the general-purpose registers in x86-64 assembly.

We do have a variable/argument equals to 25, which cannot be others except a, while for b, it should **not** be calculated...  

```bash
$ cat ./constexpr.S | grep 100
        movl    $100, %esi
```

Oops, we have literal 100 at compile time. Why?  

```bash
$ g++ -O0 -std=c++17 -S constexpr.cpp -o constexpr.S
$ cat ./constexpr.S | grep 25
        movl    $25, -12(%rbp)
        movl    $25, %esi
$ cat ./constexpr.S | grep 100
$
```

That reason is: the `-O0` optimisation does *constant folding*, while `-O2` does *constant folding* and *constant propagation*.   
    - *constant folding*: if we know a value of constant, just write the result.  
    - *constant propagation*: if we know a value of a var by another var with constant value, we write it in. - Actually `-O0` parially does this as well  


So if we use `-O0`, we can see why there `constexpr` can be calc at runtime if the param is not constant, and it should not be propagated  

```bash
$ cat << EOF > const-op.cpp
int main() {
    int x = 3*3;
    int y = x + 1;
    return x;
}
EOF
$ g++ -O0 -S const-op.cpp -o const-op.S
$ cat const-op.S | grep 10
$ g++ -O2 -S const-op.cpp -o const-op.S
$ cat const-op.S | grep 10
        movl    $10, %eax
$ 
```

Back to our example, we should not see literal 100 as it is not computed at compile time.  

```bash
$ g++ -O2 -std=c++17 -S constexpr.cpp -o constexpr.S
$ cat ./constexpr.S | grep 25
        movl    $25, %esi
$ cat ./constexpr.S | grep 100
        movl    $100, %esi
$ g++ -O0 -std=c++17 -S constexpr.cpp -o constexpr.S
$ cat ./constexpr.S | grep 25
        movl    $25, -12(%rbp)
        movl    $25, %esi
$ cat ./constexpr.S | grep 100
$
```

That's it!

## Can We Use GDB?

However, reading from `.S` file are annoying, that's why we have gdb  

```gdb
...
Breakpoint 1, main () at constexpr.cpp:6
6           constexpr int a = square(5);  // 编译期算出 25
(gdb) p a
$1 = 0
(gdb) p b
$2 = 32767
(gdb) s
7           int y = 10;
(gdb) p a
$3 = 25
(gdb) p b
$4 = 32767
(gdb) s
8           int b = square(y);            // 运行期算
(gdb) s
square (x=10) at constexpr.cpp:3
3       constexpr int square(int x) { return x*x; }
(gdb) p a
$5 = {i = {0, 1045149306}, x = 1.2904777690891933e-08, d = 1.2904777690891933e-08}
(gdb) p b
$6 = {i = {0, 1068498944}, x = 0.0625, d = 0.0625}
(gdb) 
```

We can see that `a` is set by literal 25 directly, while we must go into the `squre()` function to compute the outcome for `b`!


### However, why `a` is init as `0`?
- [ ] The local var should not be set as `0`, and there seems no corresponding process in `.S`...
