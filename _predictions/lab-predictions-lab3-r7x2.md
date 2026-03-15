---
layout: page
title: "Lab Predictions - lab3-r7x2"
lab: lab3
description: "Exam predictions for Lab3-w8: bigfile.c source code analysis, for vs while loop, exit codes, cc variable, and BSIZE access."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

<code>bigfile.c</code> includes <code>"kernel/fs.h"</code>. What does this give the user-space program access to, and why does it need it?

<div class="answer-content">

<p><strong>The includes in bigfile.c:</strong></p>
<pre><code class="language-c">#include "kernel/types.h"    // uint, uchar, etc.
#include "kernel/stat.h"     // T_FILE, T_DIR, struct stat
#include "user/user.h"       // open(), write(), close(), exit(), printf()
#include "kernel/fcntl.h"    // O_CREATE, O_WRONLY, O_RDONLY flags
#include "kernel/fs.h"       // ← BSIZE, NDIRECT, NINDIRECT, MAXFILE, struct dinode</code></pre>

<p><strong>What <code>kernel/fs.h</code> provides that bigfile.c needs:</strong></p>
<pre><code class="language-c">// From kernel/fs.h:
#define BSIZE 1024   // block size in bytes

// bigfile.c uses it here:
char buf[BSIZE];                    // declare a 1024-byte buffer
write(fd, buf, sizeof(buf));        // write exactly one block at a time
// sizeof(buf) = sizeof(char[BSIZE]) = 1024</code></pre>

<p><strong>Why BSIZE must come from the kernel header:</strong></p>
<p>BSIZE is not a standard C constant — it is specific to xv6's file system design. If a future version of xv6 changed the block size, every program that uses BSIZE from the header would automatically pick up the new value without recompilation. If bigfile.c hardcoded <code>1024</code> instead of <code>BSIZE</code>, it would break silently if the kernel ever changed block sizes.</p>

<p><strong>What bigfile.c does NOT use from kernel/fs.h:</strong></p>
<p>The struct definitions (<code>struct dinode</code>, <code>struct buf</code>) and most constants (NDIRECT, NINDIRECT) are not directly used by bigfile.c — it operates at the system call level (<code>write()</code>, <code>open()</code>) and does not manipulate kernel data structures directly. BSIZE is the only constant it needs from <code>kernel/fs.h</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What is the <code>cc</code> variable in <code>bigfile.c</code> and why is the condition <code>if(cc &lt;= 0)</code> rather than <code>if(cc &lt; 0)</code>?

<div class="answer-content">

<p><strong>The cc variable:</strong></p>
<pre><code class="language-c">for (i = 0; i &lt; TEST_BLOCKS; i++){
    *(int*)buf = blocks;
    int cc = write(fd, buf, sizeof(buf));   // cc = return value of write()
    if(cc &lt;= 0)
        break;
    blocks++;
    if (blocks % 100 == 0)
        printf(".");
}</code></pre>

<p><code>cc</code> stands for "character count" (the traditional POSIX name for the write return value). It holds the number of bytes actually written by the <code>write()</code> system call.</p>

<p><strong>What write() returns:</strong></p>

<table>
  <tr><th>Return value</th><th>Meaning</th></tr>
  <tr><td><code>BSIZE</code> (1024)</td><td>Full block written successfully — normal case</td></tr>
  <tr><td><code>0</code></td><td>Disk full or maximum file size reached — no bytes written</td></tr>
  <tr><td><code>-1</code></td><td>Error (bad fd, permission denied, etc.)</td></tr>
  <tr><td><code>1..1023</code></td><td>Partial write — possible but unusual for local file systems</td></tr>
</table>

<p><strong>Why <code>&lt;= 0</code> catches both failure cases:</strong></p>
<pre><code class="language-c">if(cc <= 0) break;
// cc == 0: disk full / max file size → stop writing (this is the expected failure mode)
// cc < 0:  error → stop writing (defensive: should not normally happen on a valid fd)

