---
title: You Probably Don't Know Thread Safety - A Definitive Guide for C++
---

**Thread safety** - one of those complex topics which almost no one seems to get right. Programmers quake at the thought of it, with a dangerous world of threads, atomics, and locks to navigate, lest they summon the elusive *data race*. Meanwhile, education materials are doing them no favours - far too many sources are at best misleading, and at worst, flat out wrong.
Particularly intimidating is thread safety in C++, where (as usual) it's left for the programmer to figure out how to avoid foot bullets.

Are you one of the developers who quake at thread safety? Or perhaps, despite learning attempts, you never really "got" how to do it properly? This is the article for you. Today I am here to set the record straight on thread safety.

## Do You Really Know Thread Safety?

I consider thread safety to be one of the most misunderstood topics in C++, for two reasons:

1. Thread safety is complex and nuanced, so quality education material is hard to come by.
2. There is so much nondeterminism (from the programmer's perspective) that it's easy to convince yourself you understand thread safety.

Without sounding too accusatory - because in large the problem is due to #1 -  there seems to be an alarming number of programmers who believe they understand thread safety, yet unknowingly have the gun a millimetre from their foot.

Some questions to consider, without judgement:

- Could you explain the terms "memory model", "sequential consistency", and "data race"?
- Do you know how `std::mutex`, `std::atomic`, `std::memory_order` work and when they are necessary?  
- Do you know how `volatile` relates to thread safety?  
- Could you explain the behaviour of the following code snippets?

```c++
int data{};
bool ready{};

std::thread producer{[&] {
    data = 42;
    ready = true;
}};
std::thread consumer{[&] {
    if (ready) {
        std::cout << data;
    }
}};
```

```c++
int data{};
bool ready{};

std::thread producer{[&] {
    data = 42;
    ready = true;
    std::atomic_thread_fence(std::memory_order_release);
}};
std::thread consumer{[&] {
    std::atomic_thread_fence(std::memory_order_acquire);
    if (ready) {
        std::cout << data;
    }
}};
```

```c++
int data{};
std::atomic<bool> ready{};

std::thread producer{[&] {
    data = 42;
    ready = true;
}};
std::thread consumer{[&] {
    if (ready) {
        std::cout << data;
    }
}};
```

```c++
int data{};
bool ready{};
std::mutex mutex;

std::thread producer{[&] {
    std::lock_guard lock{mutex};
    data = 42;
    ready = true;
}};
std::thread consumer{[&] {
    std::lock_guard lock{mutex};
    if (ready) {
        std::cout << data;
    }
}};
```

```c++
int data{};
bool ready{};

std::thread producer{[&] {
    data = 42;
    ready = true;
}};
producer.join();
std::thread consumer{[&] {
    if (ready) {
        std::cout << data;
    }
}};
```

If you thought "no" or "unsure" to any of those questions, then read on and rest assured - let us share in a thread safety journey.

## Quick Shout-out to Herb Sutter

A lot of what I know about thread safety in C++ comes from a presentation by Herb Sutter called "atomic<> Weapons: The C++ Memory Model and Modern Hardware" \[1\]. The presentation is a wealth of information explained in a digestible and practical manner. A great watch, although it is three hours long. I present this article as an explanation which may be quicker to consume.

## What Is Thread Safety?

Let's begin by solidifying what we mean by thread safety.

In concurrent systems, there may be multiple processors executing instructions at the same time on the same data. In C++, that might look like this:

```c++
int shared_data = 0;

void func1() {
    shared_data++;
}

void func2() {
    shared_data--;
}

int main() {
    std::thread thread1{func1};
    std::thread thread2{func2};
}
```

(Warning: the above code is not valid.)

On the surface, the code may appear straightforward enough, but if we really think about it, things gets ambiguous. What does it mean for two processors to operate on data at the same time? What is "data"? What is "the same time"? Computers are near-incomprehensibly complex machines; finding answers is not trivial.

The framework which enables us to reason about these concurrency questions is called a "memory model". A memory model is a contract between the user and implementer of a computer system. It is, for the most part, a magic abstraction, derived from the incredible nuance of computer hardware and software. As high level language programmers, we primarily interact with the memory model of our programming language.  
If you're thinking "memory models sound awfully vague" - that's deliberate, and in a sense, true. We will slowly add nuance and work our way up to a full explanation of the C++ memory model.

"Thread safety" is about writing code which obeys the rules of a memory model. If code plays nicely with the memory model, then we can reason about its behaviour and place assurances on its correctness - it is "thread safe" code. On the other hand, if code runs afoul of the memory model, then we say it is not thread safe, and it could exhibit weird behaviour. The code might not work at all. Or it might work 95% of the time. Or it might always work but only on one type of computer.

## Memory Model - The Hardware Perspective

Now we will look at memory models as they function at the bare metal. By reading this section, you promise not to stop here and run off with the newfound knowledge. A proper understanding of thread safety requires knowledge both hardware and software memory models (next section)!

The brain of most computers is a "central processing unit" - or CPU for short. The bulk of the CPU's role is to perform logic and math operations - called "instructions" - on data. On most commodity hardware, data used by a computer while switched on is stored in main memory (RAM), accessible to the CPU via an electrical connection[^1]. Inside the CPU chip, there can be multiple "cores", which can independently execute instructions and access memory. Cores also have fast internal storage called "registers", which act like working out space for instructions.

The purpose of a CPU's memory model is to define what happens with regards to multiple cores accessing the same main memory. For the discussion here, the "same main memory" means the same physical memory address, that is, the same physical circuits storing the same data. Generally, there is only one electrical connection from the CPU to main memory, so only one access to memory can occur at one point in time. You might think this means thread safety becomes trivial, yet unfortunately that is not the case. CPUs are complex constructions and have many tricks up their sleeves.

[^1]: This is a huge simplification, and you may point out that modern hardware does not strictly follow a Von Neumann architecture, but is a mixture of architectures. This is true, and adds to my point that hardware memory models are very complex.

### Cache

Over time, CPU operating speeds have significantly outpaced the performance of main memory. The reason is that the cheapest and dominant technology for main memory - DRAM - is constrained on an electrical level to have slower response characteristics compared to CPU logic technology. To alleviate performance bottlenecks arising from slow memory accesses, CPUs got "cache": a faster, but limited, intermediate storage between the CPU logic and main memory.

When loading data from main memory, the data is put into the cache first. If the CPU reads the same memory address again later, the cache provides the data quickly, rather than waiting to load from memory again. When the CPU writes data, it first goes into the cache in case it needs to be read soon. If at any point the cache can't fit any more data, previous data is evicted. When utilised effectively, cache enables the CPU to rarely wait for memory accesses and performance improves significantly.

The existence of cache immediately raises problems for a CPU with multiple cores. If the same cache is shared between all cores, then it becomes difficult to obtain high performance. However, if each core has a separate cache, then cores could end up with different (cached) data for the same memory address! This problem is known as "cache coherency".  
Clearly some thinking is required to determine how the cache system deals with concurrency, and this is one potential factor contributing to the memory model at the hardware level.

### Out of order execution

A fascinating aspect of modern CPU design is their knack for not executing instructions in the order the programmer specifies. Such design is known as "out of order execution" and came about to improve utilisation of available CPU resources, hence improving software performance. We won't dive into the details here, as this topic is somewhat involved, but the gist is that by reordering of instructions, throughtput is increased by finding an ordering which takes advantage of the free resources in the CPU at that time[^2]. Of course, the reordering must be done such that the correctness of the program is not affected.

Memory access instructions may be reordered too; which is interesting, since memory is an external entity the CPU may not have perfect knowledge on. Often memory addresses are not known in advance. And a single CPU core may not know what memory accesses other cores are performing at the same time. Arbitrarily reordering memory accesses will heavily influence the correctness of data communication between cores, i.e. thread safety. At the same time, performance goals pressure CPU design towards increased reordering.  
CPU designers have some control over what "correctness" means by way of memory models. The memory model will define what memory access reorderings are possible, depending on the chosen tradeoff with performance. Compilers and assembly writers must be aware of possible reorderings to ensure they use memory access instructions in an appropriate manner to produce the desired behaviour.

[^2]: I have a more detailed introduction to out of order execution available [here](https://github.com/MC-DeltaT/cpu-performance-demos/tree/main/out-of-order-execution).

### Where's the memory model?

At this point, you may be eager for an in-depth explanation of a real hardware memory model, such as for the ubiquitous x86 architecture. However, we are not going to discuss that here. While understanding hardware motivations is important, I argue knowing specifics is not important - and in fact, counterproductive - because as high level language programmers we should be focusing on software memory models. It is easy to fall victim to the trap of believing we understand the hardware memory model, thus ignoring relevant information. Some people claim, for example, that certain C++ constructs are required to enforce cache coherency on x86 - which is not true, because x86 cache hardware handles it for us. Further, others claim that out of order execution is a main component of thread safety on x86 - which is also not true, since most memory reorderings are disallowed on that architecture.  
What we really do need to know is the programming language's memory model, which we will explore next.

## Memory Model - The Software Perspective

TODO

### Read-modify-write operations

Even with the most basic, naive hardware memory model, we can't just write programs which behave correctly with multiple threads active. Consider the this sequence of CPU instructions which a compiler might generate:

1. Read the value `V` of memory location `M` into a register `R`.
2. Increment `R` by 1. I.e. `V = V + 1`.
3. Write the new value of `V` in `R` back into memory location `M`.

This is a classic "read-modify-write" operation which is the bread and butter of software. Read data from memory, do something do it, then write the result back to memory. Now, what happens if two threads execute this operation at the same time? Imagine `V` begins equal to 0.

| Time | Thread 1       | Thread 2       |
| ---- | -------------- | -------------- |
| 1    | Read 0         | Read 0         |
| 2    | Increment 0->1 | Increment 0->1 |
| 3    | Write 1        | Write 1        |

We thought we incremented `M` twice, but actually ended up with `M=1`! This is because nothing stops one thread reading the same original value while the other is busy doing something else. In order to prevent such a scenario, extra coordination is required. This coordination comes in different forms. The compiler might have to generate special CPU instructions, or it might require guidance from the programmer.

### Compiler optimisations

TODO: other subsections

## The C++ Memory Model

TODO

## References

\[1\] "atomic<> Weapons: The C++ Memory Model and Modern Hardware", Herb Sutter, https://herbsutter.com/2013/02/11/atomic-weapons-the-c-memory-model-and-modern-hardware/
