---
layout: page
title: "Lab Predictions - lab3-j3w6"
lab: lab3
description: "Exam predictions for Lab3-w8: log layer internals, begin_op/end_op mechanics, crash recovery, and transaction size limits."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

What is the xv6 write-ahead log and why must <code>sys_symlink()</code> wrap all its disk writes inside <code>begin_op()</code>/<code>end_op()</code>?

<div class="answer-content">

<p><strong>What the write-ahead log (WAL) does:</strong></p>
<p>The WAL guarantees that a group of disk writes either all succeed or all fail — even if the machine crashes mid-operation. Without it, a partial update (e.g., inode created but data not yet written) would leave the filesystem permanently inconsistent.</p>

<p><strong>How it works in three phases:</strong></p>
<pre><code class="language-c">// Phase 1: begin_op() — reserve log space, start recording
begin_op();

// Phase 2: All disk writes go through log_write() — buffered in the log header
// log_write() does NOT write to the final disk location yet.
// It pins the buffer in cache and records its block number in the log.
log_write(inode_block);     // inode creation
log_write(bitmap_block);    // block allocation
log_write(data_block);      // target path string

// Phase 3: end_op() — atomic commit
// 3a: Write all dirty buffers to the log section on disk
// 3b: Write a commit record to disk (makes the transaction "permanent")
// 3c: Copy blocks from log to their real disk locations
// 3d: Clear the log
end_op();</code></pre>

<p><strong>Why sys_symlink() specifically needs this:</strong></p>
<p>Creating a symlink requires at minimum three disk writes that must all happen atomically:</p>
<ol>
  <li>Allocate a new inode (write inode block + bitmap block)</li>
  <li>Add a directory entry pointing to the new inode (write directory data block)</li>
  <li>Write the target path string into the symlink's data block</li>
</ol>
<p>If the machine crashes after step 1 but before step 3, the symlink inode exists on disk (type=T_SYMLINK) but contains no target path — reading it would return garbage. The log ensures all three writes commit together.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What is the log region on disk? Where is it located in the xv6 disk layout, and what does it contain during and after a committed transaction?

<div class="answer-content">

<p><strong>Log region location:</strong></p>
<pre><code class="language-c">// Disk layout (FSSIZE=200000):
// Block 0:  boot
// Block 1:  superblock  (sb.logstart = 2)
// Block 2:  log header block  ← start of log
// Block 3–31: log data blocks (LOGSIZE-1 = 29 blocks for data)
// Block 32: first inode block

// In kernel/fs.h:
// sb.logstart = 2
// LOGSIZE = 30 → blocks 2..31 (1 header + 29 data slots)</code></pre>

<p><strong>What the log contains during a transaction:</strong></p>
<pre><code class="language-c">// Log header (block 2):
struct logheader {
    int n;                  // number of committed blocks (0 = no committed transaction)
    int block[LOGSIZE-1];   // real disk block numbers for each logged block
};

// During begin_op()..end_op():
// In-memory log.lh.n grows as log_write() is called.
// The actual block data is in the buffer cache (not yet on log disk).

// When end_op() commits:
// 1. Writes each dirty buffer's data to log data blocks 3..31 on disk
// 2. Writes log header with n=count, block[]=real_addresses
// 3. Copies data from log blocks to real locations
// 4. Sets log header n=0 (clears commit record)</code></pre>

<p><strong>Log contents after a committed transaction:</strong></p>
<p>The log header has <code>n = 0</code> — indicating no pending committed-but-not-installed transaction. The log data blocks (3–31) still contain the old block data, but since <code>n = 0</code>, they are ignored on reboot. A clean log means either: (a) no crash occurred, or (b) recovery already replayed and cleared the log.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What happens when xv6 reboots after a crash that occurred while <code>end_op()</code> was running? Explain the two cases: crash before the commit record, and crash after the commit record.

<div class="answer-content">

<p><strong>How recovery works (kernel/log.c — initlog()):</strong></p>
<pre><code class="language-c">void initlog(int dev, struct superblock *sb) {
    // ...
    recover_from_log();
}

static void recover_from_log(void) {
    read_head();                  // read log header from disk
    install_trans(1);             // if n > 0: replay all logged blocks
    log.lh.n = 0;
    write_head();                 // clear the log header
}</code></pre>

