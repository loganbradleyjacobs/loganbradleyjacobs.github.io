---
layout: project
title: "Systems II Shell Lab (Pt 2)"
date: 2025-10-17
tags: [Operating Systems, Shell, School]
excerpt: "Adding onto my simple shell with a history file and arrow key functionality."
thumbnail: /assets/img/filler.png
---

[When I last left off,](https://loganbradleyjacobs.github.io/2025/10/14/sys-II-shell-lab.html) I had finished implementing a simple shell, complete with `exec`ing commands, piping between processes, control character handling, background processing and more. Today I will add a history file, and enable the user to traverse through the history file with the arrow keys. I will also enable clearing the history file, and set a cap on how many commands will be stored in history.

According to the instructions, here is exactly what is needed.
- I need to create a hidden file called `.myhistory`.
    - When the shell reads a command from the command line, it should be appended to this file.
    - The file must hold `MAX_HISTORY` commands.
        - After this limit has been reached, the oldest entry should be erased, and the new entry can be added as normal.
- I need a `history` command which will display every command stored in the file, preceded by an index relative to the start of the file.
- I need an `erase history` command that will reset the history file by erasing every command stored within the file
- I need to have the up and down arrows traverse the history file, where an up arrow decreases the index (going backwards in time), and the down arrow increases the index (going forward in time).

In my version, which is commented out in part one of this article for simplicty, I already write to a file quite often: the log file. Therefore, it is easy to retrofit the shell to have a function very similar to the log writing function, but simply do the same for the `.myhistory file`. I also added a line in my `setup()` function to initialize the history file the same as I do for the log file.

<img src="/assets/img/sys-II-shell-lab-2-writeHistory.png" alt="History Writing Function" class="project-image-code">

I added the call to the above function in the `getInput()` function, as I already write the input buffer to the log file, so it is a good spot to also write to the history file.

Now that a "history write" file is implemented, it seems to also be a good time to implement a "history read" function. Instead of reading into a variable, I believe it is okay to simply print the results to stdout, as if it's not meant to go there, it will be redirected by pipes, as it should be.

Here is the function, note the `rewind()` call to reset the file pointer to the beginning of the file, as it is normally at the end because it was opened with the `fopen()` `a+` flag for appending.

<img src="/assets/img/sys-II-shell-lab-2-printHistory.png" alt="History Printing Function" class="project-image-code">

There was a dilemma I uncovered while thinking about where to put the call of this function. I could implement `history` like I did `exit`, as a builtin executed by the parent. However, this limits the shell, as it cannot pipe the results of `history` to anywhere else. If I instead insert it when processes are being delegated, however, I can simply execute the builtin instead of allowing the execvp call to take the argument if it is equal to "history". This way, pipes retain their functionality.

Next, the `erase history` command needs to be implemented. First I'll design a function to erase the `.myhistory` file, and then I'll decide where to call it from. Opening the file in write mode (with `fopen` and `w`) clears the content of the file automatically. The file descriptor is in `a+` mode under normal circumstances, so to erase the contents, we can close it, open it again in `w` mode, close it again, and open it back in `a+` mode.

<img src="/assets/img/sys-II-shell-lab-2-eraseHistory.png" alt="History Erasing Function" class="project-image-code">

I call this function in the `delegateProcesses()` stage, and simply don't `fork()` or `exec()` in the case I recieve "erase history".