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

<img src="/assets/img/sys-II-shell-lab-process-structure.png" alt="Process Structure" class="project-image">

## Assignment

In this assignment, we are tasked with creating a shell that can handle a line like: 
```
command [indentifier[identifier]]
```
and parse argv from that. We can then use `execvp()` to execute the file that contains the command. Then, `execvp()` will search for that file in the paths specified by the `$PATH` environment variable. 

We also need to handle if the last token in the command is the `&` operator, which means that the command should be executed concurrently rather than the shell waiting for the command to terminate before prompting the user again.

The shell is to be in a loop that only exits if the user enters 'exit'. `SIG_INT`, aka Ctrl-C should not interrupt the shell either. We also should return error messages if `execvp()` cannot execute. 

Lastly, we need to include pipes. Pipes are a way that two processes can communicate with one another. If I was to type `ls | grep "test"` into my terminal, that means the `ls` command's output should go into the `grep "test"` command's input. We also know that the maximum number of pipes is 2, per the assignment's requirements. With that, we are able to start implementing.

## The Code

Firstly, since the shell needs to be in a loop, lets get a main function that has an infinite loop in it, that prints a prompt, and gets user input. We'll have to `#include <stdio.h>` as we want to print to the terminal. We could use write calls, but if we always call `fflush(stdout)`, then we can use `printf`'s formatting with the same basic functionality, and without the headache that comes from `fork()`ing processes that still have characters in their stdout buffer. We can also use ANSI escape codes to deal with formatting.

After we prompt the user, we need to take input, and an easy way to do that is to use `fgets()`. `fgets()` expects a buffer to be passed into it to hold what it reads as input. This means that we need to set a maximum buffer size, or get clever and dynamically allocate memory. Right now, lets get something up and running, so we'll decide that the maximum input buffer size is 255 characters, and the maximum number of tokens in each command will be 20. Since the maximum amount of pipes is 2, we can also say the maximum number of commands is 3. 

So, we define the buffer, and pass it and its length to fgets, also error checking as we do so. We null terminate the input, and return. Now we have a program that loops, each time presenting a prompt and taking input and putting it in a buffer. My code looked like this:

<img src="/assets/img/sys-II-shell-lab-code-1.png" alt="Code 1" class="project-image-code">

As you can see, there are a few other things I needed to include to make the behavior as safe as I could. Also, don't mind the writeLog() call. When doing this project on my own, I realized how useful it was to have a logfile so I could trace back the shell's behavior when it went wrong.

Now though, we have input from the user, but we need to parse it into something useful. This can be done by a string tokenization function called `strtok`. However, we need to tokenize by 2 delimiters: first by separating the commands from each other (delimited by `|`), and second separating the tokens from each other within each command (delimited by ` `). With this pattern, it is probably easier to use `strtok_r`, another niche tokenization function that does the same thing as `strtok` but is passed an extra pointer to keep it from executing some odd behavior. You can read more [here](https://systems-encyclopedia.cs.illinois.edu/articles/c-strtok/).

<img src="/assets/img/sys-II-shell-lab-code-2.png" alt="Code 1" class="project-image-code">