// If the condition were cc < 0 only:
//   When bmap() returns 0 (doubly-indirect not implemented), write() returns 0
//   cc = 0, which is NOT < 0 → loop continues
//   Next write() also returns 0 → infinite loop! Program never exits.
//   (In practice bigfile.c uses a bounded for loop, but the cc <= 0 guard is still correct)</code></pre>

<p>The <code>&lt;= 0</code> condition is the standard defensive pattern for write loops — treat both 0 and negative as "stop writing."</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

Explain the difference between <code>exit(-1)</code> and <code>exit(0)</code> in <code>bigfile.c</code>. What does the shell use these for?

<div class="answer-content">

<p><strong>The three exit points in bigfile.c:</strong></p>
<pre><code class="language-c">// Exit 1: cannot open the file
fd = open("big.file", O_CREATE | O_WRONLY);
if(fd < 0){
    printf("bigfile: cannot open big.file for writing\n");
    exit(-1);   // ← failure: could not create file
}

// Exit 2: wrote fewer blocks than expected
if(blocks != TEST_BLOCKS) {
    printf("bigfile: file is too small\n");
    close(fd);
    exit(-1);   // ← failure: file system limit hit
}

// Exit 3: success
close(fd);
printf("bigfile done; ok\n");
exit(0);        // ← success</code></pre>

<p><strong>What the shell uses these for:</strong></p>
<p>In xv6, the parent process (the shell) calls <code>wait(&status)</code> after <code>fork()</code>ing a child. The exit code becomes the child's status. The grader script that runs bigfile captures whether the process exited successfully (0) or with an error (non-zero):</p>

<table>
  <tr><th>Exit code</th><th>Meaning</th><th>What the grader sees</th></tr>
  <tr><td><code>0</code></td><td>Success — wrote exactly 6580 blocks</td><td>Test passes</td></tr>
  <tr><td><code>-1</code></td><td>Failure — file system limit hit or open failed</td><td>Test fails</td></tr>
</table>

