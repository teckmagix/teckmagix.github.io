---
layout: page
title: "Lab Predictions - lab3-p4k1"
lab: lab3
description: "Exam predictions for Lab3-w8: sys_symlink() plumbing, begin_op/end_op, create(), writei(), and argstr()."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

List every file that must be modified to add the <code>symlink</code> system call to xv6. What role does each file play?

<div class="answer-content">

<p><strong>All files that require changes for Task 2:</strong></p>

<table>
  <tr><th>File</th><th>Change</th><th>Role</th></tr>
  <tr><td><code>kernel/stat.h</code></td><td><code>#define T_SYMLINK 4</code></td><td>Defines the new inode type so the kernel can identify symlinks</td></tr>
  <tr><td><code>kernel/fcntl.h</code></td><td><code>#define O_NOFOLLOW 0x800</code></td><td>New open flag — opens the symlink itself instead of following it</td></tr>
  <tr><td><code>kernel/syscall.h</code></td><td><code>#define SYS_symlink 22</code></td><td>Assigns a unique integer ID to the new system call</td></tr>
  <tr><td><code>user/user.h</code></td><td><code>int symlink(const char*, const char*);</code></td><td>Declaration for user-space programs to call <code>symlink()</code></td></tr>
  <tr><td><code>user/usys.pl</code></td><td><code>entry("symlink");</code></td><td>Generates the assembly stub that issues the <code>ecall</code> trap instruction</td></tr>
  <tr><td><code>kernel/syscall.c</code></td><td><code>extern</code> declaration + <code>[SYS_symlink] sys_symlink,</code></td><td>Routes trap number 22 to the correct handler in <code>sysfile.c</code></td></tr>
  <tr><td><code>kernel/sysfile.c</code></td><td>Implement <code>sys_symlink()</code>; add loop in <code>sys_open()</code></td><td>Creates symlinks on disk; follows them transparently on open</td></tr>
  <tr><td><code>Makefile</code></td><td><code>$U/_symlinktest\</code></td><td>Compiles the test binary into the xv6 disk image</td></tr>
</table>

<p><strong>Why so many files for one system call?</strong></p>
<p>A system call crosses the user-kernel boundary. Every layer of that crossing has its own file: the user-space declaration (<code>user.h</code>), the assembly trampoline (<code>usys.pl</code>), the dispatch table (<code>syscall.c</code>), and the handler (<code>sysfile.c</code>). The header files (<code>stat.h</code>, <code>fcntl.h</code>, <code>syscall.h</code>) define constants used across all those layers. Every piece must be in place or the system call cannot be invoked correctly.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What does <code>argstr(n, buf, max)</code> do in <code>sys_symlink()</code>, and why can the function not simply receive its arguments as normal C parameters?

<div class="answer-content">

<p><strong>What argstr does:</strong></p>
<pre><code class="language-c">uint64
sys_symlink(void)
{
    char target[MAXPATH], path[MAXPATH];

    if(argstr(0, target, MAXPATH) &lt; 0 || argstr(1, path, MAXPATH) &lt; 0)
        return -1;
    // target now contains e.g. "/usr/bin/sh"
    // path   now contains e.g. "/link_to_sh"
    ...
}</code></pre>

<p><code>argstr(n, buf, max)</code> fetches the nth string argument from the system call's user-space arguments. It:</p>
<ol>
  <li>Reads the nth word from the user process's trap frame (the register or stack slot that held argument n when <code>ecall</code> was executed)</li>
  <li>Treats that word as a user-space pointer to a string</li>
  <li>Copies the string from user memory into the kernel buffer <code>buf</code>, up to <code>max</code> bytes</li>
  <li>Returns the number of bytes copied, or -1 on error (invalid pointer, string too long, etc.)</li>
</ol>

