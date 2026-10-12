---
title: "Unix Processes in C: Understanding FIFOs (named pipes)"
description: "Why and how to use FIFOs for communication between unrelated processes: mkfifo, open flags, blocking behavior and O_NONBLOCK."
date: 2026-10-12 08:30:00 +0100
categories: [C/C++]
tags: [linux, c, fifo, named-pipes, ipc, mkfifo, unix]
---

*Realised by: Rahma Rahali*

## Motivation: why use FIFOs?

In Linux, unnamed pipes are useful for communication between related processes (parent-child), but if we need communication between **unrelated processes**, that's where FIFOs (First In, First Out), or **named pipes**, are used.

## Introduction

FIFOs are a type of IPC mechanism. They are represented as a special file in the filesystem, allowing any processes (that have access to the file) to exchange data. A FIFO has 2 ends, one for reading data and the other for writing, so it is a **unidirectional** communication mechanism.

FIFOs are blocking, which means:

- **Opening the read end** will block until the write end is opened by another process.
- Similarly, **writing** to a pipe/FIFO will block until there is a reader.

However, blocking behavior also introduces the **risk of deadlocks**: a situation where each process waits on the other to proceed, and neither can continue. Proper coordination is required to avoid such pitfalls.

## How FIFOs work

To work with the FIFO mechanism, we have to create a FIFO file. This file can be created using the `mkfifo` command or the `mkfifo()` system call in C.

### The mkfifo command

```bash
mkfifo [OPTION]... NAME...
```

- **NAME** is the name of the FIFO to be created.
- Among the available **options**, you can specify the FIFO's permissions using the `-m MODE` option.

### The mkfifo() system call

```c
int mkfifo(const char *pathname, mode_t mode);
```

- **pathname** is the name of the file to be created (or its path).
- **mode** specifies the permissions (modified by `mode & ~umask`).

`mkfifo` returns 0 (success of file creation) or -1 (file creation failed).

A FIFO can be opened for reading or writing like any regular file (using `open`/`read`), but you must respect the **directionality** of the FIFO.

### Opening a FIFO

To access a FIFO, use the `open()` system call:

```c
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

- **pathname**: the path of the FIFO file.
- **flags**: specifies how the file should be opened:
  - **O_RDONLY**: open the FIFO for **read-only** access.
  - **O_WRONLY**: open the FIFO for **write-only** access.
  - **O_RDWR**: open the FIFO for **both read and write**.
- **mode**: used when `O_CREAT` is specified, to define the permissions of the new file.

> Opening the read or write end of a FIFO blocks the communication until the other end is also opened (by another process or thread). If we open a FIFO for writing, the `open` call hangs until another process comes and opens the same FIFO for reading, then the process that opened it for writing continues. The two ends of the FIFO have to be opened.
{: .prompt-warning }

#### Example: a FIFO producer

```c
#include <stdio.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>

int main()
{
    if (mkfifo("fifo_file", 0777) == -1)
    {
        if (errno != EEXIST)
        {
            perror("Could not create fifo file");
            return 1;
        }
    }

    printf("Opening...\n");
    int fd = open("fifo_file", O_WRONLY); // The code hangs here if there is no reading process
    printf("Opened \n"); // Once a reading process opens the file, this line will be executed

    int x = 'Z';
    if (write(fd, &x, sizeof(x)) == -1)
    {
        perror("Write failed");
        return 2;
    }

    printf("Written \n");
    close(fd);
    printf("Closed \n");

    return 0;
}
```

1. **Run the program.** It is blocked at the step of opening, waiting for the presence of a reading process:

   ```bash
   ./fifo_producer
   ```

2. **In another terminal**, run the following command (or any other command or program that reads the FIFO file). The code will then continue executing:

   ```bash
   cat fifo_file
   ```

3. **Result** in the first terminal:

   ```text
   Opening...
   Opened
   Written
   Closed
   ```

To avoid this blocking behavior, we can use the `O_NONBLOCK` flag when opening the FIFO:

```c
int fd = open("myfifo1", O_WRONLY | O_NONBLOCK);
```
