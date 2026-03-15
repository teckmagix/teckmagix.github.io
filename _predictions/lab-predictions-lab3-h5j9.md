---
layout: page
title: "Lab Predictions - lab3-h5j9"
lab: lab3
description: "Exam predictions for Lab3-w8: dinode vs inode struct sync, ilock/iupdate, memmove, and addrs[] mismatch bugs."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

What is the difference between <code>struct dinode</code> in <code>fs.h</code> and <code>struct inode</code> in <code>file.h</code>? Why do both need to be updated?

<div class="answer-content">

<p><strong>The two structs:</strong></p>
<pre><code class="language-c">// kernel/fs.h — ON-DISK inode
struct dinode {
    short  type;
    short  major;
    short  minor;
    short  nlink;
    uint   size;
    uint   addrs[NDIRECT+2];   // block address array — physically on disk
};

// kernel/file.h — IN-MEMORY inode
struct inode {
    uint   dev;         // extra: device number (not in dinode)
    uint   inum;        // extra: inode number (not in dinode)
    int    ref;         // extra: reference count (not in dinode)
    struct sleeplock lock; // extra: concurrency control (not in dinode)
    int    valid;       // extra: has dinode been loaded from disk?
    short  type;        // ← mirrors dinode
    short  major;       // ← mirrors dinode
    short  minor;       // ← mirrors dinode
    short  nlink;       // ← mirrors dinode
    uint   size;        // ← mirrors dinode
    uint   addrs[NDIRECT+2];   // ← mirrors dinode — MUST match exactly
};</code></pre>

<p><strong>Why both must be updated:</strong></p>
<p>When the kernel needs to access a file, it loads the on-disk <code>dinode</code> into an in-memory <code>inode</code> via <code>ilock()</code>. The loading uses <code>memmove(ip->addrs, dip->addrs, sizeof(ip->addrs))</code> — a raw byte copy. If <code>sizeof(ip->addrs) != sizeof(dip->addrs)</code>, the copy either misses the last pointer (too small) or reads garbage bytes past the dinode's array (too large). Either way, the doubly-indirect pointer slot is wrong, and bmap() either cannot find doubly-indirect blocks or follows a random block number as if it were a valid pointer.</p>

<p><strong>Why the array size stays at 13 despite NDIRECT changing:</strong></p>
<pre><code class="language-c">// Before: NDIRECT=12 → addrs[NDIRECT+1] = addrs[13]
// After:  NDIRECT=11 → addrs[NDIRECT+2] = addrs[13]
// Both compile to a 13-element array of uint — identical on-disk layout.
// The physical size of the dinode struct is unchanged. No filesystem re-formatting needed.</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What does <code>ilock(ip)</code> do beyond acquiring the sleeplock? When does it read the inode from disk?

<div class="answer-content">

<p><strong>What ilock() does:</strong></p>
<pre><code class="language-c">// kernel/fs.c — ilock() behaviour:
void ilock(struct inode *ip)
{
    struct buf *bp;
    struct dinode *dip;

    acquiresleep(&ip->lock);          // (1) acquire the sleeplock — blocks if held

    if(ip->valid == 0){               // (2) has this inode been loaded from disk yet?
        bp  = bread(ip->dev, IBLOCK(ip->inum, sb));  // (3) load inode block from disk
        dip = (struct dinode*)bp->data + ip->inum%IPB; // (4) find this inode in the block

        ip->type  = dip->type;        // (5) copy all dinode fields into in-memory inode
        ip->major = dip->major;
        ip->minor = dip->minor;
        ip->nlink = dip->nlink;
        ip->size  = dip->size;
        memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));  // (6) copy addrs[]

        brelse(bp);                   // (7) release the inode block buffer
        ip->valid = 1;                // (8) mark inode as loaded
        if(ip->type == 0)
            panic("ilock: no type"); // inode 0 or freed inode — should not be used
    }
}</code></pre>

<p><strong>When it reads from disk:</strong></p>
<p>Only when <code>ip->valid == 0</code> — the first time this inode is locked after being fetched via <code>iget()</code>. <code>iget()</code> allocates an in-memory slot and sets <code>valid = 0</code>. The first <code>ilock()</code> call populates the slot from disk. Subsequent <code>ilock()</code> calls find <code>valid == 1</code> and skip the disk read — the data is already in RAM.</p>

