---
layout: page
title: "Lab Predictions - lab1-k7m2"
lab: lab1
description: "Exam predictions for Lab1-w3: handshake, sniffer, monitor."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

In <code>handshake.c</code>, why must both pipes be created <strong>before</strong> calling <code>fork()</code>?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Reason:</strong> <code>fork()</code> copies the parent's <strong>entire file descriptor table</strong> to the child. Pipes must exist before the fork so both processes share the same kernel buffers.</p>

<p><strong>Step by step — what happens if you create pipes AFTER fork:</strong></p>
<ol>
  <li>Parent calls <code>fork()</code> → child is created</li>
  <li>Parent creates <code>pipe(p2c)</code> → gets its own private buffer</li>
  <li>Child creates <code>pipe(p2c)</code> → gets a <strong>completely different</strong> private buffer</li>
  <li>Parent writes to its <code>p2c[1]</code> → data goes into parent's buffer only</li>
  <li>Child reads from its <code>p2c[0]</code> → reads from a different buffer → <strong>blocks forever</strong></li>
</ol>

<p><strong>Correct approach:</strong></p>
<pre><code class="language-c">pipe(p2c);   // create BEFORE fork()
pipe(c2p);   // create BEFORE fork()
int pid = fork();
// Now BOTH parent and child share the same kernel pipe buffers</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What does <code>fd[0]</code> and <code>fd[1]</code> represent in a pipe? Which end does the parent use to send a byte to the child?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Pipe file descriptor roles:</strong></p>
<ul>
  <li><code>fd[0]</code> = <strong>read end</strong> — pull data OUT of the pipe (think: mouth = 0 = input)</li>
  <li><code>fd[1]</code> = <strong>write end</strong> — push data INTO the pipe (think: pen = 1 = output)</li>
</ul>

<p><strong>Direction for handshake:</strong></p>
<pre><code class="language-c">// Parent → Child  (p2c pipe)
write(p2c[1], &byte, 1);   // parent WRITES via write end [1]
read(p2c[0],  &byte, 1);   // child  READS  via read  end [0]

// Child → Parent  (c2p pipe)
write(c2p[1], &byte, 1);   // child  WRITES via write end [1]
read(c2p[0],  &byte, 1);   // parent READS  via read  end [0]</code></pre>

<p>To send a byte <strong>to the child</strong>, the parent writes to <code>p2c[1]</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What happens when a process calls <code>read()</code> on a pipe that is currently empty but still has open write ends?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p>The process <strong>blocks (sleeps)</strong> — the kernel puts it in a wait queue until data arrives.</p>

<p><strong>The three pipe states:</strong></p>

<table>
  <tr><th>Situation</th><th>What happens</th></tr>
  <tr><td>Pipe empty, write end still open</td><td>Reader <strong>blocks</strong> until writer sends data</td></tr>
  <tr><td>Pipe full, reader not consuming</td><td>Writer <strong>blocks</strong> until reader drains space</td></tr>
  <tr><td>All write ends closed, pipe empty</td><td>Reader gets <strong>0 (EOF)</strong> — no more data ever</td></tr>
</table>

<p>This automatic blocking is why <code>handshake</code> works without any explicit synchronisation — the pipe acts as both a data channel and a synchronisation primitive.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does <code>monitor(1 &lt;&lt; SYS_read)</code> do? What integer value does it evaluate to if <code>SYS_read = 5</code>?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Step by step:</strong></p>
<ol>
  <li><code>SYS_read = 5</code>, so <code>1 &lt;&lt; 5 = 32</code> (binary: <code>00100000</code>)</li>
  <li><code>monitor(32)</code> is called → kernel stores <code>p-&gt;monitor_mask = 32</code></li>
  <li>Every syscall, kernel checks: <code>(p-&gt;monitor_mask &gt;&gt; num) &amp; 1</code></li>
  <li>When <code>num = 5</code>: <code>(32 &gt;&gt; 5) &amp; 1 = 1</code> → print trace</li>
  <li>For any other syscall: bit is 0 → no print</li>
</ol>

<pre><code class="language-c">// How the kernel checks the mask:
if ((p->monitor_mask >> num) & 1) {
    printf("%d: syscall %s -> %d\n",
           p->pid, syscall_names[num], p->trapframe->a0);
}</code></pre>

<p>Result: only <code>read</code> syscall calls are traced. All other syscalls are silent.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Why does <code>handshake.c</code> use <code>argv[1][0]</code> instead of <code>atoi(argv[1])</code> to get the byte to send?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>The compiler error problem:</strong></p>
<ol>
  <li>xv6 compiles with <code>-Werror=unused-variable</code> — any unused variable is a <strong>compile error</strong></li>
  <li>If you write <code>char byte = atoi(argv[1]);</code> before <code>fork()</code>...</li>
  <li>The child branch never reads <code>byte</code> directly (it does <code>read()</code> from the pipe instead)</li>
  <li>Compiler sees <code>byte</code> assigned but potentially unused in one branch → <strong>error, compilation fails</strong></li>
</ol>

