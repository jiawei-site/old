---
layout: post
title:  "Allocate Stuff in the Right Way"
date:   2021-07-10 00:07:12 +0800
categories: blog
layout: post
---

NOTE: This article is for those who are new to code optimizations. If you have already understood the design rationale of small vectors and SOO (small object optimzation), please ignore this.

## TL;DR

Try to avoid dynamic allocation in critical path. Always prefer stack/static memory, or at least leverage small vectors according to the distribution of the length of your data.

## Memory Storage Types

As what you were taught in college class, there're commonly 3 memory storage types (stuff like `thread_local` is not my story today):

1) **static**; 2) **dynamic** (or so-called heap memory); 3) **automatic/stack**

### A dummy example

Look at this example containing those 3 types of memory usage.

```c++
int static_var {233333}; // static storage

int main() {
    auto ptr_to_dynamic_var = new int{55555555}; // dynamic storage
    int stack_var {666666}; // automatic/stack storage
    delete ptr_to_dynamic_var;
}
```

Its corresponding assembly (unused labels, lib functions, directives filtered) by GCC 11.1 in `-O0 -std=c++2a` flag:

```
static_var:
        .long   233333 # non-const static storage variable will be stored in static data region (marked by its corresponding directive `.long`).
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16 # You can simply regard this as stack allocation, making rsp pointing to the end of current stack.
        mov     edi, 4
        call    operator new(unsigned long) # allocate dynamic memory according to `operator new` (usually libc's malloc by default)
        mov     DWORD PTR [rax], 55555555 # After dynamic allocation, we got the space and we ship the value (55555555) into the corresponding address.
        mov     QWORD PTR [rbp-8], rax
        mov     DWORD PTR [rbp-12], 666666 # Set the value of `stack_var`
        mov     rax, QWORD PTR [rbp-8]
        test    rax, rax
        je      .L2
        mov     esi, 4
        mov     rdi, rax
        call    operator delete(void*, unsigned long)
.L2:
        mov     eax, 0
        leave
        ret
```

To help you read the assembly, let's quickly review some important X86 registers:

- `%rax`: return value. We move the value `55555555` to `rax` which was previously set by the call `operator new`, and then we put the value of `rax` to `[rbp - 8]`
- `%rbp`: the start address of current stack (the base pointer for `rsp` (the stack pointer)). `[rbp - 8]` is where `ptr_to_dynamic_var` lives (in 64bit mode, pointers are of 8 bytes) and `[rbp - 12]` (12 = 8 + 4) is where `stack_var` lives. Make a lot more sense right? 

### Some intuitive analysis

The (de)allocation of static variable is super cheap, since you cannot see any instructions regarding its (de)allocation. Actually, (de)allocation to static varibles is handled by the OS (reserved by the compiler). Like the memory for the codes (instructions), it is created once the program starts and cleaned when the program ends. Hence, there's no runtime cost of (de)allocate them.

