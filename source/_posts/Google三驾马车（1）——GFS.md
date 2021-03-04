---
title: Google三驾马车（1）——GFS
date: 2021-03-04 20:12:49
tags: [Google, 论文]
categories: 分布式系统
---

最近阅读了号称Google分布式系统的三驾马车中的"[The Google File System](https://dl.acm.org/doi/pdf/10.1145/945445.945450?casa_token=oR7grA26w34AAAAA:foohVEqDKPuAtLOsysk87pEoO7zTgcYRPFKtOxG7QLzOMI8HnT1vybCb6kibsHO5RWxPm8C4RSRKiTw)"。由于水平尚浅，即便读的是[中译](https://duanmeng.github.io/2017/12/07/gfs-notes/)仍觉得倍感吃力，偶然间发现这个YouTube上的讲解视频——[深入浅出Google File System](https://www.youtube.com/watch?v=WLad7CCexo8)，可谓豁然开朗，打算回头再重读原文，收获颇丰。



# 架构的层次



{% asset_img 架构的层次.png 架构的层次 %}

