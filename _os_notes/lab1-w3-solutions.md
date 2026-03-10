---
layout: page
title: "Lab1-w3: handshake, sniffer & monitor"
short_title: "W3: handshake, sniffer & monitor"
lab: lab1
week: 3
order: 3
date: 2026-01-22
---

<div class="info-box">
📦 <strong>Lab zip:</strong> xv6labs-w3 &nbsp;|&nbsp; 📊 <strong>Score:</strong> 65/65 &nbsp;|&nbsp; 📅 <strong>Due:</strong> Week 3
</div>

## Task 1 — `handshake`

### What you need to do

Create two processes that exchange a byte over two pipes. Parent sends to child; child echoes back; both print a confirmation message.

**File:** `user/handshake.c`

### Pipe layout

```
parent → p2c[1]  ==pipe==  p2c[0] → child
child  → c2p[1]  ==pipe==  c2p[0] → parent
```

### Solution

```c
// user/handshake.c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    if (argc != 2) {
        fprintf(2, "Usage: handshake <byte>\n");
        exit(1);
    }

    char send_byte = (char)atoi(argv[1]);
    char recv_byte;

    int p2c[2];   // parent-to-child pipe
    int c2p[2];   // child-to-parent pipe

    pipe(p2c);
    pipe(c2p);

    int pid = fork();
    if (pid == 0) {
        // ── Child ──
        close(p2c[1]);   // child only reads from p2c
        close(c2p[0]);   // child only writes to c2p

        read(p2c[0], &recv_byte, 1);
        printf("%d: received %d from parent\n", getpid(), (int)(unsigned char)recv_byte);

        write(c2p[1], &recv_byte, 1);

        close(p2c[0]);
        close(c2p[1]);
        exit(0);
    } else {
        // ── Parent ──
        close(p2c[0]);   // parent only writes to p2c
        close(c2p[1]);   // parent only reads from c2p

        write(p2c[1], &send_byte, 1);

        read(c2p[0], &recv_byte, 1);
        printf("%d: received %d from child\n", getpid(), (int)(unsigned char)recv_byte);

        close(p2c[1]);
        close(c2p[0]);
        wait(0);
        exit(0);
    }
}
```

### Why close the unused ends?

If the parent keeps `p2c[0]` open, the child's `read()` on `p2c[0]` will block forever when the parent closes its write end, because there is still a live read-end — the OS thinks more writers exist. Always close every pipe end you don't use.

---

## Task 2 — `sniffer`

### Background: the deliberate bug

For this lab, xv6 has `memset` removed from `uvmalloc()` (in `vm.c`) and from `kalloc()`. This means newly-allocated memory pages still contain data from the previous process that used them.

When `secret` runs, it writes a secret string into its heap/stack and exits, freeing its memory. `sniffer` then allocates memory — and gets those same physical pages back, still containing the secret.

### Strategy

1. Call `sbrk()` in a loop to allocate large chunks of memory
2. Scan each chunk for a non-null string that looks like the secret
3. Print it

### Solution

```c
// user/sniffer.c
#include "kernel/types.h"
#include "user/user.h"

#define SCAN_SIZE (1024 * 1024)   // 1 MB — more than enough

int
main(void)
{
    // Allocate a big chunk; pages come from freed memory of previous processes
    char *mem = sbrk(SCAN_SIZE);
    if (mem == (char *)-1) {
        fprintf(2, "sniffer: sbrk failed\n");
        exit(1);
    }

    // Scan for a printable null-terminated string
    // secret.c writes the secret starting at a known-ish location
    for (int i = 0; i < SCAN_SIZE - 1; i++) {
        // Look for a sequence of printable chars followed by '\0'
        if (mem[i] >= 32 && mem[i] < 127) {
            int len = 0;
            while (i + len < SCAN_SIZE && mem[i + len] >= 32 && mem[i + len] < 127)
                len++;
            if (len > 0 && i + len < SCAN_SIZE && mem[i + len] == '\0') {
                printf("%s\n", &mem[i]);
                exit(0);
            }
        }
    }

    fprintf(2, "sniffer: secret not found\n");
    exit(1);
}
```

### How `secret.c` stores the secret

Looking at `user/secret.c`, it copies `argv[1]` into a local buffer (on the stack) and into a heap allocation. When it exits, those pages are freed. `sniffer` allocates memory right after — the kernel's free list is LIFO-ish so it hands back the most-recently-freed pages.

### Grader expectation