<p><strong>Why normal C parameters are impossible:</strong></p>
<p>All system call handlers in xv6 have the same signature: <code>uint64 sys_xxx(void)</code>. The kernel's trap dispatcher calls the handler through a function pointer table indexed by system call number — it cannot pass typed arguments through that call. Instead, arguments are left in the registers where the user program placed them before executing <code>ecall</code>, and handlers must explicitly extract them using helper functions like <code>argstr()</code>, <code>argint()</code>, and <code>argaddr()</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

What does <code>create(path, T_SYMLINK, 0, 0)</code> return, and what state is it in when it returns?

<div class="answer-content">

<p><strong>What create() returns:</strong></p>
<pre><code class="language-c">ip = create(path, T_SYMLINK, 0, 0);
if(ip == 0){
    end_op();
    return -1;
}</code></pre>

<p><code>create()</code> allocates a new inode at the given path with the specified type and returns a pointer to the <strong>in-memory inode struct</strong>. The inode is returned in the following state:</p>

<table>
  <tr><th>Property</th><th>State on return</th></tr>
  <tr><td>Type</td><td><code>T_SYMLINK = 4</code></td></tr>
  <tr><td>Lock</td><td><strong>Held (locked)</strong> — caller must call <code>iunlockput()</code> when done</td></tr>
  <tr><td>nlink</td><td>1 — one directory entry points to it</td></tr>
  <tr><td>size</td><td>0 — no data written yet</td></tr>
  <tr><td>addrs[]</td><td>All zero — no data blocks allocated yet</td></tr>
  <tr><td>Return on failure</td><td>0 — path already exists, disk full, or other error</td></tr>
</table>

<p><strong>What happens after create():</strong></p>
<p>Because <code>create()</code> returns with the inode locked, you can immediately call <code>writei()</code> to write data into it — <code>writei()</code> requires the inode lock to be held by the caller. After writing, <code>iunlockput(ip)</code> must be called. Using just <code>iunlock(ip)</code> would release the lock but leave the reference count incremented — the in-memory inode would never be freed, leaking kernel memory.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does <code>O_NOFOLLOW 0x800</code> mean and why must it use a bit value no other open flag uses?

<div class="answer-content">

<p><strong>The open flags:</strong></p>
<pre><code class="language-c">#define O_RDONLY   0x000   // binary: 0000 0000 0000
#define O_WRONLY   0x001   // binary: 0000 0000 0001
#define O_RDWR     0x002   // binary: 0000 0000 0010
#define O_CREATE   0x200   // binary: 0010 0000 0000
#define O_TRUNC    0x400   // binary: 0100 0000 0000
#define O_NOFOLLOW 0x800   // binary: 1000 0000 0000  ← bit 11, unique</code></pre>

<p><strong>Why each flag needs its own bit:</strong></p>
<p>Flags are combined with bitwise OR and tested with bitwise AND:</p>
<pre><code class="language-c">// User program opens a symlink without following it:
open("mylink", O_RDONLY | O_NOFOLLOW);
// omode = 0x000 | 0x800 = 0x800

