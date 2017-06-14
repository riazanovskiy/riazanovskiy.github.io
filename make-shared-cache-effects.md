# On C++ and cache effects: shared_ptr vs make_shared

A quiz for you. Suppose it's a (tough) interview question.

Is there any performance difference between the two ways of creating a `shared_ptr` in the following code snippet?

```cpp
#include <cstdio>
#include <memory>
#include <thread>
#include <vector>

constexpr size_t ITER_COUNT = 100'000'000;

volatile size_t bh = 0;

void threadFunction(std::shared_ptr<uint32_t> ptr) {
    for (size_t i = 0; i < ITER_COUNT; i++) {
        if (ptr.use_count() > 5) {
            std::puts("five refs if branch");
        }
    }
}


int main() {
#ifndef USE_MAKE_SHARED
    std::shared_ptr<uint32_t> shared(new uint32_t(42));
#else
    auto shared = std::make_shared<uint32_t>(42);
#endif

    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back(threadFunction, shared);
    }

    for (size_t i = 0; i < ITER_COUNT; ++i) {
        *shared += 1;
        bh = *shared;
    }

    for (auto& thread : threads) {
        thread.join();
    }
}
```

#### Quick recap

Why would you use `std::make_shared`?

This is answered in great detail in Scott Meyers's book (item 21) but the short version is following:
 * Code with `make_shared` is shorter and cleaner
 * `make_shared` prevents potential resource leak due to exceptions
 * `make_shared` makes one allocation instead of two: it allocates one contiguous block of memory and
    constructs the control block in one part of it and the owned object in another
     (most likely this is implemented with [placement new](http://en.cppreference.com/w/cpp/language/new))

The last point entails that the reference counter and the data are adjacent to each other in memory.
And that is exactly what is abused in the code above.

### Measuring running time

These are results from my laptop (i5-6300U).

By the way, if you ever try to measure something similar in Java, *the only* way to do so is JMH.

| Compiler                                      | make_shared, seconds | shared_ptr, seconds |
| --------------------------------------------- |:--------------------:|:-------------------:|
| [gcc 7.1.1](https://godbolt.org/g/CQQbAI)     |         3.17         |        2.13         |
| [clang 4.0](https://godbolt.org/g/l6ctta)     |         3.23         |        1.75         |
| [clang 4.0 -O3](https://godbolt.org/g/ZQHcFn) |         0.64         |        0.17         |

What the hell? `make_shared` is supposed to be faster, not almost two times slower, isn't it?
This looks like an optimizer quirk but it isn't. The generated assembly differs only
in the shared pointer creation routine, the assembly for the threadFunction is
exactly the same and both cycles survive the optimization.

If the code is almost identical, how can the performance differ so much?
Fortunately, we have hardware performance counters, here they come:

```
$ g++ -pthread test.cpp -o test1 -DUSE_MAKE_SHARED
$ g++ -pthread test.cpp -o test2
$ perf stat -r 100 -e cycles,task-clock,instructions,L1-dcache-load-misses,LLC-loads,LLC-load-misses ./test1

 Performance counter stats for './test1' (100 runs):

    28,930,940,993      cycles:u                  #    2.899 GHz                      ( +-  0.52% )
       9980.701609      task-clock:u (msec)       #    3.079 CPUs utilized            ( +-  0.52% )
    24,602,152,308      instructions:u            #    0.85  insn per cycle           ( +-  0.00% )
        77,959,313      L1-dcache-load-misses:u                                       ( +-  1.20% )
        39,048,863      LLC-loads:u               #    3.912 M/sec                    ( +-  1.20% )
             2,171      LLC-load-misses:u         #    0.01% of all LL-cache hits     ( +-  2.95% )

       3.241782842 seconds time elapsed                                          ( +-  0.40% )

$ perf stat -r 100 -e cycles,task-clock,instructions,L1-dcache-load-misses,LLC-loads,LLC-load-misses ./test2

 Performance counter stats for './test2' (100 runs):

    18,663,381,710      cycles:u                  #    2.901 GHz                      ( +-  0.23% )
       6432.534765      task-clock:u (msec)       #    2.839 CPUs utilized            ( +-  0.25% )
    24,602,150,959      instructions:u            #    1.32  insn per cycle           ( +-  0.00% )
           369,477      L1-dcache-load-misses:u                                       ( +- 15.86% )
           173,979      LLC-loads:u               #    0.027 M/sec                    ( +- 16.54% )
             2,143      LLC-load-misses:u         #    1.23% of all LL-cache hits     ( +-  3.11% )

       2.265771621 seconds time elapsed                                          ( +-  1.06% )

````

This looks like the answer: the number of cache misses differs by two orders of magnitude.
The this is explained by how the CPU caches work.
Modern Intel processors have three levels of caches. Each core has its own L1 and L2 cache and the L3 cache is shared between all cores.
When memory is accessed, a 64 byte cache line is loaded into the L1 cache of the corresponding core and into the (common) L3 cache.

Simplifying somewhat (some details are covered [here](http://sabercomlogica.com/en/), for example), if several threads read some memory location
without interfering writes, a copy of a corresponding cache like appears in each core's L1 cache in shared state and all the reads are really fast.
If any thread attempts to write something in the same cache line (not necessarily in the very same address!), all the other cores which have this cache line will invalidate it. Any subsequent read will result in a cache miss and the current value of the cache line will be transferred between cores through L3 cache or RAM.

Because `make_shared` places reference counter right next to the data, they are likely to be in the same cache line.
Although reads and writes happen at different location, they result in false sharing and lots of cache misses.

