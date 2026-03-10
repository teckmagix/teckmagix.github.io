---
layout: page
title: "Lab Predictions - W3X9F"
lab: lab1
description: "Exam predictions for Lab1-w3: handshake, sniffer, monitor."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

In `handshake.c`, why must both pipes be created **before** calling `fork()`?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

Because `fork()` copies the parent's entire file descriptor table to the child. If the pipes were created after `fork()`, each process would have its own private pair of pipes — they would not share the same kernel buffers, so communication would be impossible. Creating them before `fork()` means both parent and child end up holding file descriptors that point to the **same** kernel pipe buffers.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What does `fd[0]` and `fd[1]` represent in a pipe? Which end does the parent use to send a byte to the child?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

- `fd[0]` = **read end** — you pull data out of the pipe here.
- `fd[1]` = **write end** — you push data into the pipe here.

To send a byte **to the child**, the parent writes to `p2c[1]` (the write end of the parent-to-child pipe). The child reads from `p2c[0]`.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What happens when a process calls `read()` on a pipe that is currently empty but still has open write ends?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

The process **blocks** (sleeps) until another process writes data into the pipe. This is the automatic synchronisation property of pipes — no explicit lock or semaphore is needed. The kernel will wake the blocked reader as soon as bytes arrive.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does `monitor(1 << SYS_read)` do? What integer value does this evaluate to if `SYS_read = 5`?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

It sets bit 5 of the monitoring mask, telling the kernel to print a trace line every time the `read` syscall returns for this process (and any children it forks). `1 << 5 = 32`. So `monitor(32)` enables tracing for `read` only.

The kernel checks: `if (p->monitor_mask & (1 << syscall_num))` — if that bit is set, it prints `"PID: syscall NAME -> RETURN"`.

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Why does `handshake.c` use `argv[1][0]` instead of `atoi(argv[1])` to get the byte to send?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

xv6 compiles with `-Werror=unused-variable`. If you wrote `char byte = atoi(argv[1])` before the `fork()`, the child branch never uses `byte`, triggering a compiler error. Using `argv[1][0]` reads the first character directly from the argument string — no intermediate variable is needed, avoiding the compiler warning entirely.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

Explain the "LIFO free page list" behaviour in xv6 and why `sbrk(8*4096)` in `sniffer.c` reliably returns the same pages that `secret.c` just freed.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

xv6's `kalloc.c` maintains a **free list as a stack (LIFO)**. When `secret` exits, its 8 pages are pushed onto the top of the free list. When `sniffer` immediately calls `sbrk(8*4096)`, the kernel pops 8 pages from the top — the exact same pages `secret` just freed. Because the three `memset` zeroing calls were removed for this lab, those pages still contain `secret`'s old data, including the marker string and the secret itself.

If instead xv6 used a FIFO free list, the pages could come from anywhere and the strategy would fail.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Why does `monitor_mask` survive across an `exec()` call, even though `exec()` replaces the process's code, data, and stack?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

`exec()` replaces the **user-space memory** (text, data, stack, heap) but does **not** destroy the kernel's `struct proc` for that process. `monitor_mask` lives inside `struct proc` (kernel memory), not in user space. Since `struct proc` survives `exec()`, the monitoring mask persists and tracing continues even after the program image is replaced.

This is the same reason the process retains its PID and open file descriptors after `exec()`.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

Why does a simple ASCII scan (looking for the first long printable string) fail to find the secret in `sniffer`? What is the correct approach?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

`secret.c` writes `"This may help."` at `data[0]` (offset 0) and the secret at `data[16]` (offset 16). An ASCII scanner scanning left-to-right finds `"This may help."` first — 14 printable characters — and prints that instead of the secret.

The correct approach is to scan the allocated memory for the **exact marker string** `"This may help."`. Once found at position `i`, the secret is at `i + 16` (right after the null terminator of the marker). Use `strcmp` or `memcmp` to find the marker, then `printf("%s\n", ptr + i + 16)`.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-med">Medium</span></div>

What changes must be made to `kfork()` in `kernel/proc.c` for the monitor syscall, and why?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

Add one line after allocating the child process `np`:

```c
np->monitor_mask = p->monitor_mask;
```

Without this, a child forked from a traced process would have `monitor_mask = 0` (default), so tracing would silently stop for any child processes. The assignment requirement is that monitoring must propagate to all descendants. Since `fork()` is supposed to create a near-identical copy of the parent, copying the mask is the correct semantics.

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Describe a scenario where `handshake` could **deadlock** if the unused pipe ends are not closed. Include which process blocks and why.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

After `fork()`, both parent and child hold **all four** file descriptors: `p2c[0]`, `p2c[1]`, `c2p[0]`, `c2p[1]`.

Scenario: Suppose the child calls `read(p2c[0], ...)` and the parent finishes writing and closes `p2c[1]`. But if the **child** never closed its copy of `p2c[1]`, the pipe still has an open write end (owned by the child). The kernel sees the pipe as still writable, so `read()` in the parent on `c2p[0]` would block forever waiting for data — even after the child calls `exit()` — because the write end of `c2p` is still held open by the parent itself.

The rule: **each process must close every pipe end it does not use**, otherwise the "all write ends closed → EOF" signal never fires, and readers block indefinitely.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

In `monitor`, the print is added **after** calling the syscall handler, not before. Why does this matter, and what would go wrong if you printed before?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

The trace line includes the **return value** (`-> RETURN`). The return value is only known after the handler executes and stores its result in `p->trapframe->a0`. If you printed before calling the handler, `a0` would contain the syscall **number** (or garbage), not the return value — the output would be wrong.

Additionally, for `exec()`: if you printed before the handler, the process name and context might not yet reflect the new program, leading to misleading trace output. Printing after ensures the kernel state is fully updated.

In `kernel/syscall.c`:
```c
num = p->trapframe->a7;
if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();   // call handler first
    if (p->monitor_mask & (1 << num))     // THEN check mask
        printf("%d: syscall %s -> %d\n",
               p->pid, syscall_names[num], p->trapframe->a0);
}
```

</div>
</div>
