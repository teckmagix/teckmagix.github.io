---
layout: page
title: "Lab Predictions - lab3-e5g8"
lab: lab3
description: "Exam predictions for Lab3-w8: on-disk layout, superblock fields, mkfs internals, nmeta calculation, and inode region arithmetic."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

After changing FSSIZE to 200,000, mkfs prints <code>nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000</code>. What does each number mean and how are they all calculated?

<div class="answer-content">

<p><strong>Breaking down the nmeta line:</strong></p>
<pre><code class="language-c">// boot = 1 block (block 0) — reserved for bootloader code
// super = 1 block (block 1) — the superblock

// log blocks 30: defined by LOGSIZE in kernel/param.h
//   #define LOGSIZE (MAXOPBLOCKS*3+3) = (10*3+3) = 33... 
//   Actually LOGSIZE = 30 — check kernel/param.h: #define LOGSIZE 30

// inode blocks 13: how many blocks hold all inodes
//   NINODES = 200 (from kernel/param.h)
//   sizeof(struct dinode) = 64 bytes → IPB = 1024/64 = 16 inodes per block
//   inode blocks = ⌈200/16⌉ = ⌈12.5⌉ = 13 blocks

// bitmap blocks 25 (for FSSIZE=200000):
//   BPB = BSIZE*8 = 1024*8 = 8192 bits per bitmap block
//   bitmap blocks = ⌈200000/8192⌉ = ⌈24.41⌉ = 25 blocks

// nmeta = 1 + 1 + 30 + 13 + 25 = 70 blocks
// data blocks = 200000 - 70 = 199930</code></pre>

<p><strong>How each region begins on disk:</strong></p>

<table>
  <tr><th>Block</th><th>Region</th><th>Superblock field</th></tr>
  <tr><td>0</td><td>Boot block</td><td>—</td></tr>
  <tr><td>1</td><td>Superblock</td><td>sb.size = 200000, sb.nblocks = 199930</td></tr>
  <tr><td>2</td><td>Log start</td><td>sb.logstart = 2</td></tr>
  <tr><td>32</td><td>Inode start</td><td>sb.inodestart = 32</td></tr>
  <tr><td>45</td><td>Bitmap start</td><td>sb.bmapstart = 45 (original); 57 for FSSIZE=200000</td></tr>
  <tr><td>70</td><td>Data blocks start</td><td>—</td></tr>
</table>

<p><strong>For the original FSSIZE=2000:</strong></p>
<pre><code class="language-c">bitmap blocks = ⌈2000/8192⌉ = 1 block
nmeta = 1 + 1 + 30 + 13 + 1 = 46 blocks
data blocks = 2000 - 46 = 1954 blocks</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What does the superblock store, and which fields in it does <code>bmap()</code> and <code>ilock()</code> use directly?

<div class="answer-content">

<p><strong>The superblock struct:</strong></p>
<pre><code class="language-c">// kernel/fs.h
struct superblock {
    uint magic;       // Must be FSMAGIC (0x10203040)
    uint size;        // Size of file system image (blocks) = FSSIZE = 200000
    uint nblocks;     // Number of data blocks = 199930
    uint ninodes;     // Number of inodes = NINODES = 200
    uint nlog;        // Number of log blocks = LOGSIZE = 30
    uint logstart;    // Block number of first log block = 2
    uint inodestart;  // Block number of first inode block = 32
    uint bmapstart;   // Block number of first free map block
};

// Read from disk once at boot into global variable sb:
struct superblock sb;    // kernel/fs.c</code></pre>

<p><strong>Which fields each function uses:</strong></p>

<table>
  <tr><th>Function</th><th>Superblock fields used</th><th>For what</th></tr>
  <tr><td><code>ilock()</code></td><td><code>sb.inodestart</code></td><td>Via <code>IBLOCK(inum, sb)</code> to find which disk block holds this inode</td></tr>
  <tr><td><code>iupdate()</code></td><td><code>sb.inodestart</code></td><td>Same — to write the modified inode back to the correct disk block</td></tr>
  <tr><td><code>ialloc()</code></td><td><code>sb.ninodes</code>, <code>sb.inodestart</code></td><td>Scan all inodes to find a free slot</td></tr>
  <tr><td><code>balloc()</code></td><td><code>sb.size</code>, <code>sb.bmapstart</code></td><td>Scan bitmap blocks to find a free data block</td></tr>
  <tr><td><code>bfree()</code></td><td><code>sb.bmapstart</code></td><td>Via <code>BBLOCK(b, sb)</code> to find the bitmap block for block b</td></tr>
  <tr><td><code>bmap()</code></td><td>None directly</td><td>bmap() calls balloc() and bread() — which use sb internally</td></tr>
