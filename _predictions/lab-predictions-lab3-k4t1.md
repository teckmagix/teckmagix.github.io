---
layout: page
title: "Lab Predictions - lab3-k4t1"
lab: lab3
description: "Exam predictions for Lab3-w8: PDF step-by-step bmap/itrunc implementation, the bn=500 index example, and matching PDF labels to code."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

The lab PDF gives an example: "I want block 500, then 500/256 = 1.95 ~= 1" for Index1, and "500%256 = 244" for Index2. Using these values, trace the physical path through the map book hierarchy to find the data block.

<div class="answer-content">

<p><strong>The lab PDF's worked example — bn = 500 (after subtractions):</strong></p>
<pre><code class="language-c">// The lab PDF illustrates using bn=500 directly for pedagogy.
// (In actual code, bn=500 is the logical file block, which gets NDIRECT and NINDIRECT subtracted
//  before the division — but the PDF's example directly uses 500 for illustration.)

Index1 = 500 / 256 = 1    // integer division truncates: 1 not 2
Index2 = 500 % 256 = 244  // remainder</code></pre>

<p><strong>Physical navigation using these indices:</strong></p>
<pre><code class="language-c">// Step 1: Consult the Catalog Card (inode)
//   Slot 13 = ip->addrs[12] = e.g. disk block 500 (the Master Map Book)

// Step 2: Open the Master Map Book
bp = bread(dev, 500);           // load master map block
a  = (uint*)bp->data;           // a[] = 256 pointers to Secondary Map Books

// Step 3: Use Index1 = 1 to find which Secondary Map Book
secondary_map_addr = a[1];      // e.g. disk block 502 (Secondary Map Book #1)
// (a[0] = Secondary Map Book #0, covering logical blocks 267-522)
// (a[1] = Secondary Map Book #1, covering logical blocks 523-778)
brelse(bp);                     // ALWAYS close the Master Map before opening secondary!

// Step 4: Open Secondary Map Book #1
bp = bread(dev, 502);
a  = (uint*)bp->data;

// Step 5: Use Index2 = 244 to find the Data Block
data_block_addr = a[244];       // e.g. disk block 760

brelse(bp);                     // close the Secondary Map
return 760;                     // physical disk block for logical block 500</code></pre>

<p><strong>The complete addressing tree for this example:</strong></p>
<pre><code class="language-c">Inode (ip->addrs[12]) → Master Map Block (500)
                             ↓ entry [Index1=1]
                        Secondary Map #1 Block (502)
                             ↓ entry [Index2=244]
                        Data Block (760)  ← contains the file's actual data</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

The lab PDF's bmap() implementation guide has six lettered steps (a through f). Match each step to a specific line or block of code in the actual implementation.

<div class="answer-content">

<p><strong>The six steps from the lab PDF mapped to code:</strong></p>
<pre><code class="language-c">// (a) Check Range: "If the block number is beyond Direct and Singly-Indirect limits"
bn -= NINDIRECT;
if(bn &lt; NDINDIRECT){          // ← the range check

// (b) Level 1 — Get Master Map address, allocate if absent, then bread() it:
    // "Get the address of the doubly-indirect block: Look at inode's 13th slot"
    if((addr = ip->addrs[NDIRECT+1]) == 0){
        addr = balloc(ip->dev);          // "call balloc() to create the Master Map"
        if(addr == 0) return 0;
        ip->addrs[NDIRECT+1] = addr;
    }
    // "Load the doubly-indirect block (the first level): bread() the Master Map"
    bp = bread(ip->dev, addr);
    a  = (uint*)bp->data;

// (c) Calculate Indices:
    // "Index1: Which Secondary Map? (bn / 256)"
    uint idx1 = bn / NINDIRECT;
    // "Index2: Which Data Block inside that secondary map? (bn % 256)"
    uint idx2 = bn % NINDIRECT;

// (d) Level 2 — Secondary Map: check, allocate if absent, log_write, brelse() master, bread() secondary:
    // "Check Index1 inside the Master Map. If empty, call balloc() to create a Secondary Map."
    if((addr = a[idx1]) == 0){
        addr = balloc(ip->dev);
        if(addr){
            a[idx1] = addr;
            log_write(bp);              // "Call log_write() to mark the buffer"
        }
    }
    // "brelse() the Master Map (always close the previous book!)"
    brelse(bp);
    if(addr == 0) return 0;
    // "Load the second-level indirect block: bread() the Secondary Map"
    bp = bread(ip->dev, addr);
    a  = (uint*)bp->data;

// (e) Level 3 — Data Block: check, allocate if absent, log_write, brelse() secondary:
    // "Check Index2 inside the Secondary Map. If empty, call balloc() for the final data block."
    if((addr = a[idx2]) == 0){
        addr = balloc(ip->dev);
        if(addr){
            a[idx2] = addr;
            log_write(bp);              // "Call log_write() to mark the buffer"
        }
    }
    // "brelse() the Secondary Map"
    brelse(bp);

// (f) Return the final data block address:
    return addr;
}
panic("bmap: out of range");</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

