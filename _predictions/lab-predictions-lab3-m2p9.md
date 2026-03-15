---
layout: page
title: "Lab Predictions - lab3-m2p9"
lab: lab3
description: "Exam predictions for Lab3-w8: symlink analogies, hard link vs symlink, real-world uses, and the breadcrumb following logic."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

The lab PDF uses two analogies for a symlink: an "empty box with a sticky note" and a "postal forwarding address." Map each analogy to the actual kernel data structures and explain what each part corresponds to.

<div class="answer-content">

<p><strong>Analogy 1 — The sticky note box:</strong></p>

<table>
  <tr><th>Analogy part</th><th>Real xv6 structure</th></tr>
  <tr><td>Physical box containing a toy</td><td>Regular T_FILE inode — <code>ip->addrs[]</code> point to blocks containing actual data</td></tr>
  <tr><td>Empty box</td><td>T_SYMLINK inode — the data blocks do not contain the user's file content</td></tr>
  <tr><td>Sticky note inside</td><td>The target path string stored in the symlink's data block (e.g., <code>"/usr/bin/sh"</code>)</td></tr>
  <tr><td>"The toy is in the cabinet under the sink"</td><td>The text of the path string — tells the kernel where to look next</td></tr>
  <tr><td>Opening the box and reading the note</td><td><code>readi(ip, 0, target, 0, MAXPATH)</code> — reads the path string from the symlink's data block</td></tr>
  <tr><td>Going to the cabinet under the sink</td><td><code>iunlockput(ip); ip = namei(target); ilock(ip)</code> — follow the path to the actual file</td></tr>
</table>

<p><strong>Analogy 2 — The postal forwarding address:</strong></p>

<table>
  <tr><th>Analogy part</th><th>Real xv6 structure</th></tr>
  <tr><td>Sending mail to an address</td><td>User program calls <code>open("link_name", O_RDONLY)</code></td></tr>
  <tr><td>Post office checks the address</td><td><code>namei("link_name")</code> + <code>ilock(ip)</code> — kernel looks up and locks the inode</td></tr>
  <tr><td>Sees a forwarding note at the address</td><td><code>ip->type == T_SYMLINK</code> — kernel detects this is a symlink, not real data</td></tr>
  <tr><td>Reads the forwarding address</td><td><code>readi(ip, 0, target, 0, MAXPATH)</code> — reads the target path</td></tr>
  <tr><td>Delivers mail to the new address instead</td><td><code>ip = namei(target); ilock(ip)</code> — kernel opens the target file</td></tr>
  <tr><td>Post office forwarding limit (prevent infinite bouncing)</td><td><code>if(depth > 10) return -1</code> — depth guard prevents circular symlink loops</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

The lab PDF lists four real-world uses of symlinks: version control, space efficiency, stability, and flexibility. For each one, write a concrete xv6 <code>symlink()</code> call example and explain what problem it solves.

<div class="answer-content">

<p><strong>1. Version Control — pointing python3 to a specific version:</strong></p>
<pre><code class="language-c">symlink("/usr/bin/python3.11", "/usr/bin/python3");
// Problem solved: user-facing command "python3" doesn't need to change when you upgrade.
// Before upgrade: /usr/bin/python3 → /usr/bin/python3.10
// After upgrade:  unlink("/usr/bin/python3"); symlink("/usr/bin/python3.11", "/usr/bin/python3");
// All scripts using "python3" now use 3.11 without modification.</code></pre>

<p><strong>2. Space Efficiency — referencing one large file from multiple locations:</strong></p>
<pre><code class="language-c">// 10GB dataset exists at /data/corpus/english.txt
symlink("/data/corpus/english.txt", "/home/alice/project/training_data.txt");
symlink("/data/corpus/english.txt", "/home/bob/nlp/source.txt");
// Problem solved: each user has their own path to the file.
// Only ONE copy of the 10GB file exists on disk.
// Without symlinks: each user has a 10GB copy → 30GB total for 3 users.</code></pre>

<p><strong>3. Stability — mapping complex paths to simple ones:</strong></p>
<pre><code class="language-c">symlink("/mnt/disk_01/data/project_alpha", "/data");
// Problem solved: all code uses the simple path /data.
// When disk_01 is replaced by disk_02, only the symlink changes:
// unlink("/data"); symlink("/mnt/disk_02/data/project_alpha", "/data");
// No code changes needed — /data still works.</code></pre>

