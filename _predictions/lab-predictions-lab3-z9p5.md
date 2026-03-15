---
layout: page
title: "Lab Predictions - lab3-z9p5"
lab: lab3
description: "Exam predictions for Lab3-w8: symlinktest structure, what each assertion checks, concurrent test internals, and failing test diagnosis."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

The lab PDF says <code>symlinktest</code> has four test phases. For each phase, write the minimal xv6 system calls that would reproduce the test and explain what pass/fail looks like.

<div class="answer-content">

<p><strong>Phase 1 — Basic Link:</strong></p>
<pre><code class="language-c">// Creates file "a", symlink "b" → "a", opens "b"
int fd = open("a", O_CREATE|O_RDWR);
write(fd, "test", 4); close(fd);
symlink("a", "b");

fd = open("b", O_RDONLY);   // PASS: fd > 0, reads "test"
                              // FAIL: fd = -1 (symlink following broken)
char buf[4]; read(fd, buf, 4);
// PASS: buf == "test"
// FAIL: buf != "test" (wrong inode followed or wrong data block)</code></pre>

<p><strong>Phase 2 — Concurrent Links:</strong></p>
<pre><code class="language-c">// Forks N child processes, each does:
int pid = fork();
if(pid == 0){
    char name[16];  // unique name per child
    symlink("target", name);
    fd = open(name, O_RDONLY);   // PASS: fd > 0
    close(fd); unlink(name); exit(0);
}
// Parent waits for all children
// FAIL: any child exits with non-zero (panic or wrong fd)</code></pre>

<p><strong>Phase 3 — Chain Test:</strong></p>
<pre><code class="language-c">// link4 → link3 → link2 → link1 → file
symlink("file",  "link1");
symlink("link1", "link2");
symlink("link2", "link3");
symlink("link3", "link4");

fd = open("link4", O_RDONLY);  // PASS: fd > 0 (followed 4 hops)
                                 // FAIL: fd = -1 (loop exits too early)</code></pre>

<p><strong>Phase 4 — Circle Test:</strong></p>
<pre><code class="language-c">// linkA → linkB → linkA (infinite cycle)
symlink("linkB", "linkA");
symlink("linkA", "linkB");

fd = open("linkA", O_RDONLY);  // PASS: fd = -1 (depth limit triggered)
                                 // FAIL: kernel hangs/panics (no depth limit)
if(fd != -1){
    printf("FAIL: circular link should return -1\n");
    exit(1);
}</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What does <code>./grade-lab-fs symlink</code> check in its output? Write out the expected passing output and explain what each line confirms.

<div class="answer-content">

<p><strong>Expected passing output:</strong></p>
<pre><code class="language-c">$ ./grade-lab-fs symlink
make: 'kernel/kernel' is up to date.
== Test running symlinktest == (2.3s)
== Test symlinktest: symlinks ==
    symlinktest: symlinks: OK
== Test symlinktest: concurrent symlinks ==
    symlinktest: concurrent symlinks: OK
$</code></pre>

<p><strong>What each line confirms:</strong></p>

<table>
  <tr><th>Output line</th><th>What it confirms</th></tr>
  <tr><td><code>make: 'kernel/kernel' is up to date.</code></td><td>No recompilation needed — kernel was already built; this is not a test result</td></tr>
  <tr><td><code>== Test running symlinktest == (2.3s)</code></td><td>xv6 booted successfully, symlinktest binary exists and ran to completion in 2.3 seconds</td></tr>
  <tr><td><code>symlinktest: symlinks: OK</code></td><td>The basic + chain + circle test phases passed — single-process symlink creation, following, and depth limiting work correctly</td></tr>
  <tr><td><code>symlinktest: concurrent symlinks: OK</code></td><td>Multiple concurrent processes creating/opening symlinks simultaneously worked correctly — ilock/iunlock discipline is correct, no race conditions</td></tr>
</table>

<p><strong>What a failing output looks like:</strong></p>
<pre><code class="language-c">// If basic test fails (e.g., target[n]=0 missing):
== Test symlinktest: symlinks ==
    symlinktest: symlinks: FAIL
