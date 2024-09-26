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

- Could you explain the terms "memory model", "data race", and "sequential consistency"?
- Do you know how `std::mutex`, `std::atomic`, `std::memory_order` work and when they are necessary?  
- Do you know how `volatile` relates to thread safety?  
- Could you explain the behaviour of the following code snippets?

```c++
int data;
bool ready = false;

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
int data;
std::atomic<bool> ready = false;

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
int data;
bool ready = false;
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
int data;
bool ready = false;

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

```c++
int data;
bool ready = false;

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

If you thought "no" or "unsure" to any of those questions, then read on and rest assured - let us share in a thread safety journey.

## Quick Shout-out to Herb Sutter

A lot of what I know about thread safety in C++ comes from a presentation by Herb Sutter called "atomic<> Weapons: The C++ Memory Model and Modern Hardware" \[1\]. The presentation is a wealth of information explained in a digestible and practical manner. A great watch, although it is three hours long. I present this article as an explanation which may be quicker to consume.

## What Is Thread Safety?

Let's begin by solidifying what we mean by thread safety.

In concurrent systems, there may be multiple processors performing operations at the same time on the same data. We say there are multiple "threads" of execution. In C++, that might look like this:

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

On the surface, the code may appear straightforward enough, but if we really think about it, things gets ambiguous. What does it mean for two processors to operate on data at the same time? What is "data"? What is "the same time"?

TODO: say more about memory model also being a framework for guiding concurrent code design. like driving a car to get to your destination

The framework which enables us to reason about these concurrency questions is called a "memory model". A memory model is a contract between the user and implementer of a computer system. Then, "thread safety" is about writing code which obeys the rules of a memory model. If code plays nicely with the memory model, we can reason about its behaviour and place assurances on its correctness - it is "thread safe" code. On the other hand, if code runs afoul of the memory model, it is not thread safe, and it could exhibit weird behaviour. The code might not work at all. Or it might work 95% of the time. Or it might always work but only on one type of computer.

Understanding memory models is the key to writing thread safe code - such knowledge gives you the confidence to create concurrent programs knowing they will function as you desire. At the moment, memory models probably sound like a vague concept, which is in some part true. They are massive abstractions layered over mountains of complexity of computer systems. In order to understand them fully, we must take a trip through some of that complexity, from the bare metal up - only then can we face the C++ memory model with a truly holistic understanding.

## Memory Model - The Hardware Perspective

We start at the bottom of the world in which our software runs: the raw computer hardware.

The brain of most computers is the "central processing unit", or CPU for short. The bulk of the CPU's role is to perform logic and math operations - called "instructions" - on data. Data used by a computer while switched on is stored in main memory (RAM), which typically sits separate to the CPU. Inside the CPU chip, there can be multiple "cores", which can independently execute instructions and access memory. Cores also have fast internal storage called "registers", which act like working out space for instructions. The setup of multiple cores in one computer is useful for running multiple programs simultaneously, and for increasing the performance of individual programs.

TODO: link to further reading about CPUs

Since we have multiple cores all doing things at the same time, what happens if two cores want to access the same memory? Such scenarios are extremely common nowadays. Even before your program runs, operating systems have a plethora of shared data used for coordinating program execution, tracking resource usage, and so on. And many programs themselves use multiple threads to boost performance or enact asynchronous tasks. Clearly, concurrent memory access is everywhere, and the hardware must deal with it *somehow*. How, is where hardware memory models come in. Depending on the design of the CPU, there will be a governing memory model which defines the behaviour of concurrent memory accesses. Let's have a look at some design factors with play a role in hardware memory models.

### Cache

Over time, CPU operating speeds have significantly outpaced the performance of main memory in commodity systems due to difference in underlying technologies. To alleviate performance bottlenecks arising from slow memory accesses, CPUs got "cache": a faster, but limited, intermediate storage between the CPU logic and main memory.

Cache works like a temporary buffer. When loading data from main memory, the data is put into the cache first. If the CPU reads the same memory address again later, the cache provides the data quickly, rather than waiting to load from memory again. When the CPU writes data, it first goes into the cache in case it needs to be read soon. If at any point the cache can't fit any more data, previous data is evicted. When utilised effectively, cache allows the CPU to operate at its peak performance, avoiding the need to wait for main memory accesses. Without cache, the performance of code using main memory (i.e., almost everything) is crippled.

