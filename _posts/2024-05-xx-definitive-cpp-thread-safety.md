---
title: You Probably Don't Know Thread Safety - A Definitive Guide for C++
---

**Thread safety** - one of those complex topics which almost no one seems to get right. Programmers quake at the thought of it, with a dangerous world of threads, atomics, and locks to navigate, lest they summon the elusive *data race*. Meanwhile, education materials are doing them no favours - far too many sources are at best misleading, and at worst, flat out wrong.
Particularly intimidating is thread safety in C++, where (as usual) it's left for the programmer to figure out how to avoid foot bullets.

Are you one of the developers who quake at thread safety? Or perhaps, despite learning attempts, you never really "got" how to do it properly? This is the article for you. Today I am here to set the record straight on thread safety.

## Do You Really Know Thread Safety?

I consider thread safety to be one of the most misunderstood topics in C++. Why? Two reasons:

1. Thread safety is complex and nuanced, so quality education material is hard to come by.
2. There is so much nondeterminism (from the programmer's perspective) that it's easy to convince yourself you understand thread safety.

Without sounding too accusatory - because in large the problem is due to #2 -  there seems to be an alarming number of programmers who believe they understand thread safety, yet unknowingly have the gun a millimetre from their foot.

Some questions to consider:

- Could you explain the terms "memory model", "sequential consistency", and "data race"?
- Do you know how `std::mutex`, `std::atomic`, `std::memory_order` work and when they are necessary?  
- Do you know why `volatile` does or does not assist thread safety?  
- Could you explain the behaviour of the following code snippets, and say which are correct/incorrect?

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

If you thought "no" or "unsure" to any of those questions, then read on, and let us share in a thread safety journey.

## Quick Shout-out to Herb Sutter

A lot of what I know about thread safety in C++ comes from a presentation by Herb Sutter called "atomic<> Weapons: The C++ Memory Model and Modern Hardware" \[1\]. The presentation is a wealth of information explained in a digestible and practical manner. A great watch, although it is three hours long. I present this article as a shorter explanation.

## What Is Thread Safety?

Let's begin by solidifying what we mean by thread safety.

In concurrent software and hardware with multiple CPU cores, there may be multiple threads executing instructions at the same time. The instructions may operate on the same data.

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

| CPU    | Instructions | Data        |
| ------ |--------------| ----------- |
| Core 1 | func0        | shared_data |
| Core 2 | func1        | shared_data |

(Warning: the above code is not valid.)

If we really think about it, what does it mean for two things to operate on data *at the same time*? What is "data"? What is "the same time"? Computers are near-incomprehensibly complex machines, answering these questions is not trivial.

The framework which enables us to reason about these questions is called a "memory model". A type of CPU will have a memory model. A programming language such as C++ will also have a memory model. As high level language programmers, we primarily interact with the programming language's memory model.  
We are deliberately introducing memory models here as an abstract concept. Despite having grounds in hardware reality, memory models are very much a magic abstraction, especially in C++. We will work our way up to a full explanation of the C++ memory model.

"Thread safety" is about writing code which obeys the rules of a memory model. If code plays nicely with the memory model, then we can reason about its behaviour and place assurances on its correctness - it is "thread safe" code. On the other hand, if code runs afoul of the memory model, then we say it is not thread safe, and it could exhibit weird behaviour. The code might not work at all. Or it might work 95% of the time. Or it might always work but only on one type of computer.

## Memory Model - The Hardware Perspective

Let's start by looking at memory models as they function at the bare metal. By reading this section, you promise not to stop here and run off with the newfound CPU knowledge. A proper understanding of thread safety requires knowledge both hardware and software memory models!

TODO

## Memory Model - The Software Perspective

TODO

## The C++ Memory Model

TODO

## References

\[1\] "atomic<> Weapons: The C++ Memory Model and Modern Hardware", Herb Sutter, https://herbsutter.com/2013/02/11/atomic-weapons-the-c-memory-model-and-modern-hardware/