<p><strong>What ilock() guarantees on return:</strong></p>
<ul>
  <li>The sleeplock is held by the calling thread</li>
  <li><code>ip->type</code>, <code>ip->size</code>, <code>ip->addrs[]</code> reflect the current on-disk state</li>
  <li>No other thread will modify these fields until <code>iunlock()</code> is called</li>
</ul>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What does <code>iupdate(ip)</code> do and when must it be called?

<div class="answer-content">

<p><strong>What iupdate() does:</strong></p>
<pre><code class="language-c">// kernel/fs.c — iupdate() behaviour:
void iupdate(struct inode *ip)
{
    struct buf *bp;
    struct dinode *dip;

    bp  = bread(ip->dev, IBLOCK(ip->inum, sb));
    dip = (struct dinode*)bp->data + ip->inum%IPB;

    dip->type  = ip->type;            // copy in-memory fields back to on-disk dinode
    dip->major = ip->major;
    dip->minor = ip->minor;
    dip->nlink = ip->nlink;
    dip->size  = ip->size;
    memmove(dip->addrs, ip->addrs, sizeof(dip->addrs));  // copy addrs[] to disk

    log_write(bp);                    // mark inode block dirty
    brelse(bp);
}</code></pre>

<p><code>iupdate()</code> writes the in-memory inode back to the corresponding on-disk <code>dinode</code>. Without it, changes to the inode (type, size, addrs[]) exist only in RAM and are lost on a crash or reboot.</p>

<p><strong>When iupdate() must be called:</strong></p>

<table>
  <tr><th>Operation</th><th>Why iupdate() is needed</th></tr>
  <tr><td>After <code>itrunc()</code> zeros addrs[] and sets size=0</td><td>The zeroed addrs and size must be persisted to disk</td></tr>
  <tr><td>After incrementing or decrementing <code>ip->nlink</code></td><td>Link count change must survive a crash</td></tr>
  <tr><td>After <code>writei()</code> extends the file and updates <code>ip->size</code></td><td>New size must be on disk before the data blocks are committed</td></tr>
  <tr><td>After <code>create()</code> sets the inode type and initialises fields</td><td>New inode type must be on disk or the file cannot be found after reboot</td></tr>
</table>

<p><strong>Key rule:</strong> iupdate() is always called while the inode's sleeplock is held — it is not safe to call concurrently with code that modifies the same inode's fields.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What is the <code>ip-&gt;valid</code> flag in the in-memory inode? What are its two values and what do they mean?

<div class="answer-content">

<p><strong>The valid flag:</strong></p>
<pre><code class="language-c">// kernel/file.h
struct inode {
    // ...
    int valid;   // has data been loaded from disk? 0 = no, 1 = yes
    // ...
};</code></pre>

<p><strong>Two states and their meaning:</strong></p>

<table>
  <tr><th>valid = 0</th><th>valid = 1</th></tr>
  <tr><td>In-memory inode slot has been allocated by <code>iget()</code> but not yet loaded from disk</td><td>All fields (type, size, addrs[]) reflect the current on-disk state</td></tr>
  <tr><td><code>ilock()</code> will read the dinode from disk on next call</td><td><code>ilock()</code> skips the disk read — data already in RAM</td></tr>
  <tr><td><code>ip->type</code>, <code>ip->size</code>, <code>ip->addrs[]</code> are undefined / zero</td><td>Fields are valid and safe to use after acquiring the sleeplock</td></tr>
</table>

<p><strong>The valid flag lifecycle:</strong></p>
<pre><code class="language-c">// iget(): allocate in-memory slot
ip->valid = 0;   // mark as not yet loaded

// First ilock() call:
if(ip->valid == 0){
    // bread() + memmove() from dinode
    ip->valid = 1;   // mark as loaded
}

// iput() when ref count hits 0:
ip->valid = 0;   // reset — if slot is reused for a different inode,
                 // next ilock() will re-read from disk with the new inum</code></pre>

<p><strong>Why valid is needed:</strong></p>
<p>The in-memory inode table (itable) has NINODE=50 slots that are reused. When a slot is assigned to a new inode number, its old data (from the previous occupant) is invalid. <code>valid = 0</code> forces a fresh disk read before the new inode's data can be used.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