<p><strong>4. Flexibility — pointing to directories and across filesystems:</strong></p>
<pre><code class="language-c">symlink("/mnt/external_drive/photos", "/home/alice/photos");
// Problem solved: hard links cannot span filesystems or link to directories.
// A hard link to /mnt/external_drive/photos is impossible (different FS, different inode numbers).
// A symlink just stores a path string — it works across any filesystem boundary.</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

The lab PDF describes the "Breadcrumb Logic" for how the kernel follows symlinks in four steps. Map each step to the exact lines of code in <code>sys_open()</code>.

<div class="answer-content">

<p><strong>The four breadcrumb steps from the lab PDF:</strong></p>
<pre><code class="language-c">// Step 1: "Open 'Link B' → Kernel sees it is a Symlink"
ip = namei(path);       // get Link B's inode
ilock(ip);              // lock it
// while condition: ip->type == T_SYMLINK   ← "sees it is a Symlink"

// Step 2: "Read Target → Kernel finds the string 'File A' inside"
char target[MAXPATH];
int n = readi(ip, 0, (uint64)target, 0, MAXPATH);
target[n] = 0;          // "File A\0" is now in target[]

// Step 3: "Redirect → Kernel automatically switches to opening 'File A'"
iunlockput(ip);         // release Link B
ip = namei(target);     // look up "File A"
ilock(ip);              // lock File A's inode
// Loop condition re-evaluated: ip->type == T_FILE → exit loop
// → rest of sys_open() opens File A as if it were requested directly

// Step 4: "Loop Protection → Kernel stops after 10 jumps"
if(depth > 10){
    iunlockput(ip);
    end_op();
    return -1;          // "prevent infinite recursion (A → B → A)"
}
depth++;</code></pre>

<p><strong>Why the steps happen in this order:</strong></p>

<table>
  <tr><th>Step</th><th>Must come before</th><th>Reason</th></tr>
  <tr><td>1 (ilock)</td><td>Step 2 (readi)</td><td>readi() requires the inode lock to safely read data blocks</td></tr>
  <tr><td>4 (depth check)</td><td>Step 2 (readi)</td><td>Guard against infinite loops before doing any work</td></tr>
  <tr><td>3 (iunlockput)</td><td>namei(target)</td><td>Must release current lock before namei() acquires new locks (deadlock prevention)</td></tr>
  <tr><td>Step 2 (null-terminate)</td><td>namei(target)</td><td>namei() requires a null-terminated string; readi() does not add one</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

The lab PDF says Task 1 is "about how to handle data (making files bigger)" and Task 2 is "about how to handle metadata (making files point to other files)." What is the distinction between data and metadata in the xv6 file system?

<div class="answer-content">

<p><strong>Data vs Metadata in xv6:</strong></p>

<table>
  <tr><th>Concept</th><th>Task 1 (Large Files)</th><th>Task 2 (Symlinks)</th></tr>
  <tr><td><strong>What changes on disk</strong></td><td>Data blocks holding user content + map blocks (master map, secondary maps)</td><td>Metadata: new T_SYMLINK inode + directory entry + path string in data block</td></tr>
  <tr><td><strong>What the user cares about</strong></td><td>File can be 64 MB instead of 268 KB</td><td>One file can "be" another file — indirection</td></tr>
  <tr><td><strong>Kernel structure changed</strong></td><td>bmap() navigation + itrunc() cleanup + inode addrs[] layout</td><td>New inode type (T_SYMLINK) + new syscall (sys_symlink) + sys_open() following logic</td></tr>
</table>

<p><strong>Precise definitions in the context of xv6:</strong></p>

<p><strong>Data</strong> = the bytes the user wrote to a file. Stored in data blocks on disk, addressed through the inode's <code>addrs[]</code> array. Task 1 extends how many data blocks a file can have.</p>

<p><strong>Metadata</strong> = information <em>about</em> a file: its type, size, link count, ownership, and where its data blocks are. Stored in the inode (<code>struct dinode</code> on disk, <code>struct inode</code> in RAM) and in directory entries. Task 2 adds a new inode type (T_SYMLINK) and uses the inode's data blocks to store a path string — which is metadata about which other file to open, not user-visible content.</p>