</table>

<p><strong>Where the superblock is populated:</strong></p>
<pre><code class="language-c">// kernel/fs.c, fsinit():
void fsinit(int dev) {
    readsb(dev, &sb);         // bread() block 1 → copy to global sb
    if(sb.magic != FSMAGIC)
        panic("invalid file system");
}</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

xv6's disk has a boot block at block 0, but the lab PDF says "block 0 is never a valid data block address." Why does <code>balloc()</code> return 0 on failure instead of a valid block number, and what ensures block 0 is never allocated as a data block?

<div class="answer-content">

<p><strong>Why 0 is returned on failure:</strong></p>
<pre><code class="language-c">// In balloc():
static uint balloc(uint dev)
{
    for(b = 0; b &lt; sb.size; b += BPB){
        // ... scan bitmap ...
        if(free_bit_found){
            // ... allocate and return ...
            return b + bi;   // returns the actual block number
        }
    }
    printf("balloc: out of blocks\n");
    return 0;                // ← 0 signals failure
}</code></pre>

<p>0 is used as the "null" / "not allocated" sentinel throughout xv6's filesystem code. Every inode's <code>addrs[]</code> entry initialises to 0, meaning "not yet allocated." When bmap() checks <code>if((addr = ip->addrs[NDIRECT+1]) == 0)</code>, it uses 0 as "this slot is empty." Using 0 as the failure return from <code>balloc()</code> is consistent with this convention.</p>

<p><strong>What ensures block 0 is never allocated by balloc():</strong></p>
<p>The bitmap covers all blocks 0 through FSSIZE-1. Block 0 is the boot block. When <code>mkfs</code> builds the filesystem, it marks all metadata blocks (including block 0) as <em>already allocated</em> in the bitmap:</p>
<pre><code class="language-c">// mkfs/mkfs.c — marks all metadata blocks as in use:
for(i = 0; i &lt; nmeta; i++)
    wsect(BBLOCK(i, sb), bitmap_block_with_bit_i_set);
// nmeta = 70 → blocks 0..69 are all marked allocated in the bitmap
// balloc() scans the bitmap and skips any bit that is already 1.
// Block 0 has bit 0 = 1 (set by mkfs) → never returned by balloc().</code></pre>

<p>So <code>balloc()</code> physically cannot return 0 on a correctly formatted xv6 filesystem — the boot block's bitmap bit is always 1. Returning 0 as an error code works safely because 0 can never be a legitimately allocated data block.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What is the <code>BBLOCK</code> macro and why does xv6 need it? Give a worked example finding the bitmap block for data block 10,000 with FSSIZE=200,000.

<div class="answer-content">

<p><strong>The BBLOCK macro:</strong></p>
<pre><code class="language-c">// kernel/fs.h
#define BBLOCK(b, sb) ((b)/BPB + sb.bmapstart)
// b        = the block number whose free status you want to check or update
// BPB      = BSIZE*8 = 8192  (bits per bitmap block)
// sb.bmapstart = the disk block number where the bitmap region begins</code></pre>

<p><strong>Why it's needed:</strong></p>
<p>The free-space bitmap is spread across multiple consecutive blocks (25 blocks for FSSIZE=200,000). A single block can only track 8,192 disk blocks. To find/update the bitmap bit for block N, you must first determine which of the 25 bitmap blocks contains bit N.</p>

