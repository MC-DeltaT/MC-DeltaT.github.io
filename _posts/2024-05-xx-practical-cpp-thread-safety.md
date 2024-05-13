---
title: You Probably Don't Know Thread Safety - A Practical Guide for C++
---

**Thread safety** - one of those complex topics which almost no one seems to get right. Programmers quake at the thought of it, education material can't get its game straight, and everyone is left stumbling with half-broken code. Particularly in C++, where (as usual) it's left for the programmer to figure out lest they be shot in the foot.

An intidimating topic, for sure; yet not an untouchable one. Today I am here to set the record straight on thread safety.  
In this article we will discuss everything. You may enter with thread safety confusion, and you will exit with thread safety enlightenment.

## Why Is It So hard?

Thread safety is one of those legitimately hard problems where every which way you look, it's still hard to grapple with. The topic is technically involved, affected by both software and hardware behaviour. Reasoning about multiple simultaneous threads pushes the limits of the brain's working capacity. Even software tools can't really solve concurrency for us. Plus, synchronisation issues are notoriously random, being dependent on process scheduling, compiler configuration, and type of CPU.

The nail in the coffin comes when education materials don't keep up. So many sources out there are flat out wrong, or at best, *not holistic enough to be correct*.  
You can't approach thread safety from a software perspective only, nor a hardware perspective only. You can't slap `volatile` on everything and call it a day. You can't pretend that the C++ Standard doesn't exist.

## Quick Shout-out to Herb Sutter

A lot of what I know about thread safety in C++ comes from a presentation by Herb Sutter called "atomic<> Weapons: The C++ Memory Model and Modern Hardware". The presentation is a wealth of information explained in a digestible and practical manner. A great watch, although it is three hours long. I present this article as a shorter explanation.

## What Is Thread Safety?

Let's begin by solidifying what we mean by thread safety. This section is necessarily a bit theoretical, but bear with me.

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

If you really think about it, what does it mean for two things to operate on data *at the same time*? What is "data"? What is "the same time"? Computers are near-incomprehensibly complex machines, so these questions are not trivially answered.

There is a concept of a "memory model" which describes how such questions are answered for the perspective of someone using the system. A type of CPU will have a memory model. A programming language such as C++ will also have a memory model. As high level language programmers, we primarily interact with the programming language's memory model.

If memory models sound abstract at this stage, good. Despite having grounds in hardware reality, memory models are very very much a magic abstraction, especially in C++. Later we will examine the details of the C++ memory model.

"Thread safety" is about writing code that obeys the rules of a memory model, such that code always behaves as expected when run.  
If code doesn't obey the memory model's rules, then we say it is not thread safe, and it could exhibit weird behaviour. The code might not work at all. Or it might work 95% of the time. Or it might always work but only on one type of computer.

The main problem which arises from disobeying a memory model is named "data race" or "race condition". This is an event where multiple threads possibly affect the same data simultaneously, and the only thing preventing it is timing. In this event, the result depends on which thread executes first. For operations with multiple steps, it also depends on the sequence of executions of steps between threads (and that is where it gets hard to wrap your head around).  
Even without further information, you might guess that the previous C++ code example contains a data race, and you'd be correct.

## Memory Model - The Hardware Perspective

I give you this section of knowledge on the condition that you please do not take it and run without reading further. OK? Let's dive in.

TODO

## Memory Model - The Software Perspective

TODO

## The C++ Memory Model

TODO