The grader runs `secret <secret_string>` then immediately runs `sniffer`. Your `sniffer` must print exactly the secret string. The simple scan above works because the secret is a printable ASCII string.

---

## Task 3 — `monitor` (Syscall Tracing)

### Overview

Add a `monitor(mask)` syscall that enables tracing for the calling process (and its children). When a monitored syscall returns, print: `<pid>: syscall <name> -> <retval>`

### All files to modify

| File | Change |
|------|--------|
| `kernel/syscall.h` | Add `#define SYS_monitor 22` |
| `kernel/proc.h` | Add `uint32 monitor_mask;` to `struct proc` |
| `kernel/sysproc.c` | Implement `sys_monitor()` |
| `kernel/syscall.c` | Add extern, table entry, names array, print logic |
| `kernel/proc.c` | Copy mask in `kfork()` |
| `user/usys.pl` | `entry("monitor");` |
| `user/user.h` | `int monitor(int);` |
| `Makefile` | `$U/_monitor\` |

### `kernel/syscall.h` — add syscall number

```c
#define SYS_interpose  22
#define SYS_monitor    23   // add after existing defines
```

> **Note:** Check your lab's `syscall.h` — the number must be the next available one.

### `kernel/proc.h` — add mask field

```c
struct proc {
  // ... existing fields ...
  uint32 monitor_mask;   // bitmask of syscalls to trace
};
```

### `kernel/sysproc.c` — syscall handler

```c
uint64
sys_monitor(void)
{
    int mask;
    argint(0, &mask);
    struct proc *p = myproc();
    p->monitor_mask = (uint32)mask;
    return 0;
}
```

### `kernel/syscall.c` — names array + print logic

Add the `extern` and table entry:

```c
extern uint64 sys_monitor(void);

// in syscalls[]:
[SYS_monitor] sys_monitor,
```

Add the names array (inside or above `syscall()`):

```c
static const char *syscall_names[] = {
    [SYS_fork]      "fork",
    [SYS_exit]      "exit",
    [SYS_wait]      "wait",
    [SYS_pipe]      "pipe",
    [SYS_read]      "read",
    [SYS_kill]      "kill",
    [SYS_exec]      "exec",
    [SYS_fstat]     "fstat",
    [SYS_chdir]     "chdir",
    [SYS_dup]       "dup",
    [SYS_getpid]    "getpid",
    [SYS_sbrk]      "sbrk",
    [SYS_pause]     "pause",
    [SYS_uptime]    "uptime",
    [SYS_open]      "open",
    [SYS_write]     "write",
    [SYS_mknod]     "mknod",
    [SYS_unlink]    "unlink",
    [SYS_link]      "link",
    [SYS_mkdir]     "mkdir",
    [SYS_close]     "close",
    [SYS_monitor]   "monitor",
};
```

Modify the `syscall()` function to print when the bit is set:

```c
void
syscall(void)
{
    int num;
    struct proc *p = myproc();

    num = p->trapframe->a7;
    if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        p->trapframe->a0 = syscalls[num]();

        // ── monitoring output ──
        if ((p->monitor_mask >> num) & 1) {
            printf("%d: syscall %s -> %d\n",
                   p->pid,
                   (num < NELEM(syscall_names) && syscall_names[num])
                       ? syscall_names[num] : "?",
                   p->trapframe->a0);
        }
    } else {
        printf("%d %s: unknown sys call %d\n", p->pid, p->name, num);
        p->trapframe->a0 = -1;
    }
}
```

### `kernel/proc.c` — inherit mask on fork

In `kfork()`, after the line that sets `np->pid`:

```c
np->monitor_mask = p->monitor_mask;
```

### User program — `user/monitor.c`

```c
// user/monitor.c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    if (argc < 3) {
        fprintf(2, "Usage: monitor <mask> <command> [args...]\n");
        exit(1);
    }

    int mask = atoi(argv[1]);
    monitor(mask);

    exec(argv[2], &argv[2]);
    fprintf(2, "monitor: exec %s failed\n", argv[2]);
    exit(1);
}
```

---

## Variation Questions

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

Why does `handshake` need **two** pipes instead of one? What would happen if you used a single pipe for both directions?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

A single pipe is **unidirectional** — data flows only one way (from writer to reader). If parent and child both wrote to and read from the same pipe:

- The parent might read back what it just wrote instead of the child's reply
- Both processes could end up blocked waiting to read data that hasn't been written yet (deadlock)

With two pipes: `p2c` carries data from parent to child; `c2p` carries data from child back to parent. Each pipe has a dedicated direction, so no confusion is possible.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-med">Medium</span></div>

In `sniffer`, why does allocating memory with `sbrk()` give you pages that still contain the secret's data? Explain in terms of the kernel's memory allocator and the deliberate bug in this lab.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

Normally, `kalloc()` in `kernel/kalloc.c` calls `memset(mem, 5, PGSIZE)` before returning a page, filling it with garbage byte `0x05`. This prevents old data from leaking to a new process.

In this lab, that `memset` call is removed (`#ifndef LAB_SYSCALL`). So when `secret` frees its pages (on exit), those pages go back onto the kernel's free list **still containing the secret data**.