// In sys_open():
while(ip->type == T_SYMLINK &amp;&amp; !(omode &amp; O_NOFOLLOW)){
// omode &amp; 0x800 = 0x800, which is non-zero → condition is false → loop skipped
// The symlink itself is opened, not the target.</code></pre>

<p>If <code>O_NOFOLLOW</code> shared a bit with any existing flag, it would be impossible to set <code>O_NOFOLLOW</code> without also setting that flag. For example, if <code>O_NOFOLLOW = 0x200</code> (same as <code>O_CREATE</code>), opening a symlink with <code>O_NOFOLLOW</code> would also trigger file creation logic — a serious bug.</p>

<p><strong>Why 0x800 is safe:</strong></p>
<p>0x800 is bit 11. Every existing flag occupies a different bit. No single-bit combination of 0x000, 0x001, 0x002, 0x200, and 0x400 can produce 0x800 — it is impossible to accidentally set <code>O_NOFOLLOW</code> without explicitly including it.</p>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

Explain what <code>begin_op()</code> and <code>end_op()</code> do in <code>sys_symlink()</code>. What guarantee do they provide, and what happens if <code>end_op()</code> is never called?

<div class="answer-content">

<p><strong>How they frame the operation:</strong></p>
<pre><code class="language-c">uint64 sys_symlink(void)
{
    ...
    begin_op();              // ← start transaction

    ip = create(path, T_SYMLINK, 0, 0);
    if(ip == 0){
        end_op();            // ← must call even on the error path
        return -1;
    }

    if(writei(ip, 0, (uint64)target, 0, strlen(target)) != strlen(target)){
        iunlockput(ip);
        end_op();            // ← must call even on the error path
        return -1;
    }

    iunlockput(ip);
    end_op();                // ← commit on success
    return 0;
}</code></pre>

<p><strong>What they guarantee:</strong></p>
<p>Every disk write between <code>begin_op()</code> and <code>end_op()</code> is part of one atomic <strong>log transaction</strong>. Creating a symlink requires at least two disk writes — one to allocate the inode and one to write the target path into the data block. <code>begin_op()</code>/<code>end_op()</code> ensures these either both appear on disk or neither does. Without this, a crash between the two writes would leave an inode of type <code>T_SYMLINK</code> with no data — reading it would return garbage as the target path.</p>

<p><strong>If <code>end_op()</code> is never called:</strong></p>

<table>
  <tr><th>Consequence</th><th>Explanation</th></tr>
  <tr><td>Log transaction never committed</td><td>All buffers dirtied by <code>log_write()</code> inside the transaction are never flushed to their real disk locations</td></tr>
  <tr><td>Log space exhaustion</td><td>The transaction stays "open" indefinitely, consuming log slots. Subsequent <code>begin_op()</code> calls may block forever waiting for log space</td></tr>
  <tr><td>Kernel freeze</td><td>Any other process that calls <code>begin_op()</code> blocks waiting for the never-ending transaction to commit — the system deadlocks</td></tr>
</table>

<p>This is why every error path in <code>sys_symlink()</code> calls <code>end_op()</code> before returning — the transaction must always be closed regardless of whether the operation succeeded.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

Why does <code>sys_symlink()</code> use <code>iunlockput(ip)</code> rather than just <code>iunlock(ip)</code> after <code>writei()</code>?

<div class="answer-content">

<p><strong>What each function does:</strong></p>
<pre><code class="language-c">// iunlock(ip) — only releases the lock:
void iunlock(struct inode *ip){
    // releases sleeplock on ip
    // ip->ref is UNCHANGED — still incremented by create()
}

// iunlockput(ip) — releases lock AND decrements reference count:
void iunlockput(struct inode *ip){
    iunlock(ip);
    iput(ip);   // decrements ip->ref; if ref==0 AND nlink==0, frees the inode
}</code></pre>

<p><strong>Why iunlockput is correct here:</strong></p>
<p><code>create()</code> allocates an in-memory inode slot and increments its reference count (<code>ip->ref</code>) to 1. This reference count tracks how many kernel code paths currently hold a pointer to this inode. After <code>sys_symlink()</code> finishes writing the target path, it is done with the inode — no other code in <code>sys_symlink()</code> will use <code>ip</code> again. Calling <code>iunlockput()</code> decrements the reference count back to 0, allowing the in-memory inode slot to be reused for future operations.</p>

<p><strong>What happens if you use just iunlock(ip):</strong></p>
<pre><code class="language-c">// If iunlock(ip) is used instead:
iunlock(ip);      // lock released — OK
// ip->ref is still 1 — kernel thinks something still holds this inode
// end_op() returns — function exits
// ip is now unreachable (local variable gone from stack)
// But ip->ref == 1 keeps the inode slot permanently "in use"
// After NINODE (50) such leaks: all inode slots are marked in-use
// Next create() or iget() call: panic("iget: no inodes") or silently fails</code></pre>

<p>Each call to <code>sys_symlink()</code> that uses only <code>iunlock()</code> leaks one in-memory inode slot permanently. With NINODE=50, the system runs out of inode cache slots after 50 symlink creations.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

In <code>sys_open()</code>'s symlink loop, why is <code>iunlockput(ip)</code> called <em>before</em> <code>namei(target)</code>, not after?

<div class="answer-content">

<p><strong>The loop structure:</strong></p>
<pre><code class="language-c">ilock(ip);

int depth = 0;
while(ip->type == T_SYMLINK &amp;&amp; !(omode &amp; O_NOFOLLOW)){
    if(depth > 10){ iunlockput(ip); end_op(); return -1; }
    depth++;

    char target[MAXPATH];
    int n = readi(ip, 0, (uint64)target, 0, MAXPATH);
    if(n &lt;= 0){ iunlockput(ip); end_op(); return -1; }
    target[n] = 0;

    iunlockput(ip);              // ← release symlink inode BEFORE namei()

    ip = namei(target);          // ← look up next inode
    if(ip == 0){ end_op(); return -1; }

    ilock(ip);                   // ← lock the new inode before looping
}</code></pre>

<p><strong>Why the release must come before namei():</strong></p>
<p><code>namei()</code> resolves a path by walking directory entries. Internally it calls <code>ilock()</code> on each directory inode it visits. If we hold the lock on the current symlink's inode while <code>namei()</code> tries to lock a directory that is (or shares a lock with) that same inode, we get a deadlock:</p>
<pre><code class="language-c">// Deadlock scenario without early iunlockput:
ilock(symlink_inode);          // we hold lock on inode #12
namei(target);
  → walks /usr → ilock(root dir inode)
  → walks usr → ilock(usr dir inode)
  → finds target → ilock(target inode)
  // If target inode == symlink_inode (#12):
  //   We try to ilock(#12) which WE already hold → sleeplock deadlock
  //   Process sleeps waiting to acquire a lock it already owns — hangs forever</code></pre>

<p>More generally, holding any inode lock while calling code that may acquire other inode locks violates xv6's lock ordering rules. Releasing the current inode before <code>namei()</code> eliminates this class of deadlock entirely.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

Why does the symlink loop write <code>target[n] = 0</code> after <code>readi()</code>? What happens if this line is omitted?

<div class="answer-content">

<p><strong>What readi() does and does not do:</strong></p>
<pre><code class="language-c">int n = readi(ip, 0, (uint64)target, 0, MAXPATH);
// Copies n bytes from the symlink's data block into target[].
// The n bytes are exactly what was written by sys_symlink():
//   writei(ip, 0, (uint64)target_str, 0, strlen(target_str))
// writei wrote strlen(target_str) bytes — NOT including a null terminator.
// readi reads exactly n bytes — also NOT adding a null terminator.
target[n] = 0;   // ← we must add the null terminator ourselves</code></pre>

<p><strong>What target[] looks like without the null terminator:</strong></p>
<pre><code class="language-c">// Suppose the target was "/usr/bin/sh" (11 characters, no null)
// After readi():  target = ['/','u','s','r','/','b','i','n','/','s','h', ?, ?, ...]
//                                                                         ^^^^^^^^^^^
//                                                          undefined bytes from the stack
// namei(target) scans forward from &target[0] looking for a '\0' byte.
// It reads past the 11 valid characters into the uninitialized part of the stack buffer.
// It may find a '\0' quickly (by chance) and work correctly.
// Or it may read hundreds of bytes of garbage before finding a '\0'.
// namei() would try to resolve a nonsense path like "/usr/bin/sh\x8a\x3c..."
// → returns 0 (path not found) → sys_open() returns -1 → open() fails.</code></pre>

<p><strong>Summary:</strong></p>

<table>
  <tr><th>With <code>target[n] = 0</code></th><th>Without <code>target[n] = 0</code></th></tr>
  <tr><td>target is a valid null-terminated C string</td><td>target has no null terminator — garbage bytes follow the path</td></tr>
  <tr><td><code>namei(target)</code> resolves the correct path</td><td><code>namei(target)</code> likely returns 0 (path not found)</td></tr>
  <tr><td>Symlink following works</td><td>Every symlink open fails silently</td></tr>
</table>

<p><code>readi()</code> copies raw bytes — it is not a string function. The null terminator is not part of the stored data (it was not written by <code>writei()</code>) and must be added explicitly.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

Walk through exactly what happens when <code>symlinktest</code> calls <code>open("slowlink", O_RDONLY)</code> and <code>slowlink</code> points to <code>realfile</code>. Trace every kernel function call from <code>sys_open()</code> to the moment the file descriptor is returned.

<div class="answer-content">

<p><strong>Setup:</strong> <code>slowlink</code> is a T_SYMLINK inode whose data block contains the string <code>"realfile"</code>. <code>realfile</code> is a T_FILE inode with actual data.</p>
<pre><code class="language-c">// ── sys_open() enters, O_CREATE not set ──
nameiparent(path, name);           // (1) find parent dir of "slowlink"
ip = namei("slowlink");            // (2) look up "slowlink" → returns T_SYMLINK inode #12
ilock(ip);                         // (3) lock inode #12

// ── Symlink loop begins ──
// ip->type == T_SYMLINK and !(omode & O_NOFOLLOW)  → enter loop
depth = 0; depth++ → depth = 1;

// (4) readi(ip, 0, target, 0, MAXPATH)
//   → reads 8 bytes ("realfile") from inode #12's data block into target[]
//   → n = 8
target[8] = 0;                    // (5) null-terminate: target = "realfile\0"

iunlockput(ip);                   // (6) unlock inode #12, decrement ref count

ip = namei("realfile");           // (7) look up "realfile" in current directory
                                   //     → returns T_FILE inode #35

ilock(ip);                        // (8) lock inode #35

// ── Loop condition re-checked ──
// ip->type == T_FILE != T_SYMLINK → loop exits

// ── Directory check ──
// ip->type == T_FILE != T_DIR → no error

// ── File descriptor allocation ──
f = filealloc();                   // (9) allocate struct file from file table
fd = fdalloc(f);                   // (10) assign next free fd in process's fd table
f->type = FD_INODE;
f->ip   = ip;                      // points to T_FILE inode #35
f->off  = 0;
f->readable = 1; f->writable = 0;

iunlock(ip);                       // (11) unlock inode #35 (ref count stays ≥1)
end_op();                          // (12) commit log transaction

return fd;                         // (13) returned to user as e.g. fd = 3</code></pre>

<p><strong>Key observation:</strong> The returned file descriptor points directly to <code>inode #35</code> (the real file) — not to the symlink inode #12. The caller is completely transparent to the fact that a symlink was traversed. Subsequent <code>read(fd, ...)</code> and <code>write(fd, ...)</code> calls go directly to <code>realfile</code>'s data blocks.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

The depth limit in the symlink loop is <code>depth &gt; 10</code>, not <code>depth &gt;= 10</code>. How many symlinks does this allow in a chain, and what is the exact state of variables when the depth check triggers?

<div class="answer-content">

<p><strong>Counting the iterations:</strong></p>
<pre><code class="language-c">int depth = 0;
while(ip->type == T_SYMLINK &amp;&amp; !(omode &amp; O_NOFOLLOW)){
    if(depth > 10){               // triggers when depth = 11
        iunlockput(ip);
        end_op();
        return -1;
    }
    depth++;                      // depth becomes 1, 2, 3, ..., 10, 11
    // ... follow the symlink ...
}</code></pre>

<p><strong>Iteration trace:</strong></p>

<table>
  <tr><th>Loop iteration</th><th>depth at check</th><th>depth &gt; 10?</th><th>depth after ++</th><th>Links followed so far</th></tr>
  <tr><td>1st</td><td>0</td><td>No</td><td>1</td><td>1</td></tr>
  <tr><td>2nd</td><td>1</td><td>No</td><td>2</td><td>2</td></tr>
  <tr><td>…</td><td>…</td><td>No</td><td>…</td><td>…</td></tr>
  <tr><td>10th</td><td>9</td><td>No</td><td>10</td><td>10</td></tr>
  <tr><td>11th</td><td>10</td><td>No</td><td>11</td><td>11</td></tr>
  <tr><td>12th</td><td>11</td><td><strong>Yes — return -1</strong></td><td>—</td><td>11 (rejected)</td></tr>
</table>

<p><strong>Maximum chain length: 10 symlinks followed successfully.</strong></p>
<p>On the 12th entry to the loop body, <code>depth = 11</code> and <code>depth > 10</code> triggers. At this point: the current <code>ip</code> is the 11th symlink in the chain (inode is locked), <code>target[]</code> has not yet been read, and the function returns -1 without opening any file.</p>

<p><strong>Why 10 and not 1 or 100:</strong></p>
<p>Linux uses a depth limit of 40 for POSIX compliance. xv6 uses 10 as a simpler bound sufficient to detect the A→B→A cycle case (which loops every 2 steps and is caught at step 12). A limit of 1 would prevent useful chained symlinks (e.g. <code>/bin → /usr/bin → /usr/local/bin</code>). A limit of 100 would allow 100 iterations before detecting a cycle, wasting kernel stack and log transaction space.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

What is a <em>dangling symlink</em>? Write a concrete xv6 test sequence that creates one, and trace exactly what happens inside <code>sys_open()</code> when a dangling symlink is opened.

<div class="answer-content">

<p><strong>What a dangling symlink is:</strong></p>
<p>A dangling symlink is a symlink whose target path no longer exists. The symlink file itself is intact — it still has a valid inode and its data block still contains the target path string. But the file (or directory) that path refers to has been deleted. Opening the symlink fails because the target cannot be found.</p>

<p><strong>Creating a dangling symlink in xv6 (user program):</strong></p>
<pre><code class="language-c">// Step 1: Create the target file
int fd = open("realfile", O_CREATE | O_WRONLY);
write(fd, "hello", 5);
close(fd);

// Step 2: Create a symlink pointing to it
symlink("realfile", "mylink");
// mylink's inode (T_SYMLINK) now contains "realfile"

// Step 3: Delete the target
unlink("realfile");
// realfile's inode is freed (nlink→0), data blocks freed, directory entry removed.
// mylink still exists — its inode still contains "realfile" — but "realfile" is gone.

// Step 4: Try to open through the dangling symlink
int fd2 = open("mylink", O_RDONLY);
// Expected result: fd2 = -1 (error)</code></pre>

<p><strong>Trace through sys_open() for the dangling open:</strong></p>
<pre><code class="language-c">// sys_open("mylink", O_RDONLY):
ip = namei("mylink");             // finds T_SYMLINK inode — still valid
ilock(ip);

// Symlink loop:
depth = 0; depth++ → 1
readi(ip, 0, target, 0, MAXPATH); // reads "realfile" from data block, n=8
target[8] = 0;                    // "realfile\0"

iunlockput(ip);                   // release symlink inode — done with it

ip = namei("realfile");           // ← namei scans directory for "realfile"
                                   //   directory entry was deleted by unlink()
                                   //   namei() returns 0

if(ip == 0){
    end_op();
    return -1;                    // ← sys_open returns -1 to the user
}</code></pre>

<p><strong>What is NOT cleaned up:</strong></p>
<p>The symlink inode itself is not deleted. <code>mylink</code> still appears in directory listings. Its inode is still of type <code>T_SYMLINK</code> and its data block still contains <code>"realfile"</code>. xv6 (like Linux) does not automatically remove symlinks when their targets are deleted — the symlink persists until explicitly unlinked with <code>unlink("mylink")</code>.</p>

</div>
</div>
