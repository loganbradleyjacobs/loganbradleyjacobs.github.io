---
layout: project
title: "Systems II Shell Lab (Pt 2)"
date: 2025-10-17
tags: [Operating Systems, Shell, Programming]
excerpt: "Adding onto my simple shell with a history file and arrow key functionality."
thumbnail: /assets/img/sys-II-shell-lab-2.png
---

# Recap

[When I last left off,](https://loganbradleyjacobs.github.io/2025/10/14/sys-II-shell-lab.html) I had finished implementing a simple shell, complete with `exec`ing commands, piping between processes, control character handling, background processing and more. Today I will add a history file, and enable the user to traverse through the history file with the arrow keys. I will also enable clearing the history file, and set a cap on how many commands will be stored in history.

# Specifications

According to the instructions, here is exactly what is needed.
- I need to create a hidden file called `.myhistory`.
    - When the shell reads a command from the command line, it should be appended to this file.
    - The file must hold `MAX_HISTORY` commands.
        - After this limit has been reached, the oldest entry should be erased, and the new entry can be added as normal.
- I need a `history` command which will display every command stored in the file, preceded by an index relative to the start of the file.
- I need an `erase history` command that will reset the history file by erasing every command stored within the file
- I need to have the up and down arrows traverse the history file, where an up arrow decreases the index (going backwards in time), and the down arrow increases the index (going forward in time).

# History

In my version, which is commented out in part one of this article for simplicty, I already write to a file quite often: the log file. Therefore, it is easy to retrofit the shell to have a function very similar to the log writing function, but simply do the same for the `.myhistory file`. I also added a line in my `setup()` function to initialize the history file the same as I do for the log file.

<img src="/assets/img/sys-II-shell-lab-2-writeHistory.png" alt="History Writing Function" class="project-image-code">

I added the call to the above function in the `getInput()` function, as I already write the input buffer to the log file, so it is a good spot to also write to the history file.

Now that a "history write" file is implemented, it seems to also be a good time to implement a "history read" function. Instead of reading into a variable, I believe it is okay to simply print the results to stdout, as if it's not meant to go there, it will be redirected by pipes, as it should be.

Here is the function, note the `rewind()` call to reset the file pointer to the beginning of the file, as it is normally at the end because it was opened with the `fopen()` `a+` flag for appending.

<img src="/assets/img/sys-II-shell-lab-2-printHistory.png" alt="History Printing Function" class="project-image-code">

There was a dilemma I uncovered while thinking about where to put the call of this function. I could implement `history` like I did `exit`, as a builtin executed by the parent. However, this limits the shell, as it cannot pipe the results of `history` to anywhere else. If I instead insert it when processes are being delegated, however, I can simply execute the builtin instead of allowing the execvp call to take the argument if it is equal to "history". This way, pipes retain their functionality.

Next, the `erase history` command needs to be implemented. First I'll design a function to erase the `.myhistory` file, and then I'll decide where to call it from. Originally, I had thought that the best way to do this is by opening the file in `w` mode, clearing it, and restoring the file descriptor to `a+` mode, but this created issues. Instead, I used `ftruncate()` to truncate the file to 0, effectively clearing it.

<img src="/assets/img/sys-II-shell-lab-2-eraseHistory.png" alt="History Erasing Function" class="project-image-code">

I call this function in the `delegateProcesses()` stage, and simply don't `fork()` or `exec()` in the case I recieve "erase history".

Another thing I need to do in terms of the history file is implement a maximum size for it defined as `MAX_HISTORY`. I was hoping that there would be an efficient way to remove the first line of a file, and simply do that and append when there were `MAX_HISTORY` commands in `.myhistory`, but after looking into it, it seems inefficient and a little messy. The next thing I thought of was a [circular buffer](https://www.youtube.com/watch?v=uvD9_Wdtjtw). 

When I start the shell, I can read the `.myhistory` file, and fill the circular buffer with those values. Since the `.myhistory` file will have less than MAX_HISTORY lines, if the buffer is of size MAX_HISTORY, it should fit. Because the buffer is circular, as long as I manage the head and tail pointers, the behavior of the buffer should be perfect for the behavior I want. If a new item is added in any case, I move the head pointer 1 item forward. If a new item is added, but I have no space left, the same thing happens, and it simply overwrites the oldest item. When exiting, I can write the circular buffer back to the `.myhistory` file, and all is well.

<img src="/assets/img/sys-II-shell-lab-2-addToHistoryBuffer.png" alt="History Functions" class="project-image-code">

I had to fix a few bugs, but it now works. I ended up having to test my history-related functions in a separate file, where I was just testing the adding, writing and erasing of the circular buffer. Other things were needed, like writing the whole buffer to the history file, instead of just one entry. Here are the updated history functions:

<img src="/assets/img/sys-II-shell-lab-2-historyFunctions.png" alt="History Functions" class="project-image-code">

# Arrow Key Handling

Now all that's left to do (theoretically) is managing the arrow key inputs so that they traverse through the file. However, since we have our circular buffer in memory, we don't have to actually access the file itself for this operation. Unfortunately, "managing arrow keys" is anything but simple.

Firstly, we need to shut off canonical mode, which means input won't be line buffered anymore, instead it will be available immediately. This is done with `rawTermios.c_lflag &= ~(ICANON);` in the `startRawMode()` function. Secondly, right now we get input with `fgets()` which was fine before, but now we need to handle arrow characters immediately, so this will have to change. I must implement my own fgets, and handle each character individually.

<img src="/assets/img/sys-II-shell-lab-2-getRawInput.png" alt="Get Raw Input" class="project-image">

Now this function is set to replace `getInput()`. It uses `read()` to read each character 1 by 1, and handle them accordingly. If `c` (the character last read) is a newline, then null-terminate the buffer for history-related reasons, and print a newline. If there was a command there, write it to the history file. If the character is a backspace and there's characters on the line, delete the last character. If it's printable and there's room to print it, do so, and handle the buffer. If it's an escape sequence (used for arrow keys), read the next two characters, and if the first is a `[`, then pass the second character to the handleArrows function for further processing. I'm honestly decently proud of this function, even if it's a little hard to read at first glance.

Below is my `handleArrows()` function. If the character it receives is `A`, corresponding to the up arrow, it gets the newest command if not yet browsing history, it does nothing if already at the end of history and if it's anywhere in the middle, it gets the previous command from history and writes it to the command line.

The handling for `B` (down arrow) is similar, just the opposite. Additionally, if moving out of browsing the history, I need to clear the line and reset the buffer. `C` and `D` (right and left arrows respectively) don't do anything at the moment, although I may change this in the future.

<img src="/assets/img/sys-II-shell-lab-2-handleArrows.png" alt="Get Raw Input" class="project-image">

Minus some considerations for bugs I had, this is my shell. It's not perfect, but I had fun making it, and enjoyed learning everything I needed to, to make this project a reality. My sources will be posted at the bottom of this document.

<img src="/assets/img/sys-II-shell-lab-2-handleArrows.png" alt="Get Raw Input" class="project-image">


# Sources:
- ## Input Processing
	- `strtok` w3 schools https://www.w3schools.com/c/ref_string_strtok.php
		- strtok acting weird
			- strtok_r vs strtok https://systems-encyclopedia.cs.illinois.edu/articles/c-strtok/
	- format prompt https://gist.github.com/fnky/458719343aabd01cfb17a3a4f7296797
		- hex rgb to ansi converter https://github.com/v-amorim/hex_to_ansi
	- null terminating
		- `strcspn` https://cplusplus.com/reference/cstring/strspn/
	- stripping spaces https://stackoverflow.com/questions/1726302/remove-spaces-from-a-string-in-c
	- stripping quotes
		- `memmove` https://man7.org/linux/man-pages/man3/memmove.3.html
	- logging errors
		- opening file flags https://stackoverflow.com/questions/28466715/using-open-to-create-a-file-in-c
		- using fopen instead because it works with fprintf https://man7.org/linux/man-pages/man3/fopen.3.html
		- using dprinf instead of fprintf because unbuffered https://linux.die.net/man/3/dprintf
			- weird behavior fix: https://stackoverflow.com/questions/1716296/why-does-printf-not-flush-after-the-call-unless-a-newline-is-in-the-format-strin
		- trying to modularize code
			- variadic functions https://en.cppreference.com/w/c/variadic.html
- ## Input Handling
	- exec family https://www.geeksforgeeks.org/c/exec-family-of-functions-in-c/
- ## Signal Handling
	- ignoring `^C` https://stackoverflow.com/questions/32708086/ignoring-signals-in-parent-process
	- not having `"^C"` come up after ignoring `SIGINT` https://stackoverflow.com/questions/608916/ignoring-ctrl-c
	- restoring default signal handling in children https://thelinuxcode.com/signal_handlers_c_programming_language/
	- signalPressed flag https://en.cppreference.com/w/c/program/sig_atomic_t
- ## Pipe Handling
	- compound literals https://en.cppreference.com/w/c/language/compound_literal.html
- ## Arrow Key Detection
	- Someone doing it in bash: https://stackoverflow.com/questions/79416101/bash-arrow-key-detection-in-real-time
		- `stty raw -echo` https://stackoverflow.com/questions/22832933/what-does-stty-raw-echo-do-on-os-x
			- `man stty` https://www.man7.org/linux/man-pages/man1/stty.1.html
		- `trap cleanup EXIT`
			- `trap` https://ss64.com/bash/trap.html
		- `if IFS= read -r -t 0.1 -n 3 keypress; then`
			- `IFS=`"sets the internal field separator to an empty value (use result of read in above)" "no splitting will occur when processing input strings"
			- `read` https://linuxize.com/post/bash-read/
			- `man read` https://ss64.com/bash/read.html
		- `$'\e[A') echo "Up Arrow! ;;`
			- `\e[A`
				- why does it happen https://stackoverflow.com/questions/21384040/why-does-the-terminal-show-a-b-c-d-when-pressing-the-arrow-k
				- recognizing arrow keys with stdin https://stackoverflow.com/questions/4130048/recognizing-arrow-keys-with-stdin
				- ansi escape codes (explains `\[` and `[A` are meaningful) https://en.wikipedia.org/wiki/ANSI_escape_code
	- Someone with a better solution in comments https://stackoverflow.com/questions/79416101/bash-arrow-key-detection-in-real-time
		- `test "$c" = q && break` break if `c` is equal to `"q"` 
			- `test` https://linuxhandbook.com/bash-test-command/
		- `if test "$state" = 0 -a "$c" = $'\e'; then`
			- `-a` https://linuxhandbook.com/bash-test-operators/
	- how to put terminal in raw mode in C https://www.man7.org/linux/man-pages/man3/termios.3.html
		- `termios.h` https://pubs.opengroup.org/onlinepubs/7908799/xsh/termios.h.html
	- custom fgets https://codereview.stackexchange.com/questions/202490/dynamic-fgets-in-c
	- really helpful termios guide https://viewsourcecode.org/snaptoken/kilo/02.enteringRawMode.html
	- arrow key tips https://viewsourcecode.org/snaptoken/kilo/03.rawInputAndOutput.html
- ## History
	- reading file https://www.geeksforgeeks.org/c/c-program-to-read-contents-of-whole-file/
	- history starts at 1, not 0 https://www.ibm.com/docs/en/aix/7.2.0?topic=shell-history-lists-c
	- a+ puts pointer at end, i want it at the beginning https://www.geeksforgeeks.org/c/rewind-in-c/
	- making sure i'm set on opening modes https://stackoverflow.com/questions/1466000/difference-between-modes-a-a-w-w-and-r-in-built-in-open-function
	- MAX_HISTORY with circular buffer https://www.youtube.com/watch?v=uvD9_Wdtjtw
	- lets keep the head, tail and history file desc in a struct https://www.w3schools.com/c/c_structs.php
		- i don't want to type struct out every time https://stackoverflow.com/questions/1675351/typedef-struct-vs-struct-definitions/
		- initializing struct in setup() https://stackoverflow.com/questions/32698293/assign-values-to-structure-variables
	- ftruncate instead of opening in w mode? https://www.man7.org/linux/man-pages/man3/ftruncate.3p.html