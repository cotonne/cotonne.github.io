---
layout: post
title:  "High performance Haskell"
date:   2016-12-10 14:42:27 +0100
categories: haskell performances
---
Someone made me discover the [Google APAC][google-apac]. It is a challenge for those who want to join Google.
Well, even if I'm not interested, those problems are really clever :).


The [Robot Rock Band][robot-rock-band] was quite challenging. You have to solve the following problem that 
can be rephrased like this:
```
Given 4 sets (A, B, C, D) of numbers and a number k, 
you have to calculate size({(a, b, c, d) ∈ A x B x C x D/ a ^ b ^ c ^ d = k })
```

For the first test set, you have only 50⁴ possibilities (around 6 millions) and for the second test set,
1000⁴ possibilities. Even if you were able to test 1 million of combinaisons, you would need around 11.5
days to solve it with brute force.

So, i decided to do it ... with Haskell :)

## First naive approach 

With Haskell, it was really easy : generate the couple with sequence.

{% highlight haskell %}
calculateK x = foldl xor (head x) (tail x)
robot :: Int -> [[Int]] -> Int
robot k testset = length $ filter (==k) $ map calculateK $ sequence testset
{% endhighlight %} 

This approach was really slow. With the first test set, it was ok but, with the second test, memory started 
to explode.

## GC?

One thing I discover is that your Haskell binary is shipped with a [garbage collector][haskell-garbage-collector]
 (like in Java and a lot of non-C languages).

Let's have a look to the GC:
```
$ time cat ~/Downloads/B-small-practice.in | ./robot-exe.exe +RTS -T -s
Case #1: 2882388
Case #2: 83297
Case #3: 0
Case #4: 4080
Case #5: 5308416
Case #6: 2439795
Case #7: 76081
Case #8: 0
Case #9: 4711
Case #10: 5308416
   4,337,223,240 bytes allocated in the heap
   2,653,696,960 bytes copied during GC
       6,155,040 bytes maximum residency (259 sample(s))
         277,056 bytes maximum slop
              23 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0      8046 colls,  8046 par   22.000s   5.530s     0.0007s    0.0023s
  Gen  1       259 colls,   258 par   26.203s   5.888s     0.0227s    0.0289s

  Parallel GC work balance: 0.00% (serial 0%, perfect 100%)

  TASKS: 10 (1 bound, 9 peak workers (9 total), using -N8)

  SPARKS: 0 (0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.000s  (  0.001s elapsed)
  MUT     time    8.578s  (  4.912s elapsed)
  GC      time   48.203s  ( 11.419s elapsed)
  EXIT    time    0.000s  (  0.001s elapsed)
  Total   time   56.812s  ( 16.333s elapsed)

  Alloc rate    505,614,366 bytes per MUT second

  Productivity  15.2% of total user, 52.7% of total elapsed

gc_alloc_block_sync: 68370
whitehole_spin: 0
gen[0].sync: 0
gen[1].sync: 294

real    0m16.493s
user    0m0.000s
sys     0m0.062s
```

We can notice that a lot of memory is copied (2.6GB). More than 8000 parallel collections are done.
First, we have to improve memory consumption.

## List comprehension

The memory usage is quite high. It is due to the sequence function. This function generates
all possibilities and then, continues the computation.


List-comprehesion is designed for infinite streams. So, only one element is treated at a time,
giving use a huge improvement of memory. Moreover, the GC is less used, improving the time given
to the computation instead of managing the memory:
{% highlight haskell %}
robotListC k r = length [1|r0 <- r!!0, r1 <- r!!1, r2 <- r!!2, r3 <- r!!3, calculateK [r0, r1, r2, r3, k] == 0]
{% endhighlight %} 


Here is the GC usage :
```
time cat ~/Downloads/B-small-practice.in | ./robot-exe.exe +RTS -T -s
[... response skip ...]
   3,823,644,328 bytes allocated in the heap
         692,584 bytes copied during GC
          86,616 bytes maximum residency (2 sample(s))
          87,440 bytes maximum slop
               5 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0      7407 colls,  7407 par    5.094s   0.952s     0.0001s    0.0017s
  Gen  1         2 colls,     1 par    0.000s   0.001s     0.0006s    0.0009s

  Parallel GC work balance: 0.07% (serial 0%, perfect 100%)

  TASKS: 10 (1 bound, 9 peak workers (9 total), using -N8)

  SPARKS: 0 (0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.000s  (  0.001s elapsed)
  MUT     time    9.328s  (  4.561s elapsed)
  GC      time    5.094s  (  0.953s elapsed)
  EXIT    time    0.000s  (  0.000s elapsed)
  Total   time   14.422s  (  5.516s elapsed)

  Alloc rate    409,904,919 bytes per MUT second

  Productivity  64.7% of total user, 169.1% of total elapsed

gc_alloc_block_sync: 46999
whitehole_spin: 0
gen[0].sync: 0
gen[1].sync: 0

real    0m5.905s
user    0m0.015s
sys     0m0.061s
```