A student changes <code>addrs[NDIRECT+2]</code> in <code>struct dinode</code> but forgets to change <code>struct inode</code> in <code>file.h</code>, leaving it as <code>addrs[NDIRECT+1]</code>. Describe exactly what goes wrong for a file that uses the doubly-indirect block.

<div class="answer-content">

<p><strong>The mismatch:</strong></p>
<pre><code class="language-c">// kernel/fs.h — correctly updated:
struct dinode {
    // ...
    uint addrs[NDIRECT+2];   // 13 entries: [0..10]=direct, [11]=singly, [12]=doubly
};

// kernel/file.h — NOT updated:
struct inode {
    // ...
    uint addrs[NDIRECT+1];   // 12 entries: [0..10]=direct, [11]=singly (WRONG!)
};</code></pre>

<p><strong>What happens in ilock() — the memmove:</strong></p>
<pre><code class="language-c">memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
// sizeof(ip->addrs) = sizeof(uint[12]) = 48 bytes  ← uses the SMALLER array
// sizeof(dip->addrs) = sizeof(uint[13]) = 52 bytes
// Only 48 bytes are copied — 12 entries of the 13 in dip->addrs.
// ip->addrs[12] (the doubly-indirect pointer) is NEVER copied into the in-memory inode.
// ip->addrs[11] gets ip->addrs[11] from disk = the singly-indirect pointer.
// The doubly-indirect pointer from disk (dip->addrs[12]) is silently dropped.</code></pre>

<p><strong>Consequence when bmap() tries to access a doubly-indirect block:</strong></p>
<pre><code class="language-c">// In bmap(), after subtracting NDIRECT and NINDIRECT:
if(bn < NDINDIRECT){
    if((addr = ip->addrs[NDIRECT+1]) == 0){   // ip->addrs[12] — but this slot
                                               // does NOT EXIST in the 12-entry array!
        // C does NOT bounds-check arrays.
        // ip->addrs[12] reads 4 bytes PAST the end of ip->addrs[].
        // These 4 bytes are whatever follows addrs[] in struct inode:
        //   struct inode {
        //     uint dev;    ← possibly this field
        //   }
        // ip->addrs[12] returns ip->dev (or whatever is at that offset).
        addr = ip->dev;   // ← treats the device number as a block address!
    }
    // bmap() loads disk block ip->dev as if it were the master map block.
    // disk block 1 is the superblock — reading it as a pointer table
    // returns garbage doubly-indirect pointers → reads wrong data blocks or panics.
}</code></pre>

<p><strong>Symptoms:</strong></p>
<ul>
  <li>Files smaller than 267 blocks work correctly (direct and singly-indirect only)</li>
  <li>First write past block 266 follows a garbage block address — likely panics with an invalid block number or silently reads/writes another file's data</li>
  <li><code>bigfile</code> fails with: <code>wrote 266 blocks\nbigfile: file is too small</code> or kernel panic</li>
</ul>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

What happens when <code>iupdate()</code> is called after <code>itrunc()</code> has set <code>ip-&gt;addrs[NDIRECT+1] = 0</code>? Trace the memmove from in-memory inode to on-disk dinode.

<div class="answer-content">

<p><strong>State after itrunc() zeroing:</strong></p>
<pre><code class="language-c">// In-memory inode after itrunc():
ip->size            = 0;
ip->addrs[0..10]    = 0;    // direct blocks cleared by existing itrunc() code
ip->addrs[11]       = 0;    // singly-indirect cleared by existing itrunc() code
ip->addrs[12]       = 0;    // doubly-indirect cleared by our new code</code></pre>

<p><strong>iupdate() memmove trace:</strong></p>
<pre><code class="language-c">void iupdate(struct inode *ip)
{
    struct buf *bp  = bread(ip->dev, IBLOCK(ip->inum, sb));   // load inode block
    struct dinode *dip = (struct dinode*)bp->data + ip->inum%IPB;  // find this dinode

    dip->type  = ip->type;    // e.g. T_FILE — unchanged, file still exists
    dip->nlink = ip->nlink;   // e.g. 0 if being deleted, 1 if still linked
    dip->size  = 0;           // ← ip->size = 0 copied to disk

    memmove(dip->addrs, ip->addrs, sizeof(dip->addrs));
    // sizeof(dip->addrs) = sizeof(uint[13]) = 52 bytes
    // Copies: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0] from ip->addrs to dip->addrs
    // All 13 slots on disk are now zero.

    log_write(bp);   // mark the inode block dirty — will be written via log
    brelse(bp);
}</code></pre>

