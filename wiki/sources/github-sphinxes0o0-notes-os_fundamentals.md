---
type: source
source-type: github
title: "操作系统基础 (OS Fundamentals)"
author: "notes repo"
date: 2026-05-20
size: medium
path: raw/github/notes/os_fundamentals/
summary: "操作系统原理课程笔记：进程管理、内存管理、文件系统、IO模型、网络协议、并发控制"
tags: [linux, os]
created: 2026-05-20
---

# 操作系统基础 (OS Fundamentals)

## 来源信息

- **路径**: raw/github/notes/os_fundamentals/
- **文件数**: 39个lecture + 8个加餐
- **类型**: 课程笔记（Markdown格式）

## 核心内容

课程涵盖计算机组成原理、操作系统核心概念：
- 计算机基础：图灵机、可计算理论、P vs NP
- 进程与线程：创建开销、状态机、分时调度
- 内存管理：虚拟内存、页表、缓存置换、内存回收
- 文件系统：VFS、FAT/NTFS/Ext3、B+树、HDFS
- IO模型：select/poll/epoll、IO多路复用
- 网络协议：TCP/IP五层模型、多路复用
- 并发控制：锁、信号量、分布式锁、哲学家就餐问题
- 用户态/内核态：系统调用、中断向量

## 关键引用

> Linux中创建进程需要分配完整内存空间（堆栈、正文区等），创建线程只需确定PC指针和寄存器值。

> epoll在内核中使用红黑树管理所有待监听的fd，查找/插入/删除均为O(log n)。

## 相关页面
- [[synthesis/topic-os-fundamentals]] — 综合分析
- [[sources/github-sphinxes0o0-notes-network_fundamentals]]