The lab PDF's itrunc() guide has a three-part logic (a, b, c). Write the complete doubly-indirect cleanup code that implements exactly these three parts, labelled with comments matching the PDF.

<div class="answer-content">

<p><strong>The PDF's itrunc() doubly-indirect logic implemented:</strong></p>
<pre><code class="language-c">// Part (a): Direct & Singly-Indirect — Already handled in original code (not shown)

// Part (b): Doubly-Indirect Tree Walk
if(ip->addrs[NDIRECT+1]){   // is the Master Map present?

    struct buf *bp1, *bp2;
    uint *a1, *a2;

    // "bread() the Master Map Book"
    bp1 = bread(ip->dev, ip->addrs[NDIRECT+1]);
    a1  = (uint*)bp1->data;

    // "Loop 1: For each entry in the Master Map"
    for(int i = 0; i &lt; NINDIRECT; i++){
        // "If the entry is not empty, bread() that Secondary Map Book"
        if(a1[i]){
            bp2 = bread(ip->dev, a1[i]);
            a2  = (uint*)bp2->data;

            // "Loop 2: For each entry in the Secondary Map"
            for(int j = 0; j &lt; NINDIRECT; j++){
                // "If the entry is not empty, call bfree() on that Data Block"
                if(a2[j])
                    bfree(ip->dev, a2[j]);
            }

            // "brelse() the Secondary Map, then call bfree() on the Secondary Map block itself"
            brelse(bp2);
            bfree(ip->dev, a1[i]);
        }
    }

    // "brelse() the Master Map, then call bfree() on the Master Map block itself"
    brelse(bp1);
    bfree(ip->dev, ip->addrs[NDIRECT+1]);

    // Part (c): Clear the inode
    // "Set the 13th slot in the inode back to 0"
    ip->addrs[NDIRECT+1] = 0;
}</code></pre>

<p><strong>Why the PDF's ordering (brelse before bfree) is critical:</strong></p>
<p>The PDF explicitly says: <code>brelse()</code> the Secondary Map, THEN <code>bfree()</code> on the Secondary Map block. Freeing a block while its buffer is still locked would allow another process to allocate that block (via <code>balloc()</code>) while we still hold it in the cache — leading to two files sharing the same disk block. Release first, then free.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

The lab PDF describes the five implementation steps for Task 1 in order. For each step, name the file that must be changed and what the change achieves.

<div class="answer-content">

<p><strong>The five Task 1 implementation steps:</strong></p>

