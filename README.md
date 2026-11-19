<!---
{
  "id": "8459b214-c342-48ad-96c4-51213d49297a",
  "teaches": "Cpp: From Function Pointers to Function Objects",
  "depends_on": [
    "4ad8d26e-1c11-4940-bac3-a9d84e1eb4b6",
    "27ae3da1-b961-4922-b033-2f0a75802a36",
    "ce9d3ddc-a17a-4f76-b03a-3c85c14a154f"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-11-19",
  "keywords": [
    "C++",
    "function object",
    "functor",
    "operator()",
    "std::ref",
    "std::function",
    "std::greater",
    "templates",
    "g++"
  ]
}
--->

# Cpp: From Function Pointers to Function Objects

> In this exercise you build a class with `operator()`, give a callback its own state, find out that the standard algorithms copy their callbacks, write a function template that accepts any callable, and measure what a function pointer, a `std::function` and a function object cost. No `sudo` is required.

## Introduction

In the previous exercises a callback had no room for data. C++ solves this by combining two features you know: a class can **overload `operator()`**, and an object of that class can then be called like a function. Such an object is called a **function object**, or **functor**. It is a normal object, so it has members, a constructor and a lifetime, and the standard algorithms accept it wherever they accept a function pointer.

There is a second advantage. A function pointer has the same type for every function with that signature, so the compiler usually does not know which function will be called. The type of a function object is its class: the call is known at compile time and can be inlined. This is why `std::sort` with a function object is often faster than `std::sort` with a function pointer.

### 1.1) Further Readings and Other Sources

* [cppreference: Function objects](https://en.cppreference.com/w/cpp/utility/functional)
* [cppreference: FunctionObject (named requirement)](https://en.cppreference.com/w/cpp/named_req/FunctionObject)
* [cppreference: std::reference_wrapper](https://en.cppreference.com/w/cpp/utility/functional/reference_wrapper)
* [STEMgraph: Cpp: Function Objects - Functors](https://github.com/STEMgraph/e2c108ad-125e-45ac-8fe1-d2e3b13f48eb) (a shorter, earlier exercise on the same topic)

## Tasks

### Task 1: A class you can call

```bash
mkdir -p ~/cpp-functors && cd ~/cpp-functors
```

```c++
// file: functor_basic.cpp
#include <algorithm>
#include <iostream>
#include <vector>

// A class with operator(): objects of it can be called like functions.
struct Descending {
    bool operator()(int a, int b) const { return a > b; }
};

bool descending_fn(int a, int b) { return a > b; }

int main() {
    std::vector<int> v{3, 1, 4, 1, 5, 9, 2, 6};

    Descending d;
    std::cout << "d(2, 1) = " << d(2, 1) << std::endl;          // a call on an object

    std::sort(v.begin(), v.end(), Descending());                 // a temporary object
    for (int x : v) std::cout << x << " ";
    std::cout << std::endl;

    std::cout << "sizeof(Descending) = " << sizeof(Descending) << std::endl;
    std::cout << "sizeof(&descending_fn) = " << sizeof(&descending_fn) << std::endl;
}
```

```bash
g++ -std=c++17 -Wall -Wextra functor_basic.cpp -o functor_basic && ./functor_basic
```

`d(2, 1)` looks like a function call, but `d` is an object and the call runs `Descending::operator()`. `Descending()` creates a temporary object that `std::sort` calls for every comparison. The object has no members and takes one byte, a function pointer takes eight. Remove the `const` from `operator()`, compile again, and see whether `std::sort` still accepts it. Why is `const` still the better habit?

### Task 2: A functor with state

```c++
// file: threshold.cpp
#include <algorithm>
#include <iostream>
#include <vector>

// The state lives in a member: no global, no void* context.
class GreaterThan {
    int limit;
public:
    explicit GreaterThan(int l) : limit(l) {}
    bool operator()(int x) const { return x > limit; }
};

int main() {
    std::vector<int> v{3, 14, 15, 9, 26, 5, 35};

    for (int limit : {5, 10, 20}) {
        auto n  = std::count_if(v.begin(), v.end(), GreaterThan(limit));
        auto it = std::find_if(v.begin(), v.end(), GreaterThan(limit));
        std::cout << "limit " << limit << ": " << n << " elements, first is " << *it << std::endl;
    }
}
```

```bash
g++ -std=c++17 -Wall -Wextra threshold.cpp -o threshold && ./threshold
```

This solves the problem of the previous exercises: the limit is a member, set by the constructor, and each call of `GreaterThan(limit)` creates a predicate with its own limit. No global, no `void *`. Write a functor `InRange` with a lower and an upper bound and use it with `std::count_if`.

### Task 3: The algorithm copies your functor

```c++
// file: copies.cpp
#include <algorithm>
#include <functional>
#include <iostream>
#include <vector>

struct CountingLess {
    int calls = 0;
    bool operator()(int a, int b) {
        ++calls;
        return a < b;
    }
};

struct Sum {
    long total = 0;
    void operator()(int x) { total += x; }
};

int main() {
    std::vector<int> v{9, 8, 7, 6, 5, 4, 3, 2, 1};

    CountingLess cmp;
    std::sort(v.begin(), v.end(), cmp);                 // the algorithm gets a COPY
    std::cout << "calls after sort(cmp):           " << cmp.calls << std::endl;

    std::reverse(v.begin(), v.end());
    std::sort(v.begin(), v.end(), std::ref(cmp));       // a reference_wrapper: no copy
    std::cout << "calls after sort(std::ref(cmp)): " << cmp.calls << std::endl;

    Sum s;
    std::for_each(v.begin(), v.end(), s);               // copy again, s stays untouched
    std::cout << "s.total after for_each(s):       " << s.total << std::endl;

    Sum t = std::for_each(v.begin(), v.end(), Sum());   // for_each returns its (copied) functor
    std::cout << "t.total from the return value:   " << t.total << std::endl;
}
```

```bash
g++ -std=c++17 -Wall -Wextra copies.cpp -o copies && ./copies
```

The standard algorithms take their callback **by value**. A functor with a counter inside is copied, the copy is counted, and the original stays at zero. There are two ways out: wrap the object in `std::ref`, so that the copy is a reference to the original, or use the fact that `std::for_each` returns its functor, and read the result from the returned copy. Is the number of comparisons printed for `std::ref(cmp)` the same on every standard library? Explain why you should not rely on it.

### Task 4: A function template for any callable

```c++
// file: generic.cpp
#include <functional>
#include <iostream>
#include <vector>

// One template, any callable: the type F is deduced from the argument.
template <typename F>
void apply_all(std::vector<int> &v, F f) {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    for (int &x : v) x = f(x);
}

int  inc_fn(int x) { return x + 1; }
struct Inc {
    int operator()(int x) const { return x + 1; }
};

int main() {
    std::vector<int> v{1, 2, 3};

    apply_all(v, inc_fn);                              // F = int (*)(int)
    apply_all(v, Inc());                               // F = Inc
    apply_all(v, std::function<int(int)>(inc_fn));     // F = std::function<int(int)>

    for (int x : v) std::cout << x << " ";
    std::cout << std::endl;
}
```

```bash
g++ -std=c++17 -Wall -Wextra generic.cpp -o generic && ./generic
```

`apply_all` is a template over the type `F`, so function pointers, function objects and `std::function` all fit, each as its own instantiation. `__PRETTY_FUNCTION__` (a GCC and Clang extension) prints which one was created. Compare this with `bubble_sort` of the first exercise, which accepted only `bool (*)(int, int)`. Change the parameter to `F &f`, `const F &f` and `F &&f` one after the other and call it with a temporary `Inc()` each time: which versions compile, and what does `F` become?

### Task 5: Measure the call

```c++
// file: bench.cpp
#include <algorithm>
#include <chrono>
#include <functional>
#include <iostream>
#include <random>
#include <vector>

bool less_fn(int a, int b) { return a < b; }
struct LessObj {
    bool operator()(int a, int b) const { return a < b; }
};

template <typename F>
long long time_sort(std::vector<int> v, F cmp) {
    auto t0 = std::chrono::steady_clock::now();
    std::sort(v.begin(), v.end(), cmp);
    auto t1 = std::chrono::steady_clock::now();
    if (!std::is_sorted(v.begin(), v.end())) std::cout << "not sorted!" << std::endl;
    return std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0).count();
}

int main() {
    std::mt19937 gen(42);
    std::vector<int> data(5000000);
    for (int &x : data) x = static_cast<int>(gen());

    bool (*volatile fp)(int, int) = less_fn;                 // volatile: the compiler cannot see the target
    std::function<bool(int, int)> fn = less_fn;

    std::cout << "function pointer: " << time_sort(data, fp)         << " ms" << std::endl;
    std::cout << "std::function:    " << time_sort(data, fn)         << " ms" << std::endl;
    std::cout << "function object:  " << time_sort(data, LessObj())  << " ms" << std::endl;
}
```

```bash
g++ -std=c++17 -O2 -Wall -Wextra bench.cpp -o bench && ./bench
```

The same sort of five million numbers, three different callables. Your numbers will differ from run to run and from machine to machine: run the program several times and look at the *order*. The function object can be inlined because its type tells the compiler what is called. The `volatile` on the pointer keeps the compiler from looking through it, which is the usual situation when a function pointer comes from elsewhere. Remove `volatile` and compare again, and describe what happens.

### Task 6: Functors that the standard library already has

```c++
// file: standard_functors.cpp
#include <algorithm>
#include <functional>
#include <iostream>
#include <numeric>
#include <vector>

int main() {
    std::vector<int> v{3, 1, 4, 1, 5, 9, 2, 6};

    std::sort(v.begin(), v.end(), std::greater<int>());
    for (int x : v) std::cout << x << " ";
    std::cout << std::endl;

    std::cout << "product: " << std::accumulate(v.begin(), v.end(), 1, std::multiplies<int>()) << std::endl;
    std::cout << "sum:     " << std::accumulate(v.begin(), v.end(), 0, std::plus<int>()) << std::endl;

    std::vector<int> w(v.size());
    std::transform(v.begin(), v.end(), w.begin(), std::negate<int>());
    for (int x : w) std::cout << x << " ";
    std::cout << std::endl;
}
```

```bash
g++ -std=c++17 -Wall -Wextra standard_functors.cpp -o standard_functors && ./standard_functors
```

`<functional>` offers function objects for the usual operators: `std::less`, `std::greater`, `std::plus`, `std::multiplies`, `std::negate` and more. They replace the little helper functions like `mul` and `larger` from the earlier exercises. Use `std::less<int>()` instead of `Descending` in Task 1, and a `std::greater<>` without a type (C++14) on a `std::vector<std::string>`.

## Questions

- What do you have to write so that an object of your class can be called like a function?
- How does a functor solve the problem of the missing state that a function pointer has?
- Why does the counter in `CountingLess` stay at zero after `std::sort(v.begin(), v.end(), cmp)`, and which two ways fix this?
- Why is a function object often faster than a function pointer in `std::sort`?
- What does `std::function` offer that a template parameter `F` does not, and what does it cost?
- Which standard function objects exist for comparison and arithmetic?

## Advice

Take a function object when the callback needs data, or when it is called very often. Remember that algorithms copy it, so a functor should be cheap to copy, and results that you want to read afterwards must not sit in a copy. Writing a small class for every comparison is tedious, and the next exercise shows the shorter way to write exactly this: the lambda.
