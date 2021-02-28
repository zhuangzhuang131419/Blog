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



线性化提供了一种错觉，即每个操作在其调用(invocation)和响应之间的某个点瞬间生效，这说明



## Introduction