<p><strong>Worked example — finding the bitmap block for data block 10,000:</strong></p>
<pre><code class="language-c">// Setup: FSSIZE=200000, BSIZE=1024, BPB=8192
// nmeta = 70 → sb.bmapstart = 32+13+1... let's derive it:
//   logstart = 2, log blocks = 30 → log ends at block 31
//   inodestart = 32, inode blocks = 13 → inodes occupy blocks 32-44
//   bmapstart = 45 (original, FSSIZE=2000)
//   For FSSIZE=200000: bmapstart = 45 + (25-1) extra bitmap blocks... 
//   Actually: bmapstart = 1+1+30+13 = 45 (same, bitmap always starts after inodes)
//   Wait — for FSSIZE=200000 with 25 bitmap blocks:
//   2 (boot+super) + 30 (log) + 13 (inodes) = 45 → bmapstart = 45

// BBLOCK(10000, sb) = 10000 / 8192 + 45
//                   = 1 + 45
//                   = disk block 46

// Interpretation:
//   Block 45 (first bitmap block) covers blocks 0 - 8191
//   Block 46 (second bitmap block) covers blocks 8192 - 16383
//   Block 10000 falls in the range 8192-16383 → it's tracked by bitmap block 46

// Which bit within block 46?
bit_index = 10000 % 8192 = 1808
byte_index = 1808 / 8 = 226
bit_within_byte = 1808 % 8 = 0

// To check/set bit for block 10000:
bp = bread(dev, BBLOCK(10000, sb));   // bread block 46
m  = 1 &lt;&lt; (1808 % 8);               // = 1 &lt;&lt; 0 = 0x01
if(bp->data[226] &amp; m) → block 10000 is allocated
bp->data[226] |= m  → allocate block 10000
bp->data[226] &= ~m → free block 10000</code></pre>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

After the lab is complete, how many total disk blocks does a freshly written <code>big.file</code> consume? Account for every level: direct, singly-indirect map, singly-indirect data, doubly-indirect master map, doubly-indirect secondary maps, and doubly-indirect data blocks.

<div class="answer-content">

<p><strong>bigfile writes TEST_BLOCKS = 6580 logical blocks. Breaking down by level:</strong></p>
<pre><code class="language-c">// Direct blocks: logical blocks 0–10
direct_data    = 11 blocks

// Singly-indirect: logical blocks 11–266
singly_map     = 1 block  (the indirect block itself, addrs[11])
singly_data    = 256 blocks  (data blocks pointed to by the map)

// Doubly-indirect: logical blocks 267–6579 (6580 - 267 = 6313 doubly-indirect data blocks)
doubly_data    = 6313 blocks  (actual data, pointed to by secondary maps)

// How many secondary maps are needed?
// Each secondary map covers 256 data blocks
secondary_maps = ⌈6313 / 256⌉ = ⌈24.66⌉ = 25 secondary map blocks

// The master map block (1 block, addrs[12]):
master_map     = 1 block

// Total blocks consumed:
total = 11 + 1 + 256 + 6313 + 25 + 1 = 6607 blocks</code></pre>

<p><strong>Summary table:</strong></p>

<table>
  <tr><th>Category</th><th>Count</th><th>Source</th></tr>
  <tr><td>Direct data blocks</td><td>11</td><td>addrs[0..10]</td></tr>
  <tr><td>Singly-indirect map block</td><td>1</td><td>addrs[11]</td></tr>
  <tr><td>Singly-indirect data blocks</td><td>256</td><td>indirect_map[0..255]</td></tr>
  <tr><td>Doubly-indirect master map</td><td>1</td><td>addrs[12]</td></tr>
  <tr><td>Doubly-indirect secondary maps</td><td>25</td><td>master_map[0..24]</td></tr>
  <tr><td>Doubly-indirect data blocks</td><td>6,313</td><td>secondary_map[i][j]</td></tr>
  <tr><td><strong>Total</strong></td><td><strong>6,607 blocks</strong></td><td></td></tr>
</table>

<p><strong>How much of the 199,930 available data blocks this uses:</strong></p>
<pre><code class="language-c">6607 / 199930 ≈ 3.3%

// After one bigfile run, 199,930 - 6607 = 193,323 data blocks remain free.
// itrunc() must free all 6607 blocks when big.file is deleted.
// (If itrunc() is buggy and only frees 11+1+256 = 268 blocks,
//  6607 - 268 = 6339 blocks are leaked per run.)</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

