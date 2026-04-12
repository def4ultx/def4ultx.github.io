+++
date = '2026-04-13T03:19:36+07:00'
draft = false
title = 'Big O is lying to you. CPU cacheline is all that matter'
+++

เคยสงสัยไหมครับว่าทำไม algorithm บางตัวถึงรันได้เร็วกว่าอีกอัน แม้จริงๆมันจะมี big o ที่แย่กว่า

## Introduction

ปกติแล้วใน course Data Structure and Algorithm จะพูดถึงเรื่อง Asymptotic complexity หรือ Big O notation ว่า algorithm นั้นๆว่ามีความซับซ้อนเท่าไรในแง่ของ time and space ที่ใช้

แต่่สวนนึงที่ไม่ค่อยได้พูดถึงกันก็ตือ notation อันนี้หมายถึง growth ของตัว algorithm นั้นๆด้วย

ซึ่งทำให้เวลาที่เราเทียบว่า algorithm ไหนดีกว่า เราตัดหลายๆส่วนที่สำคัญออกไป เช่นค่า constant บางอย่างหรือแม้กระทั่ง hardware model ที่ถูก abstract ว่าไม่มีผลอะไร

ผลลัพธ์ก็คือตอนที่เราเอาของไปรันจริงๆ มันทำงานได้ดันช้ากว่าซะได้

## Memory Hierarchy

ทั่วไปแล้วเราจะมี model ในหัวว่า CPU มี L1/L2/L3, RAM และ cost ในการเข้าถึง data ต่างๆที่อยู่ในแต่ละที่นั้นใกล้ๆเคียงกัน แต่ความจริงแล้วมันไม่ใช่เลย

เราจะเห็นได้ว่า cost ในการ access memory ส่วนต่างๆที่ยิ่งไกลออกไปนั้นมันจะสูงขึ้นอย่างเร็ว (อย่างเช่นใน diagram)

โดยเฉพาะตอนที่เกิด CPU cache miss มันต้องไปดึง data จากส่วนอื่นๆที่ต้องใช้

เช่น Data Structure บางชนิดที่ไม่ได้ Access data แบบ sequentially
- Linkedlist ที่ต้องทำ pointer chasing
- Hashmap ที่ต้องเอาของจากใน underlying array ที่อยู่ไกลๆ
- หรือแม้กระทั้ง Array ของ struct ที่มี field ที่เราไม่ได้ใช้เลย

จริงๆแล้วเรื่องพวกนี้อาจจะไม่สำคัญมาก แต่ก็ทำให้เราเขียนโปรแกรมได้ดีขึ้นถ้าเราเข้าใจมันมากขึ้น (i.e. mechanical sympathy)

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

แล้ว CPU Cacheline คืออะไรกันล่ะ? 

**CPU Cacheline** (ปกติแล้วขนาด 64 หรือ 128 bytes) คือขนาดที่เล็กที่สุดของการอ่าน data ของ CPU. เช่นถ้าเราต้องการอ่าน 1 byte, CPU มันไม่ได้อ่านแต่ 1 byte จาก memory แต่มันจะอ่านทั้ง 64 byte มาเลย

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

เพราะฉะนั้นการที่เราอ่านข้อมูลมาทั้ง Cacheline แล้วใช้ข้อมูลเพียงแค่เสี้ยวเดียวในนั้นมันทำให้
- เราจะต้องอ่านข้อมูลหลายรอบมากขึ้น (เพิ่ม CPU cycle)
- เช่นใช้แค่ 1 byte จาก 64 byte (ประมาณ 1%)
- ถ้าเราสามารถยัดหรือเพิ่ม data ที่เราต้องใช้ลงไปใน ฉacheline เดียวกันได้ มันก็จะทำให้เราต้องอ่านข้อมูลน้อยลง เลยเร็วขึ้นนั้นเอง

เช่นถ้าเรามี code ในแบบตัวอย่าง


แบบที่ 1
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

แบบที่ 2
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