<p><strong>Case 1 — Crash BEFORE commit record (n = 0 on disk):</strong></p>
<pre><code class="language-c">// During end_op(), the machine crashes before writing the log header with n > 0.
// On reboot: read_head() finds n = 0.
// install_trans(1): n=0 → no blocks to replay → does nothing.
// write_head(): writes n=0 (already 0 — no change).
// RESULT: All writes from the crashed transaction are DISCARDED.
// The filesystem is in the state it was in before begin_op() was called.
// The symlink was not created — clean state.</code></pre>

<p><strong>Case 2 — Crash AFTER commit record (n > 0 on disk):</strong></p>
<pre><code class="language-c">// The log header was written with n=3 (three blocks: inode, bitmap, data).
// The log data blocks contain the correct block contents.
// But the machine crashed before writing the blocks to their real locations.
// On reboot: read_head() finds n = 3.
// install_trans(1): for each of the 3 logged blocks:
//   bread(log block) → bread(real block) → memmove → bwrite(real block) → brelse both
// write_head(): writes n=0 (marks log as clean).
// RESULT: All three blocks are now at their real disk locations.
// The symlink is fully created — exactly as if no crash occurred.</code></pre>

<p><strong>The guarantee:</strong> Either the entire transaction is committed or none of it is. There is no in-between state after recovery.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

Why is <code>log_write(bp)</code> called after modifying a buffer in <code>bmap()</code>, but the data blocks that bigfile writes are NOT logged by <code>bmap()</code>? Where does the data block get logged?

<div class="answer-content">

<p><strong>What bmap() logs vs what it doesn't:</strong></p>
<pre><code class="language-c">// bmap() logs MAP BLOCK modifications only:
// When a new secondary map entry is written:
a[idx1] = addr_of_secondary_map;
log_write(bp);    // ← log the MASTER MAP modification

// When a new data block pointer is written to secondary map:
a[idx2] = addr_of_data_block;
log_write(bp);    // ← log the SECONDARY MAP modification

// bmap() does NOT log the data block itself:
return addr_of_data_block;  // just returns the block address</code></pre>

<p><strong>Where the data block gets logged:</strong></p>
<pre><code class="language-c">// The call chain for write():
sys_write()
  → filewrite(f, addr, n)
    → writei(ip, 1, addr, off, n)
      → bmap(ip, off/BSIZE)       // returns physical block number
      → bp = bread(dev, phys_block)
      → memmove(bp->data, ...)    // copy user data into buffer
      → log_write(bp)             // ← DATA BLOCK is logged HERE, in writei()
      → brelse(bp)
  → end_op()  ← commits everything</code></pre>

<p><strong>Why this separation makes sense:</strong></p>
<p>bmap()'s job is to find/allocate the block and update the pointer structures (map blocks). writei()'s job is to put user data into the block. Logging the data block in writei() keeps bmap() focused on navigation and keeps the logging responsibility with the code that actually writes user data. The map blocks and data blocks are all committed together in the same <code>end_op()</code> transaction.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

The log has LOGSIZE=30 slots. How many <code>log_write()</code> calls does a single <code>write()</code> system call on big.file potentially make, and how does xv6 ensure this never exceeds LOGSIZE?

<div class="answer-content">

<p><strong>Maximum log_write() calls for one write() at a cold doubly-indirect block:</strong></p>
<pre><code class="language-c">// All log_write() calls triggered by one write(fd, buf, BSIZE):

// From balloc() for master map block:
//   bitmap_block for master map → log_write(bitmap_1)           = 1

// From bmap() after allocating master map:
//   master_map pointer update → not logged in bmap directly
//   (ip->addrs[NDIRECT+1] is an in-memory inode change — logged via iupdate later)

// From balloc() for secondary map block:
//   bitmap_block update → log_write(bitmap_2)                   = 2
//   (might be same bitmap block as above if blocks are nearby)

// From bmap() after secondary map allocated:
//   a[idx1] = secondary_map_addr → log_write(master_map_block) = 3

// From balloc() for data block:
//   bitmap_block → log_write(bitmap_3)                          = 4

// From bmap() after data block allocated:
//   a[idx2] = data_block_addr → log_write(secondary_map_block) = 5

// From writei() writing user data:
//   log_write(data_block)                                        = 6

// From iupdate() updating ip->size:
//   log_write(inode_block)                                       = 7

// Total worst case: 7 log_write() calls per write() ≤ LOGSIZE(30) ✓</code></pre>