<p><strong>The interesting overlap:</strong></p>
<p>A symlink's "data block" technically stores the target path string — bytes in a data block. But the purpose of those bytes is purely metadata (navigation instructions). The lab PDF correctly identifies Task 2 as being about metadata because the value of a symlink lies entirely in what it points to, not in any content it contains.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

The lab PDF says: "Unlike Hard Links, Symlinks can point to directories and across different disks." Explain precisely why hard links have each of these two limitations and why symlinks are exempt from them.

<div class="answer-content">

<p><strong>Limitation 1 — Hard links cannot span filesystems:</strong></p>
<pre><code class="language-c">// A hard link is just a directory entry:
struct dirent { ushort inum; char name[DIRSIZ]; };
// It stores an INODE NUMBER — an integer index into the inode table.

// Inode numbers are LOCAL to one filesystem.
// /dev/sda has its own inode table (inodes 1..200).
// /dev/sdb has its own inode table (inodes 1..200).
// Inode #47 on sda is a DIFFERENT file from inode #47 on sdb.

// Attempting a hard link across filesystems:
// ln /mnt/sdb/file /home/alice/link
// Would create dirent { inum=47, name="link" } in /home/alice
// But inum=47 in /home/alice's filesystem (sda) is a DIFFERENT inode
// → the link points to the wrong file!

// Hard links on different filesystems are therefore PROHIBITED by the OS.</code></pre>

<p><strong>Why symlinks are exempt:</strong></p>
<pre><code class="language-c">// A symlink stores a PATH STRING, not an inode number:
// symlink("/mnt/sdb/file", "/home/alice/link")
// The symlink inode's data block contains: "/mnt/sdb/file\0"
// When followed: namei("/mnt/sdb/file") looks up the path on the mounted filesystem
// The path lookup crosses mount points correctly — no inode number confusion.
// Symlinks work across filesystems because paths are filesystem-agnostic.</code></pre>

<p><strong>Limitation 2 — Hard links cannot link to directories:</strong></p>
<pre><code class="language-c">// If hard links to directories were allowed:
// mkdir("/home/alice/dir")     // creates dir with entries "." and ".."
// ln -d /home/alice/dir /home/bob/also_dir   // hard link to directory

// Problem 1: Cycles in the directory tree
// /home/alice/dir/parent → (via "..") → /home/alice
// /home/bob/also_dir is the same inode as /home/alice/dir
// Filesystem now has a CYCLE — a directory can be its own ancestor!
// tools like find, du, ls -R loop forever or require cycle detection

// Problem 2: Ambiguous parent (..)
// /home/alice/dir/.. should be /home/alice
// /home/bob/also_dir/.. should be /home/bob
// But they are the SAME INODE — ".." can only point to ONE parent
// Filesystem becomes inconsistent — two parents for one directory</code></pre>

<p><strong>Why symlinks to directories are safe:</strong></p>
<p>A symlink to a directory is just another T_SYMLINK file containing a path string. When accessed, <code>sys_open()</code> follows the path to the real directory inode. The real directory's <code>..</code> entry correctly points to its real parent, not the symlink's location. No cycles are created in the inode graph itself — the symlink is a separate T_SYMLINK inode that merely references the directory by path.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

The lab PDF describes the Basic Link test: "Creates a normal file 'a', then a link file 'b' pointing to 'a'. To see if opening 'b' gives the data inside 'a'." Write the exact sequence of system calls this test makes, and trace the kernel state after each call.

<div class="answer-content">

<p><strong>The complete Basic Link test sequence:</strong></p>
<pre><code class="language-c">// ── Phase 1: Create the target file ──

fd = open("a", O_CREATE|O_RDWR);
// sys_open() → create("a", T_FILE, 0, 0) → new inode #5 (T_FILE)
// fd = 3 pointing to inode #5, off=0, readable=1, writable=1

write(fd, "test data", 9);
// sys_write() → writei(ip#5, 1, buf, 0, 9)
// bmap(ip#5, 0) → allocates data block e.g. #50, writes "test data" there
// ip#5->size = 9

close(fd);
// fileclose() → ip#5 ref count decremented, inode not freed (nlink=1)

// ── Phase 2: Create the symlink ──