We have already increase the speed by 3 times. Memory copied goes from 2.6GB to 700KB.
Not so bad :).


## GCC?

GHC relies on GCC to compile your application. You can used options like -02. 
You can refer to the [manual][haskell-gcc-options-manual] for more informations.
I added some more options to **cabal configuration**:
```
  ghc-options:         -threaded -rtsopts -with-rtsopts=-N -O2 -funfolding-use-threshold=16 -optc-O3
```

Let's run it again: 
```
$ time cat ~/Downloads/B-small-practice.in | ./robot-exe.exe +RTS -T -s
[... skipped ...]
  Productivity  78.7% of total user, 142.0% of total elapsed

real    0m4.426s
```

Around 10% earn in execution duration.

## Mutli-threading

Nowdays, we can (and have to) built powerful applications based on parallel computation. You can
do the same in Haskell with [par/seq][par-seq]:
{% highlight haskell %}
parRobot :: Int -> [[Int]] -> Int -> Int
parRobot _ ([]:_) _ = 0
parRobot k r chunkSize = a `par` b `pseq` (b + a)
  where 
   firstRobot = head r
   (left, right) = splitAt chunkSize firstRobot
   one   = left  : tail r
   other = right : tail r
   a = robot k one
   b = robot k other
{% endhighlight %} 

Here, again, i decided to have a naive approach. I split the first set (namely A) and i compute parts 
(ie subset A x B x C x D). At the end, i aggregate the result.

You can foree the number of threads to use with -N<n>. If you don't specify it, Haskell will choose this
number based on the number of your CPU. Let's start with 2:

```
time cat ~/Downloads/B-small-practice.in | ./robot-exe.exe +RTS -T -s -N2

  Productivity  98.4% of total user, 88.8% of total elapsed

real    0m3.436s
```

The execution duration goes from 4.4s. to 3.4s., Another good improvement!

Second time without -N:
```
time cat ~/Downloads/B-small-practice.in | ./robot-exe.exe +RTS -T -s

real    0m4.484s

```

This time, no more gain. Maybe more improvement could be done ;)

## Bazooka technic

My computer is built in with a NVIDIA GPU. A GPU is made of hundred to thousand of specialized CPU. With Haskell,
you can use [accelerate][accelerate] and [accelerate-cuda][accelerate-cuda], the implementation for NVIDIA GPU. 
This library defines a DSL for GPU. You have to use specialized functions and types.


If you want to use this library, I strongly advice you to do it under Linux. Under windows, you will have :
 - manually modify the source code of accelerate
 - install a bunch of things like Visual Studio in order to get dependencies 


Behind the scene, with the CUDA interpreter, files are generated and compiled with the NVCC, the compiler which 
produce CUDA, the NVIDIA GPU binary.


Using CUDA, it can be possible to multiple ths speed by 1000 thousand times (maybe, maybe) due to the power of
extrem parallel processing of the GPU.


This work is still in progress. You can have a look to the [code here (dirty code ahead :)][robot-github].


## The good solution

We have seen a list of technics to improve the execution speed of our application:
 - improve memory usage. As in Java, it can be the primary bottleneck in your application. 
 - improve compilation
 - Multi-threading
 - GPU processing

Whenever, brute force was not the good solution here. In fact, you should notice this property:
```
a ^ b = c ^ d ^ k
```
So, after computing two sets A x B and C x D x k, you will have to look in the first set and count how many
elements are in the second set. Computation complexity will go from O(n⁴) to O(n² log n) if you use an hashtable
for example.


So, the best way to solve a problem is often to think before coding ;).

[haskell-garbage-collector]: https://wiki.haskell.org/GHC/Memory_Management
[google-apac]: https://code.google.com/codejam/apactest/
[robot-rock-band]: https://code.google.com/codejam/contest/8264486/dashboard#s=p1
[haskell-gcc-options-manual]: https://wiki.haskell.org/Performance/GHC
[par-seq]: https://wiki.haskell.org/Par_and_seq
[accelerate]: https://github.com/AccelerateHS/accelerate
[accelerate-cuda]: https://github.com/AccelerateHS/accelerate-cuda
[robot-github]: https://github.com/cotonne/misc/tree/master/robot