<p><strong>Why -1 rather than 1:</strong></p>
<p>On most systems, exit codes are unsigned 8-bit values (0–255). <code>exit(-1)</code> is typically stored as <code>255</code> (two's complement wrap). POSIX convention uses 0 for success and any non-zero for failure — the specific non-zero value is implementation-defined. xv6's grader checks for the presence of success strings in the output rather than the exit code directly.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

Why does <code>bigfile.c</code> use a <code>for</code> loop bounded by <code>TEST_BLOCKS</code> instead of the original <code>while(1)</code> infinite loop? What would happen if the original infinite loop were used with the completed implementation?

<div class="answer-content">

<p><strong>The bounded for loop (lab version):</strong></p>
<pre><code class="language-c">#define TEST_BLOCKS (65803/10)   // = 6580

blocks = 0;
for (i = 0; i &lt; TEST_BLOCKS; i++){    // stops at 6580
    *(int*)buf = blocks;
    int cc = write(fd, buf, sizeof(buf));
    if(cc &lt;= 0) break;
    blocks++;
    if (blocks % 100 == 0) printf(".");
}

printf("\nwrote %d blocks\n", blocks);
if(blocks != TEST_BLOCKS) {
    printf("bigfile: file is too small\n");
    exit(-1);
}</code></pre>

<p><strong>What the original infinite loop would do with the completed implementation:</strong></p>
<pre><code class="language-c">// Original while(1) loop:
while(1){
    write(fd, buf, sizeof(buf));
    // ...
}
// With a correct doubly-indirect implementation:
// write() succeeds for blocks 0 through 65802 (MAXFILE-1 = 65802)
// Block 65803: bmap() hits panic("bmap: out of range") → kernel panic OR
//              write() returns 0 → loop exits naturally
// blocks = 65803 (or close to it)
// Output: "wrote 65803 blocks"   ← NOT "wrote 6580 blocks"
// Grader string match FAILS → test marked as FAIL even though implementation is correct!</code></pre>

<p><strong>Time comparison:</strong></p>

<table>
  <tr><th>Loop type</th><th>Blocks written</th><th>Approx time</th><th>Grader result</th></tr>
  <tr><td>Bounded for (lab)</td><td>6,580</td><td>~1–3 minutes</td><td>PASS (matches "wrote 6580 blocks")</td></tr>
  <tr><td>Infinite while (original)</td><td>65,803</td><td>2–35 minutes</td><td>FAIL (wrong string) or TIMEOUT</td></tr>
</table>

<p>The lab PDF explicitly warns: "stress testing with maximum blocks can take any time between 2 to 35 minutes." The Gradescope timeout is much shorter — hence the one-tenth restriction and the bounded loop.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

<code>bigfile.c</code> declares <code>int fd, i, blocks;</code> as separate variables. Trace exactly what value each variable holds at the moment <code>printf("\nwrote %d blocks\n", blocks)</code> executes in both the failure case (original xv6) and the success case (completed lab).

<div class="answer-content">

<p><strong>Variable roles:</strong></p>
<pre><code class="language-c">int fd;      // file descriptor returned by open()
int i;       // loop counter (counts iterations, not blocks — they diverge if break fires)
int blocks;  // actual number of successfully written blocks</code></pre>

<p><strong>Failure case (original xv6, NDIRECT=12, no doubly-indirect):</strong></p>
<pre><code class="language-c">fd = open("big.file", O_CREATE | O_WRONLY);
// fd = 3 (or next available fd after stdin=0, stdout=1, stderr=2)

// Loop runs i=0..267, then write() returns 0 at i=268:
//   i=0:   write succeeds, blocks++ → blocks=1
//   i=1:   write succeeds, blocks++ → blocks=2
//   ...
//   i=267: write succeeds, blocks++ → blocks=268
//   i=268: write returns 0, break fires

// At printf:
// fd     = 3         (still valid — not closed yet)
// i      = 268       (loop counter at the iteration where break fired)
// blocks = 268       (successful write count)

printf("\nwrote %d blocks\n", blocks);   // "wrote 268 blocks"
// blocks(268) != TEST_BLOCKS(6580) → exit(-1)</code></pre>

<p><strong>Success case (completed lab, doubly-indirect implemented):</strong></p>
<pre><code class="language-c">// Loop runs i=0..6579 (all 6580 iterations complete without cc<=0):
//   i=0..6579: write succeeds each time
//   i=6579 (last): write succeeds, blocks++ → blocks=6580
//   Loop exits because i=6580 is NOT < TEST_BLOCKS(6580)? Wait — for(i=0; i<6580; i++) runs when i=0..6579 → 6580 iterations total

// At printf:
// fd     = 3
// i      = 6580      (for loop exits when i reaches TEST_BLOCKS)
// blocks = 6580      (all writes succeeded)

printf("\nwrote %d blocks\n", blocks);   // "wrote 6580 blocks"
// blocks(6580) == TEST_BLOCKS(6580) → success branch
close(fd);
printf("bigfile done; ok\n");
exit(0);</code></pre>

<p><strong>Key distinction:</strong> <code>i</code> and <code>blocks</code> stay in sync only when no write fails mid-loop. If <code>write()</code> returns 0 at iteration <code>i=K</code>, then <code>i=K</code> but <code>blocks=K</code> (blocks is incremented only after a successful write). In normal operation they always agree.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

The lab PDF says the original <code>bigfile</code> test "enters an infinite while loop to write data." The provided <code>bigfile.c</code> uses a bounded <code>for</code> loop. What does this tell you about what the lab PDF was describing vs. the actual file you must use, and what would happen if you submitted with the original infinite-loop version?

<div class="answer-content">

<p><strong>What the lab PDF was describing:</strong></p>
<p>The lab PDF describes running <code>bigfile</code> <em>before</em> any changes are made, to observe the failure. The original unmodified <code>bigfile.c</code> (provided in the xv6 base repo) uses an infinite <code>while(1)</code> loop. The lab PDF's description of "enters an infinite while loop" refers to this original version.</p>

<p><strong>The actual bigfile.c you must use:</strong></p>
<p>The lab provides a <em>replacement</em> <code>bigfile.c</code> (the file uploaded in this session) which uses the bounded <code>for</code> loop with <code>#define TEST_BLOCKS (65803/10)</code>. The lab PDF's note says: "we restrict <code>bigfile.c</code> to stress to one-tenth of the theoretical maximum." This replacement file is what the Gradescope grader expects.</p>

<p><strong>What happens if you submit with the original infinite-loop version:</strong></p>
<pre><code class="language-c">// Original while(1) bigfile.c behaviour on Gradescope:
// 1. Writes all 65,803 blocks successfully (doubly-indirect works)
// 2. Prints: "wrote 65803 blocks"
// 3. Grader checks for "wrote 6580 blocks" → NOT FOUND → FAIL
// 4. Or: takes 30+ minutes → Gradescope timeout → auto-FAIL

// Grade received: 0/50 for the bigfile portion
// Even though your kernel implementation is perfectly correct!</code></pre>

<p><strong>The two things that must match the grader exactly:</strong></p>

<table>
  <tr><th>Required string</th><th>Generated by</th><th>Requires</th></tr>
  <tr><td><code>wrote 6580 blocks</code></td><td><code>printf("\nwrote %d blocks\n", blocks)</code></td><td>Bounded for loop + TEST_BLOCKS = 65803/10 = 6580</td></tr>
  <tr><td><code>bigfile done; ok</code></td><td><code>printf("bigfile done; ok\n")</code></td><td>blocks == TEST_BLOCKS check must pass</td></tr>
</table>

<p>This is why the lab PDF shows a screenshot of the replacement <code>bigfile.c</code> with the <code>#define TEST_BLOCKS (65803/10)</code> line highlighted — it is the critical change that makes the test Gradescope-compatible.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

The <code>bigfile.c</code> opens the file with <code>O_CREATE | O_WRONLY</code>. What exactly do these two flags do in <code>sys_open()</code>, and what would happen if <code>O_CREATE</code> were omitted?

<div class="answer-content">

<p><strong>What each flag does in sys_open():</strong></p>
<pre><code class="language-c">fd = open("big.file", O_CREATE | O_WRONLY);
// O_CREATE = 0x200  — create the file if it does not exist
// O_WRONLY = 0x001  — open for write access only
// Combined: 0x201</code></pre>

<p><strong>O_CREATE behaviour in sys_open():</strong></p>
<pre><code class="language-c">// In sys_open():
if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);   // allocate new inode if "big.file" does not exist
                                        // if "big.file" exists: call itrunc() and return existing inode
    if(ip == 0){ end_op(); return -1; }
}</code></pre>