The superblock records <code>sb.ninodes = 200</code>. After running <code>bigfile</code> and <code>symlinktest</code> in the same xv6 session, roughly how many inodes are consumed and why does the system never run out?

<div class="answer-content">

<p><strong>Inodes consumed at boot (from mkfs):</strong></p>
<pre><code class="language-c">// mkfs pre-allocates inodes for all built-in binaries and directories.
// From the ls output in the lab PDF:
// inode 1 = root directory (.)
// inode 2 = README
// inode 3 = cat
// inode 4 = echo
// ...
// inode 22 = console (device)
// → mkfs allocates approximately 22 inodes at build time</code></pre>

<p><strong>Inodes consumed by bigfile:</strong></p>
<pre><code class="language-c">// bigfile creates one file: "big.file"
// → 1 inode allocated (inode 23, as shown in the ls output: big.file 2 23 6737920)
// bigfile does NOT create any symlinks → 0 symlink inodes</code></pre>

<p><strong>Inodes consumed by symlinktest:</strong></p>
<pre><code class="language-c">// symlinktest creates multiple test files and symlinks, then deletes them.
// Each symlink requires 1 inode (T_SYMLINK).
// The Basic Link test: 1 file + 1 symlink = 2 inodes
// The Chain test: 1 file + 4 symlinks = 5 inodes
// The Circle test: 2 symlinks = 2 inodes
// The Concurrent test: many processes × (1 file + 1 symlink) ≈ 20-30 inodes peak
// All test inodes are cleaned up (unlinked) before symlinktest exits.
// After symlinktest: those inode slots are freed → type=0, nlink=0, valid=0</code></pre>

<p><strong>Why the system never runs out (for this lab):</strong></p>
<p>With NINODES=200 and only ~22 permanently allocated by mkfs, there are ~178 free inode slots. bigfile uses 1. symlinktest uses up to ~30 at peak but releases them all. Total peak usage: ~53 inodes — well within the 200 limit. The only way to exhaust inodes would be to create and never delete hundreds of files, which neither bigfile nor symlinktest does.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

The lab PDF shows the <code>ls</code> output after a successful bigfile run, including <code>big.file 2 23 6737920</code>. How does <code>ls</code> obtain each of these four values, tracing through the xv6 source?

<div class="answer-content">

<p><strong>How ls gets each field:</strong></p>
<pre><code class="language-c">// xv6 ls implementation (user/ls.c):
struct stat st;
fstat(fd, &st);     // fills in the stat struct via sys_fstat() → ilock() → fills st

// struct stat (kernel/stat.h):
struct stat {
    int dev;        // device number
    uint ino;       // inode number
    short type;     // T_DIR=1, T_FILE=2, T_DEVICE=3, T_SYMLINK=4
    short nlink;    // number of hard links
    uint64 size;    // size in bytes
};

printf("%s %d %d %l\n", name, st.type, st.ino, st.size);</code></pre>

<p><strong>Tracing each value for big.file:</strong></p>

<table>
  <tr><th>Output field</th><th>Value</th><th>Source in xv6</th></tr>
  <tr><td><code>big.file</code></td><td>filename</td><td>Read from directory entry: <code>struct dirent { ushort inum; char name[DIRSIZ]; }</code> — the name field</td></tr>
  <tr><td><code>2</code></td><td>T_FILE</td><td><code>st.type = ip->type</code> — set by <code>create()</code> as T_FILE when bigfile opens with O_CREATE</td></tr>
  <tr><td><code>23</code></td><td>inode number</td><td><code>st.ino = ip->inum</code> — the integer index of this inode in the inode table; 23rd inode allocated (after the 22 built-in inodes)</td></tr>
  <tr><td><code>6737920</code></td><td>file size in bytes</td><td><code>st.size = ip->size</code> — updated by <code>writei()</code> as each block is written; final value = 6580 × 1024 = 6,737,920</td></tr>
</table>

<p><strong>The fstat() call chain:</strong></p>
<pre><code class="language-c">fstat(fd, &st)
→ sys_fstat(fd)
→ filestat(f, addr)
→ stati(f->ip, &st)       // fills st from the in-memory inode
→ st.type  = ip->type
   st.ino   = ip->inum
   st.nlink = ip->nlink
   st.size  = ip->size