Stack memory is also cheap, theoretically (let's forget program interrupt), only 2 instruction (`push rbp` and `mov rbp, rsp`) is used to initialize the stack and 1 for "allocating" the stack frame. Since we can simply regard that we have infinite memory towards the direction of stack growth, the allocation is as cheap as moving forward a stack pointer (or statically set its offset in the assembly). The behavior of stack allocation is simply defined during compilation time since it is only relavant to your codes. It is a bit dynamic because the program during runtime can determine which path to enter. Some paths may contain a deep stack while others may not. But anyways, in general, stack allocation is quite cheap because it is also very "static".

Well, as for "the type of crime", dynamic allocation is absolutely the user's (or input's) call. It is your call to tell when to allocate (e.g., `new`) and deallcoate (e.g., `delete`). The cost of flexibility is complexity and efficiency. First, **flexibility** makes it possible to `new`/`delete` anywhere you want, however, it might not be legal or well-formed. You may kill your program out of double free, deallocating a non-exist pointer, memory leak, and many others. To avoid such issues, you finally ask for some help from *garbage collection*, *smart pointer*, and many other ownership stuffs, introducing complexity to your codes (tell me the probability of successfully compiling a rust program and how much you love `unsafe`) and some performance degradation. Second, everything is a tradeoff and "allocate anywhere" does not come for free. You or the memory allocation library (e.g., [jemalloc](http://jemalloc.net/)) you refered to need to carefully maintain one or more logically contiguous buffer(s) to make sure either it allocates fast (monotonic allocator) or long/robust (compact allocator). The actual instructions executed in runtime behind that `call operator new` can be dramatic. Usually such mechanisms makes allocation cost unstable, it can be fast when there's plenty of space left in the pool, but can be extremely slow when you have a huge amount of memory fragments (did you make a lot of small objects allocted with `malloc`?). Apart from this, for small objects, stack memory is always more cache-friendly than heap ones since you are always seeing something in the stack (see the example assembly) so that stack memory can be considered always on-cache.

In a word, dynamic memory is guilty and we should try our best to prevent it if your program is performance-critical (since our compilers at this moment cannot resolve this problem at all).

Also, I'd like to quote a sentence from *Optimized C++* whose author seems to hate dynamic memory way more than me:

> > *That’s where the money is.* 
> >
> > —Bank robber Willie Sutton (1901–1980) 
> 
>This quote was attributed to Sutton as the answer to a reporter’s 1952 question, “*Why do you rob banks?*” Sutton later denied ever having said it.
> 
>Except for the use of less-than-optimal algorithms, the naïve use of dynamically allocated variables is **the greatest performance killer in C++ programs**. Improving a program’s use of dynamically allocated variables is so often “*where the money is*” that a developer can be an effective optimizer knowing nothing other than how to reduce calls into the memory manager.

But he also said:

> Lest I start a panic, let me say that the goal in optimizing memory management is not to live an ascetic life free of distraction from the many useful C++ features that use dynamically allocated variables. Rather, the goal is to remove performance-robbing, unneeded calls into the memory manager through skillful use of these same features.

## Making `std::array` and small vectors your second option

Well, talk is cheap, let's see some real C++ codes.

### Introducing `std::array`

Standard array occupies stack memory (or static memory if static-storage).

```c++
// ill. triggers dynamic allocation.
const std::vector ill_vec {0, 1, 2, 3, 4, 5, 6, 7};

// better (remember to include <experimental/array>)
const auto good_vec = std::experimental::make_array(0, 1, 2, 3, 4, 5, 6, 7);

// way better
constexpr auto better_vec = std::experimental::make_array(0, 1, 2, 3, 4, 5, 6, 7);
```

Limitation of `std::array`:

- Size are statically fixed:
  - Limited initialization;
  - Cannot append or fallback to dynamic vector;
  - No capacity-size support;

```c++
// illegal
int N;
std::cin >> N;
std::array<int, N> arr {};

// correct
std::array<int, 1000> arr {};
```

To resolve these issues, try small vectors `boost::container::small_vector`.

### Small vectors

```c++
// Try this on: https://wandbox.org/

#include <boost/container/small_vector.hpp>
#include <boost/range/irange.hpp>

int main() {
    boost::container::small_vector<int, 8> small_vec;
    for (auto i : boost::irange(10)) {
        small_vec.push_back(i);
    }
    assert(small_vec.size() == 10);
    return sizeof(small_vec); 
    // 56 = sizeof(ptr) + sizeof(size) + sizeof(capacity) + 8 * sizeof(int) 
    //    = 8           + 8            + 8                + 8 * 4
} 
```

Looks quite like a vector right? In fact, a small vector like `small_vector<int, 8>` is consist of:

- for stack array usage: 8 int on the stack. if size <= 8;
- for full-vector fallback: fallback to a normal vector if size > 8;

If your data distribution is like: most `data.size()` <= 8, you definitely wanna try a small vector since it can prevent your most of your data being shipped to the heap.

### Benchmark

We compare the `push_back` efficiency among some `std::vector` and `boost::container::small_vector`.

```c++
#include <benchmark/benchmark.h>
#include <boost/container/small_vector.hpp>
#include <boost/range/irange.hpp>

#include <chrono>
#include <iostream>
#include <random>
#include <thread>
#include <vector>

const std::vector<int> workload = [] {
    using namespace std::chrono_literals;
    std::this_thread::sleep_for(100ms);

    std::vector<int> v;

    constexpr size_t n_workload = 100;
    v.reserve(n_workload);

    std::random_device rd {};
    std::mt19937 gen { rd() };
    std::chi_squared_distribution<float> dist(8.);
    // see https://en.cppreference.com/w/cpp/numeric/random/chi_squared_distribution

    std::cout << "Dataset size distribution:\n";
    for (auto i : boost::irange(n_workload)) {
        int s = static_cast<int>(dist(gen));
        v.push_back(s);
        std::cout << s << '\t';
        if (0 == (i + 1) % 20)
            std::cout << std::endl;
    }

    return v;
}();

template <size_t N> void PushBackSmallVecBench(benchmark::State& state)
{
    for (auto _ : state) {
        for (size_t s : workload) {
            boost::container::small_vector<int, N> v;
            for (int i : boost::irange(s))
                v.push_back(i);
        }
    }
}

template <size_t N> void PushBackStdVecBench(benchmark::State& state)
{
    for (auto _ : state) {
        for (size_t s : workload) {
            std::vector<int> v;
            v.reserve(N);
            for (int i : boost::irange(s))
                v.push_back(i);
        }
    }
}

static void PushBackSmallVector_BaseSize8(benchmark::State& state) { PushBackSmallVecBench<8>(state); }
static void PushBackSmallVector_BaseSize12(benchmark::State& state) { PushBackSmallVecBench<12>(state); }
static void PushBackSmallVector_BaseSize16(benchmark::State& state) { PushBackSmallVecBench<16>(state); }
static void PushBackSmallVector_BaseSize32(benchmark::State& state) { PushBackSmallVecBench<32>(state); }

static void PushBackVector_BaseSize0(benchmark::State& state) { PushBackStdVecBench<0>(state); }
static void PushBackVector_BaseSize8(benchmark::State& state) { PushBackStdVecBench<8>(state); }
static void PushBackVector_BaseSize12(benchmark::State& state) { PushBackStdVecBench<12>(state); }
static void PushBackVector_BaseSize16(benchmark::State& state) { PushBackStdVecBench<16>(state); }
static void PushBackVector_BaseSize32(benchmark::State& state) { PushBackStdVecBench<32>(state); }

// Register the function as a benchmark
BENCHMARK(PushBackVector_BaseSize0);
BENCHMARK(PushBackVector_BaseSize8);
BENCHMARK(PushBackSmallVector_BaseSize8);
BENCHMARK(PushBackVector_BaseSize12);
BENCHMARK(PushBackSmallVector_BaseSize12);
BENCHMARK(PushBackVector_BaseSize16);
BENCHMARK(PushBackSmallVector_BaseSize16);
BENCHMARK(PushBackVector_BaseSize32);
BENCHMARK(PushBackSmallVector_BaseSize32);

// Run the benchmark
BENCHMARK_MAIN();
```

The logic of the benchmark is to create a "length set" which follows [Chi Squared distribution (dof = 8)](https://en.cppreference.com/w/cpp/numeric/random/chi_squared_distribution). For each benchmark case, we let the (small) vector to append values in that length and see its efficiency. The difference among those cases are:

1. Type: small vector or standard vectors;
2. Pre-allocated size:
   1. for `std::vector` we use `.reserve` to reserve some heap memory;
   2. for `boost::container::small_vector` we simply leverage the pre-allocated stack buffer as the reserved memory.

The results:

```text
Dataset size distribution:
5	9	10	5	10	5	6	10	5	5	6	2	6	4	8	8	4	12	13	5	
4	13	14	5	10	11	0	10	13	3	16	5	4	3	10	6	3	18	6	13	
5	16	14	25	1	10	3	7	7	14	9	11	12	7	14	7	9	4	8	5	
8	4	9	8	7	19	7	11	6	7	2	2	9	7	4	8	4	8	7	4	
8	3	7	14	2	6	8	4	5	3	8	6	7	2	9	7	8	9	7	14	
2021-07-10T14:33:01+08:00
Running /Users/ganler-mac/CLionProjects/PlayBoost/cmake-build-release/PlayBoost
Run on (8 X 2200 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x4)
  L1 Instruction 32 KiB (x4)
  L2 Unified 256 KiB (x4)
  L3 Unified 6144 KiB (x1)
Load Average: 3.08, 3.10, 3.03
-------------------------------------------------------------------------
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
PushBackVector_BaseSize0            56440 ns        56058 ns        12727
PushBackVector_BaseSize8            18255 ns        17942 ns        40257
PushBackSmallVector_BaseSize8        9183 ns         9000 ns        96773
PushBackVector_BaseSize12           21918 ns        20460 ns        34947
PushBackSmallVector_BaseSize12       6073 ns         5211 ns       100000
PushBackVector_BaseSize16           13196 ns        12863 ns        44217
PushBackSmallVector_BaseSize16       1776 ns         1753 ns       371258
PushBackVector_BaseSize32           12384 ns        12059 ns        78193
PushBackSmallVector_BaseSize32       2435 ns         2104 ns       336143
```

Conclusions:

1. reserved memory is faster;
2. stack memory is faster;
3. those whose reserved memory size is closer to the distribution is faster;

## Conclusion

1. Use stack memory if you can (but watch out stack overflow, so do it for small objects);
2. Even if in your case stack memory is not applicable (to avoid stack overflow), try to reserve some;
3. See if your data has some kind of distribution and reserve memory according to it;

---

To extend:

[VLA (variable-length array)](https://en.wikipedia.org/wiki/Variable-length_array) is a technique to allocate user-space stack memory of arbitrary size. This is a standard for C99 but not for C++.

```c
// for C99 standard.
int n = 10;
int arr[n];
```

Some points regarding VLA:

- Fast:
  - Can be allocated on the stack (for most time).
- Slow:
  - Much more instructions compared with fixed-size array allocation since it requires some additional instructions to adjust stack size. See [slide 9 of Kees' presentation](https://events19.linuxfoundation.org/wp-content/uploads/2017/11/Making-C-Less-Dangerous-3.pdf).
  - (In general cases) may not be allocated right after your physical-space stack since the size is variable and the underlying implementation might simply allocate in somewhere else and virtually "concat" it with your old stack.
- Dangerous:
  - Since it is not restricted, it is a blazing-fast trap to trigger stack overflow.