But the existence of cache immediately raises problems for a CPU with multiple cores. The best performing cache is separate per core, due to circuit design complexity. However, if each core has a different cache, then cores could end up with different (cached) data for the same memory address! Data communication between cores could be ruined. This problem is known as "cache coherency", and it requires some thinking and deliberate design to solve. How it is solved factors into the memory model guaranteed to the CPU programmer.

### Out of order execution

A fascinating aspect of modern CPU design is their knack for not executing instructions in the order the programmer specifies. Such design is known as "out of order execution" and came about to improve utilisation of available CPU resources, hence improving software performance. We won't dive into the complex details here, but the gist is that by reordering of instructions, throughtput is increased by finding an ordering which takes advantage of the free resources in the CPU at that time. Of course, the reordering must be done such that the correctness of the program is not affected.[^2]

Memory access instructions may be reordered too, if desirable for performance. This is interesting, since memory is an entity external to the CPU, and one core might not know what else is accessing memory. Obviously, arbitrarily reordering memory accesses will heavily influence the correctness of data communication between cores, i.e. thread safety. Arbitrary memory instruction ordering would probably be unusably chaotic. At the same time, performance goals pressure CPU design towards increased reordering.  
CPU designers have some control over what "correctness" means by way of memory models. The memory model will define what memory access reorderings are possible, depending on the chosen tradeoff with performance. Compilers and CPU programmers must be aware of possible reorderings to ensure they use memory access instructions in an appropriate manner to produce the desired behaviour.

TODO: fix note numbering