symlink("a", "b");
// sys_symlink() → begin_op()
//   create("b", T_SYMLINK, 0, 0) → new inode #6 (T_SYMLINK)
//   writei(ip#6, 0, "a", 0, 1) → data block #51 contains "a"
//   ip#6->size = 1
//   iunlockput(ip#6) → end_op()
// Directory now has entry: { inum=6, name="b" }

// ── Phase 3: Open through the symlink ──

fd = open("b", O_RDONLY);
// sys_open() → namei("b") → inode #6 (T_SYMLINK) → ilock(#6)
// depth=0; 0>10? No; depth++ → 1
// readi(#6): reads data block #51 → target="a\0"
// iunlockput(#6) → releases inode #6
// ip = namei("a") → inode #5 (T_FILE) → ilock(#5)
// Loop: ip->type == T_FILE → exit loop
// T_DIR check: T_FILE ≠ T_DIR → pass
// filealloc() → f->ip = #5, f->off = 0, f->readable=1
// fdalloc() → fd = 3
// return 3  (fd points to T_FILE inode #5, NOT to T_SYMLINK #6)

// ── Phase 4: Read through the fd ──

read(fd, rbuf, 9);
// fileread() → readi(ip#5, 1, rbuf, 0, 9)
// reads from data block #50: returns "test data"
// rbuf == "test data" → test passes ✓</code></pre>

<p><strong>Kernel state after each phase:</strong></p>

<table>
  <tr><th>After phase</th><th>Inode #5 (file a)</th><th>Inode #6 (link b)</th></tr>
  <tr><td>Phase 1</td><td>T_FILE, size=9, data="test data"</td><td>Does not exist</td></tr>
  <tr><td>Phase 2</td><td>Unchanged</td><td>T_SYMLINK, size=1, data="a"</td></tr>
  <tr><td>Phase 3</td><td>locked, ref incremented via iget</td><td>Unlocked, ref decremented via iunlockput</td></tr>
  <tr><td>Phase 4</td><td>read position advances to 9</td><td>Unchanged</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

The lab PDF explains that the Circle Test creates "Link A pointing to B, and Link B pointing to A" and verifies the depth counter prevents an infinite loop. If the depth limit is set to 10, what is the maximum legitimate chain length a user can create, and what error does the user see when the limit is exceeded?

<div class="answer-content">

<p><strong>How many hops are allowed with <code>depth > 10</code>:</strong></p>
<pre><code class="language-c">int depth = 0;
while(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)){
    if(depth > 10){      // triggers at depth = 11
        iunlockput(ip);
        end_op();
        return -1;
    }
    depth++;             // depth becomes 1, 2, ..., 10, 11
    // ... follow the link ...
}
// depth=11 on iteration 12 entry → error

// Maximum successful hops: 10 (depth goes 1→2→...→10 during iterations 1-10)
// 11th hop: depth=10, 10>10? No → depth++ → 11 → follow
// 12th entry: depth=11, 11>10? YES → return -1</code></pre>

<p><strong>Maximum legitimate chain length: 10 symlinks.</strong></p>

<p><strong>What error the user sees:</strong></p>
<pre><code class="language-c">// When open() returns -1:
fd = open("link_11_hops", O_RDONLY);
// fd = -1

// User program checking:
if(fd < 0){
    // In a POSIX system: errno = ELOOP (Too many levels of symbolic links)
    // In xv6: no errno — just fd = -1
    printf("open failed\n");
    exit(-1);
}</code></pre>

<p><strong>Practical implication for the Circle Test:</strong></p>
<pre><code class="language-c">// Circle test setup:
symlink("b", "a");   // a → b
symlink("a", "b");   // b → a

// open("a") follows: a→b→a→b→a→b→a→b→a→b→a (10 hops to reach a again)
// At the 12th loop entry: depth=11 → return -1
// symlinktest expects this -1 → Circle Test passes ✓

// If depth limit were 1: any 2+ hop chain would fail → Chain Test would fail
// If depth limit were removed: Circle Test causes infinite loop → kernel hangs
// The value 10 balances: allows useful chains, stops clearly circular ones quickly</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

The lab PDF states the Concurrent Links test "checks if locking logic (<code>ilock</code>/<code>iunlock</code>) is correct and prevents Race Conditions." Identify three specific race conditions that could occur in a buggy <code>sys_symlink()</code> implementation without proper locking, and explain how the existing xv6 locks prevent each one.