<p><code>O_CREATE</code> means: "create the file if absent; truncate it to zero if present." This is why running <code>bigfile</code> twice in a row works — the second run truncates the file from the first run and starts fresh. Without O_CREATE, the second run would also work (the else branch handles existing files). The first run would fail without O_CREATE if "big.file" doesn't exist yet.</p>

<p><strong>What happens if O_CREATE is omitted:</strong></p>
<pre><code class="language-c">fd = open("big.file", O_WRONLY);   // O_CREATE not set
// sys_open() goes to the else branch:
//   ip = namei("big.file");
//   if(ip == 0){ end_op(); return -1; }  ← file doesn't exist → returns -1!

// bigfile.c then hits:
if(fd < 0){
    printf("bigfile: cannot open big.file for writing\n");
    exit(-1);
}</code></pre>

<p>On the very first run, "big.file" does not exist — <code>namei("big.file")</code> returns 0 → <code>open()</code> returns -1 → bigfile exits immediately with the error message. On subsequent runs (after at least one successful O_CREATE run), the file exists and O_WRONLY alone would open it for writing at position 0, overwriting existing content — functionally similar to O_CREATE|O_TRUNC, but only by coincidence.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

<code>bigfile.c</code> checks <code>if(blocks != TEST_BLOCKS)</code> after the loop. Why is this check necessary given that the loop already has <code>if(cc &lt;= 0) break</code>? What scenario would cause <code>blocks != TEST_BLOCKS</code> even without the <code>break</code> triggering?

<div class="answer-content">

<p><strong>The check and the break — two independent guards:</strong></p>
<pre><code class="language-c">for (i = 0; i &lt; TEST_BLOCKS; i++){
    *(int*)buf = blocks;
    int cc = write(fd, buf, sizeof(buf));
    if(cc &lt;= 0)
        break;             // ← stops the loop early on write failure
    blocks++;
    if (blocks % 100 == 0) printf(".");
}