→ copyout(p->pagetable, addr, &st)  // copy st to user space</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

What is <code>MAXOPBLOCKS</code> and how does it constrain the number of blocks that can be written in a single <code>begin_op()</code>/<code>end_op()</code> transaction? Why does <code>bigfile</code> stay within this limit even though it writes 6,580 blocks?

<div class="answer-content">

<p><strong>MAXOPBLOCKS:</strong></p>
<pre><code class="language-c">// kernel/param.h
#define MAXOPBLOCKS  10   // max # of blocks any FS op writes in one transaction
#define LOGSIZE      (MAXOPBLOCKS*3+3)   // = 33 log slots total</code></pre>

<p><code>MAXOPBLOCKS</code> is the maximum number of <code>log_write()</code> calls that can be made between a single <code>begin_op()</code>/<code>end_op()</code> pair. If a transaction would exceed this, <code>begin_op()</code> sleeps until the log has enough space.</p>

<p><strong>How bmap() stays within MAXOPBLOCKS per write() call:</strong></p>
<p>Each user <code>write(fd, buf, BSIZE)</code> call is one <code>begin_op()</code>/<code>end_op()</code> transaction. For one write at a cold doubly-indirect block, bmap() makes at most 3 <code>log_write()</code> calls:</p>
<pre><code class="language-c">// Worst case for one write() call in bmap():
log_write(bitmap_block);         // from balloc() for master map (if new)  → 1
log_write(master_map);           // after updating a1[idx1]               → 2
log_write(bitmap_block);         // from balloc() for secondary map        → 3
log_write(secondary_map);        // after updating a2[idx2]               → 4
log_write(bitmap_block);         // from balloc() for data block           → 5
// writei also logs:
log_write(data_block);           // the actual data                        → 6
log_write(inode_block);          // iupdate() for ip->size change          → 7
// Total: 7 log_write() calls ≤ MAXOPBLOCKS(10) ✓</code></pre>

<p><strong>Why bigfile stays within the limit:</strong></p>
<p>bigfile's loop calls <code>write(fd, buf, BSIZE)</code> — one <code>BSIZE</code> bytes at a time — which is one system call = one transaction. Each individual write touches at most 7 log blocks. This is well below MAXOPBLOCKS=10. If bigfile tried to write all 6,580 blocks in one giant write() call (which it doesn't), it would attempt 6580 × 7 = 46,060 log_write() calls in one transaction — causing <code>panic("log write too big")</code>.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

mkfs builds the filesystem with <code>NDIRECT</code> baked in. Explain what happens at the byte level if you compile the kernel with <code>NDIRECT=11</code> but boot with an <code>fs.img</code> that was built with <code>NDIRECT=12</code>. Trace a specific ilock() call for a file with 13 addrs[] entries on the old disk.

<div class="answer-content">

<p><strong>The mismatch scenario:</strong></p>
<pre><code class="language-c">// Old fs.img — built with NDIRECT=12:
// struct dinode { ... uint addrs[13]; }  → sizeof = 2+2+2+2+4+52 = 64 bytes
// dinode layout: [0..11]=direct(12), [12]=singly-indirect
// A file with a singly-indirect block has: addrs[12] = 500 (the indirect block)

// New kernel — compiled with NDIRECT=11:
// struct dinode { ... uint addrs[13]; }  → sizeof still 64 bytes (same!)
// dinode layout: [0..10]=direct(11), [11]=singly-indirect, [12]=doubly-indirect
// From the kernel's perspective: addrs[12] is the DOUBLY-INDIRECT pointer</code></pre>

<p><strong>Byte-level analysis — what the kernel reads:</strong></p>
<pre><code class="language-c">// On-disk dinode for the old file (NDIRECT=12 layout):
// Offset 0:  type  = 0x0002  (T_FILE)
// Offset 2:  major = 0
// Offset 4:  minor = 0
// Offset 6:  nlink = 0x0001
// Offset 8:  size  = 0x0000_2800  (10240 bytes = 10 blocks)
// Offset 12: addrs[0..11] = direct block addresses (e.g. 70,71,72,...,81) — 48 bytes
// Offset 60: addrs[12] = 500   ← SINGLY-INDIRECT pointer in old layout