//   (symlinktest printed "test symlinks: FAILED" instead of "ok")

// If kernel panics during test:
== Test running symlinktest == FAIL
//   (QEMU crashed — no output from symlinktest at all)</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

After both tasks are complete and <code>make grade</code> is run, what is the maximum possible score and how is it divided between the two tasks?

<div class="answer-content">

<p><strong>Maximum score and breakdown:</strong></p>
<pre><code class="language-c">$ make clean && make grade
== Test running bigfile ==
    running bigfile: OK (91.2s)         ← Task 1: Large Files
== Test running symlinktest ==
    (2.0s)
== Test symlinktest: symlinks ==
    symlinktest: symlinks: OK           ← Task 2: sub-test 1
== Test symlinktest: concurrent symlinks ==
    symlinktest: concurrent symlinks: OK ← Task 2: sub-test 2
Score: 100/100</code></pre>

<p><strong>Score allocation:</strong></p>

<table>
  <tr><th>Test</th><th>Points</th><th>Task</th></tr>
  <tr><td>running bigfile</td><td>40</td><td>Task 1: Large Files</td></tr>
  <tr><td>symlinktest: symlinks</td><td>30</td><td>Task 2: Symbolic Links (basic)</td></tr>
  <tr><td>symlinktest: concurrent symlinks</td><td>30</td><td>Task 2: Symbolic Links (concurrent)</td></tr>
  <tr><td><strong>Total</strong></td><td><strong>100</strong></td><td></td></tr>
</table>

<p><strong>Partial credit scenarios:</strong></p>

<table>
  <tr><th>What's implemented</th><th>Score</th></tr>
  <tr><td>Only Task 1 (bigfile passes)</td><td>40/100</td></tr>
  <tr><td>Task 1 + basic symlinks (no concurrent)</td><td>70/100</td></tr>
  <tr><td>Both tasks complete</td><td>100/100</td></tr>
  <tr><td>Task 1 wrong but Task 2 passes</td><td>60/100 — unlikely since Task 2 doesn't depend on Task 1</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

The grader runs <code>make zipball</code> to create <code>lab.zip</code> for Gradescope submission. What files does <code>lab.zip</code> contain, and which specific files from this lab did you modify?

<div class="answer-content">

<p><strong>What make zipball creates:</strong></p>
<p><code>make zipball</code> creates <code>lab.zip</code> containing the entire xv6 source tree — all kernel files, user files, Makefile, and scripts. The grader on Gradescope compiles everything from source and runs the tests.</p>

<p><strong>Files modified in this lab (10 files total):</strong></p>

<table>
  <tr><th>File</th><th>Task</th><th>Change</th></tr>
  <tr><td><code>kernel/param.h</code></td><td>Task 1</td><td>FSSIZE: 2000 → 200000</td></tr>
  <tr><td><code>kernel/fs.h</code></td><td>Task 1</td><td>NDIRECT=11, add NDINDIRECT, update MAXFILE, addrs[NDIRECT+2] in struct dinode</td></tr>
  <tr><td><code>kernel/file.h</code></td><td>Task 1</td><td>addrs[NDIRECT+2] in struct inode</td></tr>
  <tr><td><code>kernel/fs.c</code></td><td>Task 1</td><td>bmap() doubly-indirect section + itrunc() doubly-indirect cleanup</td></tr>
  <tr><td><code>user/bigfile.c</code></td><td>Task 1</td><td>Replace with professor's version (TEST_BLOCKS = 65803/10)</td></tr>
  <tr><td><code>kernel/stat.h</code></td><td>Task 2</td><td>T_SYMLINK = 4</td></tr>
  <tr><td><code>kernel/fcntl.h</code></td><td>Task 2</td><td>O_NOFOLLOW = 0x800</td></tr>
  <tr><td><code>kernel/syscall.h</code></td><td>Task 2</td><td>SYS_symlink = 22</td></tr>
  <tr><td><code>user/user.h</code></td><td>Task 2</td><td>int symlink(const char*, const char*);</td></tr>
  <tr><td><code>user/usys.pl</code></td><td>Task 2</td><td>entry("symlink");</td></tr>
  <tr><td><code>kernel/syscall.c</code></td><td>Task 2</td><td>extern declaration + dispatch table entry</td></tr>
  <tr><td><code>kernel/sysfile.c</code></td><td>Task 2</td><td>sys_symlink() implementation + sys_open() symlink loop</td></tr>
  <tr><td><code>Makefile</code></td><td>Both</td><td>$U/_bigfile\ (already present) + $U/_symlinktest\</td></tr>