<p><strong>How xv6 ensures the limit is respected — begin_op():</strong></p>
<pre><code class="language-c">// kernel/log.c — begin_op():
void begin_op(void) {
    acquire(&log.lock);
    while(1){
        if(log.committing){
            sleep(&log, &log.lock);    // wait if a commit is in progress
        } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
            // Would this new operation overflow the log?
            sleep(&log, &log.lock);    // wait for space to free up
        } else {
            log.outstanding++;          // reserve space for this op
            release(&log.lock);
            break;
        }
    }
}</code></pre>

<p>Before allowing a new transaction, begin_op() checks: <code>current_log_entries + (outstanding_ops + 1) × MAXOPBLOCKS ≤ LOGSIZE</code>. Each outstanding operation is reserved MAXOPBLOCKS=10 slots. This conservative estimate ensures that even if every pending operation uses its maximum log space, the log will not overflow.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

A student implements bmap() correctly but accidentally puts <code>log_write(bp)</code> BEFORE updating the buffer's data (before <code>a[idx1] = addr</code>). Explain what this causes and under what conditions it manifests as a bug.

<div class="answer-content">

<p><strong>The buggy code:</strong></p>
<pre><code class="language-c">// WRONG — log_write before the data modification:
if((addr = a[idx1]) == 0){
    addr = balloc(ip->dev);
    if(addr){
        log_write(bp);        // ← called BEFORE modifying a[idx1]!
        a[idx1] = addr;       // modification happens AFTER log_write
    }
}
brelse(bp);</code></pre>

<p><strong>What log_write(bp) actually records:</strong></p>
<p><code>log_write(bp)</code> marks the buffer as "dirty" and records its block number in the pending transaction log. It does NOT take a snapshot of the current buffer contents at the moment it's called. The buffer's <em>current contents at the time end_op() commits</em> are what gets written to disk.</p>

<p><strong>Does the bug actually matter?</strong></p>
<pre><code class="language-c">// Timeline:
// T1: log_write(bp) called — buffer flagged as dirty
// T2: a[idx1] = addr — buffer contents modified IN THE SAME BUFFER
// T3: brelse(bp) — buffer stays in cache (pinned by log_write)
// T4: end_op() — writes bp->data to the log section on disk
//     At T4, bp->data CONTAINS a[idx1]=addr (modified at T2)
//     The committed log data is correct!

// So in normal operation: NO BUG. The buffer is modified before end_op() flushes it.
// log_write() just flags the buffer — the actual data that gets flushed
// is whatever is in bp->data at flush time (T4), which includes the T2 modification.</code></pre>

<p><strong>When it DOES matter — concurrent modification scenario:</strong></p>
<pre><code class="language-c">// If another process modifies the SAME master map block between T1 and T4:
// T1: Process A calls log_write(master_map_bp) with a[idx1]=0
// T2: Process B modifies master_map_bp (different entry): a[idx2_B] = some_addr
// T3: Process A sets a[idx1] = addr
// T4: end_op() flushes master_map_bp — contains BOTH changes

// In this case the log records a merged state.
// If process B's change was in a SEPARATE transaction that committed first,
// process A's log_write "extends" the already-committed state — 
// this is actually the correct behaviour (idempotent log writes).

// The REAL risk: if log_write is forgotten entirely (not just reordered).
// Reordering log_write before vs after the modification doesn't affect correctness
// as long as the modification and log_write are within the same transaction.</code></pre>

<p><strong>Conclusion:</strong> The reordering is a style issue, not a correctness bug in normal single-threaded operation. The correct pattern (modify first, then log_write) is conventional and clearer, but both work because the buffer contents at end_op() time are what matters.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

What does <code>log.outstanding</code> track and why does begin_op() use <code>(log.outstanding+1)*MAXOPBLOCKS</code> as the reservation formula rather than just checking the current log size?

<div class="answer-content">

<p><strong>What log.outstanding tracks:</strong></p>
<pre><code class="language-c">// kernel/log.c
struct log {
    struct spinlock lock;
    int start;          // log start block
    int size;           // log size in blocks
    int outstanding;    // number of begin_op()'s that have not yet called end_op()
    int committing;     // 1 if a commit is in progress
    int dev;
    struct logheader lh; // in-memory log header (tracks currently logged block numbers)
};

// begin_op() increments outstanding:
log.outstanding++;

// end_op() decrements outstanding:
log.outstanding--;
// When outstanding hits 0: trigger a commit</code></pre>

<p><strong>Why the reservation formula uses (outstanding+1)*MAXOPBLOCKS:</strong></p>
<pre><code class="language-c">// Conservative worst-case reservation:
// Each outstanding operation (including the one being started) could use
// up to MAXOPBLOCKS log slots before it calls end_op().

