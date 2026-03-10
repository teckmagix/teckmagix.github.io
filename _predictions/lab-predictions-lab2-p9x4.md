---
layout: page
title: "Lab Predictions - lab2-p9x4"
lab: lab2
description: "Exam predictions for Lab2-w5: uthread context switching and pthreads hash table."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

What is the purpose of the <code>ra</code> register in <code>thread_context</code>, and what value is it set to when a <strong>new</strong> thread is first created?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>What ra does in thread_switch:</strong></p>
<ol>
  <li><code>thread_switch</code> ends with <code>ret</code> instruction</li>
  <li><code>ret</code> = jump to whatever address is currently in <code>ra</code></li>
  <li>For a <strong>new thread</strong>: <code>ra</code> is set to the thread function → CPU jumps straight into <code>thread_a()</code></li>
  <li>For a <strong>resuming thread</strong>: <code>ra</code> holds the address right after where it last called <code>thread_yield()</code> → execution continues from there</li>
</ol>

<p><strong>How it's set in thread_create():</strong></p>
<pre><code class="language-c">void thread_create(void (*func)()) {
    struct thread *t;
    for (t = all_thread; t &lt; all_thread + MAX_THREAD; t++)
        if (t->state == FREE) break;

    t->state = RUNNABLE;
    t->ctx.ra = (uint64)func;  // ← ra = entry point of thread function
    t->ctx.sp = (uint64)(t->stack + STACK_SIZE);  // sp = top of stack
}</code></pre>

<p>On first schedule: <code>thread_switch</code> loads <code>ra = func</code>, executes <code>ret</code> → CPU jumps into <code>thread_a()</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

Why must the stack pointer (<code>sp</code>) be initialised to the <strong>end</strong> (highest address) of the stack array, not the beginning?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>RISC-V stack direction — it grows DOWNWARD:</strong></p>
<pre><code class="language-c">t->stack[0]              ← LOW address  (0x1000)
t->stack[1]              ↑ stack grows UP this way? NO!
...                      ↓ stack actually grows DOWN
t->stack[STACK_SIZE-1]   ← HIGH address (0x1000 + STACK_SIZE)
                         ↑
                    sp STARTS HERE</code></pre>