</table>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

The concurrent symlinks test forks multiple child processes. What specific race would occur if <code>sys_symlink()</code> did not use <code>begin_op()</code>/<code>end_op()</code>, and what would the visible symptom be when running <code>symlinktest</code>?

<div class="answer-content">

<p><strong>Without begin_op/end_op — what creates() does internally:</strong></p>
<pre><code class="language-c">// sys_symlink() without transaction wrapping:
ip = create(path, T_SYMLINK, 0, 0);
// create() calls:
//   ialloc() → allocates inode, writes type=T_SYMLINK to disk immediately
//   dirlink() → adds directory entry immediately
writei(ip, 0, target, 0, strlen(target));
// writei() writes target path to data block immediately
iunlockput(ip);
return 0;</code></pre>

<p><strong>The race — two processes hit the window between ialloc and dirlink:</strong></p>
<pre><code class="language-c">// Process A: ialloc() → inode #24 allocated, type=T_SYMLINK written to disk
// [context switch]
// Process B: ialloc() → inode #25 allocated, type=T_SYMLINK written to disk
// Process B: dirlink(root, "linkB", 25) → root dir updated
// Process B: writei(ip25, "targetB") → data written
// [context switch]
// Process A: dirlink(root, "linkA", 24) → root dir updated
// Process A: writei(ip24, "targetA") → data written

// This actually succeeds — the race above is benign for simple creation.
// The real race is in dirlink() modifying the directory block without a transaction:</code></pre>

<p><strong>The actual dangerous race — directory block corruption:</strong></p>
<pre><code class="language-c">// Both processes need to write to the root directory's data block.
// Without a transaction, dirlink() does:
//   bread(dir_block) → modify a dirent → bwrite(dir_block)
// Process A: bread(dir_block) → reads current state with 20 entries
// [context switch]
// Process B: bread(dir_block) → reads SAME state with 20 entries
// Process B: writes entry for "linkB" at slot 21 → bwrite(dir_block)
// [context switch]
// Process A: writes entry for "linkA" at slot 21 → bwrite(dir_block)
// RESULT: "linkB" is overwritten by "linkA" — "linkB" disappears from the directory!
// Process B returns success but its link is gone.</code></pre>

<p><strong>Visible symptom in symlinktest:</strong></p>
<pre><code class="language-c">// In the concurrent test, each child does:
open(name, O_RDONLY)   // try to open the just-created symlink

// Some children find their symlink is missing (overwritten by another process):
// open() → namei(name) → returns 0 (not found) → fd = -1
// Child exits with -1 → parent wait() sees non-zero status

// symlinktest detects this:
// "test concurrent symlinks: FAILED"
// → Grade: symlinktest: concurrent symlinks: FAIL</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

The lab PDF's symlinktest description shows a "Chain Test" that creates a 4-hop chain. What is the minimum implementation of the while loop in sys_open() that passes the Chain Test but fails the Circle Test?

<div class="answer-content">

<p><strong>The Chain Test requirement:</strong></p>
<pre><code class="language-c">// Chain: link4 → link3 → link2 → link1 → file
// Needs to follow 4 hops successfully
// open("link4") must return a valid fd pointing to "file"</code></pre>

<p><strong>The Circle Test requirement:</strong></p>
<pre><code class="language-c">// Circle: linkA → linkB → linkA → ...
// Must return -1 (not hang)
// open("linkA") must return -1 after at most 10 hops</code></pre>