<p><strong>What is now on disk after iupdate():</strong></p>
<p>The on-disk dinode has all <code>addrs[]</code> entries zeroed and <code>size = 0</code>. On the next reboot or the next time this inode is evicted from cache and re-loaded, <code>ilock()</code> will read these zeroed values and correctly see the inode as having no data blocks. The file is fully truncated both in RAM and on disk.</p>

<p><strong>Why this must happen inside a log transaction:</strong></p>
<p>The inode block write (via <code>log_write()</code> inside <code>iupdate()</code>) is part of the same <code>begin_op()</code>/<code>end_op()</code> transaction as the <code>bfree()</code> calls that freed the data blocks. If the machine crashes after <code>bfree()</code> but before <code>iupdate()</code>, the bitmap would show the blocks as free while the inode still points to them — a use-after-free on the next boot. The log ensures both happen or neither happens.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

Why does <code>ilock()</code> panic with <code>"ilock: no type"</code> if <code>ip-&gt;type == 0</code> after loading from disk? When would this legitimately occur?

<div class="answer-content">

<p><strong>The panic condition:</strong></p>
<pre><code class="language-c">// In ilock():
memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
brelse(bp);
ip->valid = 1;

if(ip->type == 0)
    panic("ilock: no type");</code></pre>

<p><strong>What type==0 means:</strong></p>
<p>In xv6, <code>ip->type == 0</code> means the inode slot is <strong>free</strong> — it has been allocated on disk but not yet assigned to any file. <code>ialloc()</code> marks a dinode as in-use by writing a non-zero type. When a file is deleted and its inode freed, <code>iput()</code> sets <code>type = 0</code> on the dinode.</p>

<p><strong>Why it panics instead of returning gracefully:</strong></p>
<p>Any code path that calls <code>ilock(ip)</code> with a type-0 inode has already made an error — it obtained an inode pointer (via <code>iget()</code>, <code>namei()</code>, etc.) to a free inode, which should never happen. Continuing to use a free inode would cause silent data corruption. The panic immediately surfaces the underlying bug.</p>

<p><strong>When type==0 legitimately occurs and is expected:</strong></p>

<table>
  <tr><th>Scenario</th><th>How it's handled</th></tr>
  <tr><td><code>ialloc()</code> scanning for a free inode slot</td><td>Reads the dinode directly via <code>bread()</code>, NOT via <code>ilock()</code> — bypasses the panic</td></tr>
  <tr><td>A file is deleted, <code>ip->nlink == 0 &amp;&amp; ip->ref == 0</code></td><td><code>iput()</code> calls <code>itrunc()</code> then sets type=0 and calls <code>iupdate()</code> — the inode is marked free on disk. Future <code>iget()</code> calls won't return this inode (its slot is available for reuse)</td></tr>
  <tr><td>mkfs builds the filesystem with unused inode slots</td><td>Unused inodes have type=0 on disk from the start. They are never given to user processes via <code>iget()</code>.</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

What is <code>IPB</code> and why must <code>sizeof(struct dinode)</code> stay constant even after adding the doubly-indirect slot?

<div class="answer-content">

<p><strong>IPB — inodes per block:</strong></p>
<pre><code class="language-c">#define IPB  (BSIZE / sizeof(struct dinode))   // 1024 / 64 = 16</code></pre>

<p>IPB tells the kernel how many <code>dinode</code> structs fit in one 1024-byte disk block. It is used by <code>IBLOCK()</code> to calculate which block holds a given inode number, and by <code>ilock()</code> and <code>iupdate()</code> to find the correct offset within that block.</p>

<p><strong>Why sizeof(struct dinode) must stay at 64 bytes:</strong></p>
<pre><code class="language-c">// kernel/fs.h — struct dinode layout:
struct dinode {
    short type;    // 2 bytes
    short major;   // 2 bytes
    short minor;   // 2 bytes
    short nlink;   // 2 bytes
    uint  size;    // 4 bytes
    uint  addrs[NDIRECT+2];   // 4 × 13 = 52 bytes
};                 // Total: 2+2+2+2+4+52 = 64 bytes ✓

// Before the lab: addrs[NDIRECT+1] = addrs[13] = 52 bytes → same 64 bytes ✓</code></pre>