<table>
  <tr><th>Step</th><th>File</th><th>What to change</th><th>What it achieves</th></tr>
  <tr><td>1</td><td><code>kernel/param.h</code></td><td>FSSIZE: 2000 → 200000</td><td>Disk image large enough to hold 65,803-block files; mkfs output changes from nmeta 46 to nmeta 70</td></tr>
  <tr><td>2</td><td><code>kernel/file.h</code></td><td>struct inode: <code>addrs[NDIRECT+2]</code></td><td>In-memory inode has the doubly-indirect slot; matches struct dinode exactly so memmove copies all 13 entries</td></tr>
  <tr><td>3</td><td><code>kernel/fs.h</code></td><td>NDIRECT=11; add NDINDIRECT; update MAXFILE; struct dinode: <code>addrs[NDIRECT+2]</code></td><td>On-disk inode layout redefined: 11 direct + 1 singly + 1 doubly; MAXFILE = 65,803</td></tr>
  <tr><td>4</td><td><code>kernel/fs.c</code> — bmap()</td><td>Add doubly-indirect navigation logic</td><td>Kernel can find/allocate data blocks beyond the 268-block singly-indirect limit</td></tr>
  <tr><td>5</td><td><code>kernel/fs.c</code> — itrunc()</td><td>Add doubly-indirect tree walk and free</td><td>Deleting a large file returns all allocated blocks to the free pool — no disk leak</td></tr>
</table>

<p><strong>The order matters:</strong></p>
<p>Step 3 (fs.h) must come before Step 4 (bmap) and Step 5 (itrunc) because both functions reference NDIRECT, NINDIRECT, NDINDIRECT, and MAXFILE. Step 2 (file.h) must match Step 3 (fs.h) — both must use <code>addrs[NDIRECT+2]</code> simultaneously or the memmove in ilock() will be wrong. Steps 1 and 2–3 can be done in any order relative to each other.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

The lab PDF says the doubly-indirect allocation in bmap() "needs an extra step compared to the singly-indirect block allocation." Identify this extra step precisely, and explain the asymmetry between singly-indirect and doubly-indirect navigation.

<div class="answer-content">

<p><strong>Singly-indirect navigation (one level of indirection):</strong></p>
<pre><code class="language-c">// Singly-indirect: one map block → one data block
if(bn &lt; NINDIRECT){
    // Step 1: Get/allocate the indirect block
    if((addr = ip->addrs[NDIRECT]) == 0){
        addr = balloc(ip->dev);
        ip->addrs[NDIRECT] = addr;
    }
    // Step 2: Read the map block
    bp = bread(ip->dev, addr);
    a  = (uint*)bp->data;
    // Step 3: Get/allocate the data block
    if((addr = a[bn]) == 0){
        addr = balloc(ip->dev); a[bn] = addr; log_write(bp);
    }
    brelse(bp);
    return addr;
}</code></pre>

<p><strong>Doubly-indirect navigation (two levels of indirection):</strong></p>
<pre><code class="language-c">// Doubly-indirect: master map block → secondary map block → data block
bn -= NINDIRECT;
if(bn &lt; NDINDIRECT){
    // Step 1: Get/allocate the MASTER MAP block
    if((addr = ip->addrs[NDIRECT+1]) == 0){
        addr = balloc(ip->dev);
        ip->addrs[NDIRECT+1] = addr;
    }
    // Step 2: Read the MASTER MAP block
    bp = bread(ip->dev, addr);
    a  = (uint*)bp->data;
    uint idx1 = bn / NINDIRECT, idx2 = bn % NINDIRECT;

    // Step 3: Get/allocate the SECONDARY MAP block  ← THE EXTRA STEP
    if((addr = a[idx1]) == 0){
        addr = balloc(ip->dev); a[idx1] = addr; log_write(bp);
    }
    brelse(bp);   // ← must release master BEFORE reading secondary
    if(addr == 0) return 0;

    // Step 4: Read the SECONDARY MAP block  ← equivalent of singly's step 2
    bp = bread(ip->dev, addr);
    a  = (uint*)bp->data;

    // Step 5: Get/allocate the DATA block  ← equivalent of singly's step 3
    if((addr = a[idx2]) == 0){
        addr = balloc(ip->dev); a[idx2] = addr; log_write(bp);
    }
    brelse(bp);
    return addr;
}</code></pre>

