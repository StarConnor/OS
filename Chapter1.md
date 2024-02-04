# Operating system interfaces

- `kernel`: provides services to running programs
- `process`: each running program consists of memory containing instructions, data, and a stack
- `system call`: when a `process` needs to invoke a kernel service, it invokes a `system call`
  - a `system call` enter the `kernel`
  - the `kernel` performs the service and returns
  - THUS: a `process` alternates between executing in `user space` and `kernel space`
- `hardware protection mechanisms`:
  - the `kernel` executes with the hardware privileges required to implement these protections
  - `user programs` execute without the privileges

![1706843164906](image/Chapter1/1706843164906.png)

## 1.1 Processes and memory

**In xv6 os**:

- an process

  - user-space memory(instructions, data, and stack)
  - per-process state private to the kernel
- `xv6` time-shares the processes:

  - transparently switches the available CPUs among the set of processes waiting to execute
- `PID`: a process identifier associated with each process
- `fork`: create a new process by a process

  - returns in both the original and new processes
    - original: return new process's `PID`
    - new: return `zero`

  ![1706844161746](image/Chapter1/1706844161746.png)
- `exit`: release memory and open files, takes an `int`, 0 to succeed and 1 to fail
- `wait`: return the `PID` of an exited child of the current process and copies the exit status of the child to the address passed to `wait`

  - if has no children, immediately returns -1
- `exec`: load a file which has a particular format: ELF

  - when `exec` succeeds, execute at the entry
  - args: name of the executable, an array of string arguments
  - ```c
    char *argv[3];
    argv[0] = "echo";
    argv[1] = "hello";
    argv[2] = 0;
    exec("/bin/echo", argv);
    printf("exec error\n");
    ```
## I/O and File descriptors
`file descriptor` a small integer representing a kernel-managed object that a process may read from or write to.

- Obtain by: opening a file, directory, or device, creating a pipe, duplicating an existing descriptor

- `file descriptor` is an index into a per-process table

- Default: a process reads from `file descriptor 0`(standard input) and writes output to `file descriptor 1`(standard output) and writes error messages to `file descriptor 2`

- `read` and `write`: from open files named by `file descriptors`
  - `read(fd, buf, n)`: reads at most `n` bytes from `fd`, copies them to `buf`, returns the number of bytes read
    - each `fd` has an offset associated with the file
    - reads data from current file offset and then advances that offset by teh number of bytes read
  - `write(fd, buf, n)`: writes n bytes from `buf` to the `fd` and return the number of bytes wirtten.

example:
```c
char buf[512];
int n;

for (;;){
  n = read(0, buf, sizeof buf);
  if (n == 0)
    break;
  if (n < 0){
    fprintf(2, "read error\n");
    exit(1);
  }
  if (write(1, buf, n) != n){
    fprintf(2, "write error\n");
    exit(1);
  }
}
```

- `close` system call: releases a `fd`
- a newly allocated `fd` is always the lowest-numbered unused `fd` of current process

new version of `cat <input.txt`:
```c
char *argv[2];
argv[0] = "cat";
argv[1] = 0;
if(fork() == 0){
  close(0);
  open("input.txt", O_RDONLY);
  exec("cat", argv);
}
```
- `open`'s second agument: 
  - defined in file control(`fcntl`) header
  - include `O_RDONLY, O_WRONLY, ORDWR`

- child and parent process share the offset of `fd`
  ```c
  if(fork() == 0) {
    write(1, "hello ", 6);
    exit(0);
  } else {
    wait(0);
    write(1, "world\n", 6);
  }
  ```
- `dup`: duplicates an existing `fd`, return a new one 
  ```c
  fd = dup(1);
  write(1, "hello ", 6);
  write(fd, "world\n", 6);
  ```
  - Usage: help construct the command like `ls existing-file non-existing-file > tmp1 2>&1

## 1.3 Pipes

- a pair of `fd`, one for reading and one for writing
- *example*:
  ```c
  int p[2];
  char *argv[2];

  argv[0] = "wc";
  argv[1] = 0;

  pipe(p);
  if (fork() == 0){
    close(0);
    dup(p[0]);
    close(p[0]);
    close(p[1]);
    exec("/bin/wc", argv);
  } else {
    close(p[0]);
    write(p[1], "hello world\n", 12);
    close(p[1]);
  }
  ```