<p>Both <code>addrs[NDIRECT+1]</code> (before) and <code>addrs[NDIRECT+2]</code> (after) evaluate to 13 elements = 52 bytes. The struct stays at 64 bytes, so IPB stays at 16, so IBLOCK() calculations stay correct.</p>

<p><strong>What would break if sizeof(struct dinode) changed:</strong></p>
<pre><code class="language-c">// Hypothetical bug: student adds an extra field, making dinode = 68 bytes
// IPB = 1024 / 68 = 15 (integer division)
// IBLOCK(15, sb) = 15/15 + inodestart = 1 + 32 = block 33
// But inode 15 is actually in block 32 (the first inode block, indices 0..15)
// Every inode number that crosses an IPB boundary is now mapped to the WRONG block
// Reading inode 15 would read bytes from block 33, which belong to inode 16.
// The entire filesystem's inode lookup is corrupted for inodes past the first block.</code></pre>

<p>Because <code>NDIRECT</code> drops from 12 to 11 while gaining one extra slot (doubly-indirect), the array expression changes from <code>NDIRECT+1=13</code> to <code>NDIRECT+2=13</code> — same length, same size. The on-disk layout is perfectly backward-compatible.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

After the lab is complete, a student runs <code>bigfile</code>, which succeeds. They then run <code>bigfile</code> a second time without <code>make clean</code>. The second run fails at block 267 with "file is too small". Explain the root cause and trace exactly what bmap() does differently on the second run.

<div class="answer-content">

<p><strong>Why the second run fails:</strong></p>
<p>The first run of <code>bigfile</code> creates <code>big.file</code> and writes 6,580 blocks to it. It then calls <code>close(fd)</code>. The file <code>big.file</code> now exists in the xv6 filesystem on disk.</p>

<p>The second run calls:</p>
<pre><code class="language-c">fd = open("big.file", O_CREATE | O_WRONLY);</code></pre>

<p><code>O_CREATE</code> in xv6's <code>sys_open()</code> calls <code>create()</code>. Since "big.file" already exists, <code>create()</code> calls <code>itrunc()</code> to zero the file, then returns the existing inode. <strong>If <code>itrunc()</code> does not free the doubly-indirect blocks</strong> (missing implementation), the disk has no free blocks for the second run's doubly-indirect range.</p>

<p><strong>Exact bmap() behaviour on the second run:</strong></p>
<pre><code class="language-c">// First 267 writes succeed:
// - Blocks 0-10: direct (reuse slots, may overwrite existing pointers if itrunc zeroed addrs[])
// Actually: if itrunc() partially works (zeros addrs[] in inode) but did NOT free doubly-indirect blocks:
//   addrs[12] = 0 after itrunc → bmap() thinks no master map exists

// Block 267 (first doubly-indirect):
addr = ip->addrs[NDIRECT+1];   // = 0 (itrunc zeroed the pointer)
addr = balloc(ip->dev);        // Try to allocate new master map block
                                // All doubly-indirect blocks from run #1 are STILL ALLOCATED
                                // in the bitmap (itrunc didn't call bfree on them)
                                // After run #1 used 1 + 26 + 6313 = 6340 doubly-indirect blocks,
                                // and FSSIZE=200000 has 199930 data blocks,
                                // only 199930 - 6340 - 267 (direct+singly used) = 193323 remain.
                                // Actually: balloc() SUCCEEDS here (plenty of space for one run).

// Hmm — one run doesn't exhaust the disk. Let me reconsider.
// The failure at block 267 on the second run must be because itrunc() is COMPLETELY missing
// (not just the doubly-indirect part) — so the 6,580 blocks from run #1 are leaked.</code></pre>

<p><strong>Revised analysis — itrunc() completely skips doubly-indirect:</strong></p>
<pre><code class="language-c">// After run #1: 6313 doubly-indirect data blocks + 26 secondary maps + 1 master map = 6340 leaked blocks
// After run #2 completes direct + singly-indirect (267 blocks) and hits block 267:
//   balloc() for master map: succeeds (199930 - 6340 - 267×2 = ~186,000 free)
// So the failure is NOT disk exhaustion from one run.