<p><strong>Minimum implementation that passes Chain but fails Circle:</strong></p>
<pre><code class="language-c">// Implementation with NO depth limit:
while(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)){
    char target[MAXPATH];
    int n = readi(ip, 0, (uint64)target, 0, MAXPATH);
    if(n <= 0){ iunlockput(ip); end_op(); return -1; }
    target[n] = 0;
    iunlockput(ip);
    ip = namei(target);
    if(ip == 0){ end_op(); return -1; }
    ilock(ip);
}
// Chain Test: follows 4 hops, exits when ip->type == T_FILE ✓ PASSES
// Circle Test: loops forever (linkA→linkB→linkA→...) → kernel stack overflow → PANIC ✗ FAILS</code></pre>

<p><strong>Another failing implementation — fixed iteration limit too low:</strong></p>
<pre><code class="language-c">// Implementation with depth limit of 3 (too small for Chain Test):
int depth = 0;
while(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)){
    if(depth >= 3){          // ← incorrectly rejects chains of length 3+
        iunlockput(ip); end_op(); return -1;
    }
    depth++;
    // ... follow link ...
}
// Chain Test (4 hops): on 4th hop depth=3, 3>=3 → return -1 ✗ FAILS
// Circle Test: returns -1 on 4th iteration ✓ PASSES (but Chain fails)</code></pre>

<p><strong>The implementation that passes BOTH:</strong></p>
<pre><code class="language-c">// Correct: depth > 10 allows exactly 10 successful hops
// Chain Test (4 hops): depth never exceeds 4 → PASSES
// Circle Test: depth reaches 11 on 12th entry → return -1 → PASSES</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

What is a "dangling symlink" in the context of symlinktest, and how would you write a minimal xv6 test that creates one and verifies the kernel handles it correctly?

<div class="answer-content">

<p><strong>What a dangling symlink is:</strong></p>
<p>A symlink whose target no longer exists. The symlink itself is intact (valid inode, valid target path string stored in data block), but the file at the stored path has been deleted. Opening a dangling symlink should return -1, not crash.</p>

<p><strong>Test program:</strong></p>
<pre><code class="language-c">// user/danglingtest.c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

int main(void)
{
    // Step 1: Create a target file
    int fd = open("ephemeral", O_CREATE|O_RDWR);
    write(fd, "data", 4);
    close(fd);

    // Step 2: Create a symlink to it
    symlink("ephemeral", "dangling_link");

    // Step 3: Verify the symlink works while target exists
    fd = open("dangling_link", O_RDONLY);
    if(fd < 0){
        printf("FAIL: link to existing file should work\n");
        exit(-1);
    }
    close(fd);

    // Step 4: Delete the target — creating a dangling symlink
    unlink("ephemeral");

    // Step 5: Try to open through the dangling symlink
    fd = open("dangling_link", O_RDONLY);
    if(fd != -1){
        printf("FAIL: dangling link should return -1, got fd=%d\n", fd);
        close(fd);
        exit(-1);
    }

    // Step 6: Verify the symlink itself still exists (just dangling)
    fd = open("dangling_link", O_NOFOLLOW|O_RDONLY);
    if(fd < 0){
        printf("FAIL: symlink itself should still be openable with O_NOFOLLOW\n");
        exit(-1);
    }
    char buf[64];
    int n = read(fd, buf, sizeof(buf));
    buf[n] = 0;
    // buf should contain "ephemeral" — the stored target path
    if(strcmp(buf, "ephemeral") != 0){
        printf("FAIL: wrong target stored: '%s'\n", buf);
        exit(-1);
    }
    close(fd);

    // Step 7: Clean up
    unlink("dangling_link");

    printf("danglingtest: all checks passed\n");
    exit(0);
}</code></pre>

<p><strong>What each step verifies:</strong></p>