เราจะเห็นได้่ว่าแบบที่ 1 (**Array of Struct, AoS**) นั้นรันได้ช้ากว่าแบบที่ 2 (**Struct of Array, SoA**) เพราะว่า CPU cycle ที่ใช้อ่าน data มันน้อยกว่านั้นเอง โดยที่ data layout จะเป็นประมาณใน diagram
- ถ้าเราต้อง sum แบบ AoS, CPU จะต้องอ่านถึง 3 cacheline ในการ sum ทั้งหมด
- แต่ถ้าเรา sum แบบ SoA แทน, CPU นั้นอ่านเพียงแค่ cacheline เดียวก็พอ
- ถ้าเราเอา **SIMD** มาใช้เพิ่มเข้าไปอีก มันยิ่งทำให้โค้ดเราเร็วขึ้นไปอีก (Auto vectorization ก็ทำได้)
- i.e. การที่เราใช้ linked-list (List type in scala หรือหลายๆภาษา) ที่ต้อง resolve pointer แทนก็ทำให้ data เราไม่ต้องเป็น sequential


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

ปกติแล้ว modern CPU pipeline จะมี component ที่เรียกว่า branch predictor อยู่ ซึ่งจะทำหน้าที่คอยทำนายว่าตอนที่เจอ branch (เช่น if-else) code เรานั้นจะรันไปทางไหน แล้วก็ทำการ streamline process เหล่านั้นให้ทำงานได้เร็วขึ้นโดยที่ไม่ต้องรอผลจาก if-else
- เช่นถ้าไป condition A บ่อยๆก็รัน subroutine ของ condition A ไปเลย
- หรือหากเกิดว่ารัน subroutine condition A ไปแล้วแต่จริงๆต้อง take condition B แทน ก็ทำการ retire CPU pipeline นั้นทิ้ง (โดยที่เราก็เสียเวลารอรันใหม่สักนิดหน่อย)

```
No branch prediction — CPU รอ condition

  Time -->
  +--------+--------+--------+--------+--------+--------+
  | Fetch  | Decode |Execute |  ???   |  ???   |  ???   |
  | if(x>0)|        |evaluate| STALL  | STALL  | Fetch  |
  |        |        | x > 0  | waiting| waiting| branch |
  +--------+--------+--------+--------+--------+--------+
                         |                        ^
                         +--- result ready --------+
                              (wasted cycles)

branch prediction — CPU ไม่รอแล้วรันเลย ถ้าถูกก็ใช้ result แต่ถ้าผิดก็ retire แล้วรันใหม่

  Time -->
  +--------+--------+--------+--------+--------+--------+
  | Fetch  | Decode |Execute | Fetch  | Decode |Execute |
  | if(x>0)|        |evaluate| branch | branch | branch |
  |        |        | x > 0  | (guess)| (guess)| (guess)|
  +--------+--------+--------+--------+--------+--------+
```

ยกตัวอย่างเช่นการที่เรา sum ข้อมูลใน array ที่ sort และไม่ sort ด้วย code ชุดเดียวกัน
- ใน hot loop บางอันเราสามารถทำให้เป็น branchless เลยก็ได้

```
for (x in arr)
    if (x > 0) sum += x
```

จะเห็นได่ว่า หาก branch prediction ทำงานได้ดี มันช่วยให้ CPU เรารันได้เร็วขึ้น เพราะ CPU ของเราไม่ต้องรอหรือ retire pipeline แล้ว

```
// Unsorted:    [3, -1, 5, -7, 2, -4, 8, -2, ...]
// Prediction:   T   F  T   F  T   F  T   F

// Sorted:      [-7, -4, -2, -1, 2, 3, 5, 8, ...]
// Prediction:   F   F   F   F  T  T  T  T
```

ใน C/C++ หรือบางภาษาเราสามารถ hint ให้ compiler ช่วย branch predictor ได้ด้วย `[[likely]]` / `[[unlikely]]`

## PS

Always profile and measure นะครับ ของพวกนี้เราควรจะวัดด้วย tools ที่มีเพื่อ make sure ว่ามันช่วยแก้ปัญหาจริงๆ การที่เรา blind optimize นั้นนอกจากจะไม่ช่วยแล้ว อาจจะทำให้แย่กว่าเดิมได้อีก
- บางที bottleneck มันก็ไปเกิดตรงที่เราไม่คาดคิดได้เช่นกัน

แล้วก็ยังมีเรื่องอื่นๆอีกเช่น Data oriented design, False sharing, NUMA

ไว้มาต่อกันตอนใหม่นะครับ

## References
- [CppCon 2014: Mike Acton "Data-Oriented Design and C++"](https://www.youtube.com/watch?v=rX0ItVEVjHc)
- [Cpu Caches and Why You Care](https://www.youtube.com/watch?v=WDIkqP4JbkE)
- [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
- [What Every Programmer Should Know about How CPUs Work • Matt Godbolt • GOTO 2024](https://www.youtube.com/watch?v=-HNpim5x-IE)
