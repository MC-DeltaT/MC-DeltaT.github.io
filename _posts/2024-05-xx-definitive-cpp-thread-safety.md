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

A fascinating aspect of modern CPU design is their knack for not executing instructions in the order the programmer specifies. Such design is known as "out of order execution" and came about to improve utilisation of available CPU resources, hence improving software performance. We won't dive into the complex details here, but the gist is that by reordering of instructions, throughtput is increased by finding an ordering which takes advantage of the free resources in the CPU at that time. Of course, the reordering must be done such that the correctness of the program is not affected.[^2]

Memory access instructions may be reordered too, if desirable. This is interesting, since memory is an external entity the CPU may not have perfect knowledge on: often memory addresses are not known in advance; and a single CPU core may not know what memory accesses other cores are performing at the same time. Obviously, arbitrarily reordering memory accesses will heavily influence the correctness of data communication between cores, i.e. thread safety. At the same time, performance goals pressure CPU design towards increased reordering.  
CPU designers have some control over what "correctness" means by way of memory models. The memory model will define what memory access reorderings are possible, depending on the chosen tradeoff with performance. Compilers and assembly writers must be aware of possible reorderings to ensure they use memory access instructions in an appropriate manner to produce the desired behaviour.

[^2]: I have a more detailed introduction to out of order execution available [here](https://github.com/MC-DeltaT/cpu-performance-demos/tree/main/out-of-order-execution).

### Memory access alignment

TODO

### Where's the memory model?

At this point, you may be eager for an in-depth explanation of a real hardware memory model, such as for the ubiquitous x86 architecture. However, we are not going to discuss that here. While understanding hardware motivations is important, I argue knowing specifics is not important - and in fact, counterproductive - because as high level language programmers we should be focusing on software memory models. It is easy to fall victim to the trap of believing we understand the hardware memory model, thus ignoring relevant information. Some people claim, for example, that certain C++ constructs are required to enforce cache coherency on x86 - which is not true, because x86 cache hardware handles it for us. Further, others claim that out of order execution is a main component of thread safety on x86 - which is also not true, since most memory reorderings are disallowed on that architecture.  
What we really do need to know is the programming language's memory model, which we will explore next.

## Memory Model - The Software Perspective

Programming languages vary greatly in how they approach memory and concurrency, depending on their design goals. Some languages pretend concurrency doesn't exist, while others embrace it in their core features. Likewise, some languages may force you deep into the weeds of memory management, and others do not even expose the concept of memory[^3]. Further, under the surface, compilers perform all sorts of magic to make code run on many hardware platforms and with high performance.

Like a programming language's syntax defines what sequence of characters are legal, a language's memory model is a contract between the programmer and the compiler (and thus, hardware) on legal behaviour of concurrent memory accesses. In this section, we will see the factors which go into a programming language's memory model.

[^3]: Some programming languages, particularly functional languages, do not have the concept of shared mutable memory. Thread safety is effectively "solved" in such languages.

### Read-modify-write operations

Even with the most basic hardware memory model underneath us, we can't guilelessly write correct programs with multiple threads active. Consider the case of incrementing an integer in memory, which a compiler might translate to this sequence of CPU instructions:

1. Read the value `V` of memory location `M` into a register `R`.
2. Increment `R` by 1. I.e. `V = V + 1`.
3. Write the new value of `V` in `R` back into memory location `M`.

This is a classic "read-modify-write" operation and is the bread and butter of software. Read data from memory, do something do it, then write the result back to memory. Now, what happens if two threads execute this operation at the same time? Imagine `V` begins equal to 0.

| Time | Thread 1       | Thread 2       |
| ---- | -------------- | -------------- |
| 1    | Read 0         | Read 0         |
| 2    | Increment 0->1 | Increment 0->1 |
| 3    | Write 1        | Write 1        |

We thought we incremented `M` twice, but actually ended up with `M=1`! This is because nothing stops one thread reading the same original value while the other is busy doing something else. This problem is known generally as a "data race", because the result depends on the threads "racing" each other to execute. In order to prevent such a scenario, extra coordination - known as "synchronisation" is required. This coordination comes in different forms, such as special CPU instructions, or language-level guidance from the programmer.

In fact, data races are a fundamental problem in concurrency that exist beyond specifics of programming languages, compilers, and CPU hardware. If you think about it, *any* system where multiple entities are performing read-modify-write operations are subject to this problem!

### Compiler optimisations

Languages like C++ are liked in part for their runtime speed, which is helped by the awesome optimisations done by compilers. Modern C/C++ compilers can be quite aggressive in rearranging the code we write into fast machine code. To paraphrase Herb Sutter in his memory model talk\[1\]: the compiler accepts our source code and gives us back the "code we meant to write" to yield optimal performance.

The topic of code optimisation is huge and you could spend your whole career on it, but all we need to know here is the compiler will happily add, remove, and reorder memory accesses to improve performance. (Sound familiar? Yes, similar vein as out of order execution.) To illustrate, consider this simple function which performs math on some arrays:

```c++
float func(float const* a, float const* b, unsigned size) {
    float result = 0;
    for (unsigned i = 0; i < size; ++i) {
        for (unsigned j = 0; j < size; ++j) {
            result += a[i] * b[j];
        }
    }
    return result;
}
```

You may notice that `a[i]` accesses memory every iteration in the inner loop, yet `i` is not changing. The compiler could move the read of `a[i]` outside the inner loop and save the value in a register to be more efficient; something like this:

```c++
float func(float const* a, float const* b, unsigned size) {
    float result = 0;
    for (unsigned i = 0; i < size; ++i) {
        float const a_i = a[i];
        for (unsigned j = 0; j < size; ++j) {
            result += a_i * b[j];
        }
    }
    return result;
}
```

Not only does this change reduce the number of instructions executed within the inner loop, it likely opens the door to other optimisations too.

But wait! What if another thread updates `a[i]` while the inner loop is running? Then the result could be different with and without the optimisation. Perhaps you want the optimisation, or perhaps not - how should the compiler know? There is a tradeoff between freedom to optimise and simplicity for the programmer. Software memory models specify where the tradeoff is set, defining which optimisations the compiler may do and what assumptions the programmer may make surrounding memory access.

## The C++ Memory Model

Hopefully by now, you have a good idea of what is at play behind the scenes of thread safety, and what motivates the existence of memory models. That means you are ready to tackle the tricky topic of the C++ memory model. Buckle up and get ready for some serious C++ knowledge.

### A small history lesson

Fun fact: C++ first standardised its memory model in C++11, over 25 years after the inception of the language. I might guess that it took so long, as many things do in C++, due to the huge variety of use cases and platforms on which it is run (although I have no evidence for this). Before C++11, the official language standard provided no guarantees on the behaviour of concurrent code. Of course people did write concurrent code before 2011, but correctness came from specific compilers and hardware platforms. That meant that code you wrote that worked on one machine may not work on another, and C++ didn't care. The pre-11 C++ standard dealt in terms of single threaded applications only.

Naturally, it would be nice for the language to specify one way to go about concurrency which is efficient and portable across many systems, especially as multithreaded applications became more prevalent in consumer software. So, C++11 finally introduced one officialy memory model.[2]

### C++11 memory model - overview

As mentioned a few times so far, a memory model is a contract between a user and implementer. In the case of C++, the user is the programmer, and the implementer is the compiler plus CPU hardware. The memory model sets forth three broad strokes of specification:

1. Identifies what patterns of concurrent memory access are problematic and illegal - data races;
2. Specifies new language constructs necessary to avoid problem scenarios - synchronisation;
3. Defines how a program will behave when in accordance with points 1 and 2 - e.g. hardware and optimisations.

Points 1 and 2 are the contract required to be upheld by the programmer, and point 3 is the contract upheld in return by the compiler. The contract is relatively simple: don't write code containing a data race, and your program will behave like you wrote it. Otherwise, your code may not work as expected. The power of the memory model is that all we need to do is obey it, and the compiler takes care of everything for us, no matter what environment our code executes within.

### Data races and synchronisation

Data races are the driving concept of the C++ memory model. A program containing a data race produces undefined behaviour, meaning we can't rely on it functioning correctly 100% of the time. What is a data race? The C++ Standard has a specific definition[3]:

> When a thread accesses a memory location and a different thread modifies the same memory location, a data race occurs unless:
>
> - both memory accesses are atomic, or
> - one of the accesses *happens-before* the other.

Without explicitly using any C++11 synchronisation constructs, neither of these conditions are met, so memory accesses between threads are data races, such as in this code:

```c++
// DATA RACE

int shared_data = 0;

void read() {
    std::cout << shared_data << std::endl;
}

void write() {
    shared_data = 42;
}

int main() {
    std::thread thread1{read};
    std::thread thread2{write};
}
```

The definition of a data race gives us two directions on how to prevent them.  
The first is atomics, AKA `std::atomic`: special types with magically thread-safe read and write semantics.  
The second direction refers to memory ordering, which governs how memory accesses may be reordered, and is particularly significant to nonatomic memory accesses. Memory ordering can be devilishly complex and nuanced, so we will ease into it by starting with atomics.

### Atomics

TODO

TODO: acquire-release semantics?

TODO: other memory orderings?

## Practical examples

TODO: revisit code from the start

## References

\[1\] "atomic<> Weapons: The C++ Memory Model and Modern Hardware", Herb Sutter, https://herbsutter.com/2013/02/11/atomic-weapons-the-c-memory-model-and-modern-hardware/

\[2\] "The New C++: Lay down your guns, knives, and clubs", Gavin Clarke, https://www.theregister.com/2011/06/11/herb_sutter_next_c_plus_plus

\[3\] "Multi-threaded executions and data races", https://en.cppreference.com/w/cpp/language/multithread