<table>
  <tr><th>Step</th><th>Tests</th><th>Kernel path exercised</th></tr>
  <tr><td>3</td><td>Normal symlink following works</td><td>while loop follows 1 hop</td></tr>
  <tr><td>5</td><td>Dangling symlink returns -1</td><td>namei(target) returns 0 → return -1</td></tr>
  <tr><td>6</td><td>Symlink itself survives target deletion</td><td>O_NOFOLLOW opens T_SYMLINK inode directly</td></tr>
  <tr><td>6 (read)</td><td>Target path string is intact in symlink's data</td><td>readi() on T_SYMLINK inode returns correct bytes</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

If a student submits with the symlink-following loop in <code>sys_open()</code> but accidentally uses <code>depth >= 10</code> instead of <code>depth > 10</code>, which of the four symlinktest phases fail, which pass, and what is the resulting Gradescope score?

<div class="answer-content">

<p><strong>Analysis of depth >= 10 vs depth > 10:</strong></p>
<pre><code class="language-c">// depth >= 10: error fires when depth = 10 (on 11th loop entry)
//   → maximum 9 successful hops (depth goes 0→1→...→9 during hops 1-9)
//   → 10th hop: depth=9, 9>=10? No, depth++ → 10. Follow hop 10.
//   → 11th entry: depth=10, 10>=10? YES → return -1
//   Wait, let me retrace:
//   Entry 1: depth=0, 0>=10? No, depth++=1, follow
//   Entry 2: depth=1, 1>=10? No, depth++=2, follow
//   ...
//   Entry 10: depth=9, 9>=10? No, depth++=10, follow  ← follows 10th hop!
//   Entry 11: depth=10, 10>=10? YES → return -1
// So: MAXIMUM 10 successful hops. Error fires at 11th entry, SAME AS depth > 10.
// 
// WAIT — let me recount:
// depth > 10: error fires at depth=11 (after 11th depth++, i.e. 12th entry)
//   → 11 successful hops maximum
// depth >= 10: error fires at depth=10 (10th entry)
//   → 9 successful hops maximum (entry 10: depth=9 pre-check, ++ to 10, follows)
//   Actually: entry 10 has depth=9 BEFORE the check. 9>=10? No. depth++=10. Follow.
//   Entry 11: depth=10 BEFORE check. 10>=10? YES → return -1
//   So 10 successful hops maximum before error.</code></pre>

<p><strong>Test results:</strong></p>

<table>
  <tr><th>Test phase</th><th>Hops needed</th><th>depth >= 10 result</th><th>Pass/Fail</th></tr>
  <tr><td>Basic Link</td><td>1 hop</td><td>1 &lt; 10 → success</td><td>✓ PASS</td></tr>
  <tr><td>Chain Test (4 hops)</td><td>4 hops</td><td>4 &lt; 10 → success</td><td>✓ PASS</td></tr>
  <tr><td>Circle Test (needs -1)</td><td>≤ 10 hops then -1</td><td>Returns -1 after 10 hops</td><td>✓ PASS</td></tr>
  <tr><td>Concurrent Test</td><td>1 hop each</td><td>1 &lt; 10 → success</td><td>✓ PASS</td></tr>
</table>

<p><strong>Gradescope score with depth >= 10:</strong></p>
<p><strong>100/100</strong> — all four phases pass because the symlinktest chain is only 4 hops deep (well within both limits), and the circle test just needs any finite return. The difference between depth >= 10 and depth > 10 only matters for chains of exactly 11 hops, which symlinktest doesn't test.</p>

<p><strong>When depth >= 10 WOULD fail:</strong></p>
<p>A test that creates an 11-hop chain and expects it to succeed would fail with depth >= 10 (returns -1 at hop 11) but pass with depth > 10. The lab's symlinktest uses at most 4 hops, so this bug is undetected by the grader — it's a latent correctness issue, not a grader failure.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

A student passes all tests locally (<code>make grade: Score 100/100</code>) but fails on Gradescope. Describe five distinct causes for a local pass / Gradescope fail discrepancy, specific to this lab.

<div class="answer-content">