<p><strong>The exact extra step:</strong></p>
<p>Singly-indirect: <strong>1 map block</strong> (indirect block) → data block (2 levels total). Doubly-indirect: <strong>2 map blocks</strong> (master map + secondary map) → data block (3 levels total). The extra step is: reading the master map to find the secondary map address, then reading the secondary map to find the data block address. This requires an additional <code>bread()</code>/<code>brelse()</code> pair and an additional <code>balloc()</code>/<code>log_write()</code> pair for the secondary map — plus the <code>idx1</code>/<code>idx2</code> index calculation that replaces singly-indirect's single <code>bn</code> index.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

The lab PDF states the implementation has six hints. Apply all six hints as a debugging checklist — for each hint, write a concrete test you could run or observation you could make to confirm you followed it correctly.

<div class="answer-content">

<p><strong>Hint 1 — addrs[] must match in struct inode (file.h) and struct dinode (fs.h):</strong></p>
<pre><code class="language-c">// Test: check sizeof both structs are equal
// In kernel code: static_assert(sizeof(((struct inode*)0)->addrs) == sizeof(((struct dinode*)0)->addrs), "addrs mismatch");
// Observation: both files must show addrs[NDIRECT+2] — grep for "addrs" in both files
// Failure symptom: bigfile writes succeed but reads return garbage data</code></pre>

<p><strong>Hint 2 — Run make clean after changing NDIRECT:</strong></p>
<pre><code class="language-c">// Test: after any change to fs.h, always run make clean before make qemu
// Observation: the nmeta line should change from "nmeta 46" to "nmeta 70"
// If still showing "nmeta 46": make did NOT rebuild fs.img → run make clean
// Failure symptom: kernel compiles with new NDIRECT but runs with old on-disk layout → every inode is misread</code></pre>

<p><strong>Hint 3 — Delete fs.img from Unix (not xv6) to reset a corrupt filesystem:</strong></p>
<pre><code class="language-c">// Test: if xv6 panics on boot or shows strange errors → exit QEMU (Ctrl+A X)
//   then from Linux: rm fs.img && make qemu
// NEVER run inside xv6: the rm command would try to delete fs.img from xv6's filesystem (which doesn't contain it)
// Observation: after "rm fs.img && make", mkfs regenerates a clean disk image</code></pre>

<p><strong>Hint 4 — Always brelse() every bread():</strong></p>
<pre><code class="language-c">// Test: write a test that opens and writes many files in a loop
// If brelse() is missing: the loop will hang or panic after ~30 iterations (NBUF=30 buffer slots exhausted)
// Observation: run usertests — any test that performs many file operations will panic if brelse() is missing
// Manual check: for every "bp = bread(" line in your added code, count a matching "brelse(bp)" after use</code></pre>

<p><strong>Hint 5 — Allocate blocks only as needed (lazy allocation):</strong></p>
<pre><code class="language-c">// Test: create a small file (1 block) and check disk usage with ls
// If lazy: ls shows the file size as 1024 bytes — no extra map blocks allocated
// If eager: disk has allocated 1 master map + 256 secondary maps = 257 extra blocks even for a 1-block file
// Observation: run "ls" and count available blocks — with 10 small files, ~2570 blocks should NOT be consumed
// Code check: every balloc() call should be inside an "if(addr == 0)" guard</code></pre>

<p><strong>Hint 6 — itrunc() must free all blocks including doubly-indirect:</strong></p>
<pre><code class="language-c">// Test: run bigfile twice in the same xv6 session (without make clean between runs)
// If itrunc() is correct: second run succeeds ("wrote 6580 blocks")
// If itrunc() leaks doubly-indirect blocks: second run fails around block 267
//   (old doubly-indirect blocks still marked allocated in bitmap)
// Stronger test: run bigfile 30 times → if it fails, itrunc() leaks blocks
// Observation: check that ip->addrs[NDIRECT+1] = 0 at the end of itrunc()'s doubly-indirect cleanup</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

