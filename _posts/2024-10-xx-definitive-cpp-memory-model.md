---
title: You Probably Don't Know Thread Safety - Definitive Guide to the C++ Memory Model
---

**Thread safety** is a difficult topic, to say the least. What's harder than writing code? Writing multiple parallel streams of code that interact in weird and wonderful ways, of course. It seems like almost no one gets thread safety quite right. The situation in C++ is particularly intimidating, where (as usual) it's left to the programmer to do the hard work to avoid footguns. I would consider it one of the most C++ misunderstood topics, due to its nuance and nondeterminism. Based on what I've seen, an alarming number of programmers believe they understand thread safety, yet unknowingly have the gun a millimetre from their foot.  
Are you one of the developers who quake at thread safety? Or perhaps, despite learning attempts, it never really "clicked"? This is the article for you. Today I am here to set the record straight on thread safety.

## Quick Shout-out to Herb Sutter

A lot of what I know about thread safety in C++ comes from a presentation by Herb Sutter called "atomic<> Weapons: The C++ Memory Model and Modern Hardware" \[1\]. The presentation is a wealth of information explained in a digestible and practical manner. A great watch, although it is three hours long. I present this article as an explanation which may be quicker to consume.

## What Does Thread Safety Mean?

First, we should briefly clarify what we mean by "thread safety", for anyone who needs a refresher. "Thread safety" is about creating correct software involving multiple "threads". Threads are separate sequences of code which might be executed at the same time on multiple processors. The topic of thread safety is an important one, since most of our computer systems today involve more than one processor. We've discovered that having multiple processors do things simultaneously ("concurrency") is useful, since it means we can do more work in the same amount of real time.

Why is thread safety significant and difficult? Any multithreaded software which does something interesting needs to move data in and out of threads, and its correctness depends on the interactions between the threads and that shared data. A very simple C++ example may look like this:

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

One thread increments `shared_data`, while the other decrements it. If you run this code on a modern computer, there's a good chance the threads run at the same time, on different processors. And this is where the problems begin. Innocuous as it may seem on the surface, things get ambiguous if you think more about it. What does it mean for two processors to operate on data at the same time? What is a processor? What is "data"? What is "the same time"? What is actually going on inside the computer here? We can't make this code correct if we don't know what's happening, and that's why we need to know about thread safety.

## The Heart of the Problem - Memory Models

While there is a lot of thread unsafe code out there, I can't really blame anyone. Thread safety is genuinely hard. Creating thread safe software is as much an architectural and design problem as anything; you can't learn it overnight. I wish I could write a zero-to-hero guide on thread safety, but it's not easy (I tried, failed, and ended up with this article instead). Perhaps such a guide will come in the future. But, today we are going to discuss one critical aspect of thread safety, one that's right at the heart of the thread safety problem: memory models. Specifically, the C++ memory model.

"What is a memory model?" you may rightly ask! A memory model is the book of rules of the thread safety game. It's the groundwork, the foundation, that lets you build robust, portable multithreaded programs. Without having a watertight understanding of the appropriate memory model, you will inevitably fail to produce thread safe code.

Formally, a memory model is a contract between the user and implementer of a computer system regarding concurrent data access. If code plays nicely with the memory model, then we can reason about its behaviour and place assurances on its correctness. On the other hand, if code runs afoul of the memory model, it could exhibit unexpected behaviour, and this is not great. I, and hopefully also you, don't like software that fails 10% of the time or randomly breaks on other systems.

In this article, we are going from zero to hero on memory models, with a focus on C++, all the way from the hardware up to the Standard Library.

## Memory Models - The Hardware Perspective

We start at the bottom of the world in which our software runs: the raw computer hardware.

If you're not aware of the basics of compute hardware, let me explain. The brain of most computers is the "central processing unit", or CPU for short. The bulk of the CPU's role is to perform logic and math operations - called "instructions" - on data. Data used by a computer while switched on is stored in main memory (RAM), which typically sits separate to the CPU. Inside the CPU chip, there is fast internal storage called "registers", which act like working out space for instructions. You can imagine a CPU as a machine which inputs memory, does some internal magic with registers, then outputs to memory.