printf("\nwrote %d blocks\n", blocks);
if(blocks != TEST_BLOCKS) {     // ← final correctness check
    printf("bigfile: file is too small\n");
    close(fd); exit(-1);
}</code></pre>

<p><strong>Why the check is necessary even with the break:</strong></p>
<p>The <code>break</code> fires when <code>cc &lt;= 0</code>. After <code>break</code>, the program reaches <code>printf</code> with <code>blocks &lt; TEST_BLOCKS</code>. Without the <code>if(blocks != TEST_BLOCKS)</code> check, the program would print <code>"wrote 268 blocks"</code> and then proceed to <code>printf("bigfile done; ok\n")</code> and <code>exit(0)</code> — reporting success despite writing far fewer blocks than required.</p>

<p><strong>Scenario: partial write success without break triggering:</strong></p>
<pre><code class="language-c">// Hypothetical: write() returns a partial count (1..1023 bytes) instead of BSIZE
// cc = 512 (positive but not BSIZE)
// cc > 0 → break does NOT fire
// blocks++ increments normally
// But only 512 bytes were written, not 1024

// In this case:
//   i and blocks still both reach TEST_BLOCKS
//   blocks == TEST_BLOCKS → final check PASSES
// But the file only has 512 × 6580 = ~3 MB instead of 1024 × 6580 = ~6 MB

// The check does NOT catch partial writes — it only checks COUNT, not content.
// For this lab, write() always writes 0 or BSIZE bytes (no partial writes in xv6's writei())
// So in practice, break and the final check always agree.</code></pre>

<p><strong>The real purpose of the check:</strong></p>
<p>The final check is the <strong>assertion</strong> that makes bigfile.c behave as a proper test program. It separates "the program ran" from "the program validated its expectation." The break is just loop control. The final check is what makes bigfile.c emit the grader-detectable success/failure strings.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

The lab PDF's index1 example says: "I want block 500" then "500/256 = 1.95 ~= 1". Explain why C integer division gives 1 (not 2), trace all the way from the user's <code>write()</code> call to the final physical disk block address for logical block 500, including every subtraction step and both index calculations.

<div class="answer-content">

<p><strong>Why C integer division gives 1, not 2:</strong></p>
<pre><code class="language-c">// C integer division truncates toward zero (floor division for positive numbers):
500 / 256 = 1      // NOT 2 — the decimal .95 is discarded
// This is correct: idx1=1 means "use secondary map slot #1"
// Secondary map 0 covers bn 0..255 (256 entries)
// Secondary map 1 covers bn 256..511 (256 entries) ← bn=500 falls here</code></pre>

<p><strong>Full trace from write(500th block) to physical disk block:</strong></p>
<pre><code class="language-c">// User program: write(fd, buf, BSIZE) for the 501st time (0-indexed block 500)
// This reaches writei() which calls bmap(ip, 500):

// Step 1: Is it direct? bn = 500 < NDIRECT(11)? 500 < 11? No.
// Step 2: Subtract direct range:
bn = 500 - NDIRECT = 500 - 11 = 489

// Step 3: Is it singly-indirect? 489 < NINDIRECT(256)? 489 < 256? No.
// Step 4: Subtract singly-indirect range:
bn = 489 - NINDIRECT = 489 - 256 = 233
// bn is now the LOCAL offset within the doubly-indirect range

// Step 5: Is it doubly-indirect? 233 < NDINDIRECT(65536)? Yes.

// Step 6: Calculate indices:
idx1 = 233 / 256 = 0    // integer division: 0 remainder 233
idx2 = 233 % 256 = 233

// (Note: the lab PDF example uses the UNSUBSTRACTED bn=500 for its example.
// The actual code subtracts NDIRECT first, then NINDIRECT, giving bn=233 not 500.
// The lab PDF's "500/256=1" skips the subtraction steps for illustration purposes.)