The lab PDF's itrunc() guide says "brelse() the Secondary Map, then call bfree() on the Secondary Map block itself." If these two are reversed (bfree before brelse), trace the exact sequence of events that leads to data corruption.

<div class="answer-content">

<p><strong>The reversed order (WRONG):</strong></p>
<pre><code class="language-c">// WRONG — bfree BEFORE brelse:
bp2 = bread(ip->dev, a1[i]);     // load secondary map, bp2 locked
a2  = (uint*)bp2->data;
for(j = 0; j &lt; NINDIRECT; j++)
    if(a2[j]) bfree(ip->dev, a2[j]);  // free all data blocks

bfree(ip->dev, a1[i]);           // ← free the secondary map block FIRST (WRONG)
                                   //   bp2 is still held/locked at this point!
brelse(bp2);                     // ← release it after (too late)</code></pre>

<p><strong>The corruption sequence step by step:</strong></p>
<pre><code class="language-c">// Time 0: bfree(dev, secondary_map_block=501)
//   Loads bitmap block → clears bit 501 → log_write bitmap → brelse bitmap
//   Block 501 is now MARKED FREE in the bitmap
//   BUT: bp2 for block 501 is still locked in the buffer cache!
//   The buffer cache still has a VALID entry for block 501

// Time 1: Another process calls balloc()
//   Scans bitmap → finds bit 501 = 0 (free!) → sets bit 501 = 1 → brelse bitmap
//   Calls bzero(dev, 501) → bread(dev, 501) ...
//   BUT the buffer cache still has the OLD locked bp2 for block 501!
//   bget() finds the existing cache entry for block 501
//   bcache.lock logic: block 501 is ALREADY locked by the first process
//   The new process BLOCKS waiting for bp2 to be released

// Time 2: Original process calls brelse(bp2)
//   bp2 is released. Block 501 is now unlocked in the cache.
//   The new process wakes up, gets bp2 (which still has the OLD secondary map data!)
//   bzero() writes zeros over it → new file's data block 0 is block 501 → zeroed correctly.
//   But: bp2->data still has the secondary map pointers in the cache from the old content.
//   Any code that had saved pointers from bp2->data is now working with data from a block
//   that belongs to another file.

// ACTUAL CORRUPTION:
// If the new process gets bp2 via bread() BEFORE bzero() runs:
//   It reads old secondary map data as if it were the new file's inode block.
//   Chaos follows.</code></pre>

<p><strong>The correct order prevents this entirely:</strong></p>
<p><code>brelse(bp2)</code> first → the cache entry for block 501 becomes unlocked and available for reuse → <code>bfree(dev, 501)</code> then clears the bitmap bit. Any subsequent <code>balloc()</code> that gets block 501 will call <code>bzero()</code>, which calls <code>bread(dev, 501)</code> → gets an unlocked cache slot → overwrites with zeros safely. No process can be simultaneously holding the old secondary map buffer.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

The lab PDF says to "add code for doubly-indirect block allocation after the direct and singly-indirect block allocation" in bmap(). What specific lines of existing code mark the boundaries where your addition should be inserted, and what must be true about the value of <code>bn</code> at the point where your code begins?

<div class="answer-content">

<p><strong>The existing bmap() structure (before your changes):</strong></p>
<pre><code class="language-c">static uint bmap(struct inode *ip, uint bn)
{
    uint addr, *a;
    struct buf *bp;

    // ── Direct block range ──
    if(bn &lt; NDIRECT){
        if((addr = ip->addrs[bn]) == 0)
            ip->addrs[bn] = addr = balloc(ip->dev);
        return addr;
    }
    bn -= NDIRECT;      // ← bn is now relative to start of singly-indirect range

    // ── Singly-indirect range ──
    if(bn &lt; NINDIRECT){
        if((addr = ip->addrs[NDIRECT]) == 0)
            ip->addrs[NDIRECT] = addr = balloc(ip->dev);
        bp = bread(ip->dev, addr);
        a  = (uint*)bp->data;
        if((addr = a[bn]) == 0){ a[bn] = addr = balloc(ip->dev); log_write(bp); }
        brelse(bp);
        return addr;
    }
    bn -= NINDIRECT;    // ← bn is now relative to start of doubly-indirect range

    // ── INSERT YOUR DOUBLY-INDIRECT CODE HERE ──
    // At this point:
    //   bn = original_logical_block - NDIRECT - NINDIRECT
    //   bn is relative to the start of the doubly-indirect range
    //   bn=0 means the FIRST doubly-indirect block (logical block 267)
    //   bn=65535 means the LAST doubly-indirect block (logical block 65802)

    panic("bmap: out of range");  // ← this line should be AFTER your addition
}</code></pre>