<p><strong>Cause 1 — Stale fs.img submitted:</strong></p>
<pre><code class="language-c">// Local: student ran "make clean && make grade" → rebuilt fs.img with FSSIZE=200000
// Submission: ran "make zipball" WITHOUT running make clean first
// In lab.zip: fs.img built with old FSSIZE=2000 but kernel compiled with 200000
// Gradescope: builds from source but fs.img is in the zip → uses stale fs.img
// Result: nmeta 46 not 70, disk too small → bigfile fails at block ~267
// Fix: always "make clean" before "make zipball"</code></pre>

<p><strong>Cause 2 — bigfile.c is the original infinite-loop version:</strong></p>
<pre><code class="language-c">// Local: student tested with the replaced bounded-for-loop bigfile.c → wrote 6580 blocks
// But: student also edited the original while(1) version for testing and forgot to replace it
// Gradescope: compiles student's while(1) bigfile.c → writes 65803 blocks
// Grader sees "wrote 65803 blocks" not "wrote 6580 blocks" → FAIL
// Fix: ensure user/bigfile.c has the #define TEST_BLOCKS (65803/10) version</code></pre>

<p><strong>Cause 3 — Missing log_write() causes silent corruption under testing load:</strong></p>
<pre><code class="language-c">// Local: single-run tests pass (buffer cache keeps dirty data in RAM, never crashes)
// Gradescope: runs multiple test suites back-to-back, may do more concurrent ops
// Under heavier load, the buffer cache evicts dirty pages that were never log_write()'d
// On the next access, the evicted blocks revert to pre-modification state on disk
// Doubly-indirect blocks appear to be zero → bmap() allocates new blocks, wrong data returned
// Fix: ensure log_write(bp) is called after every master map and secondary map modification</code></pre>

<p><strong>Cause 4 — addrs[] size mismatch only breaks on file sizes > 11 blocks:</strong></p>
<pre><code class="language-c">// Local: student tested with small files in manual testing
// struct inode has addrs[NDIRECT+1] (old), struct dinode has addrs[NDIRECT+2] (new)
// bigfile writes 6580 blocks → all go through doubly-indirect
// With the mismatch: memmove copies only 12 entries, ip->addrs[12] = ip->dev
// bmap(ip, 267) follows ip->dev as a block address → garbage → panic
// But on a lucky local run with a specific QEMU version: might not panic (UB)
// Gradescope: different QEMU version/platform → immediately panics
// Fix: grep both fs.h and file.h for "addrs" and verify both say NDIRECT+2</code></pre>

<p><strong>Cause 5 — itrunc() doubly-indirect cleanup missing (local runs bigfile once, Gradescope runs it multiple times):</strong></p>
<pre><code class="language-c">// Local: student runs bigfile once → passes (no disk exhaustion yet)
// Gradescope: grader may run bigfile multiple times to check idempotence
// Or: grade-lab-fs runs symlinktest AFTER bigfile in the same xv6 session
// Without itrunc() fix: first bigfile run leaks 6340 blocks
// 199930 - 6340 = 193590 remaining → second bigfile run still passes
// But if grader boots xv6, runs bigfile, then creates many symlinks → total 6340+symlink_blocks
// Eventually: balloc() fails at ~31 runs × 6340 = 196540 blocks leaked
// Fix: implement the complete doubly-indirect cleanup in itrunc()</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Design a comprehensive xv6 user-space test program that checks all four symlinktest phases (Basic, Concurrent, Chain, Circle) PLUS two additional edge cases not covered by the lab's symlinktest: opening a symlink to a device file, and a symlink chain that ends in a directory.

<div class="answer-content">

<p><strong>The extended test program:</strong></p>
<pre><code class="language-c">// user/extlinktest.c — extended symlink tests
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fcntl.h"

static int pass = 0, fail = 0;

void check(char *name, int cond) {
    if(cond){ pass++; printf("%s: ok\n", name); }
    else     { fail++; printf("%s: FAILED\n", name); }
}