<p><strong>What happens if sp starts at the wrong end:</strong></p>
<ol>
  <li>Suppose <code>sp = t-&gt;stack</code> (lowest address)</li>
  <li>First function call: compiler does <code>addi sp, sp, -32</code> to allocate a frame</li>
  <li><code>sp</code> is now <strong>below</strong> <code>t-&gt;stack[0]</code> — outside the array!</li>
  <li>Stack writes go into adjacent memory (another thread's context, or code segment)</li>
  <li>Result: <strong>silent memory corruption</strong> → crash or wrong output</li>
</ol>

<p><strong>Correct initialisation:</strong></p>
<pre><code class="language-c">t->ctx.sp = (uint64)(t->stack + STACK_SIZE);
// t->stack + STACK_SIZE = address just past the end = highest valid sp
// First push: sp -= 8 → writes to t->stack[STACK_SIZE-1] → VALID</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What is a race condition in the hash table? Give a concrete step-by-step example of two threads losing a key.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Race condition:</strong> Two threads read-modify-write the same data concurrently without a lock → one write silently overwrites the other.</p>

<p><strong>Step-by-step losing a key (both threads insert into bucket 3):</strong></p>
<ol>
  <li>Thread A reads <code>table[3]</code> → sees head = <code>NULL</code></li>
  <li>Thread B reads <code>table[3]</code> → also sees head = <code>NULL</code> (before A writes)</li>
  <li>Thread A creates entry_A with <code>next = NULL</code>, writes <code>table[3] = entry_A</code></li>
  <li>Thread B creates entry_B with <code>next = NULL</code>, writes <code>table[3] = entry_B</code></li>
  <li>Result: <code>table[3] → entry_B → NULL</code></li>
  <li><strong>entry_A is orphaned</strong> — not in the list, not reachable → key permanently lost</li>
</ol>

<pre><code class="language-c">// The dangerous region in put() — NO lock:
struct entry *e = table[bucket];    // Thread A and B both read NULL here
// ... both create entries with next = NULL ...
table[bucket] = new_entry;          // whoever writes LAST wins, first is lost</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

Why does <code>get()</code> not require a mutex lock even though the hash table is shared between threads?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Two reasons get() is safe without a lock:</strong></p>

<p><strong>Reason 1 — Temporal separation (phase separation):</strong></p>
<ol>
  <li>The test harness runs all <code>put()</code>s in Phase 1 — all threads inserting</li>
  <li>A <strong>barrier</strong> (<code>pthread_barrier_wait</code>) ensures every thread finishes Phase 1 before any starts Phase 2</li>
  <li>By the time <code>get()</code> runs, zero <code>put()</code> calls are happening → no concurrent writes</li>
</ol>

<p><strong>Reason 2 — Read-only access:</strong></p>
<ol>
  <li><code>get()</code> only traverses the linked list: <code>e = e-&gt;next</code></li>
  <li>It never modifies <code>table[bucket]</code> or any <code>entry-&gt;next</code> pointer</li>
  <li>Multiple readers can read the same data simultaneously without conflict</li>
</ol>

<p>If <code>get()</code> ran <strong>concurrently with put()</strong>, a lock would be required — a <code>put()</code> modifying <code>table[3]</code> while <code>get()</code> is traversing it could cause <code>get()</code> to follow a dangling pointer.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Why do we save only <strong>callee-saved</strong> registers in <code>thread_switch</code>, not caller-saved ones like <code>a0–a7</code> or <code>t0–t6</code>?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>RISC-V calling convention — two categories:</strong></p>

<table>
  <tr><th>Category</th><th>Registers</th><th>Who saves them?</th><th>Why</th></tr>
  <tr><td>Caller-saved</td><td>a0–a7, t0–t6</td><td>The <strong>caller</strong></td><td>May be clobbered by any function call</td></tr>
  <tr><td>Callee-saved</td><td>ra, sp, s0–s11</td><td>The <strong>callee</strong> (thread_switch)</td><td>Must be preserved across function calls</td></tr>
</table>

<p><strong>Why caller-saved registers don't need saving in thread_switch:</strong></p>
<ol>
  <li>A thread calls <code>thread_yield()</code> → which calls <code>thread_schedule()</code> → which calls <code>thread_switch()</code></li>
  <li>Before each of those calls, the C compiler automatically saves caller-saved registers to the <strong>current thread's stack</strong></li>
  <li>By the time <code>thread_switch</code> executes, <code>a0–a7</code> and <code>t0–t6</code> are already saved on the stack</li>
  <li>The stack itself is preserved via <code>sp</code> (which IS saved) → so those values are safe</li>
</ol>

<p><strong>What thread_switch must save:</strong></p>
<pre><code class="language-asm">/* Only callee-saved — 14 registers total */
sd ra,   0(a0)   /* return address */
sd sp,   8(a0)   /* stack pointer — critical! */
sd s0,  16(a0)   /* s0-s11: long-lived variables */
/* ... s1-s11 ... */</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

What is per-bucket locking and why is it faster than a single global lock?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Per-bucket locking — one mutex per bucket:</strong></p>
<pre><code class="language-c">pthread_mutex_t bucket_locks[NBUCKET];  // 5 separate locks

void put(int key, int value) {
    int bucket = key % NBUCKET;
    pthread_mutex_lock(&amp;bucket_locks[bucket]);   // only lock THIS bucket
    insert(key, value, &amp;table[bucket], table[bucket]);
    pthread_mutex_unlock(&amp;bucket_locks[bucket]);
}</code></pre>

<p><strong>Performance comparison — 2 threads inserting simultaneously:</strong></p>

<table>
  <tr><th>Scenario</th><th>Single global lock</th><th>Per-bucket lock</th></tr>
  <tr><td>Thread A: bucket 0, Thread B: bucket 4</td><td>B <strong>waits</strong> for A even though no conflict</td><td>A locks [0], B locks [4] → <strong>parallel</strong> ✓</td></tr>
  <tr><td>Thread A: bucket 2, Thread B: bucket 2</td><td>B waits (same as per-bucket)</td><td>B waits for A (contention on same bucket)</td></tr>
</table>

<p><strong>Step by step — parallel execution with per-bucket:</strong></p>
<ol>
  <li>Thread A calls <code>put(key=0)</code> → hashes to bucket 0 → locks <code>bucket_locks[0]</code></li>
  <li>Thread B calls <code>put(key=4)</code> → hashes to bucket 4 → locks <code>bucket_locks[4]</code></li>
  <li>Both proceed simultaneously — <strong>no waiting</strong></li>
  <li>Both unlock when done</li>
  <li>Result: ~2× throughput vs single lock</li>
</ol>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Trace through exactly what happens when <code>thread_switch(old_ctx, new_ctx)</code> executes. What is the CPU state just before <code>ret</code>?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Phase 1 — SAVE (freeze old thread into RAM):</strong></p>
<pre><code class="language-asm">sd ra,   0(a0)   /* save old thread's return address → RAM */
sd sp,   8(a0)   /* save old thread's stack pointer → RAM */
sd s0,  16(a0)   /* save s0-s11 → RAM */
/* ... 14 stores total ... */
/* After this: old thread is fully frozen in its thread_context struct */</code></pre>

<p><strong>Phase 2 — RESTORE (thaw new thread from RAM into CPU):</strong></p>
<pre><code class="language-asm">ld ra,   0(a1)   /* load new thread's return address → CPU ra register */
ld sp,   8(a1)   /* load new thread's stack pointer → CPU sp register */
                 /* ← CRITICAL: sp now points to NEW thread's stack! */
ld s0,  16(a1)   /* load s0-s11 from new thread's context */
/* ... 14 loads total ... */</code></pre>

<p><strong>CPU state just before <code>ret</code>:</strong></p>
<ol>
  <li><code>ra</code> = new thread's resume address (either thread function or <code>thread_yield</code> return site)</li>
  <li><code>sp</code> = new thread's stack — all subsequent stack operations go to new thread's memory</li>
  <li><code>s0–s11</code> = new thread's saved variable values</li>
  <li>Old thread: fully frozen in RAM, invisible to CPU until next switch back</li>
</ol>

<p><strong><code>ret</code> fires:</strong> CPU jumps to <code>ra</code> → new thread is now running. From its perspective, <code>thread_yield()</code> just returned normally.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

What is the correct order of lock, insert, and unlock in <code>put()</code>? Why does order matter?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Correct order:</strong></p>
<pre><code class="language-c">void put(int key, int value) {
    int bucket = key % NBUCKET;

    pthread_mutex_lock(&amp;bucket_locks[bucket]);    // 1. LOCK first

    // 2. READ head pointer (must be atomic with write below)
    struct entry *head = table[bucket];
    // 3. CHECK if key exists, update or INSERT
    insert(key, value, &amp;table[bucket], head);

    pthread_mutex_unlock(&amp;bucket_locks[bucket]);  // 4. UNLOCK last
}</code></pre>

<p><strong>Why order matters — the critical window:</strong></p>
<ol>
  <li><code>insert()</code> does: read <code>table[bucket]</code> → create new entry → write <code>table[bucket] = new_entry</code></li>
  <li>The <strong>read and write must happen atomically</strong> — no other thread can modify <code>table[bucket]</code> between them</li>
  <li>If you unlock between the read and write: another thread inserts → <strong>lost update</strong></li>
  <li>If you lock after the read: the read itself is already a race — two threads both see the old head</li>
</ol>

<p><strong>Wrong — unlock too early:</strong></p>
<pre><code class="language-c">pthread_mutex_lock(&amp;bucket_locks[bucket]);
struct entry *head = table[bucket];       // read head
pthread_mutex_unlock(&amp;bucket_locks[bucket]);  // ← UNLOCK HERE = WRONG
// Another thread can now modify table[bucket]!
table[bucket] = new_entry;                // write with stale head → lost key</code></pre>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

What would happen if <code>thread_create</code> set <code>sp</code> to <code>t-&gt;stack</code> (lowest address)? Describe the failure mode in detail.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Step-by-step failure trace:</strong></p>
<ol>
  <li>New thread is scheduled → <code>thread_switch</code> loads <code>sp = t-&gt;stack[0]</code> (address 0x1000, say)</li>
  <li>CPU jumps to <code>thread_a()</code> via <code>ra</code></li>
  <li>Compiler-generated prologue: <code>addi sp, sp, -32</code> → <code>sp = 0x1000 - 32 = 0x0FE0</code></li>
  <li><code>0x0FE0</code> is <strong>below</strong> <code>t-&gt;stack[0]</code> = below the array bounds</li>
  <li>Stack frame writes go to <code>0x0FE0..0x1000</code> — which is the memory just <strong>before</strong> the stack array</li>
  <li>That memory likely contains: previous thread's stack, global variables, or code</li>
</ol>

<p><strong>What gets corrupted:</strong></p>
<pre><code class="language-c">// Memory layout in all_thread array:
all_thread[0].stack[0..STACK_SIZE-1]   // thread 0's stack
all_thread[1].stack[0..STACK_SIZE-1]   // thread 1's stack  ← underflowing thread writes HERE
all_thread[1].ctx                      // thread 1's context struct

// If thread 1's sp starts at all_thread[1].stack[0]:
// Stack underflow writes into all_thread[0].stack top OR thread 1's context.ra/sp
// → thread 0 or thread 1 jumps to garbage address on next switch</code></pre>

<p><strong>Why it's hard to debug:</strong> The crash doesn't happen at the underflow — it happens later when the corrupted thread gets scheduled and jumps to a garbage <code>ra</code>. The error appears far from the cause.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Implement the complete <code>thread_switch</code> assembly for RISC-V. Explain what each group of instructions accomplishes.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<pre><code class="language-asm">    .text
    .globl thread_switch
thread_switch:
    # a0 = pointer to OLD thread's context struct
    # a1 = pointer to NEW thread's context struct
    # Field layout: ra@0, sp@8, s0@16, s1@24 ... s11@104

    # ── GROUP 1: SAVE old thread (14 stores) ──
    # Freeze the old thread's CPU state into its context struct in RAM
    sd ra,    0(a0)   # save return address (where to resume old thread)
    sd sp,    8(a0)   # save stack pointer (old thread's stack)
    sd s0,   16(a0)
    sd s1,   24(a0)
    sd s2,   32(a0)
    sd s3,   40(a0)
    sd s4,   48(a0)
    sd s5,   56(a0)
    sd s6,   64(a0)
    sd s7,   72(a0)
    sd s8,   80(a0)
    sd s9,   88(a0)
    sd s10,  96(a0)
    sd s11, 104(a0)

    # ── GROUP 2: RESTORE new thread (14 loads) ──
    # Thaw the new thread's state from RAM into CPU registers
    ld ra,    0(a1)   # load return address → CPU will jump here on ret
    ld sp,    8(a1)   # load stack pointer → CPU now uses new thread's stack
    ld s0,   16(a1)
    ld s1,   24(a1)
    ld s2,   32(a1)
    ld s3,   40(a1)
    ld s4,   48(a1)
    ld s5,   56(a1)
    ld s6,   64(a1)
    ld s7,   72(a1)
    ld s8,   80(a1)
    ld s9,   88(a1)
    ld s10,  96(a1)
    ld s11, 104(a1)

    # ── GROUP 3: JUMP ──
    ret   # = jalr x0, ra, 0 → jump to ra (new thread's entry or resume point)</code></pre>

<p><strong>The three groups:</strong></p>
<ul>
  <li><strong>Group 1 (sd)</strong>: Writes CPU → RAM. Old thread is now safely stored.</li>
  <li><strong>Group 2 (ld)</strong>: Writes RAM → CPU. After <code>ld sp</code>, the CPU uses a completely different stack.</li>
  <li><strong>Group 3 (ret)</strong>: The "magic jump" — no explicit branch, just return to wherever <code>ra</code> now points.</li>
</ul>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

Thread A holds <code>lock[2]</code> and waits for <code>lock[3]</code>. Thread B holds <code>lock[3]</code> and waits for <code>lock[2]</code>. Identify the problem, explain why it can't happen in the current design, and propose two fixes for a redesign that does require two locks.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>The problem: DEADLOCK</strong></p>
<ol>
  <li>Thread A: holds lock[2], tries to acquire lock[3] → <strong>blocks</strong></li>
  <li>Thread B: holds lock[3], tries to acquire lock[2] → <strong>blocks</strong></li>
  <li>Neither can proceed → system hangs forever</li>
  <li>This is a circular wait — one of the four Coffman deadlock conditions</li>
</ol>

<p><strong>Why this CAN'T happen with current put() design:</strong></p>
<pre><code class="language-c">void put(int key, int value) {
    int bucket = key % NBUCKET;
    pthread_mutex_lock(&amp;bucket_locks[bucket]);   // acquire ONE lock
    insert(...);
    pthread_mutex_unlock(&amp;bucket_locks[bucket]); // release it immediately
}
// A thread NEVER holds two locks at once → circular wait impossible</code></pre>

<p><strong>If we added a "move key" operation needing 2 locks — two fixes:</strong></p>

<p><strong>Fix 1 — Lock ordering (always lock lower index first):</strong></p>
<pre><code class="language-c">void move_key(int from_bucket, int to_bucket, int key) {
    int lo = from_bucket &lt; to_bucket ? from_bucket : to_bucket;
    int hi = from_bucket &lt; to_bucket ? to_bucket : from_bucket;
    pthread_mutex_lock(&amp;bucket_locks[lo]);   // always lock lower first
    pthread_mutex_lock(&amp;bucket_locks[hi]);   // then higher
    // ... move key ...
    pthread_mutex_unlock(&amp;bucket_locks[hi]);
    pthread_mutex_unlock(&amp;bucket_locks[lo]);
    // Both threads try lo first → one blocks → other completes → no circular wait
}</code></pre>

<p><strong>Fix 2 — Try-lock with backoff:</strong></p>
<pre><code class="language-c">while (1) {
    pthread_mutex_lock(&amp;bucket_locks[from]);
    if (pthread_mutex_trylock(&amp;bucket_locks[to]) == 0)
        break;           // got both → proceed
    pthread_mutex_unlock(&amp;bucket_locks[from]);  // release and retry
    usleep(rand() % 100);  // random backoff breaks symmetry
}</code></pre>

</div>
</div>