<p><strong>Exact insertion point:</strong></p>
<pre><code class="language-c">bn -= NINDIRECT;   // ← this line is the last line before your addition
                    // after this line, bn is relative to the doubly-indirect range

// YOUR CODE GOES HERE:
if(bn &lt; NDINDIRECT){
    // ... doubly-indirect logic ...
    return addr;
}

panic("bmap: out of range");  // ← this stays at the end</code></pre>

<p><strong>What must be true about bn at entry to your code:</strong></p>

<table>
  <tr><th>Property</th><th>Value</th><th>Why</th></tr>
  <tr><td>bn has been subtracted by NDIRECT</td><td>Yes — done before singly-indirect section</td><td>The direct range (0..NDIRECT-1) has already been handled and returned</td></tr>
  <tr><td>bn has been subtracted by NINDIRECT</td><td>Yes — done just before your insertion point</td><td>The singly-indirect range (0..NINDIRECT-1 after first subtraction) has been handled</td></tr>
  <tr><td>Value range of bn</td><td>0 ≤ bn ≤ (original_bn - NDIRECT - NINDIRECT)</td><td>If original_bn was 267, bn = 267-11-256 = 0 (first doubly-indirect)</td></tr>
  <tr><td>bn < NDINDIRECT check</td><td>Your code adds this check</td><td>Values ≥ NDINDIRECT (65536) would be out of range → panic</td></tr>
</table>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

The lab PDF says bmap()'s doubly-indirect step (d) specifies: "brelse() the Master Map (always close the previous book!)." Write a proof-by-contradiction showing that if brelse(master_map) is NOT called before bread(secondary_map), the system will eventually deadlock or panic even with a correct implementation otherwise.

<div class="answer-content">

<p><strong>Setup — assume brelse(master_map) is omitted:</strong></p>
<pre><code class="language-c">// Buggy code — master map buffer held while loading secondary:
bp1 = bread(ip->dev, master_map_addr);   // LOCK slot 1 (master map)
a   = (uint*)bp1->data;
// ... brelse(bp1) is NOT called ...

bp2 = bread(ip->dev, secondary_map_addr); // LOCK slot 2 (secondary map)
// Now 2 buffer slots are locked simultaneously</code></pre>

<p><strong>Proof that panic occurs:</strong></p>
<p>Let <code>N = NBUF = 30</code> (total buffer cache slots).</p>
<pre><code class="language-c">// Consider N/2 = 15 concurrent processes, each calling write() on a large file.
// Each write() calls bmap() which (with the bug) holds 2 buffer slots simultaneously.
// 15 processes × 2 slots = 30 slots — exactly NBUF.
// When process #16 calls write() → bmap() → bread(master_map):
//   bget() scans all 30 slots → ALL 30 are locked → no free slot found
//   bget() panics: "bget: no buffers"

// More precisely, bget() calls sleep() to wait for a slot to become free.
// But ALL 30 slots are locked by processes waiting for their secondary map bread() to return.
// Those bread() calls are also waiting for a slot (via bget())... which will never free up.
// Circular wait: every bget() caller is sleeping, none can proceed to brelse() to free a slot.
// This is a deadlock among 15+ processes, all blocked in bget().