<div class="answer-content">

<p><strong>Race condition 1 — Two processes allocate the same inode:</strong></p>
<pre><code class="language-c">// Buggy: ialloc() without buffer lock
// Process A finds inode #7 is free (type=0)
//   [context switch]
// Process B also finds inode #7 is free → both claim it!
// Two symlinks now share inode #7 → writing to one corrupts the other

// Prevention:
// ialloc() reads and writes the inode block via bread()/log_write()
// The buffer cache lock (bcache.lock inside bget()) ensures that
// reading a block and marking it allocated is atomic with respect to other processes.
// Only one process can hold the buffer for the inode block at a time.</code></pre>

<p><strong>Race condition 2 — Directory entry added before inode type is written:</strong></p>
<pre><code class="language-c">// Buggy: without begin_op/end_op wrapping both operations
// Process A: ialloc() creates inode #8 with type=0 (not yet T_SYMLINK)
//   [crash or context switch]
// Process B: namei("mylink") finds the directory entry pointing to inode #8
//   ilock(#8) → ip->type = 0 → panic("ilock: no type")!
// Process A: ip->type = T_SYMLINK → too late, system already panicked

// Prevention:
// begin_op()/end_op() wraps the entire create() + writei() sequence.
// The log ensures all writes (inode type, directory entry, target path) are
// committed atomically — either all appear after end_op() or none do after a crash.</code></pre>

<p><strong>Race condition 3 — Reading target path while it's being written:</strong></p>
<pre><code class="language-c">// Buggy: sys_open() reads target path without inode lock
// Process A: sys_symlink() calls writei() to write "realfile" to symlink's data block
//   [context switch after writing 4 bytes: "real"]
// Process B: sys_open() calls readi() on the same symlink inode
//   Reads 4 bytes: target = "real" (incomplete path)
//   target[4] = 0 → target = "real\0"
//   namei("real") → returns 0 (no file named "real") → open() returns -1
// Process A: writes remaining 4 bytes: "file" → writei() completes

// Prevention:
// sys_open() calls ilock(ip) before readi() — the inode's sleeplock is held
// during the entire read. sys_symlink() calls iunlockput(ip) after writei()
// completes. The reader and writer cannot overlap — the reader always sees
// the complete target path or waits until writing finishes.</code></pre>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

The lab PDF says Task 1 increases the maximum file size and Task 2 adds symlinks. Write a single user-space xv6 program that exercises BOTH tasks simultaneously: it creates a large file (requiring doubly-indirect blocks) and accesses it through a symlink chain, verifying that both features work together correctly.

<div class="answer-content">

<p><strong>The combined test program:</strong></p>
<pre><code class="language-c">// user/combined_test.c — tests Task 1 + Task 2 interaction
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fcntl.h"
#include "kernel/fs.h"

#define TARGET_BLOCK 300   // well into doubly-indirect range (>267)
#define MAGIC 0xDEADBEEF   // unique pattern to verify data integrity

