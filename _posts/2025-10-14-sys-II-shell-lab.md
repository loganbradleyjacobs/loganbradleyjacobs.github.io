---
layout: project
title: "Systems II Shell Lab"
date: 2025-10-14
tags: [Reverse Engineering, Crackme, Radare2]
excerpt: "A simple shell, complete with pipes, that I'm working on for my CSC 251 class at WFU."
thumbnail: /assets/img/sys-II-shell-lab.png
---

## Overview
This is a post about my Shell lab for Systems II, a class I'm taking at WFU. I show the problems I needed to solve, and how I solved them.

## Class Background
In out Systems II (CSC 251) class this semester, we have been learning about how operating systems work, including how they abstract the concept of work to be done by "processes", which can be more easily controlled.

Processes need to `fork()` from another process to be created, over copying all of their information, whether it be the text section where code is stored, the data and bss sections with initialized and uninitialized data, the stack or the heap, where function calls and automatic variables are stored respectively, or the U-Area, which is how the operating system can store the data needed to track and handle the process and its tasks. There are some small exceptions to this, such as the PID (mentioned later), but essentially a child (the newly fork()ed process) and its parent (the process the child was fork()ed from) are copies of one another.

Processes also need to `exec()` to do anything useful. `exec()` is a family of system calls that allow the process to replace its user-space data (anything but the U-Area) with that of a different program, along with any arguments that program might need to operate. The U-Area is largely untouched, although there are some [exceptions](https://stackoverflow.com/questions/2333637/is-it-possible-for-a-signal-handler-to-survive-after-exec) that are good to know about.

Lastly, processes need to `wait()` when they are parents. `wait()` pauses execution of the parent program until one of its child processes has finished execution. `waitpid()`, an extension of `wait()` allows a programmer to specify the child process they would like to wait for by passing the child's `PID` or process identifier. Every process has their own unique PID, which is stored in the U-Area.

Below is a diagram of process structure in Unix. You can see the different sections of the process, and the U-Area previously mentioned. This image is from the course slides, Chp 2, slide 24.

![Process Structure](assets/img/sys-II-shell-lab-process-structure.png)

## Assignment