// New kernel's ilock() calls:
// memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
// sizeof(ip->addrs) = sizeof(uint[13]) = 52 bytes  ← NEW kernel struct inode has 13 entries
// Copies bytes at offset 12..63 from dinode to ip->addrs[0..12]
// ip->addrs[11] = old addrs[11] (old direct block 11 — some data block, e.g. 81)
// ip->addrs[12] = old addrs[12] = 500  ← treated as DOUBLY-INDIRECT by new kernel!

// When the file is read:
// New bmap() for logical block 11:
//   bn = 11; 11 < NDIRECT(11)? No! (11 is NOT less than 11)
//   bn -= NDIRECT; bn = 0
//   0 < NINDIRECT? Yes → singly-indirect path
//   ip->addrs[NDIRECT=11] = old addrs[11] = 81  ← wrong! old addrs[11] was a DIRECT block
//   bread(dev, 81) → loads data block 81 as if it were an INDIRECT MAP
//   Reads data block 81's contents as block addresses
//   Returns garbage data addresses → reads completely wrong disk blocks for logical block 11</code></pre>

<p><strong>Symptoms:</strong></p>
<ul>
  <li>Files with ≤ 10 logical blocks (≤ 10 KB): read correctly (direct blocks 0–10 are the same in both layouts)</li>
  <li>Files with exactly 11 blocks: logical block 10 reads from old addrs[10] (correct), logical block 11 reads from old addrs[11] treated as an indirect block pointer (completely wrong)</li>
  <li>Files with singly-indirect blocks (>12 blocks in old layout): the singly-indirect block address (old addrs[12]) is now treated as a doubly-indirect block — reading it as a master map returns garbage secondary map addresses</li>
</ul>

<p><strong>Fix:</strong> Always <code>make clean</code> when changing NDIRECT to force mkfs to rebuild fs.img with the new layout.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

Calculate the exact disk block numbers where the master map, all secondary maps, and the first data block of <code>big.file</code> reside, given that the filesystem starts fresh (only the pre-built xv6 binaries exist) and <code>bigfile</code> is the first user program to run.

<div class="answer-content">

<p><strong>Starting state after mkfs (FSSIZE=200000):</strong></p>
<pre><code class="language-c">// Metadata blocks (already allocated, bitmap bits=1):
// Blocks 0-69: boot, super, log, inodes, bitmap = nmeta = 70 blocks

// Pre-allocated data blocks for xv6 binaries (from ls output):
// README(2305 bytes), cat, echo, forktest, grep, init, kill, ln, ls, mkdir, rm, sh,
// stressfs, usertests, grind, wc, zombie, bigfile, symlinktest = 19 files
// Root directory + subdirs = a few directory blocks

// Rough estimate: each binary averages ~40KB = 40 blocks
// 19 binaries × 40 blocks + a few small files ≈ ~800 data blocks used
// Data blocks used by mkfs: approximately blocks 70–870
// Next free block: approximately block 871

// NOTE: actual block numbers depend on exact binary sizes.
// For this problem, assume next free block = 871.</code></pre>

<p><strong>bigfile allocation sequence:</strong></p>
<pre><code class="language-c">// open("big.file", O_CREATE|O_WRONLY):
//   ialloc() → finds free inode, assigns e.g. inode #23
//   dirlink(root_dir, "big.file", 23) → may extend root dir

// First write (logical block 0 — direct):
//   bmap(ip, 0): addrs[0]=0 → balloc() → block 871
//   ip->addrs[0] = 871

// writes for logical blocks 1-10 (direct):
//   addrs[1..10] = blocks 872..881

// write for logical block 11 (first singly-indirect):
//   bmap(ip, 11): addrs[11]=0 → balloc() for indirect map → block 882
//   bread(882); a[0]=0 → balloc() for data → block 883
//   ip->addrs[11] = 882 (the singly-indirect MAP block)
//   first singly-indirect DATA block = 883

// writes for logical blocks 12-266 (singly-indirect data):
//   indirect map[1..255] = blocks 884..1138