<p><strong>The fix — use <code>argv[1][0]</code> directly:</strong></p>
<pre><code class="language-c">// BAD: compiler may warn about 'byte' being unused in child branch
char byte = atoi(argv[1]);
write(p2c[1], &byte, 1);

// GOOD: no intermediate variable needed
write(p2c[1], &argv[1][0], 1);   // argv[1][0] is the first char of the arg string</code></pre>

<p>No variable = no unused-variable warning = no compile error.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

Explain the "LIFO free page list" behaviour in xv6 and why <code>sbrk(8*4096)</code> in <code>sniffer.c</code> reliably returns the same pages that <code>secret.c</code> just freed.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>How xv6's memory allocator works (LIFO stack):</strong></p>
<ol>
  <li><code>kalloc.c</code> keeps freed pages in a <strong>linked list used as a stack</strong></li>
  <li>When a page is freed: it's <strong>pushed to the top</strong> of the stack</li>
  <li>When a page is allocated: it's <strong>popped from the top</strong></li>
  <li>Result: the most recently freed page is the next one allocated</li>
</ol>

<p><strong>Why this makes sniffer work:</strong></p>
<ol>
  <li><code>secret</code> writes the secret into its memory (8 pages)</li>
  <li><code>secret</code> exits → its 8 pages are <strong>pushed onto the free list</strong></li>
  <li>Normally <code>kalloc</code> would call <code>memset(page, 0, PGSIZE)</code> when freeing — but <strong>this memset is removed in the lab</strong></li>
  <li><code>sniffer</code> calls <code>sbrk(8*4096)</code> → kernel pops those <strong>same 8 pages</strong> from the top</li>
  <li>Pages still contain secret's data → sniffer can read it</li>
</ol>

<pre><code class="language-c">// In sniffer.c — why 8*4096 exactly:
char *mem = sbrk(8 * 4096);  // request same size as secret used
// mem now points to secret's old pages (LIFO guarantee)
// scan mem for the marker string to find the secret</code></pre>

<p><strong>If xv6 used FIFO instead of LIFO</strong>, the pages from <code>secret</code> might not be at the top → sniffer would fail.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Why does <code>monitor_mask</code> survive across an <code>exec()</code> call, even though <code>exec()</code> replaces the process's code, data, and stack?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>What exec() replaces vs. what it keeps:</strong></p>

<table>
  <tr><th>Replaced by exec()</th><th>Survives exec()</th></tr>
  <tr><td>User-space text (code)</td><td>PID</td></tr>
  <tr><td>User-space data / BSS</td><td>Open file descriptors</td></tr>
  <tr><td>User-space stack</td><td><code>monitor_mask</code> ✓</td></tr>
  <tr><td>User-space heap</td><td>All <code>struct proc</code> fields</td></tr>
</table>

<p><strong>Where monitor_mask lives:</strong></p>
<ol>
  <li><code>monitor_mask</code> is stored in <code>struct proc</code> inside the <strong>kernel</strong></li>
  <li><code>exec()</code> only replaces <strong>user-space memory</strong> — it calls <code>proc_freepagetable()</code> on the old address space</li>
  <li>The <code>struct proc</code> in kernel memory is <strong>not touched</strong> by exec</li>
  <li>So <code>monitor_mask</code> persists → the new program is still traced</li>
</ol>

<pre><code class="language-c">// In monitor.c — this is why it works:
monitor(mask);        // sets p->monitor_mask in kernel struct proc
exec(argv[2], ...);   // replaces user code BUT struct proc survives
// The new program (e.g. grep) still has monitor_mask set!</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

Why does a simple ASCII scan fail to find the secret in <code>sniffer</code>? What is the correct approach?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Memory layout of secret.c's data array:</strong></p>
<pre><code class="language-c">// secret.c writes TWO things into its data array:
data[0..14]  = "This may help.\0"   // marker string at offset 0
data[16..]   = "ict1012\0"          // actual secret at offset 16</code></pre>

<p><strong>Why a naive ASCII scan fails:</strong></p>
<ol>
  <li>Scanner scans left-to-right looking for printable chars</li>
  <li>Finds <code>"This may help."</code> at offset 0 — 14 printable chars</li>
  <li>Prints <code>"This may help."</code> → <strong>wrong answer</strong></li>
  <li>Scanner exits — never reaches the actual secret at offset 16</li>
</ol>

<p><strong>Correct step-by-step approach:</strong></p>
<ol>
  <li>Allocate memory: <code>char *mem = sbrk(8 * 4096);</code></li>
  <li>Scan for the exact marker string <code>"This may help."</code> using <code>memcmp</code></li>
  <li>When found at position <code>i</code>, the secret is at <code>mem + i + 16</code></li>
  <li>Print it: <code>printf("%s\n", mem + i + 16);</code></li>
</ol>

<pre><code class="language-c">char *mem = sbrk(8 * 4096);
for (int i = 0; i &lt; 8 * 4096 - 16; i++) {
    if (memcmp(mem + i, "This may help.", 14) == 0) {
        printf("%s\n", mem + i + 16);  // secret is right after marker
        exit(0);
    }
}</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-med">Medium</span></div>

