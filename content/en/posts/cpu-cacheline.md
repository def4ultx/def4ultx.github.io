+++
date = '2026-04-13T03:19:36+07:00'
draft = false
title = 'Big O is lying to you. CPU cacheline is all that matter'
+++

Have you ever wonder why some algorithms run faster than others, even when they have a worse Big O complexity?

## Introduction

In typical Data Structures and Algorithms course, we learn about the Asymptotic complexity or Big O notation to describe how complex or efficient an algorithm is in terms of time and space.

But on thing that's rarely discussed is that this notation describes the *growth* of the algorithm.

This means that when we compare which algorithm is "better", we're stripping away many important factors such as constant factors or even the hardware model which is abstracted away as if it doesn't even matter.

The result? When we actually run things in practice or production, the "better" algorithm can end up being slower.

## Memory Hierarchy

We generally have a mental model of CPU with L1/L2/L3 cache, ram and cost of the accessing data in each level is roughly similar. But in reality that's far from the truth.

The cost of accessing memory in different level is dramatically increase the further away you go from the CPU (as shown in the diagram below).

This is especially noticable when a CPU cache miss occurs and the CPU have fetch data from slower level.

Some data structures that don't access data sequentially are particularly affected:
- Linkedlist that required pointer chasing
- Hashmaps that fetch the data from distance location from underlying array
- Even array of structs with field you never actually use

In practice, these details might not important or always matter - but understanding these model makes you a better programmer (i.e. mechanical sympathy).

```
+---------------------------+
|         CPU Core          |
|  +---------------------+  |
|  |   Registers (1 ns)  |  |  <-- Fastest, smallest
|  +---------------------+  |
|  | L1 Cache  (~1 ns)   |  |  32-64 KB per core
|  +---------------------+  |
|  | L2 Cache  (~4 ns)   |  |  256 KB - 1 MB per core
|  +---------------------+  |
+---------------------------+
|  | L3 Cache  (~10 ns)  |  |  8-64 MB shared across cores
|  +---------------------+  |
+---------------------------+
           |
           v
+---------------------------+
| Main Memory (~100 ns)     |  8-128+ GB
+---------------------------+
           |
           v
+---------------------------+
| Disk / SSD (~10,000+ ns)  |  TB+
+---------------------------+
```

## CPU Cacheline and data locality

So what exactly is a CPU cacheline?

A **CPU Cacheline** (typical 64 or 128 bytes) is the smallest unit of data that the CPU reads from memory. For example if you want to read just 1 byte, the CPU doesn't fetch only that 1 byte - it loads the entire 64-byte cache line.

```
                        Main Memory
 Address:  0x0000  0x0040  0x0080  0x00C0  0x0100
           |-------|-------|-------|-------|-------  ...
           | 64 B  | 64 B  | 64 B  | 64 B  | 64 B
           |       |       |       |       |

  You read address 0x0052:
           |       |       |
           |  +====|=======+
           |  | This entire|  <-- Entire 64-byte cache line
           |  | cache line |      is fetched into L1
           |  | is loaded  |
           |  +====|=======+
           |       |       |
```

This mean that if we fetch an entire cacheline but only use a fraction of it
- We end up doing more memory read, More CPU cycle
- For example, using just 1 byte out of 64 bytes (roughly 1% utilization)
- If we can pack more data we actually need into same cache line, we reduce the number of reads and that make thing faster

Consider the following code examples

Approach 1
```
data class Vec3(
    val x: Double,
    val y: Double,
    val z: Double
)

fun sumX(xs: Array<Vec3>): Double {
    return xs.sumOf { it.x }
}
```

Approach 2
```
data class Vec3List(
    val x: Array<Double>,
    val y: Array<Double>,
    val z: Array<Double>,
)

fun sumX(xs: Vec3List): Double {
    return xs.x.sum()
}
```

You'll notice that Approach 1 (**Array of Struct, AoS**) runs slower than Approach 2 (**Struct of Array, SoA**) because it requires fewer CPU cycles to read the data. The data layout looks roughly like the diagram below:

- With AoS, the CPU needs to read up to 3 cache lines to sum all x values
- With SoA, the CPU only needs a single cache line
- Adding **SIMD** on top of this makes things even faster (auto-vectorization can do this for you)
- Conversely, using a linked list (e.g. `List` type in Scala and many other languages) that requires pointer chasing means your data won't be laid out sequentially

