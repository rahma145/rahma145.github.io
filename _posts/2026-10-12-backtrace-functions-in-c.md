---
title: "Backtrace Functions in C"
description: "Capture the call stack with backtrace(), backtrace_symbols() and backtrace_symbols_fd(), and turn raw addresses into source lines with addr2line."
date: 2026-10-12 08:50:00 +0100
categories: [C/C++]
tags: [c, debugging, backtrace, addr2line, segfault, linux]
---

The backtrace functions are used to obtain a list of function calls (the **call stack**) that led to a certain point in a program. They are particularly useful for debugging situations such as segmentation faults, where it is important to identify which functions were called before the crash occurred in order to locate the source of the problem.

The main functions used for this purpose are `backtrace()`, `backtrace_symbols()`, and `backtrace_symbols_fd()`.

> All of them are declared in `<execinfo.h>`, which must be included in your program.
{: .prompt-info }

## The functions

### backtrace()

```c
int backtrace(void *buffer[size], int size);
```

`backtrace()` records the current call stack (the sequence of function calls that led to the current point of execution) in the array pointed to by `buffer`. The parameter `size` specifies the maximum number of stack frames to record. For example, if `size` is set to 5, the function stores up to the last five return addresses from the call stack.

Each element of `buffer` is a `void *`, a generic pointer to a memory address. Each of these addresses is a **return address**, i.e. the point in the program where execution continues after the current function returns.

The function returns the actual number of addresses stored, which may be less than `size` if the stack is shallower.

### backtrace_symbols()

```c
char **backtrace_symbols(void *const buffer[], int size);
```

`backtrace_symbols()` converts the raw memory addresses obtained from `backtrace()` into strings that describe each stack frame. Each string typically includes:

- the function name (if available),
- the offset within that function,
- the actual return address.

It returns an array of string pointers that is dynamically allocated (using `malloc()` internally). It is therefore the programmer's responsibility to `free()` this memory after use to avoid memory leaks.

### backtrace_symbols_fd()

```c
void backtrace_symbols_fd(void *const buffer[size], int size, int fd);
```

Instead of returning an array of strings, this function writes the strings to the file descriptor `fd`, for example standard error (`STDERR_FILENO`) or a log file.

> Static functions are **not** explored by the backtrace: they are not exported as symbols, so they show up only as raw addresses.
{: .prompt-warning }

## Example 1: printing the call stack

This program demonstrates how to print the current call stack. The function `func2()` creates an array to store up to 10 return addresses from the active stack frames, collected with `backtrace()`. These addresses are then converted into symbols and written directly to the standard error output using `backtrace_symbols_fd()`. Execution flows from `main()` to `func1()` and then to `func2()`, so the backtrace shows the sequence of calls that led to the point where `backtrace()` is invoked.

```c
#include <execinfo.h>
#include <unistd.h>

void func2(void)
{
    void *array[10]; // array to store return addresses

    /* read the size of the backtrace (number of functions called) */
    int size = backtrace(array, 10);

    /* convert addresses to symbols and redirect them to stderr */
    backtrace_symbols_fd(array, size, STDERR_FILENO);
}

void func1(void)
{
    func2();
}

int main()
{
    func1();

    return 0;
}
```

When executing this program we get this output:

```text
./example1(+0x11b5)[0x56b507c671b5]
./example1(+0x11f2)[0x56b507c671f2]
./example1(+0x1202)[0x56b507c67202]
/lib/x86_64-linux-gnu/libc.so.6(+0x29d90)[0x784c67a29d90]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x80)[0x784c67a29e40]
./example1(+0x10c5)[0x56b507c670c5]
```

The output shows the call stack at the point where `backtrace()` was called. Each line is a stack frame and includes the binary name, an offset, and the actual memory address of the return point. For example, `./example1(+0x11b5)[0x56b507c671b5]` is a return address at offset `0x11b5` in the `example1` binary.

This identifies the sequence of function calls, but raw addresses and offsets are **not immediately readable**. This is where `addr2line` comes in: it translates addresses into readable source locations (file names and line numbers).

> To make the addresses in the backtrace map consistently to locations in the source, compile with the `-no-pie` option.
>
> By default, Linux toolchains produce **Position-Independent Executables (PIE)** to improve security. The addresses of functions then change at each execution, which makes it difficult to map backtrace addresses reliably with tools like `addr2line`. Compiling with `-no-pie` produces a fixed-address executable, so backtrace addresses can be translated accurately into file names and line numbers.
{: .prompt-tip }

## Getting readable output

To get readable information from a backtrace, follow these steps.

**1. Compile with `-g` and `-no-pie`.** `-g` adds debug information to the binary, and `-no-pie` gives functions static addresses.

```bash
gcc -g example1.c -no-pie -o example1
```

**2. Run the program.** You get something like:

```text
./example1[0x4011a2]
./example1[0x4011df]
./example1[0x4011ef]
/lib/x86_64-linux-gnu/libc.so.6(+0x29d90)[0x73d025e29d90]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x80)[0x73d025e29e40]
./example1[0x4010b5]
```

**3. Use `addr2line` to find the matching line in your code.**

```bash
addr2line -e example1 0x4011a2
```

## Example 2: backtrace on a segmentation fault

Here we use `backtrace()` in a situation involving a segmentation fault. The idea is to install a signal handler for `SIGSEGV` and, inside it, call `backtrace()` to capture the call stack at the moment of the crash. To simulate the fault, we dereference a null pointer.

```c
#include <execinfo.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/* Signal handler to catch segmentation faults */
void handler(int sig) {
    void *array[10];
    size_t size;

    // Get the array of return addresses
    size = backtrace(array, 10);

    // Print error message
    fprintf(stderr, "\nError: signal %d (segmentation fault)\n", sig);

    // Print the backtrace
    backtrace_symbols_fd(array, size, STDERR_FILENO);

    // Exit after printing
    exit(1);
}

void crash() {
    int *ptr = NULL;    // Null pointer
    *ptr = 42;          // This will trigger the segmentation fault
}

void func2() { crash(); }
void func1() { func2(); }

int main() {
    // Install signal handler for SIGSEGV signal
    signal(SIGSEGV, handler);

    func1();

    return 0;
}
```

The output:

```text
Error: signal 11 (segmentation fault)
./example2[0x4011e5]
/lib/x86_64-linux-gnu/libc.so.6(+0x42520)[0x7a4a7de42520]
./example2[0x401243]
./example2[0x40125e]
./example2[0x401273]
./example2[0x40129c]
/lib/x86_64-linux-gnu/libc.so.6(+0x29d90)[0x7a4a7de29d90]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x80)[0x7a4a7de29e40]
./example2[0x4010f5]
```

`0x4011e5` is the most recent frame, i.e. the handler where `backtrace()` was executed. The next application address is `0x401243`, so we look it up with `addr2line`:

```bash
addr2line -e example2 0x401243
```

```text
/Bureau/test_backtrace/example2.c:27
```

Line 27 of the source is:

```c
*ptr = 42; // This will trigger the segmentation fault
```

So this address corresponds to line 27, exactly where the segmentation fault occurred and where the handler was triggered.

## Takeaways

| Step | Tool | Purpose |
|------|------|---------|
| Capture the stack | `backtrace()` | Fill an array with return addresses |
| Print it | `backtrace_symbols_fd()` | Write symbolic frames to a file descriptor |
| Or get strings | `backtrace_symbols()` | Returns `malloc`'d strings, remember to `free()` |
| Build for debugging | `gcc -g -no-pie` | Debug info and fixed addresses |
| Map to source | `addr2line -e bin 0xADDR` | File name and line number |