// current reserved  = log.outstanding * MAXOPBLOCKS  (already started ops)
// new op's reserve  = MAXOPBLOCKS                    (the op being started)
// total projected   = (log.outstanding + 1) * MAXOPBLOCKS

// Check: will adding one more operation potentially overflow the log?
if(log.lh.n + (log.outstanding + 1) * MAXOPBLOCKS > LOGSIZE) → wait

// Example: 2 outstanding ops, each reserved 10 slots, LOGSIZE=30, current lh.n=5
// (2+1) * 10 = 30 > 30 - 5 = 25... 
// Actually: lh.n + (outstanding+1)*MAXOPBLOCKS > LOGSIZE
//           5    + 3*10                        > 30 → 35 > 30 → TRUE → must wait</code></pre>

<p><strong>Why not just check current log size:</strong></p>
<p>The currently committed log entries (<code>log.lh.n</code>) represent what's already logged. But outstanding operations haven't called <code>log_write()</code> yet — they will use more log space in the future. If begin_op() only checked the current log size, it might admit an operation that would overflow the log mid-transaction. The reservation is forward-looking: it ensures that all currently outstanding operations PLUS the new one can complete without running out of log space.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

The lab's <code>sys_symlink()</code> calls <code>begin_op()</code> at the start and <code>end_op()</code> at the end (and on every error path). What specifically does <code>end_op()</code> do, and when does it actually trigger a commit to disk vs just decrementing a counter?

<div class="answer-content">

<p><strong>What end_op() does step by step:</strong></p>
<pre><code class="language-c">// kernel/log.c
void end_op(void)
{
    int do_commit = 0;

    acquire(&log.lock);

    log.outstanding--;      // this operation is done

    if(log.committing)
        panic("log.committing");

    if(log.outstanding == 0){
        do_commit = 1;      // all begin_op callers have called end_op → commit now
        log.committing = 1;
    } else {
        // other operations still pending → wake any waiters but don't commit yet
        wakeup(&log);
    }

    release(&log.lock);

    if(do_commit){
        commit();           // ← ACTUAL DISK COMMIT
        acquire(&log.lock);
        log.committing = 0;
        wakeup(&log);       // wake processes waiting in begin_op()
        release(&log.lock);
    }
}</code></pre>

<p><strong>When it actually commits vs just decrements:</strong></p>

<table>
  <tr><th>Condition</th><th>Action</th></tr>
  <tr><td><code>log.outstanding > 0</code> after decrement</td><td>Just decrement and wake waiters — no commit yet. Other operations are still in progress.</td></tr>
  <tr><td><code>log.outstanding == 0</code> after decrement</td><td>Set <code>do_commit = 1</code>, then call <code>commit()</code> — this writes all dirty buffers to disk.</td></tr>
</table>

<p><strong>What commit() does:</strong></p>
<pre><code class="language-c">static void commit()
{
    if(log.lh.n > 0){
        write_log();      // copy dirty buffers from cache to log blocks on disk
        write_head();     // write log header with n=count (commit record)
        install_trans(0); // copy from log blocks to real disk locations
        log.lh.n = 0;
        write_head();     // clear commit record (n=0) — log is clean
    }
}</code></pre>

<p><strong>Implication for sys_symlink():</strong></p>
<p>If <code>sys_symlink()</code> is the only operation in the log when its <code>end_op()</code> runs, <code>outstanding</code> goes from 1 to 0 → full commit. If another filesystem operation (e.g., a concurrent open() or write()) is also between begin_op() and end_op(), the commit is deferred until the last one calls end_op(). This group-commit optimisation means multiple filesystem operations can share one commit, reducing the number of expensive disk writes.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

Construct a scenario where <code>sys_symlink()</code> is called concurrently by two processes, and trace how the log's group-commit mechanism ensures both symlinks appear atomically even though they share a single commit.

<div class="answer-content">

<p><strong>The setup:</strong></p>
<pre><code class="language-c">// Process A: symlink("fileA", "linkA")
// Process B: symlink("fileB", "linkB")
// Both start at approximately the same time.</code></pre>