```
  One Cache Line (64 bytes = 512 bits)
  +--+--+--+--+--+--+--+--+--+--+--+--+-- ... --+--+--+--+
  |A0|A1|A2|A3|A4|A5|A6|A7|A8|A9|AA|AB|   ...   |AE|AF|
  |B0|B1|B2|B3|B4|B5|B6|B7|B8|B9|BA|BB|   ...   |BE|BF|
  |C0|C1|C2|C3|C4|C5|C6|C7|C8|C9|CA|CB|   ...   |CE|CF|
  +--+--+--+--+--+--+--+--+--+--+--+--+-- ... --+--+--+--+
  |<------------------ 64 bytes ------------------>|


  Array of Struct (AoS) — Vec3 = 24 bytes (3 x 8-byte Double)
  sumX() reads only x, but y and z get loaded too

  One cacheline = 64 bytes = 2 full Vec3s (48B) + 2/3 of Vec3[2]
  |<------------------------- 64 bytes -------------------------->|
  +--------+--------+--------+--------+--------+--------+---------+
  |   x0   |   y0   |   z0   |   x1   |   y1   |   z1   | x2 ... |
  |  8B    |  8B    |  8B    |  8B    |  8B    |  8B    |  16B   |
  +--------+--------+--------+--------+--------+--------+---------+
    ^used^   wasted   wasted   ^used^   wasted   wasted   partial
  [------ Vec3[0] ------] [------ Vec3[1] ------] [- Vec3[2] -]

  sumX utilization: 2 x-values per cacheline → 16 / 64 = 25%


  Struct of Array (SoA) — xs, ys, zs are separate contiguous arrays
  sumX() walks only the xs array

  xs array — one cacheline:
  |<------------------------- 64 bytes -------------------------->|
  +--------+--------+--------+--------+--------+--------+--------+--------+
  |   x0   |   x1   |   x2   |   x3   |   x4   |   x5   |   x6   |   x7   |
  |  8B    |  8B    |  8B    |  8B    |  8B    |  8B    |  8B    |  8B    |
  +--------+--------+--------+--------+--------+--------+--------+--------+
    ^used^   ^used^   ^used^   ^used^   ^used^   ^used^   ^used^   ^used^

  sumX utilization: 8 x-values per cacheline → 64 / 64 = 100%
```

## Branch prediction

Modern CPU pipelines have a component called a **branch predictor**. Its job is to predict which direction a branch (e.g. if-else) will take, and speculatively execute instructions down that path without waiting for the actual result.

- If condition A is taken frequently, the CPU will speculatively execute condition A's subroutine ahead of time
- If the prediction turns out to be wrong (e.g. condition B should have been taken instead), the CPU retires that pipeline and re-executes with the correct path (costing a small delay)

```
No branch prediction — CPU waits for the condition

  Time -->
  +--------+--------+--------+--------+--------+--------+
  | Fetch  | Decode |Execute |  ???   |  ???   |  ???   |
  | if(x>0)|        |evaluate| STALL  | STALL  | Fetch  |
  |        |        | x > 0  | waiting| waiting| branch |
  +--------+--------+--------+--------+--------+--------+
                         |                        ^
                         +--- result ready --------+
                              (wasted cycles)

With branch prediction — CPU doesn't wait, executes speculatively. If correct, use the result. If wrong, retire and re-execute.

  Time -->
  +--------+--------+--------+--------+--------+--------+
  | Fetch  | Decode |Execute | Fetch  | Decode |Execute |
  | if(x>0)|        |evaluate| branch | branch | branch |
  |        |        | x > 0  | (guess)| (guess)| (guess)|
  +--------+--------+--------+--------+--------+--------+
```

For example, summing values in a sorted vs. unsorted array using the same code:

- In some hot loops, you can even make the code branchless entirely

```
for (x in arr)
    if (x > 0) sum += x
```

As you can see, when branch prediction works well, it helps the CPU run faster because it doesn't have to stall or retire pipelines.

```
// Unsorted:    [3, -1, 5, -7, 2, -4, 8, -2, ...]
// Prediction:   T   F  T   F  T   F  T   F

// Sorted:      [-7, -4, -2, -1, 2, 3, 5, 8, ...]
// Prediction:   F   F   F   F  T  T  T  T
```

In C/C++ and some other languages, you can hint the compiler to help the branch predictor using `[[likely]]` / `[[unlikely]]`.

## PS

Always profile and measure. You should use proper tools to make sure your optimizations actually help. Blind optimization not only doesn't help — it can make things worse.

- Sometimes the bottleneck is in a place you'd never expect

There are also other related topics such as data-oriented design, false sharing, and NUMA.

Stay tuned for the next post!

## References
- [CppCon 2014: Mike Acton "Data-Oriented Design and C++"](https://www.youtube.com/watch?v=rX0ItVEVjHc)
- [Cpu Caches and Why You Care](https://www.youtube.com/watch?v=WDIkqP4JbkE)
- [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
- [What Every Programmer Should Know about How CPUs Work • Matt Godbolt • GOTO 2024](https://www.youtube.com/watch?v=-HNpim5x-IE)