Now these days, "a CPU" is more like "multiple CPUs inside the same packaging". We figured out that we can cram more processors into a CPU chip, with each processor independently executing their own instructions, to get more work done. These individual processors are known as "cores". ("CPU" and "core" are often used interchangeably; usually it's obvious from context if we mean one core or multiple cores together.)  
Here's where the problems begin: how do we share data (memory) between cores? Inevitably, cores will pass data between each other. On a multi-core system, even before your code runs, the operating system has conjured a plethora of shared data used for coordinating program execution, tracking resource usage, and so on. What happens if one cores writes to memory, and around the same time, another core reads the same memory? Does the CPU explode? Probably not; it should have some useful behaviour. And what governs that behaviour is the CPU's memory model. The rules of the memory model depends on the CPU's design goals and physical constraints. Let's have a look at some factors which play a role in that design.

TODO: link to further reading about CPUs

### Cache

TODO: reword

Over time, CPU operating speeds have significantly outpaced the performance of main memory in commodity systems due to difference in underlying technologies. To alleviate performance bottlenecks arising from slow memory accesses, CPUs got "cache": a faster, but limited, intermediate storage between the CPU logic and main memory.

Cache works like a temporary buffer. When loading data from main memory, the data is put into the cache first. If the CPU reads the same memory address again later, the cache provides the data quickly, rather than waiting to load from memory again. When the CPU writes data, it first goes into the cache in case it needs to be read soon. If at any point the cache can't fit any more data, previous data is evicted. When utilised effectively, cache allows the CPU to operate at its peak performance, avoiding the need to wait for main memory accesses. Without cache, the performance of code using main memory (i.e., almost everything) is crippled.

But the existence of cache immediately raises problems for a CPU with multiple cores. The best performing cache is separate per core, due to circuit design complexity. However, if each core has a different cache, then cores could end up with different (cached) data for the same memory address! Data communication between cores could be ruined. This problem is known as "cache coherency", and it requires some thinking and deliberate design to solve. How it is solved factors into the memory model guaranteed to the CPU programmer.

### Out of Order Execution

TODO: reword

A fascinating aspect of modern CPU design is their knack for not executing instructions in the order the programmer specifies. Such design is known as "out of order execution" and came about to improve utilisation of available CPU resources, hence improving software performance. We won't dive into the complex details here, but the gist is that by reordering of instructions, throughtput is increased by finding an ordering which takes advantage of the free resources in the CPU at that time. Of course, the reordering must be done such that the correctness of the program is not affected.[^2]

Memory access instructions may be reordered too, if desirable for performance. This is interesting, since memory is an entity external to the CPU, and one core might not know what else is accessing memory. Obviously, arbitrarily reordering memory accesses will heavily influence the correctness of data communication between cores, i.e. thread safety. Arbitrary memory instruction ordering would probably be unusably chaotic. At the same time, performance goals pressure CPU design towards increased reordering.  
CPU designers have some control over what "correctness" means by way of memory models. The memory model will define what memory access reorderings are possible, depending on the chosen tradeoff with performance. Compilers and CPU programmers must be aware of possible reorderings to ensure they use memory access instructions in an appropriate manner to produce the desired behaviour.

TODO: fix note numbering

[^2]: I have a more detailed introduction to out of order execution available [here](https://github.com/MC-DeltaT/cpu-performance-demos/tree/main/out-of-order-execution).

### Memory Access Alignment

TODO: reword

In some CPU designs, the hardware can only directly access memory at memory addresses which are a multiple of the CPU's natural data size. For example, the CPU might like to deal in terms of 8 bytes, and can only fetch memory addresses that are a multiple of 8. We say these addresses are "aligned".[^3] Such a design is favourable for reducing complexity and improving efficiency of the circuit design.

Some CPU designs disallow unaligned accesses, but often in modern hardware, lack of alignment is permitted and handled by the CPU for convenience. Unfortunately, convenience usually comes at the cost of something. If memory can only be accessed in an aligned manner, then fulfilling an unaligned request requires piecing together data from multiple separate aligned accesses. The question arises of what happens if the data is changed by another core in between the two accesses? It's plausible we could get half old, half new data - gross. The memory model of the CPU will define what the behaviour is surrounding this scenario. Compilers and assembly writers might need to ensure all data is aligned correctly, or use special instructions to avoid the problem.

[^3]: I'm purposely using the word "memory" loosely here, referring to both main memory access and cache access. Similar issues arise for both:

- Main memory technology and/or its connection to the CPU may only electrically support aligned accesses by design.
- Cache is generally split into small chunks according to memory address, so accessing data which straddles two chunks may incur overhead or be disallowed.

### Where's the Memory Model?

TODO: reword

At this point, you may be eager for an in-depth explanation of a real hardware memory model, such as for the ubiquitous x86 architecture. However, we are not going to discuss that here. While understanding hardware motivations is important, I argue knowing specifics is counterproductive, because as high level language programmers we should be focusing on software memory models. It is easy to believe we understand the hardware memory model and that the buck stops there. In fact, on the x86 architecture, the topics we covered don't play as large a role in thread safety as you might think, because the hardware memory model is quite generous![^4] Yet some people will try to claim, for example, that cache coherency is the reason why you must use `std::atomic`.  
What we really do need to know, and what should inform our decisions the most, is the programming language's memory model, which we will explore next.

[^4]: The x86 memory model is one of the most CPU-programmer-friendly out there, as much is solved invisibly by the hardware. Cache coherency is basically not a concern from the programmer's perspective, and most memory instruction reorderings are disallowed. Only unaligned memory access incurs some thread safety hazards. However, we pay a continual cost in performance, efficiency, and chip complexity in exchange for such a friendly model. Newer chips, such as ARM, have shifted the tradeoff closer to efficiency.