<p><strong>Interleaved execution trace:</strong></p>
<pre><code class="language-c">// Process A begins:
A: begin_op() → log.outstanding = 1
A: create("linkA", T_SYMLINK) → ialloc(#24), dirlink, inode_block logged
   log_write(inode_block_A) → log.lh.n = 1
   log_write(bitmap_block)  → log.lh.n = 2
   log_write(dir_block_A)   → log.lh.n = 3

// Context switch to Process B:
B: begin_op() → log.outstanding = 2  ← A is still outstanding
B: create("linkB", T_SYMLINK) → ialloc(#25), dirlink
   log_write(inode_block_B) → log.lh.n = 4
   log_write(bitmap_block)  → log.lh.n = 5 (same bitmap block if nearby inodes)
   log_write(dir_block_B)   → log.lh.n = 6

// Process A continues, calls writei:
A: writei(ip_A, "fileA", ...) → log_write(data_block_A) → log.lh.n = 7
A: iunlockput(ip_A)
A: end_op() → log.outstanding = 2-1 = 1 → NOT zero → no commit, wake waiters

// Process B calls writei:
B: writei(ip_B, "fileB", ...) → log_write(data_block_B) → log.lh.n = 8
B: iunlockput(ip_B)
B: end_op() → log.outstanding = 1-1 = 0 → COMMIT TRIGGERED

// Process B's end_op() calls commit():
//   write_log():  writes log.lh.n=8 dirty buffers to log blocks 3-10 on disk
//   write_head(): writes log header {n=8, blocks=[inode_A, bitmap, dir_A, inode_B, bitmap, dir_B, data_A, data_B]}
//   install_trans: copies all 8 blocks from log to real locations
//   write_head(): clears log header (n=0)

// RESULT: Both symlinks committed atomically in ONE disk commit.</code></pre>

<p><strong>Crash safety of the group commit:</strong></p>
<pre><code class="language-c">// Crash scenario 1: crash before write_head() with n=8
//   On reboot: log header shows n=0 → NO recovery → both symlinks lost
//   Both processes see their symlink creation "failed" on reboot — clean state ✓

// Crash scenario 2: crash after write_head() with n=8 but before install_trans completes
//   On reboot: log header shows n=8 → install_trans replays all 8 blocks
//   Both symlinks appear on disk — exactly as committed ✓

// Either both appear or neither does — the group commit preserves atomicity for both operations simultaneously.</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

The bmap() doubly-indirect allocation in the lab makes three potential calls to <code>balloc()</code>. Each <code>balloc()</code> call internally does a <code>log_write()</code> on the bitmap block. Could the same bitmap block be logged multiple times in one transaction, and what does <code>log_write()</code> do when it receives a block that's already in the log?

<div class="answer-content">

<p><strong>The three balloc() calls in one bmap() invocation:</strong></p>
<pre><code class="language-c">// Cold doubly-indirect path — all three levels absent:
balloc(dev);  // 1: allocates master map block → log_write(bitmap_block)
balloc(dev);  // 2: allocates secondary map block → log_write(bitmap_block)
balloc(dev);  // 3: allocates data block → log_write(bitmap_block)

// All three might update the SAME bitmap block if the three allocated blocks
// fall within the same BPB=8192 range (very likely for sequentially allocated blocks).</code></pre>

<p><strong>What log_write() does on duplicate block numbers:</strong></p>
<pre><code class="language-c">// kernel/log.c — log_write():
void log_write(struct buf *b)
{
    int i;

    acquire(&log.lock);
    if(log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
        panic("too big a transaction");
    if(log.outstanding < 1)
        panic("log_write outside of trans");

    // ABSORPTION: check if this block is already in the log
    for(i = 0; i &lt; log.lh.n; i++){
        if(log.lh.block[i] == b->blockno)   // already logged!
            break;
    }
    log.lh.block[i] = b->blockno;
    if(i == log.lh.n)
        log.lh.n++;        // new entry only if NOT already present

    b->flags |= B_DIRTY;   // mark buffer as dirty regardless
    release(&log.lock);
}</code></pre>

<p><strong>The absorption optimisation:</strong></p>
<p>If the same block number appears in <code>log.lh.block[]</code>, the new <code>log_write()</code> call reuses the existing slot (<code>i &lt; log.lh.n</code> → no increment). This is called <strong>log absorption</strong>. The buffer remains pinned in cache and <code>B_DIRTY</code> is set, but no new log slot is consumed.</p>

<p><strong>Concrete example — three balloc() calls updating the same bitmap block:</strong></p>
<pre><code class="language-c">// balloc() 1: bitmap_block #45, log.lh.n = 5
//   log.lh.block[5] = 45; log.lh.n = 6

// balloc() 2: same bitmap_block #45
//   for(i=0; i<6; i++) if(lh.block[i]==45) → found at i=5
//   log.lh.block[5] = 45 (unchanged), log.lh.n stays 6
//   → ABSORBED: no new log slot consumed!

// balloc() 3: same bitmap_block #45
//   → ABSORBED again: log.lh.n stays 6

// Net effect: 3 balloc() calls → only 1 log slot used for the bitmap block!
// The bitmap buffer accumulates all 3 bit-set operations in memory.
// commit() writes the final combined bitmap state once.</code></pre>

<p>This absorption is critical for performance — without it, a file write that allocates 3 blocks would use 3 log slots for the bitmap alone, quickly exhausting LOGSIZE.</p>

</div>
</div>

<div class="var-question">
<invoke class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

If a student adds a new disk write inside bmap() that is NOT wrapped in the caller's <code>begin_op()</code>/<code>end_op()</code> — for example, by calling <code>bwrite(bp)</code> directly instead of <code>log_write(bp)</code> — describe all the ways this breaks the filesystem's crash safety guarantees.

<div class="answer-content">

<p><strong>The buggy code:</strong></p>
<pre><code class="language-c">// WRONG — direct bwrite instead of log_write:
if((addr = a[idx1]) == 0){
    addr = balloc(ip->dev);
    if(addr){
        a[idx1] = addr;
        bwrite(bp);         // ← direct write, bypasses the log!
        // Should be: log_write(bp);
    }
}</code></pre>

<p><strong>Problem 1 — Atomicity violated: master map written without its transaction partners:</strong></p>
<pre><code class="language-c">// bwrite(bp) writes the master map block to its real disk location IMMEDIATELY.
// The data block (which bmap() returns and writei() will fill) has NOT been written yet.
// The bitmap update (from balloc()) has been logged but not yet committed.

// Crash scenario:
// T1: bwrite(master_map) → master map on disk shows a[idx1] = secondary_map_addr
// T2: CRASH before writei() writes the actual user data

// On reboot:
// Recovery replays only the LOG — the bitmap log_write() is replayed.
// But the master map's bwrite() is NOT in the log — it was written directly.
// Master map on disk: a[idx1] = secondary_map_addr (from bwrite before crash)
// Secondary map: never allocated or filled (balloc() for secondary not yet reached)
// But wait — actually we're mid-bmap for Level 2, so:
// ip->addrs[NDIRECT+1] (master map address) was set in RAM but iupdate() not yet called.
// After reboot: the inode still shows addrs[12] = 0 (not yet persisted).
// The bwrite'd master map block exists on disk but is unreferenced.
// The secondary map was never allocated. The bitmap shows one block allocated with no owner.
// RESULT: 1 block permanently leaked (bitmap bit=1, no inode references it).</code></pre>

<p><strong>Problem 2 — Log ordering violated: bwrite bypasses the buffer cache's dirty tracking:</strong></p>
<pre><code class="language-c">// bwrite(bp) bypasses the log's B_DIRTY+log.lh tracking.
// The log doesn't know this block was written.
// If the buffer cache later decides to flush this block again (e.g., as part of a real commit),
// it might write the SAME block twice — once the bwrite()'d version and once the log version.
// If the two versions have different content (due to another modification between them),
// the log version would overwrite the bwrite()'d version — wrong order of operations.</code></pre>

<p><strong>Problem 3 — Transaction isolation violated: reader sees partial state:</strong></p>
<pre><code class="language-c">// While Process A is in bmap() between bwrite(master_map) and writei() completing:
// Process B calls bmap() for the SAME logical block.
// Process B reads the master map (now on disk with a[idx1] = secondary_map_addr).
// Process B follows to the secondary map — which is zeroed (data block not yet allocated).
// Process B's bmap() attempts to allocate the data block again!
// Two processes both allocate the same logical block position → two different physical blocks.
// Later: one of them is written correctly, the other is abandoned (disk leak).</code></pre>

<p><strong>The correct solution: always use log_write, never bwrite in transaction context:</strong></p>
<pre><code class="language-c">// log_write(bp) marks the buffer as dirty and records it in the transaction.
// The actual disk write is deferred until end_op() → commit() → install_trans().
// All modifications within begin_op()..end_op() are flushed together as one atomic unit.
// No intermediate state is visible to other processes (readers wait for the transaction lock).
// Crash recovery replays the complete set or none — no partial state can persist.</code></pre>

</div>
</div>
