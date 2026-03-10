---
layout: page
title: "Lab Predictions - M4T7K"
lab: lab2
description: "Exam predictions for Lab2-w5: uthread context switching and pthreads hash table."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

What is the purpose of the `ra` (return address) register in `thread_context`, and what value is it set to when a **new** thread is first created?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

`ra` stores the address where execution will jump when `ret` is executed at the end of `thread_switch`. For a **brand-new** thread, `ra` is set to the **thread function pointer** (e.g., `thread_a`). So when the scheduler switches to this new thread for the first time, `ret` jumps directly into `thread_a()` — starting the thread's execution from the beginning.

```c
// In thread_create():
t->context.ra = (uint64)func;  // jump here on first switch
```

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

Why must the stack pointer (`sp`) be initialised to the **end** (highest address) of the stack array, not the beginning?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

Stacks on RISC-V (and x86) **grow downward** — each function call pushes a frame by **subtracting** from `sp`. If `sp` started at the lowest address, the very first function call would write below the array, corrupting adjacent memory. Starting at the highest address gives the stack room to grow down into the array.

```c
t->context.sp = (uint64)(t->stack + STACK_SIZE);
// t->stack[0] is low address, t->stack[STACK_SIZE-1] is high
```

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What is a race condition in the context of the multi-threaded hash table? Give a concrete example of how two threads inserting at the same time can lose a key.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

A race condition occurs when the result depends on the unpredictable timing of two threads. For the hash table:

1. Thread A and Thread B both call `put()` for keys that hash to bucket 3.
2. Both read `table[3]` and see the same head pointer (say `NULL`).
3. Both create a new entry and set `entry->next = NULL`.
4. Thread A writes its entry as the new head of bucket 3.
5. Thread B then **overwrites** the head with its own entry — A's entry is now orphaned and lost.

Neither thread overwrote the other's **entry struct**, but they both wrote to the same `table[3]` pointer, so whichever wrote last wins and the other key is permanently dropped.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

Why does `get()` not require a mutex lock in this lab, even though the hash table is shared between threads?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

Two reasons:
1. **Temporal separation**: All `put()` calls (Phase 1) complete before any `get()` calls (Phase 2) begin. A barrier ensures no thread starts `get()` while another is still doing `put()`.
2. **Read-only**: `get()` only traverses the linked list — it never modifies `next` pointers or the bucket head. Since no writes happen during the get phase, there is no chance of a data race.

If `get()` ran concurrently with `put()`, a lock would be required.

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Why do we only save **callee-saved** registers (`ra`, `sp`, `s0–s11`) in `thread_switch`, and not caller-saved registers like `a0–a7` or `t0–t6`?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

The RISC-V calling convention divides registers into two groups:

- **Caller-saved** (`a0–a7`, `t0–t6`): The *caller* must save these before any function call if it wants to keep them. The C compiler automatically generates code to spill them to the stack before calling `thread_switch`. So by the time `thread_switch` executes, these are already safe on the stack.

- **Callee-saved** (`ra`, `sp`, `s0–s11`): The *callee* (`thread_switch`) must preserve these. If we didn't save them, the old thread's saved register values would be corrupted when it eventually resumes.

By saving only callee-saved registers, `thread_switch` follows the calling convention — from the C compiler's perspective, it just looks like a regular function call.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

What is **per-bucket locking** and why does it provide better performance than a single global lock for the hash table?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

Per-bucket locking assigns one mutex to each of the 5 buckets (`pthread_mutex_t locks[NBUCKET]`). A thread only acquires the lock for the specific bucket it is inserting into.

With a single global lock: Thread A inserting into bucket 0 and Thread B inserting into bucket 4 would **block each other** even though they access completely different memory — the lock serialises all insertions.

With per-bucket locking: Thread A locks bucket 0, Thread B locks bucket 4 — they run **in parallel** because there is no shared state between the two buckets. Threads only block each other when they hash to the **same** bucket.

Result: Near-linear speedup with thread count, while still guaranteeing 0 missing keys.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Trace through what happens when `thread_switch(old_ctx, new_ctx)` is called. Describe the CPU state before `ret` executes.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

1. **Save phase**: `sd ra, 0(a0)` through `sd s11, 104(a0)` — all 14 callee-saved registers of the **old** thread are written into its `thread_context` struct in RAM.

2. **Load phase**: `ld ra, 0(a1)` through `ld s11, 104(a1)` — the CPU registers are overwritten with values from the **new** thread's `thread_context`.

3. **Crucial moment after `ld sp, 8(a1)`**: The CPU's stack pointer now points to the new thread's stack. Any subsequent stack operations (local variables, return addresses) belong to the new thread.