int main(void)
{
    char buf[BSIZE];
    char rbuf[BSIZE];
    int fd, i;

    // ── Phase 1: Create a file that requires doubly-indirect blocks ──
    fd = open("large.file", O_CREATE|O_RDWR);
    if(fd < 0){ printf("cannot create large.file\n"); exit(-1); }

    // Write TARGET_BLOCK blocks (first 267 use direct/singly-indirect, rest doubly-indirect)
    for(i = 0; i < TARGET_BLOCK; i++){
        *(int*)buf = i;                      // store block index
        if(i == TARGET_BLOCK - 1)
            *(int*)(buf+4) = MAGIC;          // store magic in last block's bytes 4-7
        if(write(fd, buf, BSIZE) != BSIZE){
            printf("write failed at block %d\n", i);
            exit(-1);
        }
    }
    close(fd);
    printf("Phase 1: wrote %d blocks (includes doubly-indirect) ok\n", TARGET_BLOCK);

    // ── Phase 2: Create a two-hop symlink chain to the file ──
    symlink("large.file", "link1");   // link1 → large.file
    symlink("link1", "link2");         // link2 → link1 → large.file
    printf("Phase 2: created symlink chain link2 -> link1 -> large.file\n");

    // ── Phase 3: Open through the symlink chain ──
    fd = open("link2", O_RDONLY);
    if(fd < 0){ printf("open through symlink chain failed\n"); exit(-1); }
    printf("Phase 3: opened large.file through two-hop symlink chain ok\n");

    // ── Phase 4: Seek to the last block (TARGET_BLOCK-1) and verify data ──
    // xv6 has no lseek: read and discard TARGET_BLOCK-1 blocks
    for(i = 0; i < TARGET_BLOCK - 1; i++){
        if(read(fd, rbuf, BSIZE) != BSIZE){
            printf("seek-read failed at block %d\n", i);
            exit(-1);
        }
    }

    // Read the last block (doubly-indirect block at logical block TARGET_BLOCK-1)
    if(read(fd, rbuf, BSIZE) != BSIZE){
        printf("read of doubly-indirect block failed\n");
        exit(-1);
    }

    int stored_index = *(int*)rbuf;
    int stored_magic = *(int*)(rbuf+4);

    if(stored_index != TARGET_BLOCK - 1){
        printf("FAIL: wrong block index: expected %d got %d\n",
               TARGET_BLOCK-1, stored_index);
        exit(-1);
    }
    if(stored_magic != MAGIC){
        printf("FAIL: wrong magic: expected 0x%x got 0x%x\n",
               MAGIC, stored_magic);
        exit(-1);
    }

    close(fd);
    printf("Phase 4: doubly-indirect block data verified through symlink chain ok\n");

    // ── Cleanup ──
    unlink("link2");
    unlink("link1");
    unlink("large.file");
    printf("combined_test: all phases passed!\n");
    exit(0);
}</code></pre>

<p><strong>What each phase tests:</strong></p>

<table>
  <tr><th>Phase</th><th>Tests Task 1</th><th>Tests Task 2</th></tr>
  <tr><td>1: Write 300 blocks</td><td>Yes — blocks 267–299 go through doubly-indirect</td><td>No</td></tr>
  <tr><td>2: Create symlink chain</td><td>No</td><td>Yes — sys_symlink() + writei() called twice</td></tr>
  <tr><td>3: Open through chain</td><td>No</td><td>Yes — while loop follows 2 hops</td></tr>
  <tr><td>4: Verify doubly-indirect block data</td><td>Yes — bmap() for block 299 verified via read-back</td><td>Yes — file was accessed through symlink, not directly</td></tr>
</table>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

The lab PDF says a symlink "contains a string (a path) to another file." In the xv6 implementation, this string is stored using <code>writei()</code>. Explain exactly how much disk space a symlink consumes compared to a regular empty file, and trace every disk block allocated when <code>symlink("/usr/bin/sh", "/mylink")</code> is called.

<div class="answer-content">

<p><strong>Disk space consumed by a symlink:</strong></p>
<pre><code class="language-c">// Creating symlink("/usr/bin/sh", "/mylink"):
// target = "/usr/bin/sh"  (11 characters)
// path   = "/mylink"

// Allocations:
// 1. One inode slot (from the inode region — not a separate "block")
//    → ip = create("/mylink", T_SYMLINK, 0, 0)
//    → ialloc() finds a free dinode in the inode blocks (e.g. dinode #24 in inode block 33)
//    → balloc() is NOT called by ialloc() — inodes are preallocated by mkfs

// 2. One directory entry in the parent directory ("/")
//    → create() calls dirlink(dp, "/mylink", inum)
//    → This may allocate a new directory data block if current dir block is full
//      (for a mostly-empty xv6 root directory, it just uses existing space)

// 3. One data block for the symlink's content
//    → writei(ip, 0, "/usr/bin/sh", 0, 11)
//    → bmap(ip, 0): ip->addrs[0] == 0 → balloc() allocates e.g. disk block 48
//    → 11 bytes of "/usr/bin/sh" written to block 48 (1024 bytes allocated, 1013 wasted)

// Total new blocks allocated: 1 data block (block 48 for the path string)
// Inode slot used: 1 (shared with existing inode region — no new inode block)</code></pre>

<p><strong>Comparison with a regular empty file:</strong></p>

