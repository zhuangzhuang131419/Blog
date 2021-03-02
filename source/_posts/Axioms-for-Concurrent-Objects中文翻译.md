---
title: Axioms for Concurrent Objects中文翻译
date: 2021-02-28 18:01:26
tags: [论文翻译, 一致性]
categories: 分布式系统
---

# 并发对象的公理

## Abstract

Specification and verification techniques for abstract data types that have been successful for sequential programs can be extended in a natural way to provide the same benefits for concurrent programs. We propose an approach to specifying and verifying concurrent objects based on a novel correctness condition, which we call “linearizability.”



抽象数据类型的规范和验证技术已经成功应用于顺序程序(sequential program)，自然而然的，我们就想把优点扩展到并发程序。我们提出一个基于新的正确性条件来指定和验证并发对象的方法，我们称之为”线性化(linearizability)“。



Linearizability provides the illusion that each operation takes effect instantaneously at some point between its invocation and its response, implying that the meaning of a concurrent object’s operations can still be given by pre- and post-conditions. In this paper, we will define and discuss linearizability, and then give examples of how to reason about concurrent objects and verify their implementations based on their (sequential) axiomatic specifications.



线性化提供了一种错觉，即每个操作在其调用(invocation)和响应之间的某个点瞬间生效，这说明并发操作的意义仍然可以由前置和后置条件给出。在这篇文章中，我们将定义和讨论线性化，然后举例说明如何根据并发对象的（顺序）公理规范来推理和验证它们的实现。



## Introduction

This paper shows that the specification and verification techniques for abstract data types that have been successful for sequential programs can be extended in a natural way to provide the same benefits for concurrent programs. Our two main contributions are:

* New techniques for using (sequential) axiomatic specifications to reason about concurrent objects; and
* A novel correctness condition, which we call linearizability.



本文表明，抽象数据类型的规范和验证技术已经成功地应用于顺序程序，自然而然地，可以以某种方式进行扩展，从而为并发程序提供同样的好处。我们的两个主要贡献是：

* 使用（顺序）公理规范对并发对象进行推理的新方法；
* 一个新的正确性条件，我们称之为线性化。



Informally, a concurrent system consists of a collection of sequential processes that communicate through shared typed objects. This model is appropriate for multiprocessor systems in which processors communicate through reliable, high-bandwidth shared memory. Whereas “memory” suggests registers with read and write operations, we use the term *concurrent object* to suggest a richer semantics. Each object has a type, which defines a set of possible values and a set of primitive operations that provide the only means to create and manipulate that object. We can give an *axiomatic specification* for a typed object to define the meaning of its operations when they are invoked one at a time by a single process. In a concurrent system, however, an object’s operations can be invoked by concurrent processes, and it is necessary to give a meaning to possible interleavings of operation invocations.



不严谨地说，一个并发系统由一组通过共享类型对象进行通信的顺序进程组成。该模型适用于处理器通过可靠的高带宽共享内存进行通信的多处理器系统。虽然“内存”表示寄存器具有读写操作，但我们使用术语“并发对象”来表示更丰富的语义。每个对象都有一个类型，它定义了一组可能的值和一组基本操作，这些操作提供了创建和操作该对象的唯一方法。我们可以为一个类型化对象提供一个*公理化规范*，当单个进程一次调用一个操作时，定义其操作的含义。然而，在并发系统中，对象的操作可以由并发进程调用，因此有必要对操作调用的可能交错(interleaving)给出一个含义。



Our approach to specifying and verifying concurrent objects is based on the notion of *linearizability*. A concurrent computation is linearizable if it is “equivalent,” in a sense formally defined in Section 3, to a legal sequential computation. We interpret a data type’s (sequential) axiomatic specification as permitting only linearizable interleavings. Instead of leaving data uninterpreted, linearizability exploits the semantics of abstract data types; it permits a high degree of concurrency, yet it permits programmers to specify and reason about concurrent objects using known techniques from the sequential domain. Unlike alternative correctness conditions such as sequential consistency [17] or serializability [26], linearizability is a local property: a system is linearizable if each individual object is linearizable. Locality enhances modularity and concurrency, since objects can be implemented and verified independently, and run-time scheduling can be completely decentralized. Linearizability is a simple and intuitively appealing correctness condition that generalizes and unifies a number of correctness conditions both implicit and explicit in the literature.



我们指定和验证并发对象的方法基于*线性化*的概念。如果并发计算在第3节正式定义的意义上与合法的顺序计算“等价”，那么它是可线性化的(linearizable)。我们将数据类型的（顺序）公理规范解释为只允许线性化交错(linearizable interleaving)。线性化利用了抽象数据类型的语义，而不是不解释数据；它允许高度的并发性，但允许程序员使用顺序域中的已知技术来指定和推理并发对象。与顺序一致性[17]或可串行化[26]等其他正确性条件不同，线性化是一种局部性质：如果每个单独的对象都是可线性化的，则系统是可线性化的。局部性增强了模块性和并发性，因为对象可以独立地实现和验证，并且运行时调度可以完全分散。线性化是一个简单而直观的正确性条件，它概括和统一了文献中的一些隐式和显式的正确性条件。