// Step 7: Navigate the master map:
addr = ip->addrs[NDIRECT+1];          // = ip->addrs[12] (doubly-indirect slot)
if(addr == 0): addr = balloc(dev);    // allocate if not yet created
ip->addrs[12] = addr;                 // e.g. disk block 500

bp = bread(dev, 500);                 // load master map block
a  = (uint*)bp->data;                 // a[] = 256-entry array of secondary map addresses

// Step 8: Navigate to secondary map:
if((a[idx1=0]) == 0): a[0] = balloc(dev);  // allocate secondary map if needed, e.g. block 501
log_write(bp);
brelse(bp);                           // RELEASE master map BEFORE loading secondary

bp = bread(dev, 501);                 // load secondary map #0
a  = (uint*)bp->data;

// Step 9: Find/allocate the data block:
if((a[idx2=233]) == 0): a[233] = balloc(dev); // allocate data block, e.g. block 760
log_write(bp);
brelse(bp);                           // release secondary map

return 760;                           // physical block for logical block 500</code></pre>

<p><strong>Why the lab PDF example says 500/256=1 while the code gives idx1=0:</strong></p>
<p>The lab PDF illustrates index calculation using the raw <code>bn=500</code> before subtraction, purely for pedagogical clarity. The real code subtracts <code>NDIRECT</code> and <code>NINDIRECT</code> first (giving 233), then divides. For <code>bn=500</code>: 233/256=0 (idx1=0), 233%256=233 (idx2=233). The lab PDF's shortcut of 500/256=1 is an approximation to explain the concept, not the actual code.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

<code>bigfile.c</code> writes <code>*(int*)buf = blocks</code> before every write, but never reads back to verify. Design the complete code to add a read-back verification pass to <code>bigfile.c</code> using only the xv6 system calls available (<code>open</code>, <code>read</code>, <code>close</code>, <code>unlink</code>), and explain which implementation bugs this would catch that the current write-only test misses.

<div class="answer-content">

<p><strong>The complete verification pass to add after the write loop:</strong></p>
<pre><code class="language-c">// After: close(fd); printf("bigfile done; ok\n");
// ADD before exit(0):

// ── Verification pass ──
fd = open("big.file", O_RDONLY);
if(fd < 0){
    printf("bigfile: cannot reopen for read\n");
    exit(-1);
}

char rbuf[BSIZE];
int verify_errors = 0;

for(int b = 0; b < TEST_BLOCKS; b++){
    int n = read(fd, rbuf, BSIZE);
    if(n != BSIZE){
        printf("bigfile: short read at block %d\n", b);
        verify_errors++;
        break;
    }
    int stored_index = *(int*)rbuf;
    if(stored_index != b){
        printf("bigfile: block %d has wrong index: expected %d got %d\n",
               b, b, stored_index);
        verify_errors++;
        if(verify_errors >= 5) break;   // stop after 5 to avoid flooding
    }
}

close(fd);

if(verify_errors > 0){
    printf("bigfile: VERIFY FAILED with %d errors\n", verify_errors);
    exit(-1);
}
printf("bigfile: verify ok\n");
exit(0);</code></pre>

<p><strong>Bugs the current write-only test misses but this verification would catch:</strong></p>

<table>
  <tr><th>Bug</th><th>Write-only behaviour</th><th>Read-back catches it?</th></tr>
  <tr><td>Wrong idx1/idx2 calculation (off-by-one)</td><td>Writes succeed (balloc always finds a free block)</td><td>Yes — block N reads back index M ≠ N</td></tr>
  <tr><td>Missing brelse(bp1) before bread(bp2)</td><td>Kernel panic — both tests fail</td><td>Panic before even reaching read</td></tr>
  <tr><td>Missing log_write() on secondary map</td><td>Writes succeed (cache keeps data in RAM)</td><td>Maybe — only catches it after a crash+reboot. The read-back in the same run finds cached (correct) data.</td></tr>
  <tr><td>Two different logical blocks mapping to the same physical block</td><td>Write succeeds — last write wins</td><td>Yes — one block reads back the other block's index</td></tr>
  <tr><td>Boundary error at bn=267 (first doubly-indirect)</td><td>Write at 267 succeeds, write at 268 also succeeds</td><td>Yes — if bmap maps 267 to the wrong level, block 267 reads back an incorrect index</td></tr>
  <tr><td>itrunc() not zeroing addrs[NDIRECT+1] (second run)</td><td>Second run succeeds (overwrites old blocks)</td><td>No difference — the new data is written over the old</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