int main(void)
{
    int fd;
    char buf[64];

    // ── Phase 1: Basic Link ──
    fd = open("realfile", O_CREATE|O_RDWR);
    write(fd, "hello", 5); close(fd);
    symlink("realfile", "basiclink");
    fd = open("basiclink", O_RDONLY);
    check("basic link opens", fd >= 0);
    if(fd >= 0){
        int n = read(fd, buf, 5);
        buf[n] = 0;
        check("basic link reads correct data", strcmp(buf, "hello") == 0);
        close(fd);
    }
    unlink("basiclink"); unlink("realfile");

    // ── Phase 2: O_NOFOLLOW ──
    fd = open("target2", O_CREATE|O_RDWR);
    write(fd, "world", 5); close(fd);
    symlink("target2", "nofollowlink");
    fd = open("nofollowlink", O_NOFOLLOW|O_RDONLY);
    check("O_NOFOLLOW opens symlink itself", fd >= 0);
    if(fd >= 0){
        int n = read(fd, buf, 64);
        buf[n] = 0;
        check("O_NOFOLLOW reads path not data", strcmp(buf, "target2") == 0);
        close(fd);
    }
    unlink("nofollowlink"); unlink("target2");

    // ── Phase 3: 4-hop Chain ──
    fd = open("chainfile", O_CREATE|O_RDWR);
    write(fd, "chain", 5); close(fd);
    symlink("chainfile", "c1");
    symlink("c1", "c2");
    symlink("c2", "c3");
    symlink("c3", "c4");
    fd = open("c4", O_RDONLY);
    check("4-hop chain opens", fd >= 0);
    if(fd >= 0){ close(fd); }
    unlink("c4"); unlink("c3"); unlink("c2"); unlink("c1"); unlink("chainfile");

    // ── Phase 4: Circle (must return -1) ──
    symlink("cirlcB", "circleA");
    symlink("circleA", "circleB");
    fd = open("circleA", O_RDONLY);
    check("circular link returns -1", fd == -1);
    if(fd >= 0) close(fd);
    unlink("circleA"); unlink("circleB");

    // ── Edge case 1: Symlink to a device file ──
    // /dev/console exists as T_DEVICE with major=1
    symlink("console", "devlink");
    fd = open("devlink", O_RDONLY);
    // Opening a symlink to a device: should succeed and return device fd
    // Reading from console might block, so just verify fd is valid
    check("symlink to device file opens", fd >= 0);
    if(fd >= 0) close(fd);
    unlink("devlink");

    // ── Edge case 2: Symlink chain ending in a directory ──
    mkdir("testdir");
    fd = open("testdir/subfile", O_CREATE|O_RDWR);
    write(fd, "indir", 5); close(fd);

    symlink("testdir", "dirlink");    // dirlink → testdir (a directory)
    // Opening a symlink to a directory for reading should succeed (O_RDONLY)
    fd = open("dirlink", O_RDONLY);
    check("symlink to directory opens (O_RDONLY)", fd >= 0);
    if(fd >= 0) close(fd);

    // Opening a symlink to a directory for writing should fail (T_DIR check)
    fd = open("dirlink", O_RDWR);
    check("symlink to directory fails (O_RDWR)", fd == -1);
    if(fd >= 0) close(fd);

    unlink("testdir/subfile");
    unlink("dirlink");
    // Note: rmdir not available in basic xv6, just leave testdir

    printf("\nextlinktest: %d passed, %d failed\n", pass, fail);
    exit(fail > 0 ? -1 : 0);
}</code></pre>

<p><strong>What the two extra edge cases test:</strong></p>

<table>
  <tr><th>Edge case</th><th>Tests</th><th>What would fail</th></tr>
  <tr><td>Symlink to device file</td><td>The while loop exits when ip->type == T_DEVICE (not just T_FILE), allowing device I/O through a symlink</td><td>If the loop only exits on T_FILE, a symlink to a device would loop forever (T_DEVICE ≠ T_FILE but also ≠ T_SYMLINK → loop exits correctly)</td></tr>
  <tr><td>Symlink to directory (read)</td><td>Verifies symlink resolution gives the directory inode to the T_DIR check, not the symlink inode</td><td>If T_DIR check runs before symlink resolution, dirlink would be type T_SYMLINK ≠ T_DIR → passes incorrectly with O_RDWR</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