// FORMAL PROOF:
// Assume N processes each hold exactly 2 buffer locks (master + secondary).
// Each process needs one more slot to return from bmap().
// No process will brelse() its held slots until bmap() returns.
// bmap() cannot return until a slot is available.
// A slot is available only when some process calls brelse().
// No process will call brelse() until bmap() returns.
// CIRCULAR: bmap_return requires free_slot requires brelse requires bmap_return.
// By contradiction: the assumption "N processes hold 2 slots each" leads to deadlock.
// For N ≥ NBUF/2 = 15 processes: deadlock is guaranteed.</code></pre>

<p><strong>Why the correct brelse(bp1) prevents this:</strong></p>
<pre><code class="language-c">// Correct code:
bp1 = bread(dev, master_map);    // lock slot 1
addr = a[idx1];                   // extract needed pointer
brelse(bp1);                      // ← IMMEDIATELY release slot 1
bp2 = bread(dev, secondary_map); // now only 1 slot locked
// ...
brelse(bp2);                      // release slot 2

// Peak simultaneous lock count: 1 (never 2).
// 30 concurrent processes × 1 slot = 30 slots → still fits in NBUF.
// No deadlock possible — each process holds at most 1 slot while waiting for the next.</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

The lab PDF says the itrunc() cleanup must "clear the inode: set the 13th slot in the inode back to 0." But the PDF does not mention calling iupdate(). Where does iupdate() get called for this change to reach disk, and what would happen if iupdate() were never called?

<div class="answer-content">

<p><strong>Where iupdate() is called — not directly by itrunc():</strong></p>
<pre><code class="language-c">// In kernel/fs.c, the call chain from unlink():
sys_unlink()
  → begin_op()
  → nameiparent(path, name)
  → dirlookup(dp, name, &off)
  → ip = iget(dp->dev, de.inum)
  → ilock(ip)
  → ip->nlink--
  → iupdate(ip)           // ← writes nlink change to disk
  → iunlockput(ip)        // → iput(ip) → if ref==0 && nlink==0: iput calls itrunc()
  
// Inside iput() when ref==0 && nlink==0:
void iput(struct inode *ip){
    // ...
    if(ip->valid && ip->nlink == 0){
        itrunc(ip);            // ← free all blocks
        ip->type = 0;
        iupdate(ip);           // ← THIS is where itrunc's changes get written to disk!
        ip->valid = 0;
    }
    // ...
}</code></pre>

<p><strong>What iupdate() writes to disk after itrunc():</strong></p>
<pre><code class="language-c">// After itrunc(), ip has:
ip->addrs[0..NDIRECT-1]  = 0   // direct slots cleared
ip->addrs[NDIRECT]       = 0   // singly-indirect cleared
ip->addrs[NDIRECT+1]     = 0   // doubly-indirect cleared  ← your addition
ip->size                 = 0   // file size zeroed
ip->type                 = 0   // (set by iput AFTER itrunc)

// iupdate() copies all of these to the on-disk dinode via:
memmove(dip->addrs, ip->addrs, sizeof(dip->addrs));
// The zeroed addrs[NDIRECT+1] is persisted to disk.</code></pre>

<p><strong>What happens if iupdate() is never called after itrunc():</strong></p>
<pre><code class="language-c">// In-memory inode: ip->addrs[NDIRECT+1] = 0  ← correctly zeroed by itrunc()
// On-disk dinode:  dip->addrs[12] = 500       ← still has old master map address!

// Scenario 1: System continues running
//   The in-memory inode cache still holds the zeroed version.
//   Any subsequent ilock() that finds ip->valid=1 uses the correct zeroed in-memory inode.
//   The old master map block (500) has been freed by itrunc() → its bitmap bit is 0.
//   balloc() may allocate block 500 to a new file.
//   New file writes data to block 500.
//   Everything works — the in-memory inode has ip->addrs[12]=0, never follows block 500.