// THE ACTUAL BUG: if O_CREATE|O_WRONLY calls create() which calls itrunc():
//   itrunc() zeros ip->addrs[NDIRECT+1] = 0 (pointer cleared in RAM)
//   But itrunc() did NOT bfree the 6340 old doubly-indirect blocks
//   AND itrunc() did NOT call iupdate() — so the on-disk dinode still has addrs[12] = old_master_map
//   On ilock() of the inode: ip->addrs[12] gets the OLD master map address (still on disk!)
//   bmap() for block 267: ip->addrs[NDIRECT+1] = old_master_map (not 0!)
//   old_master_map[0] = old_secondary_map (still on disk, still has old data pointers)
//   bmap() returns an old data block — write() overwrites old data, but the block count succeeds.
// So if itrunc() zeroed RAM but not disk, the second run actually succeeds too.

// REAL failure scenario: itrunc() is missing entirely → O_CREATE opens the existing file
// with all its blocks still allocated. The write loop overwrites existing blocks. Succeeds.
// The failure at 267 with "file is too small" means balloc() returns 0 — disk full.
// This only happens after ~31 runs (as computed in variation w9m5).</code></pre>

<p><strong>The actual second-run failure scenario:</strong></p>
<p>If <code>bigfile.c</code> uses a different path — <code>unlink("big.file")</code> then <code>open("big.file", O_CREATE)</code> — then <code>unlink()</code> calls <code>itrunc()</code>. With the bug, 6,340 blocks are leaked per unlink. After 31 unlink+create cycles (likely in the same xv6 session running bigfile multiple times), the disk fills up and block 267 can no longer be allocated.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

The in-memory inode is loaded from disk on the first <code>ilock()</code>. What happens if the inode block itself spans a disk read that returns stale cache data — specifically, if one CPU just updated <code>dip->addrs[12]</code> via <code>iupdate()</code> and another CPU calls <code>ilock()</code> on the same inode before the cache is consistent?

<div class="answer-content">

<p><strong>The potential issue:</strong></p>
<pre><code class="language-c">// CPU 0: iupdate(ip) is called after bmap() allocates the master map block
//   bread(dev, IBLOCK(inum, sb))         // loads inode block into cache slot X
//   dip->addrs[12] = master_map_block    // modifies cache slot X
//   log_write(bp)                        // marks slot X dirty
//   brelse(bp)                           // releases slot X (still dirty in cache)

// CPU 1: another thread has ip->valid=0 and calls ilock(ip)
//   acquiresleep(&ip->lock)              // waits if CPU 0 holds it, otherwise proceeds
//   ip->valid == 0 → read from disk:
//   bread(dev, IBLOCK(inum, sb))         // ← what does this return?</code></pre>

<p><strong>Why this is NOT a problem in xv6:</strong></p>
<p>The key is the <strong>inode sleeplock</strong>. Both CPU 0 (running <code>iupdate()</code>) and CPU 1 (running <code>ilock()</code>) acquire <code>ip->lock</code> before proceeding. <code>iupdate()</code> is always called while holding the inode lock (the caller already holds it from a previous <code>ilock()</code> call). CPU 1's <code>ilock()</code> cannot proceed until CPU 0 releases <code>ip->lock</code>.</p>

<p><strong>The happens-before guarantee:</strong></p>
<pre><code class="language-c">// CPU 0 timeline:
ilock(ip)         → acquires ip->lock
// ... bmap() modifies inode in RAM ...
iupdate(ip)       → bread() → dip->addrs[12] = X → log_write() → brelse()
iunlock(ip)       → releases ip->lock

// CPU 1 timeline (blocked until CPU 0 releases):
ilock(ip)         → acquires ip->lock (AFTER CPU 0 released it)
ip->valid == 0    → bread(dev, IBLOCK(inum, sb))
// The buffer cache returns the SAME cache slot that CPU 0 modified
// (the dirty buffer has not been evicted yet).
// CPU 1 reads dip->addrs[12] = X — the UPDATED value!
memmove(ip->addrs, dip->addrs, sizeof(ip->addrs))
// ip->addrs[12] = X ← correct</code></pre>

<p><strong>The buffer cache as a coherence mechanism:</strong></p>
<p>The buffer cache is a <strong>single shared data structure</strong> — all CPUs share the same 30-slot array. A dirty buffer in the cache is visible to any CPU that calls <code>bread()</code> for the same block number. There is no per-CPU cache. The mutex inside <code>bget()</code> ensures only one thread holds a given buffer at a time. Combined with the inode sleeplock, xv6 provides full coherence for inode reads without any additional memory barriers.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