<table>
  <tr><th>Property</th><th>Empty regular file</th><th>Symlink to "/usr/bin/sh"</th></tr>
  <tr><td>Inode slots used</td><td>1</td><td>1</td></tr>
  <tr><td>Data blocks allocated</td><td>0</td><td>1 (for the path string)</td></tr>
  <tr><td>Directory entry</td><td>1 dirent record</td><td>1 dirent record</td></tr>
  <tr><td>ip->size</td><td>0</td><td>11 (strlen of "/usr/bin/sh")</td></tr>
  <tr><td>ip->type</td><td>T_FILE (2)</td><td>T_SYMLINK (4)</td></tr>
</table>

<p><strong>Why a symlink uses 1 data block regardless of path length (up to BSIZE):</strong></p>
<p>xv6's block allocation is always in units of BSIZE (1024 bytes). The path string "/usr/bin/sh" is 11 bytes, but <code>bmap(ip, 0)</code> allocates a full 1024-byte block. The remaining 1013 bytes are zeroed (by <code>balloc()</code>'s internal <code>bzero()</code>) but not used. For paths longer than 1024 bytes, a second block would be allocated — but MAXPATH=128 in xv6 ensures all paths fit in one block.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

The lab PDF lists four "Core Uses" of symlinks. For each one, describe a scenario where using a symlink would cause a problem — situations where the symlink's indirection actually creates a worse outcome than using the original file directly.

<div class="answer-content">

<p><strong>Use 1 — Version Control (python3 → python3.11):</strong></p>
<pre><code class="language-c">// Setup: python3 → python3.11

// Problem scenario: Security audit or forensics
// A security scanner checking which binary runs as "python3" must follow the symlink.
// A naive scanner reads /usr/bin/python3 → sees "this is a symlink to python3.11"
//   → must make an additional system call to get the real binary
//   → if the scanner doesn't handle symlinks, it only audits the link, not the binary
// Malicious actor creates: python3 → /tmp/evil_script  (and evil_script → python3.11)
//   A user running "python3" runs /tmp/evil_script instead
//   If the scanner only checks the symlink target one level deep, it sees python3.11 and misses the attack

// Scenario where direct file is better:
// Scripts that require a STABLE binary at a known inode (e.g. integrity checks using inode numbers)
// would fail with symlinks since the symlink has a different inode than the real binary.</code></pre>

<p><strong>Use 2 — Space Efficiency (reference a large file from multiple locations):</strong></p>
<pre><code class="language-c">// Setup: /home/alice/project/data → /archive/10gb_file.dat

// Problem scenario: File deletion order matters
unlink("/archive/10gb_file.dat");  // delete the real file
// All symlinks pointing to it are now DANGLING
// open("/home/alice/project/data") → namei() returns 0 → -1
// The symlinks still exist — they appear in ls but cannot be opened
// Confusion: disk space is freed (good) but the apparent references still exist

// With a hard link instead:
// nlink would be incremented for each link
// Deleting the original file would just decrement nlink
// The last remaining hard link would keep the file alive
// Symlinks provide NO reference counting — deletion order matters critically</code></pre>

<p><strong>Use 3 — Stability (map /mnt/disk_01/data to /data):</strong></p>
<pre><code class="language-c">// Setup: /data → /mnt/disk_01/data

// Problem scenario: The symlink itself becomes a point of failure
// If the symlink is accidentally deleted:
unlink("/data");
// Every program using /data now gets -1 from open()
// The actual data is still at /mnt/disk_01/data — untouched
// But nothing can access it via /data until the symlink is recreated

// Worse: if /data is a mount point in the original design (before symlinks),
// using a symlink instead means the shortcut can disappear independently of the data.
// A mount point is protected by the OS mount table — a symlink is just a file that can be deleted.</code></pre>

<p><strong>Use 4 — Flexibility (point to directories and across different disks):</strong></p>
<pre><code class="language-c">// Setup: /home/alice/photos → /mnt/external/photos

// Problem scenario: the external disk is not mounted
// symlink("/mnt/external/photos", "/home/alice/photos")
// External disk unmounted (or not present at boot):
// open("/home/alice/photos") → namei("/mnt/external/photos") → returns 0 → -1
// The symlink silently fails as if the directory does not exist
// No error message explains that the problem is a missing disk/mount point

// For a local directory, open("/home/alice/photos") always works if the filesystem is healthy.
// For a symlink to an external disk, availability depends on whether the disk is mounted —
// a transient failure creates confusing "file not found" errors.</code></pre>

</div>
</div>
