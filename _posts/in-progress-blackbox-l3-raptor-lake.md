---
title: Building L3 Eviction Sets for Raptor Lake CPUs
---

# Building Blackbox L3 Eviction Sets for Intel Raptor Lake CPUs via Cache Coloring

Author: Andrea Di Dio

If you have any further questions or suggestions after reading this writeup feel free to contact me at a.di.dio@vu.nl or on X ([@hammertux](https://x.com/hammertux "hammertux")). I will try to answer any questions or adopt any suggestions :)

## Introduction

I was playing around with some cache-based timing side-channels on an Intel Raptor Lake CPU (i9-13900K) and tried building eviction sets for the L3 cache (LLC) and found that on newer CPUs, Intel has changed the inclusivity policy. This makes it a bit more challenging to effectively do cache coloring as some others algorithms propose (e.g., [Theory and Practice of Finding Eviction Sets](https://arxiv.org/pdf/1810.01497)). In this writeup I will walk you through my findings and propose a modified algorithm to efficiently build last level cache eviction sets on CPUs which no longer use inclusive LLCs.

## Caches