---
title: How does the Linux kernel manage your memory?
author: Andrea Di Dio
---

# `#include <linux/mm.h>`

If you have any further questions or suggestions after reading this writeup feel free to contact me at a.didio@student.vu.nl or on Twitter ([@hammertux](https://twitter.com/hammertux)). I will try to answer any questions or adopt any suggestions :)

## Introduction

Memory management in the Linux kernel is among the hardest topics which can be studied. Throughout the years, the `mm` sub-tree of the kernel has evolved rapidly to accommodate numerous architectures and to increase performance. In this writeup I will try to explain how the Linux kernel manages the memory available in a system by walking through the most important structures used to give concrete examples of the concepts used. This writeup is based on the 5.5.13 Linux kernel (latest stable release at the time of writing --- _Sat 28 Mar 15:56:40 CET 2020_ ). I will explain how things work on x86\_64 systems for the time being due to it being the predominant architecture nowadays.

## Physical Memory

### Pages

The most basic unit for memory management in Linux (and most other operating systems) is the **page** (physical page or page frame. Not to be confused with a virtual page.) which is 4KB (defined as the `PAGE_SIZE` macro in `<asm/page_types.h>`) for a _normal page_ and 2MB or 1GB for _huge pages_. The reason behind using the page as the standard unit is that the Memory Management Unit (_MMU_) which performs virtual to physical address translations via the page tables works at the page granularity. In Linux, each page is represented by the `struct page` (`<linux/mm_types.h>`), inarguable one of the most important structures used within the kernel. The page structure is 64B in size, which is ideal, allowing it to be cache-line aligned and contains all the information needed to represent and manipulate a page as I will discuss below.

* _For those of you who are interested in random fun statistics: given that the `struct page` is 64B and a standard page is 4KB, 64/4096 gives us 0.0156 ca., meaning that 1.56\% of all the DRAM available on a Linux system is taken up by page structures_.

Below I will explain some of the most interesting members of the page structure. I will skip over those related to the slab allocator because there is a separate [dedicated post](https://hammertux.github.io/slab-allocator) that covers the internals of the slab and buddy allocators.

| Member | Description |
|:------:|-------------|
| `unsigned long flags`{:.c} | bla |
|  