What changes must be made to <code>kfork()</code> in <code>kernel/proc.c</code> for the monitor syscall, and why?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>The one-line change:</strong></p>
<pre><code class="language-c">// In kfork(), after: np->pid = alloc_pid();
// Add this line:
np->monitor_mask = p->monitor_mask;</code></pre>

<p><strong>Why this is needed — step by step:</strong></p>
<ol>
  <li>User runs: <code>monitor 2 usertests forkforkfork</code></li>
  <li><code>monitor(2)</code> sets <code>p-&gt;monitor_mask = 2</code> (trace <code>fork</code>)</li>
  <li><code>exec(usertests)</code> runs → usertests calls <code>fork()</code> many times</li>
  <li>Each child is created by <code>kfork()</code></li>
  <li><strong>Without the copy</strong>: child gets <code>monitor_mask = 0</code> (default) → child's forks are not traced</li>
  <li><strong>With the copy</strong>: child gets <code>monitor_mask = 2</code> → child's forks ARE traced too</li>
</ol>

<p><strong>Expected output with the fix:</strong></p>
<pre><code class="language-c">$ monitor 2 usertests forkforkfork
6: syscall fork -> 7
7: syscall fork -> 8    // child 7 also traced because it inherited mask
8: syscall fork -> 9    // grandchild also traced</code></pre>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Describe a scenario where <code>handshake</code> deadlocks if unused pipe ends are not closed. Trace which process blocks and why.

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>Setup — after fork(), both processes hold ALL 4 fds:</strong></p>
<pre><code class="language-c">// After fork(), both parent AND child have:
p2c[0], p2c[1]   // both ends of parent-to-child pipe
c2p[0], c2p[1]   // both ends of child-to-parent pipe</code></pre>

<p><strong>Deadlock scenario — step by step:</strong></p>
<ol>
  <li>Parent writes to <code>p2c[1]</code> ✓ → child wakes, reads from <code>p2c[0]</code> ✓</li>
  <li>Child writes to <code>c2p[1]</code> and calls <code>exit(0)</code></li>
  <li>On exit: child closes its <code>c2p[1]</code> — but the <strong>parent still holds its own copy of <code>c2p[1]</code></strong> (it never closed it)</li>
  <li>Parent calls <code>read(c2p[0], ...)</code> waiting for child's reply</li>
  <li>Kernel checks: "are all write ends of <code>c2p</code> closed?" → <strong>NO</strong> — parent's copy of <code>c2p[1]</code> is still open</li>
  <li>Parent's <code>read()</code> blocks forever → <strong>deadlock</strong></li>
</ol>

<p><strong>The fix — close every end you don't use:</strong></p>
<pre><code class="language-c">// In child:
close(p2c[1]);  // child never writes to p2c
close(c2p[0]);  // child never reads from c2p

// In parent:
close(p2c[0]);  // parent never reads from p2c
close(c2p[1]);  // parent never writes to c2p
// Now when child exits and closes c2p[1], ALL write ends are closed
// → parent's read() returns 0 (EOF) instead of blocking</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

In <code>monitor</code>, the print is added <strong>after</strong> calling the syscall handler. Why does this matter? What would go wrong if you printed before?

<button class="answer-toggle">▼ Show Answer</button>
<div class="answer-body">

<p><strong>What's in register a0 at each point:</strong></p>

<table>
  <tr><th>Moment</th><th>a0 contains</th></tr>
  <tr><td>Before calling handler</td><td>Syscall <strong>number</strong> (from a7 already decoded) or first argument</td></tr>
  <tr><td>After calling handler</td><td><strong>Return value</strong> — the actual result</td></tr>
</table>

<p><strong>Step by step — what goes wrong if you print before:</strong></p>
<ol>
  <li>User calls <code>read(fd, buf, 512)</code> → <code>a7 = SYS_read = 5</code></li>
  <li><code>a0</code> currently holds the <strong>first argument</strong> = <code>fd</code> value (e.g. 3)</li>
  <li>If you print now: <code>"3: syscall read -&gt; 3"</code> → wrong! Shows fd not return value</li>
  <li>After handler runs, <code>a0 = 1023</code> (bytes read)</li>
  <li>Correct output: <code>"3: syscall read -&gt; 1023"</code></li>
</ol>

<p><strong>Correct implementation:</strong></p>
<pre><code class="language-c">void syscall(void) {
    int num;
    struct proc *p = myproc();
    num = p->trapframe->a7;

    if (num > 0 && num &lt; NELEM(syscalls) && syscalls[num]) {
        // Step 1: EXECUTE the handler → return value written to a0
        p->trapframe->a0 = syscalls[num]();

        // Step 2: THEN check mask and print (a0 now has correct return value)
        if ((p->monitor_mask >> num) & 1)
            printf("%d: syscall %s -> %d\n",
                   p->pid, syscall_names[num], p->trapframe->a0);
    }
}</code></pre>

</div>
</div>