// Scenario 2: System reboots
//   The in-memory inode was evicted (valid → 0) or lost entirely.
//   The on-disk dinode still has addrs[12] = 500 (never updated by iupdate()).
//   Block 500 now belongs to a different file (reused by balloc()).
//   On reboot, if the original file's inode (somehow still with nlink=1 from a race)
//   is accessed: ilock() reads dip->addrs[12] = 500 into ip->addrs[12].
//   bmap() follows addrs[12] = 500 → reads the new file's content as if it were map data.
//   Crash, corruption, or silent wrong data returned to any reader of the old file.</code></pre>

<p>The log transaction wrapping iput()'s itrunc() + iupdate() ensures both operations reach disk together — either the inode is fully zeroed and updated, or neither change is visible after a crash.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

The lab PDF lists the five implementation steps for Task 1 in a specific order (param.h → file.h → fs.h → bmap → itrunc). Prove that performing them in a different order — specifically fs.h before file.h — will cause a compilation error or runtime bug even if all changes are eventually made correctly.

<div class="answer-content">

<p><strong>Scenario — updating fs.h (NDIRECT=11) before file.h:</strong></p>
<pre><code class="language-c">// After fs.h change but before file.h change:
// kernel/fs.h:   #define NDIRECT 11; struct dinode { ... addrs[NDIRECT+2] = addrs[13] }
// kernel/file.h: struct inode { ... addrs[NDIRECT+1] } ← still old! = addrs[12]</code></pre>

<p><strong>Compilation consequence — does it compile?</strong></p>
<pre><code class="language-c">// fs.h defines NDIRECT = 11
// file.h uses NDIRECT for struct inode:
//   uint addrs[NDIRECT+1];  → uint addrs[12];  (11+1)
//   This is DIFFERENT from fs.h's dinode:
//   uint addrs[NDIRECT+2];  → uint addrs[13];  (11+2)
// C DOES compile this — no compilation error.
// The two structs have different array sizes but the compiler doesn't check consistency between files.</code></pre>

<p><strong>Runtime consequence — the memmove mismatch:</strong></p>
<pre><code class="language-c">// In ilock() — the critical copy:
memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
//                              ↑ uses sizeof(struct inode::addrs)
//                              = sizeof(uint[12]) = 48 bytes  (file.h not yet updated)
// dip->addrs is sizeof(uint[13]) = 52 bytes  (fs.h already updated)

// Only 48 bytes copied → 12 entries copied → ip->addrs[12] (doubly-indirect slot) NOT COPIED
// The doubly-indirect pointer from disk is silently dropped.

// When bmap(ip, 267) is called (first doubly-indirect block):
addr = ip->addrs[NDIRECT+1];  // = ip->addrs[12]
// ip->addrs[] only has indices 0..11 (12 entries).
// ip->addrs[12] is OUT OF BOUNDS — reads 4 bytes past the end of the array.
// Those 4 bytes are wherever comes next in struct inode memory layout.
// In NDIRECT+1=12 old file.h, ip->addrs has 12 entries.
// ip->addrs[12] reads ip->dev (the next field in struct inode, a uint).
// addr = ip->dev  → e.g. 1 (device number for the disk)
// bmap() calls bread(dev, 1) to load the "master map"
// Block 1 is the SUPERBLOCK — reading it as pointer data returns garbage
// → bmap() follows garbage pointers → kernel panic or silent data corruption.</code></pre>

<p><strong>Conclusion — why the lab PDF's order matters:</strong></p>
<p>The correct order ensures that both <code>file.h</code> and <code>fs.h</code> are updated to <code>addrs[NDIRECT+2]</code> before any code that uses <code>sizeof(ip->addrs)</code> (ilock, iupdate) or accesses <code>ip->addrs[NDIRECT+1]</code> (bmap, itrunc) is compiled. If <code>fs.h</code> is updated first and you immediately try to compile with <code>make</code>, the kernel builds with mismatched struct sizes — compilation succeeds but the runtime behavior is wrong in exactly the way described above. Always update both headers together before recompiling.</p>

</div>
</div>