// write for logical block 267 (FIRST doubly-indirect):
//   bmap(ip, 267):
//   bn = 267 - 11 - 256 = 0
//   addrs[12]=0 → balloc() for MASTER MAP → block 1139
//   master_map[0]=0 → balloc() for SECONDARY MAP #0 → block 1140
//   secondary_map_0[0]=0 → balloc() for DATA block → block 1141

// ANSWERS:
Master map block:          1139
Secondary map #0 block:    1140
First doubly-indirect data: 1141</code></pre>

<p><strong>Note on actual numbers:</strong></p>
<p>The exact block numbers depend on the actual sizes of the pre-built binaries. The lab PDF's <code>ls</code> output shows <code>big.file 2 23 6737920</code> — inode 23 confirms bigfile's inode is the 23rd allocated. The pattern above correctly shows the ordering: master map is allocated before secondary map #0, which is allocated before the first data block.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

The xv6 filesystem has <code>NINODES=200</code> maximum inodes. Construct a user-space program that deliberately exhausts the inode table, explain what error every subsequent file creation returns, and describe how <code>ialloc()</code> signals the failure.

<div class="answer-content">

<p><strong>The inode exhaustion program:</strong></p>
<pre><code class="language-c">// user/inodetest.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fcntl.h"

int main(void)
{
    char name[16];
    int count = 0;

    // mkfs pre-allocates ~22 inodes (root dir + all binaries).
    // NINODES = 200 → 178 free inodes remaining.
    // Create files until we hit the limit.

    while(1){
        // Generate unique filename: f0, f1, f2, ...
        name[0] = 'f';
        name[1] = '0' + (count / 10) % 10;
        name[2] = '0' + count % 10;
        name[3] = '\0';

        int fd = open(name, O_CREATE|O_WRONLY);
        if(fd < 0){
            printf("inode table full at file #%d\n", count);
            printf("open() returned -1: no more inodes\n");
            break;
        }
        close(fd);
        count++;
        if(count % 10 == 0) printf("created %d files\n", count);
    }

    // Try one more to confirm failure:
    int fd = open("extra", O_CREATE|O_WRONLY);
    printf("extra file open: %d (should be -1)\n", fd);

    exit(0);
}</code></pre>

<p><strong>What ialloc() does when all inodes are used:</strong></p>
<pre><code class="language-c">// kernel/fs.c — ialloc():
struct inode* ialloc(uint dev, short type)
{
    int inum;
    struct buf *bp;
    struct dinode *dip;

    for(inum = 1; inum &lt; sb.ninodes; inum++){   // scan all 200 inodes
        bp = bread(dev, IBLOCK(inum, sb));
        dip = (struct dinode*)bp->data + inum%IPB;
        if(dip->type == 0){                      // type==0 means free
            memset(dip, 0, sizeof(*dip));
            dip->type = type;                    // mark as allocated
            log_write(bp);
            brelse(bp);
            return iget(dev, inum);              // return the inode
        }
        brelse(bp);
    }
    printf("ialloc: no inodes\n");               // ← printed to console
    return 0;                                    // return 0 = failure
}</code></pre>

<p><strong>What happens when ialloc() returns 0:</strong></p>
<pre><code class="language-c">// create() calls ialloc():
struct inode* create(char *path, short type, short major, short minor)
{
    struct inode *ip = ialloc(dp->dev, type);
    if(ip == 0)
        return 0;   // propagates 0 up
    // ...
}

// sys_open() receives 0 from create():
ip = create(path, T_FILE, 0, 0);
if(ip == 0){
    end_op();
    return -1;       // ← -1 returned to user program
}

// User program sees:
fd = open("newfile", O_CREATE|O_WRONLY);
// fd = -1</code></pre>

<p><strong>Expected output of the test program:</strong></p>
<pre><code class="language-c">// mkfs uses ~22 inodes. NINODES=200 → ~178 free.
// Program creates files f00, f01, ..., f177:
created 10 files
created 20 files
...
created 170 files
inode table full at file #178     ← or wherever the limit is hit
open() returned -1: no more inodes
ialloc: no inodes                 ← printed by kernel to console
extra file open: -1 (should be -1)</code></pre>

</div>
</div>