The lab PDF's flow diagram shows a branch: "Is O_NOFOLLOW set? Yes → Proceed with normal open." Explain exactly what "proceed with normal open" means for a T_SYMLINK inode — what does the user get, can they read from it, and what happens to the T_DIR check with this inode type?

<div class="answer-content">

<p><strong>What "proceed with normal open" means for T_SYMLINK + O_NOFOLLOW:</strong></p>
<pre><code class="language-c">// The while loop condition:
while(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)){
    // ... follow the link ...
}
// With O_NOFOLLOW: !(omode & O_NOFOLLOW) = false → loop NEVER executes

// ip remains pointing to the T_SYMLINK inode (never followed)
// Execution falls through to:</code></pre>

<p><strong>The T_DIR check with a T_SYMLINK inode:</strong></p>
<pre><code class="language-c">if(ip->type == T_DIR && omode != O_RDONLY){
    iunlockput(ip); end_op(); return -1;
}
// ip->type == T_SYMLINK ≠ T_DIR → condition is FALSE → NO ERROR
// The T_DIR check passes trivially for T_SYMLINK inodes.
// Even with O_RDWR, O_WRONLY: the T_DIR check does not block a T_SYMLINK open.</code></pre>

<p><strong>What the user receives:</strong></p>
<pre><code class="language-c">// After the T_DIR check, filealloc() creates:
f->type     = FD_INODE
f->ip       = ip          // points to the T_SYMLINK inode
f->off      = 0
f->readable = (omode & O_RDONLY || omode & O_RDWR)
f->writable = (omode & O_WRONLY || omode & O_RDWR)

// The user gets a file descriptor pointing to the SYMLINK INODE ITSELF</code></pre>

<p><strong>Can the user read from it?</strong></p>
<pre><code class="language-c">// read(fd, buf, n):
// fileread() → readi(ip, 1, buf, f->off, n)
// ip is the T_SYMLINK inode
// ip->size = strlen(target_path)  (e.g. 11 for "/usr/bin/sh")
// ip->addrs[0] = the data block containing the path string

// read() returns the TARGET PATH STRING:
// buf = "/usr/bin/sh\0"  (or whatever path was stored by sys_symlink)
// n   = 11

// This is EXACTLY what readlink() functionality provides on Linux.
// In xv6 there is no readlink() syscall — O_NOFOLLOW + read() achieves the same effect.</code></pre>

<p><strong>Can the user write to it?</strong></p>
<pre><code class="language-c">// write(fd_nofollow, "newpath", 7) with f->writable=1:
// writei(ip, 1, "newpath", 0, 7)
// Overwrites the target path string in the symlink's data block!
// After this write: open("mylink", O_RDONLY) would follow "newpath" instead

// This is POWERFUL and DANGEROUS:
// A process with write access to a symlink can change where it points.
// xv6 does not implement permission checking on symlinks — any process can do this.
// In a real OS (Linux): symlinks always have mode 0777 but are owned by creator;
// you can only delete (unlink) a symlink if you own the directory,
// and you cannot "overwrite" a symlink's target via open — you must unlink and recreate.</code></pre>

<p><strong>Summary of what O_NOFOLLOW + T_SYMLINK gives you:</strong></p>

<table>
  <tr><th>Operation</th><th>Result</th></tr>
  <tr><td><code>open("link", O_NOFOLLOW|O_RDONLY)</code></td><td>fd to the symlink inode itself; read returns the stored target path</td></tr>
  <tr><td><code>open("link", O_NOFOLLOW|O_WRONLY)</code></td><td>fd to the symlink; write overwrites the stored target path (changes where it points)</td></tr>
  <tr><td><code>fstat(fd_nofollow, &st)</code></td><td>st.type = T_SYMLINK(4), st.size = strlen(target_path)</td></tr>
  <tr><td><code>open("link", O_NOFOLLOW|O_RDONLY)</code> then delete target</td><td>fd remains valid — data still accessible via symlink's own data block</td></tr>
</table>

</div>
</div>