When `sniffer` calls `sbrk()`, the kernel calls `kalloc()` to get a fresh page — and gets one of those recently-freed pages. Since `memset` was skipped, the physical memory still holds `secret`'s data. `sniffer` can now read it.

This demonstrates why **zeroing freed memory** is a critical security measure: without it, any process can steal sensitive data from previously-run processes.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-med">Medium</span></div>

Modify `monitor` to also print the **system call arguments** for `SYS_write`. The `write` syscall takes `(int fd, char *buf, int n)`. Print: `<pid>: syscall write(fd=<fd>, n=<n>) -> <retval>`

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

You need to read the arguments from the trapframe **before** the syscall executes (because `a0` gets overwritten with the return value). Capture them in `syscall()`:

```c
// Before calling syscalls[num]():
int arg0 = p->trapframe->a0;   // first argument
// int arg1 = p->trapframe->a1; // second (pointer — unsafe to dereference in kernel)
int arg2 = p->trapframe->a2;   // third argument

p->trapframe->a0 = syscalls[num]();

if ((p->monitor_mask >> num) & 1) {
    if (num == SYS_write) {
        printf("%d: syscall write(fd=%d, n=%d) -> %d\n",
               p->pid, arg0, arg2, p->trapframe->a0);
    } else {
        printf("%d: syscall %s -> %d\n", p->pid, syscall_names[num], p->trapframe->a0);
    }
}
```

Note: argument 1 (the buffer pointer) is a user-space address — you'd need `copyinstr()` or `copyin()` to safely read it from kernel context, which is why we skip it here.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-hard">Hard</span></div>

If `monitor` is called from a shell script that uses `exec` to replace itself with a new process, does the monitor mask persist? Explain what happens to `monitor_mask` when `exec` is called.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

**No, the mask does not persist across `exec`.**

When `exec` loads a new program, it calls `proc_freepagetable()` to free the old address space, then builds a fresh one. However, `exec` does **not** reset the `proc` struct's `monitor_mask` field — it only replaces the user-space memory.

So actually: **the mask does persist** because `monitor_mask` lives in `struct proc` (kernel memory), not in user space. `exec` doesn't zero out the proc struct fields.

This means if you do:
```sh
monitor 32 sh        # sets mask=32 on the shell process
# inside the new sh:
exec grep ...        # exec replaces sh's user space
                     # BUT monitor_mask=32 is still set
```

The `grep` process inherits the mask from the shell because it's the same proc-table entry. This is actually the **intended behavior** for the fourth test case in the lab.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-hard">Hard</span></div>

Implement a `monitorlog` system call (similar to `monitor`) that writes trace output to a kernel ring buffer of 64 entries instead of immediately printing. Add a second `monitorread` system call that lets a user process read and clear the ring buffer.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

**Ring buffer in `kernel/monitor_log.c`:**

```c
#define LOG_SIZE 64
struct log_entry {
    int pid;
    int syscall_num;
    int retval;
} log_buf[LOG_SIZE];
int log_head = 0, log_tail = 0, log_count = 0;
struct spinlock log_lock;
```

**Append in `syscall()`:**
```c
if (log_count < LOG_SIZE) {
    log_buf[log_tail] = (struct log_entry){ p->pid, num, (int)p->trapframe->a0 };
    log_tail = (log_tail + 1) % LOG_SIZE;
    log_count++;
}
```

**`sys_monitorread()`** copies entries out to user space via `copyout()`:
```c
uint64
sys_monitorread(void)
{
    uint64 addr;
    int n;
    argaddr(0, &addr);
    argint(1, &n);
    // copyout min(n, log_count) entries, reset log_count
}
```

This is the pattern used by real kernel event buffers (e.g. Linux's `ftrace` and `perf`).

</div>
</div>