4. **`ret`**: Jumps to whatever address is now in `ra`. For a new thread, this is the thread function's entry point. For a resuming thread, this is wherever it last called `thread_yield`.

From this point on, the CPU is executing the new thread as if `thread_switch` was never called.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

What is the correct order to `lock`, `insert`, and `unlock` in `put()` for per-bucket locking, and why does order matter?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

```c
void put(int key, int value) {
    int bucket = key % NBUCKET;
    pthread_mutex_lock(&locks[bucket]);     // 1. lock BEFORE reading head
    insert(key, value, &table[bucket], table[bucket]);
    pthread_mutex_unlock(&locks[bucket]);   // 2. unlock AFTER insert
}
```

Order matters because `insert()` reads `table[bucket]` (the head pointer) and then writes to it. Both the **read** and the **write** must happen atomically. If you unlock between the read and the write, another thread could insert between those two steps, causing the same lost-update race condition as before.

Locking too late (after the read) or unlocking too early (before the write) defeats the purpose of the mutex.

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

What would happen if `thread_create` set `sp` to `t->stack` (the lowest address) instead of `t->stack + STACK_SIZE`? Describe the failure mode in detail.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

The stack would immediately **underflow**. The moment the new thread is scheduled and begins executing `thread_a()`, the function prologue subtracts from `sp` to allocate a stack frame (e.g., `addi sp, sp, -32`). This would put `sp` at a **negative** address (below `t->stack[0]`), writing the saved registers and local variables into memory that doesn't belong to this stack — likely into another thread's context struct or global variables.

The result would be a **silent memory corruption** bug: other threads' `ra` or `sp` values get overwritten, causing them to jump to garbage addresses or use wrong stacks when they next run. The program would crash or produce wrong output in a non-deterministic way that is extremely hard to debug.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Implement the complete `thread_switch` assembly function for RISC-V. Explain what each group of instructions does.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

```asm
    .text
    .globl thread_switch
thread_switch:
    /* SAVE old thread's callee-saved registers into its context (a0) */
    sd ra,   0(a0)
    sd sp,   8(a0)
    sd s0,  16(a0)
    sd s1,  24(a0)
    sd s2,  32(a0)
    sd s3,  40(a0)
    sd s4,  48(a0)
    sd s5,  56(a0)
    sd s6,  64(a0)
    sd s7,  72(a0)
    sd s8,  80(a0)
    sd s9,  88(a0)
    sd s10, 96(a0)
    sd s11,104(a0)

    /* RESTORE new thread's callee-saved registers from its context (a1) */
    ld ra,   0(a1)
    ld sp,   8(a1)
    ld s0,  16(a1)
    ld s1,  24(a1)
    ld s2,  32(a1)
    ld s3,  40(a1)
    ld s4,  48(a1)
    ld s5,  56(a1)
    ld s6,  64(a1)
    ld s7,  72(a1)
    ld s8,  80(a1)
    ld s9,  88(a1)
    ld s10, 96(a1)
    ld s11,104(a1)

    ret   /* jump to ra — either thread_a/b/c entry or resume point */
```

- **`sd` block**: Freezes the old thread's CPU state into RAM. The offsets (0, 8, 16…) match the 8-byte field layout of `struct thread_context`.
- **`ld` block**: Thaws the new thread's state from RAM back into CPU registers. After `ld sp, 8(a1)`, the CPU is using the new thread's stack.
- **`ret`**: The "magic jump" — uses the newly loaded `ra` to continue execution in the new thread.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

In a per-bucket locking scheme with 2 threads and 5 buckets, Thread A holds `lock[2]` and is waiting for `lock[3]`. Thread B holds `lock[3]` and is waiting for `lock[2]`. Identify the problem and propose two solutions.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

This is a **deadlock** — both threads are waiting for a lock the other holds, and neither can proceed.

**This scenario cannot happen in the current `put()` design** because each `put()` only ever holds **one** lock at a time (lock, insert, unlock — done). A thread never holds two bucket locks simultaneously, so circular wait (one of the four Coffman conditions) is impossible.

However, if the design were changed (e.g., a "move key from bucket X to bucket Y" operation), deadlock could occur. Two solutions:

1. **Lock ordering**: Always acquire locks in a fixed order (e.g., by bucket index, lowest first). If both threads must lock buckets 2 and 3, they both try 2 first → one blocks → the other completes and releases → first proceeds. Circular wait is broken.

2. **Try-lock with backoff**: Use `pthread_mutex_trylock()`. If a thread can't acquire the second lock, it releases the first and retries after a random delay, breaking the symmetry.

</div>
</div>
