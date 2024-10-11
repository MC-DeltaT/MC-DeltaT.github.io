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