[^2]: I have a more detailed introduction to out of order execution available [here](https://github.com/MC-DeltaT/cpu-performance-demos/tree/main/out-of-order-execution).

### Memory access alignment

In some CPU designs, the hardware can only directly access memory at memory addresses which are a multiple of the CPU's natural data size. For example, the CPU might like to deal in terms of 8 bytes, and can only fetch memory addresses that are a multiple of 8. We say these addresses are "aligned".[^3] Such a design is favourable for reducing complexity and improving efficiency of the circuit design.

Some CPU designs disallow unaligned accesses, but often in modern hardware, lack of alignment is permitted and handled by the CPU for convenience. Unfortunately, convenience usually comes at the cost of something. If memory can only be accessed in an aligned manner, then fulfilling an unaligned request requires piecing together data from multiple separate aligned accesses. The question arises of what happens if the data is changed by another core in between the two accesses? It's plausible we could get half old, half new data - gross. The memory model of the CPU will define what the behaviour is surrounding this scenario. Compilers and assembly writers might need to ensure all data is aligned correctly, or use special instructions to avoid the problem.

[^3]: I'm purposely using the word "memory" loosely here, referring to both main memory access and cache access. Similar issues arise for both:

- Main memory technology and/or its connection to the CPU may only electrically support aligned accesses by design.
- Cache is generally split into small chunks according to memory address, so accessing data which straddles two chunks may incur overhead or be disallowed.

### Where's the memory model?

At this point, you may be eager for an in-depth explanation of a real hardware memory model, such as for the ubiquitous x86 architecture. However, we are not going to discuss that here. While understanding hardware motivations is important, I argue knowing specifics is counterproductive, because as high level language programmers we should be focusing on software memory models. It is easy to believe we understand the hardware memory model and that the buck stops there. In fact, on the x86 architecture, the topics we covered don't play as large a role in thread safety as you might think, because the hardware memory model is quite generous![^4] Yet some people will try to claim, for example, that cache coherency is the reason why you must use `std::atomic`.  
What we really do need to know, and what should inform our decisions the most, is the programming language's memory model, which we will explore next.

[^4]: The x86 memory model is one of the most CPU-programmer-friendly out there, as much is solved invisibly by the hardware. Cache coherency is basically not a concern from the programmer's perspective, and most memory instruction reorderings are disallowed. Only unaligned memory access incurs some thread safety hazards. However, we pay a continual cost in performance, efficiency, and chip complexity in exchange for such a friendly model. Newer chips, such as ARM, have shifted the tradeoff closer to efficiency.

## Memory Model - The Software Perspective

Programming languages vary greatly in how they approach memory and concurrency, depending on their design goals. Some languages pretend concurrency doesn't exist, while others embrace it in their core features. Likewise, some languages may force you deep into the weeds of memory management, and others do not even expose the concept of memory[^5]. Further, under the surface, compilers perform all sorts of magic to make code run on many hardware platforms and with high performance.

Like a programming language's syntax defines what sequence of characters are legal, a language's memory model is a contract between the programmer and the compiler (and thus, hardware) on legal behaviour of concurrent memory accesses. In this section, we will see the factors which go into a programming language's memory model.

[^5]: Some programming languages, particularly functional languages, do not have the concept of shared mutable memory. Thread safety is effectively "solved" in such languages.

### Read-modify-write operations

Even with the friendliest hardware memory model with us, we can't guilelessly write correct programs when multiple threads are at play. Consider the case of incrementing an integer in memory, which a compiler might translate to this sequence of CPU instructions:

1. Read the value `V` of memory location `M` into a register `R`.
2. Increment `R` by 1. I.e. `V = V + 1`.
3. Write the new value of `V` in `R` back into memory location `M`.

This is a classic "read-modify-write" operation and is the bread and butter of computation. Read data from memory, do something do it, then write the result back to memory. Now, what happens if two threads execute this operation at the same time? Imagine `V` begins equal to 0.

| Time | Thread 1        | Thread 2        |
| ---- | --------------- | --------------- |
| 1    | Read 0          | Read 0          |
| 2    | Increment 0 → 1 | Increment 0 → 1 |
| 3    | Write 1         | Write 1         |

We thought we incremented `M` twice, but actually ended up with `M=1`! This is because nothing stops the threads' execution from overlapping. This problem is in fact a very fundamental issue in any system where read-modify-write operations occur concurrently, from the small scale of one CPU, to the huge scale of the internet. Dealing with concurrent read-modify-writes is a core concern of writing correct concurrent software. Solutions depend on the scale and complexity of the concurrent operations, ranging from special CPU instructions to specific algorithm design.

### Compiler optimisations

Languages like C++ are liked in part for their runtime speed, which is assisted by the awesome optimisations done by compilers. Modern C/C++ compilers can be quite aggressive in rearranging source code into fast machine code. To paraphrase Herb Sutter in his memory model talk\[1\]: the compiler accepts our source code and gives us back the "code we meant to write" to yield optimal performance.

The topic of code optimisation is huge and you could spend your whole career on it, but all we need to know here is the compiler will happily add, remove, and reorder memory accesses to improve performance. (Sound familiar? Yes, similar vein as out of order execution.) Common optimisations include: eliding adjacent reads, merging adjacent writes, and placing data in registers intead of memory.

To illustrate, consider this function which sums the elements of an array:

```c++
void sum(std::vector<int> const& array, long long* result) {
    *result = 0;
    for (int e : array) {
        *result += e;
    }
}
```

If dereferencing `result` incurs a memory access, then this function has the potential to perform a lot of memory accesses. Given that memory is relatively slow, the compiler would love to find an alternative method, and that is just what it will do. After optimisation, the code might resemble this:

```c++
void sum(std::vector<int> const& array, long long* result) {
    *result = 0;
    long long tmp = 0;  // CPU register
    for (int e : array) {
        tmp += e;
    }
    *result = tmp;  // Memory access
}
```

The compiler has used a temporary, fast CPU register to accumulate the result, rather than writing to slow memory every iteration. Brilliant!  
But wait - what if another thread was monitoring `result`? Before optimisation, it would see many partial sums, but now, it will only see 0 and the final sum. You may say that in this example, of course there is no other thread watching. Congratulations; as a human programmer you know the intent behind the code. Unfortunately, the compiler may not know the context. Perhaps you want the optimisation, or perhaps not - how should the compiler know for certain? There is a tradeoff between freedom to optimise and simplicity for the programmer. Software memory models specify where the tradeoff is set, defining how the programmer specifies their intent and which optimisations the compiler may do surrounding memory access.

## The C++ Memory Model

Hopefully by now, you have a good idea of what is at play behind the scenes of thread safety, and what motivates the existence of memory models. That means you are ready to tackle the tricky topic of the C++ memory model. Buckle up and get ready for some serious C++ knowledge.

### History and context

C++ first standardised its memory model in C++11, over 25 years after the inception of the language. I might guess that it took so long, as many things do in C++, due to the huge variety of use cases and platforms on which it is run (although I have no evidence for this). Before C++11, the official language standard provided no guarantees on the behaviour of concurrent code. Of course people did write concurrent code before 2011, but correctness came from specific compilers and hardware platforms. That meant code you wrote that worked on one machine may not work on another, and C++ didn't care. The pre-C++11 Standard dealt in terms of single threaded applications only.

Naturally, it would be nice for the language to specify one way to go about concurrency which is efficient and portable across many systems, especially as multithreaded applications became more prevalent in consumer software. So with C++11, one official memory model was finally introduced.[2]

### C++11 memory model - overview

As mentioned a few times so far, a memory model is a contract between a user and implementer. In the case of C++, the user is the programmer, and the implementer is the compiler plus CPU hardware. C++'s memory model specifies how a program may behave in scenarios where multiple threads are active. If you write C++ code with good understanding of the memory model, the compiler will make sure the program behaves how you intend. Otherwise, the program may not work as expected.

The first step taken by C++ memory model is to forbid all scenarios where data is shared between threads, unless special C++11 concurrency constructs are used. Specifically, it says[3]:

> When a thread accesses a memory location and a different thread modifies the same memory location, a data race occurs unless \[special conditions here\].
>
> If a data race occurs, the behavior of the program is undefined.

This is quite a harsh landscape to step into. Even the simplest code with multiple threads is unequivocally *broken* unless we take explicit action according to the memory model. Here is an example of what we cannot do:

```c++
// DATA RACE - UNDEFINED BEHAVIOUR

int shared_data = 0;

void read_data() {
    std::cout << shared_data << std::endl;
}

void write_data() {
    shared_data = 42;
}

std::thread thread1{read_data};
std::thread thread2{write_data};
```

The probable reason C++ takes this stance is for performance. Thread safety is somewhat at odds with performance, and C++ loves to prioritise performance, so by default the compiler and hardware are allowed to go wild with optimisation. You may argue this approach is suboptimal, and you may be right - but you cannot deny the reality. If we want thread safety, we must force the compiler's hand.

### Atomics - a step towards sanity

One problem we have established about memory accesses is that computers love to jumble them around. For example, they might be split up, reordered, or outright removed. With memory being such a fundamental component in software, these facts leave us with disastrously shaky foundations on which to attempt to build programs. "Atomics" provide critical initial stabilisation to enable safe memory concurrency.

The idea of an atomic operation is that of an operation which is fundamental and indivisible, either done completely or not at all. You can think of it as an operation which occurs at an infinitely small point in time. One nanosecond, nothing has happened, and the next, the operation is completed. No matter what you do, you cannot observe an atomic operation "in progress", nor force two of them to overlap in execution.

If memory accesses could be atomic, then they would align with our naive intuition of how memory should work, and provide a baseline on which to reason about memory concurrency. A memory write would just be a write, and a read just a read - no catches. Such a dream is granted by C++11's [`std::atomic`](https://en.cppreference.com/w/cpp/atomic/atomic). This type wraps another trivial type, for example `int`, and provides an interface with atomic semantics blessed by the Standard. If an object is `std::atomic`, then accessing it concurrently is no longer undefined behaviour. Compiler and hardware shenanigans are (mostly) guaranteed to not ruin attempts at using the object.  
Hence, atomics are one of the "special conditions" mentioned earlier[3]:

> When a thread accesses a memory location and a different thread modifies the same memory location, a data race occurs unless:  
>
> - **both conflicting evaluations are atomic operations**
> - \[...\]

Revisiting the previously-undefined-behaviour code example, it can be made well-defined via application of `std::atomic` to the shared data.

```c++
// DEFINED BEHAVIOUR

std::atomic<int> shared_data = 0;

void read_data() {
    std::cout << shared_data.load() << std::endl;
}

void write_data() {
    shared_data.store(42);
}

std::thread thread1{read_data};
std::thread thread2{write_data};
```

The C++ memory model has promising things to say about this code. `read_data()` cannot print some garbage value, nor crash the program, nor launch nuclear weapons. It can print either 0 or 42, depending on which of `thread1` or `thread2` happens to execute first.

Now that we have regained the most basic memory access soundness with `std::atomic`, our discussion moves to the behaviours between many memory accesses.

### Memory order and visibility

In the previous example, we saw the final value of `shared_data` can be different depending on which thread executes first. Such nondeterminism is a common occurrence in concurrent programs thanks to the various sources of memory access reordering we have learnt about. The order of memory accesses as written in source code likely doesn't match the order executed by hardware. Naturally, we have no hope in writing correct programs if order means naught. Hence, enter the next two foundational elements of the C++ memory model: memory ordering, and memory visibility.

Memory ordering encompasses rules regarding how memory accesses may or may not be reordered. Memory visibility is a closely related concept which describes how the effects of memory accesses are seen in other threads. For example, considering the following code, what combinations of values of `x` and `y` might another thread read?

```c++
std::atomic<int> x = 0;
std::atomic<int> y = 0;

x.store(10);
y.store(20);
```

The first thing to know about memory ordering in C++ is there is no one "memory order", but a collection of rather complex rules. The programmer has some choice over how strict or relaxed they want the ordering to be via [`std::memory_order`](https://en.cppreference.com/w/cpp/atomic/memory_order). Read that link at your own risk: it's a rabbit hole of Standardese terms and cut-throat logic. To be quite honest, I do not fully understand all aspects of `std::memory_order`[^6].  
Fortunately, the second thing to know about memory ordering in C++ is that in the sort of everyday C++ I would recommend you write, most of the complexities of memory ordering aren't relevant, and there are only a few scenarios to concern ourselves with.

The ubiquitous memory ordering scenario is "acquire-release". In this scenario, one thread "releases" a memory resource for use, while another thread "acquires" it for consumption. Typically, an acquire is a memory read, while a release is a memory store. (If you have heard of the producer-consumer pattern, this is the memory ordering at play.) The critical aspect is the relationship between a release by one thread and a subsequent acquire by another thread, which forms a memory ordering. The C++ memory model says\[4\]:

- If thread 1 performs a write `W` of value `V` to memory `M`, which is a release operation;
- And thread 2 reads value `V` from `M`, which is an acquire operation;
- Then thread 2 subsequently observes all memory access effects performed by thread 1 up to `W`.

In other words, once thread 1 performs the release, it "signs off" on all previous memory operations as safe to observe. If another thread acquires that release, then it will see everything that was signed off. C++ calls this a "happens-before" relationship - memory access in thread 1 really do happen before and are observable by thread 2.  
Happens-before is another special condition that escapes data races[3]:

> When a thread accesses a memory location and a different thread modifies the same memory location, a data race occurs unless:  
>
> - \[...\]
> - **one of the conflicting evaluations happens-before another**

To illustrate acquire-release, consider this pseudocode:

```c++
int x = 0;
int M = 0;

void release_data() {
    x = 42;
    release_operation(M = 10);  // RELEASE
}

void acquire_data() {
    if (acquire_operation(M) == 10) {   // ACQUIRE
        std::cout << x << std::endl;
    }
}

std::thread thread1{release_data};
std::thread thread2{acquire_data};
```

(Note `acquire_operation()` and `release_operation()` are not real constructs, but represent the concept of an acquire and release operation, respectively. We will discuss real acquire/release functionality shortly.)

What values of `x` might `thread2` print? If `thread2` acquires `M == 10`, then it *must* read `x == 42`, because that is the value released by `thread1`. It is not possible for 0 to be printed, nor any other value. However, it is possible for `thread2` to run before `thread1`, in which case `M == 0` will be acquired and nothing will be printed.

`std::atomic` reads and writes are acquire and release operations, respectively. The above pseudocode can be made into valid C++ like so:

```c++
int x = 0;
std::atomic<int> M = 0;

void release_data() {
    x = 42;
    M.store(10);    // RELEASE
}

void acquire_data() {
    if (M.load() == 10) {   // ACQUIRE
        std::cout << x << std::endl;
    }
}

std::thread thread1{release_data};
std::thread thread2{acquire_data};
```

Notice that `x` need not be atomic, because the acquire-release memory ordering set up by the atomic `M` applies to all memory accesses, not just atomics.

It's important to stress that the presence of acquire-release operations only safeguards memory accesses where the happens-before relationship exists, and does not preclude the possibility of other data races. For example, the following adjustment of our code creates a data race, because writing `x = 50` conflicts with reading `x` in the other thread:

```c++
// UNDEFINED BEHAVIOUR

void release_data() {
    x = 42;
    M.store(10);    // RELEASE
    x = 50;         // BAD
}

void acquire_data() {
    if (M.load() == 10) {   // ACQUIRE
        std::cout << x << std::endl;
    }
}
```

Another classic construct which provides acquire-release semantics is a mutex, represented in C++ by [`std::mutex`](https://en.cppreference.com/w/cpp/thread/mutex). A mutex provides two operations, "lock" i.e. acquire, and "unlock" i.e. release, with the additional guarantee that only one thread has locked the mutex at any time (called "mutual exclusion"). Memory access ordering is provided by mutexes even if the data is not `std::atomic`. We'll look at `std::mutex` more closely in a later section.

In a suprisingly user-friendly twist, C++ tries to default to a memory ordering even stricter than acquire-release - a model called "sequential consistency for data-race-free programs". Under this model, when using `std::atomic`, not only does acquire-release ordering apply, but there is also a single ordering of all `std::atomic` memory accesses observed by all threads. Effectively, `std::atomic` by default behaves like our naive understanding of memory access, sans compiler optimisations and hardware trickery!  
Going back to a previous example:

```c++
std::atomic<int> x = 0;
std::atomic<int> y = 0;

x.store(10);
y.store(20);
```

When any other thread reads `x` and `y`, the only possible outcomes are:

1. `x == y == 0`
2. `x == 10`, `y == 0`
3. `x == 10`, `y == 20`

Since there is a single global ordering as written in the source code. (Of course another thread may still observe midway through the code, resulting in outcome 2.)  
The behaviour may be modified via `std::memory_order`, which can provide a small increase in memory access performance on some CPU architectures. But I **strongly** recommend you stay with the default, because sequential consistency is the safest and most intuitive option. For nearly all applications, the small performance improvement will not be worth the risk of incorrect code and subsequent time and mental health lost to debugging. Testament to sequential consistency's fundamental value is its adoption in the memory models of other programming languages such as Java, Go, and Rust.

[^6]: Amusingly, it seems many C++ experts are confused too, as `std::memory_order::consume` is discouraged from use while its specification is revised.

## C++ Synchronisation APIs

At this point, we hopefully have a good understanding of the C++ memory model. Concurrent memory accesses cause data races and undefined behaviour, unless we take care to abide by memory ordering and visibility rules. The mechanisms on which we build thread safety are in general called "synchronisation" mechanisms. `std::atomic` is such a mechanism, which we have touched on already. In this section, we will discuss more synchronisation mechanisms provided by the C++ Standard Library.

### `std::atomic`

[`std::atomic`](https://en.cppreference.com/w/cpp/atomic/atomic) is the basic building block of thread safe memory access. As discussed earlier, it wraps a type and provide atomic memory access semantics. The wrapped type usually must be [TriviallyCopyable](https://en.cppreference.com/w/cpp/named_req/TriviallyCopyable): generally speaking, fundamental types (such as `int`, `float`, and pointers), and arrays and structs thereof. Complex types such as `std::vector` are not compatible, and require other sychronisation mechanisms. The reason is that atomicity is generally implemented via special CPU instructions which can only operate on primitive data types. However, as a reuslt of its primitiveness, `std::atomic` is often the cheapest synchronisation mechanism in terms of performance.

`std::atomic` provides atomic [`load()`](https://en.cppreference.com/w/cpp/atomic/atomic/load) and [`store()`](https://en.cppreference.com/w/cpp/atomic/atomic/store) member functions for reading and writing. It also provides some atomic read-modify-write operations, which are handy for building higher-level synchronisation. For example, [`operator++()`](https://en.cppreference.com/w/cpp/atomic/atomic/operator_arith) is overloaded for integral `std::atomic` to atomically increment the contained value by 1.

### `std::mutex`

[`std::mutex`](https://en.cppreference.com/w/cpp/thread/mutex) is a ubiquitous synchronisation mechanism that provides acquire-release and mutual exclusion semantics. It is used to ensure that only one thread executes a section of code at a time, which is necessary when the data and code is more complex than can be managed with `std::atomic`.

```c++
std::vector<int> data;
std::mutex mutex;

void produce() {
    mutex.lock();
    data.push_back(10);
    mutex.unlock();
}

void consume() {
    mutex.lock();
    if (!data.empty()) {
        std::cout << data.front() << std::endl;
    }
    mutex.unlock();
}

std::thread thread1{produce};
std::thread thread2{consume};
```

In the above example, an instance of `std::mutex` provides thread safety for a complex data type `std::vector`. `mutex.lock()` waits if the mutex is already locked, and also performs an acquire operation. `mutex.unlock()` performs a release operation and unlocks the mutex (allowing another thread to lock it). Thus memory ordering is achieved, and only one thread can modify `data` at any time.

C++ provides variants of `std::mutex` which add features, such as [`std::timed_mutex`](https://en.cppreference.com/w/cpp/thread/timed_mutex) and [`std::recursive_mutex`](https://en.cppreference.com/w/cpp/thread/recursive_mutex).  
Additionally, [`std::lock_guard`](https://en.cppreference.com/w/cpp/thread/lock_guard) and friends provide RAII wrappers to make locking and unlocking more foolproof.

### `std::condition_variable`

TODO

### `std::counting_semaphore`

TODO

### `std::thread`

TODO

TODO: discuss C++ synchronisation APIs: atomic, mutex, thread, fence, etc

### A note on `volatile`

Far too many sources claim thread safety can be achieved by declaring variables as `volatile`. This is theoretically and practically incorrect. To put it bluntly: `volatile` does not provide any inter-thread synchronisation, and attempting to use `volatile` to share data between threads is undefined behaviour.

So what *does* `volatile` do? It effectively forces the compiler to generate a memory access instruction, even where the compiler may wish to not do so due to optimisation. While this guarantee might look promising for thread safety, note that this does not mitigate memory access trickery done by hardware - the memory access may still, for example, be reordered through out of order execution. `volatile` is intended to be used for [signal handlers](https://en.cppreference.com/w/cpp/utility/program/signal) and memory locations with special hardware side effects (e.g. microcontroller registers).

Unfortunately, `volatile` may appear to work as a thread safety mechanism on platforms where the hardware memory model is generous. On the common x86 CPU architecture, memory access reorderings are limited, so a normal memory access instruction emitted from use of a `volatile` object can result in a functioning program. It is probably due to this fact that `volatile` has been proclaimed as a thread safety tool. However, the same code that works on x86 likely will not work on a different architecture, such as ARM, because that code invokes undefined behaviour. There is no reason to play with fire trying to use `volatile` rather than proper thread safety mechanisms on a modern compiler. Just use `std::atomic`, please!

## Practical examples

TODO: revisit code from the start

## References

\[1\] "atomic<> Weapons: The C++ Memory Model and Modern Hardware", Herb Sutter, https://herbsutter.com/2013/02/11/atomic-weapons-the-c-memory-model-and-modern-hardware/

\[2\] "The New C++: Lay down your guns, knives, and clubs", Gavin Clarke, https://www.theregister.com/2011/06/11/herb_sutter_next_c_plus_plus

\[3\] "Multi-threaded executions and data races", https://en.cppreference.com/w/cpp/language/multithread

\[4\] "std::memory_order: Release-Acquire ordering", https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Acquire_ordering