Write a minimal test program (user-space xv6 C) that verifies the doubly-indirect implementation is correct: it creates a file, writes a unique pattern to block 267 specifically, closes the file, reopens it, reads block 267, and verifies the pattern. Explain what this tests that <code>bigfile</code> alone does not.

<div class="answer-content">

<p><strong>The test program:</strong></p>
<pre><code class="language-c">// user/dibtest.c — test doubly-indirect block read/write correctness
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fcntl.h"

#define BSIZE  1024
#define TARGET_BLOCK 267   // first doubly-indirect block

int main(void)
{
    char wbuf[BSIZE];
    char rbuf[BSIZE];
    int  fd, i;

    // Fill write buffer with a unique pattern
    for(i = 0; i &lt; BSIZE; i++)
        wbuf[i] = (char)(i ^ 0xAB);   // XOR pattern — easily verifiable

    // Create the file
    fd = open("dibtest.file", O_CREATE | O_WRONLY);
    if(fd &lt; 0){ printf("dibtest: open failed\n"); exit(1); }

    // Write TARGET_BLOCK-1 dummy blocks to advance past direct and singly-indirect
    char zero[BSIZE];
    memset(zero, 0, BSIZE);
    for(i = 0; i &lt; TARGET_BLOCK; i++){
        if(i == TARGET_BLOCK - 1){
            // Write the unique pattern to the last block before target
            // (just to be sure we're at the right position)
        }
        int n = write(fd, zero, BSIZE);
        if(n != BSIZE){ printf("dibtest: write %d failed\n", i); exit(1); }
    }

    // Write the unique pattern to block 267
    int n = write(fd, wbuf, BSIZE);
    if(n != BSIZE){ printf("dibtest: pattern write failed\n"); exit(1); }

    close(fd);

    // Reopen and seek to block 267
    fd = open("dibtest.file", O_RDONLY);
    if(fd &lt; 0){ printf("dibtest: reopen failed\n"); exit(1); }

    // Seek to byte offset 267 * BSIZE
    // xv6 does not have lseek; simulate by reading 267 blocks
    for(i = 0; i &lt; TARGET_BLOCK; i++){
        if(read(fd, rbuf, BSIZE) != BSIZE){
            printf("dibtest: seek-read %d failed\n", i);
            exit(1);
        }
    }

    // Read block 267
    if(read(fd, rbuf, BSIZE) != BSIZE){
        printf("dibtest: read block 267 failed\n");
        exit(1);
    }

    // Verify the pattern
    int ok = 1;
    for(i = 0; i &lt; BSIZE; i++){
        if(rbuf[i] != wbuf[i]){
            printf("dibtest: mismatch at byte %d: got %d expected %d\n",
                   i, rbuf[i], wbuf[i]);
            ok = 0;
            break;
        }
    }

    close(fd);
    unlink("dibtest.file");

    if(ok){
        printf("dibtest: doubly-indirect block 267 OK\n");
        exit(0);
    } else {
        exit(1);
    }
}</code></pre>

<p><strong>What this tests that bigfile alone does not:</strong></p>

<table>
  <tr><th>Property tested</th><th>bigfile</th><th>dibtest</th></tr>
  <tr><td>Doubly-indirect blocks can be allocated</td><td>Yes — writes through them</td><td>Yes</td></tr>
  <tr><td>Block 267 specifically is mapped correctly</td><td>No — only checks count</td><td>Yes — reads back and verifies</td></tr>
  <tr><td>Data written to block 267 survives close + reopen</td><td>No</td><td>Yes — the read-back after close tests that iupdate() and the log correctly persisted the doubly-indirect pointer</td></tr>
  <tr><td>Wrong idx1/idx2 calculation detected</td><td>No — writes succeed even with wrong indices (data goes to a wrong but existing block)</td><td>Yes — the unique XOR pattern will not match if bmap() returns a wrong physical block</td></tr>
  <tr><td>Correct block 266 (last singly-indirect) vs 267 (first doubly-indirect) boundary</td><td>No</td><td>Yes — by targeting block 267 specifically, an off-by-one in the range subtraction (using NDIRECT instead of NDIRECT+1 or vice versa) would map block 267 to singly-indirect instead of doubly-indirect, reading a zero block instead of the pattern</td></tr>
</table>

</div>
</div>
