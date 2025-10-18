---
layout: project
title: "Systems II Shell Lab (Pt 1)"
date: 2025-10-14
tags: [Operating Systems, Shell, School]
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
command [identifier[identifier]]
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

Now though, we have input from the user, but we need to parse it into something useful. This can be done by a string tokenization function called `strtok`. However, we need to tokenize by 2 delimiters: first by separating the commands from each other (delimited by `|`), and second separating the tokens from each other within each command (delimited by a space). With this pattern, it is probably easier to use `strtok_r`, another niche tokenization function that does the same thing as `strtok` but is passed an extra pointer to keep it from executing some odd behavior. You can read more [here](https://systems-encyclopedia.cs.illinois.edu/articles/c-strtok/).

<img src="/assets/img/sys-II-shell-lab-code-2.png" alt="Code 2" class="project-image-code">

Great! So now we have a prompt, and tokenized input. The only thing to do now is say what we're going to do with the tokenized input. We tokenized it like this:
```
[token from cmd 1][token from cmd 1]...[NULL]
[token from cmd 2][NULL][NULL]...[NULL]
[token from cmd 3][token from cmd 3]...[NULL]
```
We have a 3x20 array that holds 20 tokens in each row, designating a command, and 3 rows, because we have only a maximum of 3 commands to worry about. If we had input `ls | grep "test" | wc` then we would have parsed:
```
[ls][NULL]...[NULL]
[grep]["test"][NULL]...[NULL]
[wc][NULL]...[NULL]
```
Note that `"test"` would actually be parsed as `test` due to the `stripQuotes()` function call. In simple terms, we need to fork a child process for each command, and then `execvp()` that command, passing the tokens as arguments. We can name the top level function that just takes in the tokenizedInput `delegateProcesses()`. This function will have to do a lot of heavy lifting, so let's modularize our work. First, we can check if the process is a background process (ending with `&`), so that can be its own function: `checkBackground()`. Then, if appropriate, we will have to create the pipes that connect the input and output of commands, using `createPipes()`. Those pipes are useless though, until we create the processes that will run each command using `createProcesses()`. The pipes are still not operational, they need to be "hooked up" to each process in the right way, let's do this in `redirectIO()`. The processes aren't doing anything yet, they need to actually `exec()` the commands, so let's make them do that with `execCommand()`. Lastly we need to close all the pipes in the parent, so that undefined behavior doesn't occur. We can do this in `closeAllPipes().` Lastly, we need to wait for each child process in the parent process, but I didn't decide to do this inside a function, I just call a for loop that calls `waitpid()` for each process. This all is done in the code that follows:

<img src="/assets/img/sys-II-shell-lab-code-3.png" alt="Code 3" class="project-image-code">

The last thing to do after this is ensure a user can't `Ctrl-C` out of the program, and must use `exit` from the command line to leave. An acceptable way to do this is to set up a signal handler that does nothing, thus ignoring the SIG_INT signal from Ctrl-C, but I went a little overboard and tried to emulate the behavior I see from my own terminal: going to the next line, and spawning a new prompt. To do this, I needed to stop `^C` from showing up on the line when I hit those keys. The way I did this is to save the terminal's settings, turn off the terminal's `ECHOCTL` (echo control characters) setting, and restore the terminal's settings when I was done. I did this with the `startRawMode()`, `endRawMode()` and `setup()` helper functions.

Lastly, without `^C` to exit the program, we would need to handle our alternative: typing "exit" in our shell. I did this with a `checkExit()` function.

Below is the final code for these specifications of the lab. There are optional pieces to the assignment I haven't introduced yet, such as a working `history` command, and using the up and down arrow keys to control that command.

<img src="/assets/img/sys-II-shell-lab-code-4.png" alt="Code 4" class="project-image-code">