The lab PDF says <code>bigfile.c</code>'s primary purpose is to "verify whether the kernel can handle a file significantly larger than the original 268-block limit." Given that <code>TEST_BLOCKS = 65803/10 = 6580</code>, at exactly which block numbers does bigfile exercise each of the three addressing levels, and what is the minimum value of TEST_BLOCKS that would still verify the doubly-indirect implementation is working?

<div class="answer-content">

<p><strong>Block number ranges for each addressing level:</strong></p>

<table>
  <tr><th>Addressing level</th><th>Logical block range</th><th>Count</th><th>bigfile exercises this?</th></tr>
  <tr><td>Direct (addrs[0..10])</td><td>0 – 10</td><td>11 blocks</td><td>Yes — blocks 0–10 in bigfile's loop</td></tr>
  <tr><td>Singly-indirect (addrs[11])</td><td>11 – 266</td><td>256 blocks</td><td>Yes — blocks 11–266</td></tr>
  <tr><td>Doubly-indirect (addrs[12])</td><td>267 – 65,802</td><td>65,536 blocks</td><td>Yes — blocks 267–6,579</td></tr>
</table>

<p><strong>Breaking down the 6,580 blocks bigfile writes:</strong></p>
<pre><code class="language-c">Blocks 0–10:    11 blocks  (direct — addrs[0..10])
Blocks 11–266: 256 blocks  (singly-indirect — addrs[11] → 256-entry map)
Blocks 267–6579: 6313 blocks  (doubly-indirect — addrs[12] → master → secondary → data)

// Doubly-indirect breakdown:
// Blocks 267–522:  256 blocks → all in secondary map #0  (idx1=0, idx2=0..255)
// Blocks 523–778:  256 blocks → all in secondary map #1  (idx1=1, idx2=0..255)
// ...
// Blocks 267 + 256*k through 267 + 256*(k+1)-1 → secondary map #k

// How many secondary maps does bigfile touch?
doubly_indirect_blocks = 6580 - 11 - 256 = 6313
secondary_maps_touched = ⌈6313 / 256⌉ = ⌈24.66⌉ = 25 secondary maps
// Secondary maps #0 through #23 are full (256 entries each = 6144 blocks)
// Secondary map #24 has 6313 - 6144 = 169 entries</code></pre>

<p><strong>Minimum TEST_BLOCKS to verify doubly-indirect works:</strong></p>
<pre><code class="language-c">// Minimum requirement: write at least one doubly-indirect block
// First doubly-indirect block is at logical block 267
// Therefore: TEST_BLOCKS >= 268

// But 268 only tests:
//   - Allocation of the master map block
//   - Allocation of secondary map #0
//   - Allocation of one data block (secondary map #0, slot 0)
// It does NOT test:
//   - Secondary map #0 being fully used (slot 255, block 267+255=522)
//   - Transition to secondary map #1 (block 523)
//   - Many secondary maps being allocated

// Minimum to also test the secondary map boundary (idx1 changes from 0 to 1):
// First block in secondary map #1 = 267 + 256 = 523
// TEST_BLOCKS >= 524 (to write block 523)

// Minimum to verify the complete calculation logic and one boundary:
// TEST_BLOCKS = 524  (11 direct + 256 singly + 257 doubly = 524 total)

// In practice the lab uses 6580 to:
// 1. Exercise 25 different secondary maps (tests idx1 calculation for idx1=0..24)
// 2. Give a large enough file that timing issues in block allocation are more likely to surface
// 3. Produce visible progress (65 dots) showing the test is actually running</code></pre>

<p>The absolute minimum to verify doubly-indirect works is <code>TEST_BLOCKS = 268</code> (just one doubly-indirect block). The lab uses 6580 to provide stronger verification and visible progress output.</p>

</div>
</div>
