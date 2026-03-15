---
layout: page
title: "Lab Predictions - lab3-v2k4"
lab: lab3
description: "Exam predictions for Lab3-w8: O_NOFOLLOW, open() flag combinations, sys_open return values, and T_DIR check positioning."
---

## Easy

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

What is the <code>O_NOFOLLOW</code> flag and what exactly does <code>open("mylink", O_NOFOLLOW)</code> return compared to <code>open("mylink", O_RDONLY)</code>?

<div class="answer-content">

<p><strong>The flag definition:</strong></p>
<pre><code class="language-c">// kernel/fcntl.h
#define O_NOFOLLOW 0x800</code></pre>

<p><strong>What open("mylink", O_RDONLY) returns — follows the symlink:</strong></p>
<pre><code class="language-c">open("mylink", O_RDONLY)
// sys_open() → namei("mylink") → T_SYMLINK inode #12
// ilock(ip): type == T_SYMLINK and !(O_RDONLY & O_NOFOLLOW) → follow
// readi: reads "realfile" from inode #12's data block
// iunlockput(#12)
// namei("realfile") → T_FILE inode #35
// ilock(#35)
// Returns fd → pointing to inode #35 (the real file)
// Reading fd returns the content of "realfile"</code></pre>

<p><strong>What open("mylink", O_NOFOLLOW) returns — opens the symlink itself:</strong></p>
<pre><code class="language-c">open("mylink", O_NOFOLLOW)
// sys_open() → namei("mylink") → T_SYMLINK inode #12
// ilock(ip): type == T_SYMLINK BUT (O_NOFOLLOW & O_NOFOLLOW) != 0 → DO NOT follow
// while loop condition: !(omode & O_NOFOLLOW) = false → loop never executes
// Returns fd → pointing to inode #12 (the symlink itself)
// Reading fd returns "realfile" — the path string stored in the symlink's data block</code></pre>

<p><strong>Comparison:</strong></p>

<table>
  <tr><th>Call</th><th>fd points to</th><th>read() returns</th></tr>
  <tr><td><code>open("mylink", O_RDONLY)</code></td><td>inode #35 (T_FILE realfile)</td><td>Content of realfile</td></tr>
  <tr><td><code>open("mylink", O_NOFOLLOW)</code></td><td>inode #12 (T_SYMLINK mylink)</td><td><code>"realfile"</code> — the path string</td></tr>
</table>

<p><code>O_NOFOLLOW</code> is used by programs that need to inspect or delete the symlink itself — for example, <code>unlink("mylink")</code> internally uses the non-following path to find the symlink's own inode to remove the directory entry, not the target.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-easy">Easy</span></div>

What must be added to <code>kernel/syscall.c</code> for the new <code>sys_symlink</code> handler, and why are both changes necessary?

<div class="answer-content">

<p><strong>Both changes required in syscall.c:</strong></p>
<pre><code class="language-c">// 1. extern declaration — near the top of syscall.c with the other externs:
extern uint64 sys_symlink(void);

// 2. Dispatch table entry — in the syscalls[] array:
[SYS_symlink] sys_symlink,</code></pre>

<p><strong>Why both are necessary:</strong></p>

<p><strong>The extern declaration:</strong></p>
<p><code>sys_symlink()</code> is defined in <code>kernel/sysfile.c</code>, not in <code>kernel/syscall.c</code>. In C, a function defined in another translation unit is invisible by default. The <code>extern</code> declaration tells the compiler: "this function exists somewhere in the project; do not generate a 'undefined reference' error." Without it, the compiler does not know the type of <code>sys_symlink</code> and either rejects the dispatch table entry or silently treats it as returning <code>int</code> (wrong type).</p>

<p><strong>The dispatch table entry:</strong></p>
<pre><code class="language-c">// syscall.c
static uint64 (*syscalls[])(void) = {
    // ...
    [SYS_symlink] sys_symlink,   // ← links trap number 22 to the handler function
};

void syscall(void){
    int num = p->trapframe->a7;  // system call number from register
    if(num > 0 &amp;&amp; num &lt; NELEM(syscalls) &amp;&amp; syscalls[num]){
        p->trapframe->a0 = syscalls[num]();  // call the handler
    } else {
        printf("unknown sys call %d\n", num);
        p->trapframe->a0 = -1;
    }
}</code></pre>

<p>Without the table entry, <code>syscalls[22]</code> is 0 (null pointer). When a process calls <code>symlink()</code>, the kernel reaches <code>syscalls[22] == 0</code>, takes the else branch, prints "unknown sys call 22", and returns -1 to the user — as if the system call does not exist.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-easy">Easy</span></div>

After adding the symlink loop to <code>sys_open()</code>, why is the T_DIR check still necessary, and what does it prevent?

<div class="answer-content">

<p><strong>The T_DIR check:</strong></p>
<pre><code class="language-c">// In sys_open(), after the symlink loop:
if(ip->type == T_DIR &amp;&amp; omode != O_RDONLY){
    iunlockput(ip);
    end_op();
    return -1;
}</code></pre>

<p><strong>What it prevents:</strong></p>
<p>Opening a directory for writing. Directories are regular files in xv6 — their data blocks contain <code>struct dirent</code> records. If a process could open a directory with <code>O_WRONLY</code> and call <code>write(fd, arbitrary_bytes, n)</code>, it could overwrite the dirent records with garbage, corrupting the directory's name-to-inode mapping. Every file in that directory would become unreachable — effectively deleting the entire directory tree without going through the proper <code>unlink()</code>/<code>rmdir()</code> paths.</p>

<p><strong>Why this check applies to the final resolved target, not the symlink:</strong></p>
<p>A symlink can legitimately point to a directory:</p>
<pre><code class="language-c">symlink("/home/alice", "/shortcut");
// /shortcut → /home/alice (a directory)

open("/shortcut", O_RDONLY);
// After following: ip points to /home/alice (T_DIR)
// omode = O_RDONLY → check passes → opens the directory read-only (allowed)</code></pre>

<p>If the T_DIR check happened before the symlink loop (on the symlink's own inode, which is T_SYMLINK), the check would trivially pass (<code>T_SYMLINK != T_DIR</code>) and a symlink pointing to a directory could be opened for writing — the write would corrupt the symlink's data block, not the directory, but it would still silently destroy the symlink's target path. By placing the check after symlink resolution, the check always evaluates the actual file being opened.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-easy">Easy</span></div>

What does <code>symlinktest</code> check with the <code>O_NOFOLLOW</code> portion of its test? Write out the expected sequence of calls.

<div class="answer-content">

<p><strong>The O_NOFOLLOW test sequence (from symlinktest.c):</strong></p>
<pre><code class="language-c">// Create a real file
fd = open("target", O_CREATE|O_RDWR);
write(fd, "hello", 5);
close(fd);

// Create a symlink to it
symlink("target", "link1");

// Open the symlink WITH O_NOFOLLOW — should get the symlink itself
fd = open("link1", O_NOFOLLOW|O_RDONLY);
// Expected: fd > 0 (success), fd is opened on the T_SYMLINK inode

// Read the symlink's content — should get the path string, not "hello"
char buf[64];
int n = read(fd, buf, sizeof(buf));
// Expected: n = strlen("target") = 6, buf = "target"

// Compare
if(n != 6 || memcmp(buf, "target", 6) != 0){
    printf("O_NOFOLLOW test failed\n");
    exit(1);
}
close(fd);

// Open WITHOUT O_NOFOLLOW — should follow and get the real file
fd = open("link1", O_RDONLY);
n = read(fd, buf, sizeof(buf));
// Expected: n = 5, buf = "hello"

if(n != 5 || memcmp(buf, "hello", 5) != 0){
    printf("follow test failed\n");
    exit(1);
}</code></pre>

<p><strong>What this verifies:</strong></p>

<table>
  <tr><th>Assertion</th><th>Tests</th></tr>
  <tr><td><code>open("link1", O_NOFOLLOW)</code> succeeds</td><td>T_SYMLINK inode can be opened directly when O_NOFOLLOW is set</td></tr>
  <tr><td>Reading the O_NOFOLLOW fd returns the path string</td><td>The symlink's data block contains the target path, not random data</td></tr>
  <tr><td><code>open("link1", O_RDONLY)</code> returns data from "target"</td><td>The while loop correctly follows the symlink when O_NOFOLLOW is absent</td></tr>
</table>

</div>
</div>

---

## Medium

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-med">Medium</span></div>

What are all the conditions under which the symlink-following loop in <code>sys_open()</code> terminates normally (without returning -1)? Trace a two-hop chain A → B → C where C is a regular file.

<div class="answer-content">

<p><strong>Normal termination conditions:</strong></p>
<pre><code class="language-c">while(ip->type == T_SYMLINK &amp;&amp; !(omode &amp; O_NOFOLLOW)){
    // ...
}</code></pre>

<p>The loop exits normally when either condition in the <code>while</code> becomes false:</p>
<ol>
  <li><code>ip->type != T_SYMLINK</code> — the current inode is NOT a symlink (it's T_FILE, T_DIR, or T_DEVICE)</li>
  <li><code>omode &amp; O_NOFOLLOW != 0</code> — the O_NOFOLLOW flag is set (stop following even if current node is a symlink)</li>
</ol>

<p><strong>Two-hop chain trace: A → B → C (C is T_FILE):</strong></p>
<pre><code class="language-c">// Initial state:
// A is T_SYMLINK, data = "B"
// B is T_SYMLINK, data = "C"
// C is T_FILE

// sys_open("A", O_RDONLY):
ip = namei("A");               // inode #10, T_SYMLINK
ilock(ip);                     // lock #10

// ── Loop iteration 1 ──
// ip->type == T_SYMLINK, !O_NOFOLLOW → enter loop
depth = 0; 0 > 10? No. depth++ → 1
readi(#10, target):  target = "B\0"
iunlockput(#10);               // release A
ip = namei("B");               // inode #20, T_SYMLINK
ilock(#20);                    // lock B

// ── Loop iteration 2 ──
// ip->type == T_SYMLINK, !O_NOFOLLOW → enter loop
depth = 1; 1 > 10? No. depth++ → 2
readi(#20, target): target = "C\0"
iunlockput(#20);               // release B
ip = namei("C");               // inode #35, T_FILE
ilock(#35);                    // lock C

// ── Loop condition check ──
// ip->type == T_FILE ≠ T_SYMLINK → loop condition FALSE → EXIT loop

// After loop: ip = #35 (T_FILE, locked)
// Proceed with directory check, file descriptor allocation, etc.
// Return fd pointing to C</code></pre>

<p><strong>The invariant on loop exit:</strong></p>
<p>When the loop exits normally, <code>ip</code> points to a non-symlink inode with its sleeplock held. All intermediate symlink inodes have been unlocked and released via <code>iunlockput()</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q6 <span class="diff diff-med">Medium</span></div>

What is the behaviour of <code>open("link", O_CREATE|O_NOFOLLOW)</code>? Does the symlink-following loop run? Does O_CREATE conflict with O_NOFOLLOW?

<div class="answer-content">

<p><strong>What sys_open() does with O_CREATE:</strong></p>
<pre><code class="language-c">// In sys_open():
if(omode &amp; O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){ end_op(); return -1; }
} else {
    if((ip = namei(path)) == 0){ end_op(); return -1; }
    ilock(ip);
    // ← symlink loop is HERE (in the else branch only)
    if(ip->type == T_DIR &amp;&amp; omode != O_RDONLY){ ... }
}</code></pre>

<p><strong>The O_CREATE path does NOT have the symlink loop.</strong> The symlink-following code was added only inside the <code>else</code> branch.</p>

<p><strong>Behaviour of open("link", O_CREATE|O_NOFOLLOW):</strong></p>

<table>
  <tr><th>Case</th><th>Behaviour</th></tr>
  <tr><td>"link" does not exist</td><td>create() creates a new T_FILE inode at "link". O_NOFOLLOW is irrelevant — there is nothing to follow. Returns fd to new file.</td></tr>
  <tr><td>"link" exists as T_FILE</td><td>create() finds the existing inode, calls itrunc() to zero it, returns it. O_NOFOLLOW ignored. Returns fd to truncated file.</td></tr>
  <tr><td>"link" exists as T_SYMLINK</td><td>create() finds the existing T_SYMLINK inode. In xv6's create(), if the path exists, it returns the existing inode regardless of type. No symlink-following. Returns fd pointing to the symlink itself — O_NOFOLLOW is effectively honoured by accident.</td></tr>
</table>

<p><strong>Why O_CREATE and O_NOFOLLOW don't conflict:</strong></p>
<p>O_CREATE means "create the file if it does not exist". O_NOFOLLOW means "do not follow symlinks when opening". These operate on different code paths. O_NOFOLLOW only affects the <code>else</code> branch (non-create opens). When O_CREATE is set, the create() path always refers to the named path directly — there is no symlink traversal to suppress.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q7 <span class="diff diff-med">Medium</span></div>

The symlink loop calls <code>readi(ip, 0, (uint64)target, 0, MAXPATH)</code>. What do each of the five arguments mean, and what happens if the target path stored in the symlink is longer than MAXPATH?

<div class="answer-content">

<p><strong>The five arguments to readi():</strong></p>
<pre><code class="language-c">int readi(struct inode *ip, int user_dst, uint64 dst, uint off, uint n)
//         ↑ inode to read from
//                   ↑ 0 = dst is kernel address; 1 = dst is user-space address
//                          ↑ destination buffer address
//                                   ↑ byte offset within the file to start reading
//                                       ↑ number of bytes to read

readi(ip, 0, (uint64)target, 0, MAXPATH)
// ip     = the symlink inode
// 0      = target[] is in kernel memory (on the kernel stack in sys_open)
// target = cast to uint64 — kernel address of the destination buffer
// 0      = start reading from byte 0 (the beginning of the symlink's data)
// MAXPATH = read at most MAXPATH bytes (typically 128 in xv6)</code></pre>

<p><strong>What happens if the target path is longer than MAXPATH:</strong></p>
<pre><code class="language-c">// sys_symlink() writes: writei(ip, 0, (uint64)target, 0, strlen(target))
// strlen(target) can be at most MAXPATH-1 because:
//   argstr(0, target, MAXPATH)   ← reads at most MAXPATH bytes including null terminator
// So the stored length is at most MAXPATH-1 bytes (MAXPATH includes space for '\0')

// However: the stored data has NO null terminator (writei writes strlen bytes, not strlen+1)
// readi reads min(MAXPATH, stored_size) bytes.
// If stored_size == MAXPATH-1 (the maximum possible), readi reads MAXPATH-1 bytes.
// target[MAXPATH-1] = 0  ← correct null termination

// If somehow (via direct disk manipulation) more than MAXPATH-1 bytes were stored:
// readi reads exactly MAXPATH bytes.
// target[MAXPATH] = 0  ← BUFFER OVERFLOW! target is declared char target[MAXPATH]
// Writing target[MAXPATH] is one past the end of the array.
// On xv6's kernel stack, this overwrites adjacent local variables — undefined behaviour.</code></pre>

<p><strong>Why this is not a practical concern in the lab:</strong></p>
<p><code>argstr()</code> limits input to MAXPATH bytes, and <code>writei()</code> receives exactly <code>strlen(target)</code> bytes. The stored symlink path can never exceed MAXPATH-1 bytes through normal <code>sys_symlink()</code> calls. The bounds check exists implicitly through the argument fetching layer.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q8 <span class="diff diff-med">Medium</span></div>

Explain the difference between <code>iunlock(ip)</code> and <code>iunlockput(ip)</code>. Write a scenario where using <code>iunlock(ip)</code> in <code>sys_open()</code>'s symlink loop instead of <code>iunlockput(ip)</code> causes a resource leak.

<div class="answer-content">

<p><strong>What each does:</strong></p>
<pre><code class="language-c">// iunlock(ip):
void iunlock(struct inode *ip){
    releasesleep(&ip->lock);   // releases the sleeplock ONLY
    // ip->ref is unchanged — the in-memory inode slot stays "in use"
}

// iunlockput(ip):
void iunlockput(struct inode *ip){
    iunlock(ip);    // release the sleeplock
    iput(ip);       // decrement ip->ref; if ref==0 AND nlink==0: free the inode
}</code></pre>

<p><strong>Where ip->ref comes from in the symlink loop:</strong></p>
<pre><code class="language-c">// In the symlink loop:
iunlockput(ip);       // ← decrement ref for the symlink inode we're done with

ip = namei(target);   // namei() calls iget() internally
                       // iget() increments ip->ref for the new inode
ilock(ip);            // lock the new inode</code></pre>

<p><code>namei()</code> internally calls <code>iget()</code> which increments <code>ip->ref</code>. Every <code>iget()</code> call must be matched by an <code>iput()</code> call. <code>iunlockput()</code> provides that <code>iput()</code>. <code>iunlock()</code> alone does not.</p>

<p><strong>The resource leak scenario:</strong></p>
<pre><code class="language-c">// Symlink chain: A → B → C (C is T_FILE)
// Using iunlock() instead of iunlockput():

// Loop 1: following A
ip_A = namei("A");     // iget(A): ip_A->ref = 1
ilock(ip_A);
iunlock(ip_A);          // BUG: releases lock but NOT reference
                         // ip_A->ref is still 1 — inode A is still "in use"

ip_B = namei("B");     // iget(B): ip_B->ref = 1
ilock(ip_B);
iunlock(ip_B);          // BUG: ip_B->ref still 1

ip_C = namei("C");     // iget(C): ip_C->ref = 1
ilock(ip_C);
// Loop exits. Return fd pointing to C.

// After open() returns:
// ip_A->ref = 1 → inode A's in-memory slot is permanently marked "in use"
// ip_B->ref = 1 → inode B's in-memory slot is permanently marked "in use"
// The itable has NINODE=50 slots. After 25 two-hop open() calls:
//   50 slots consumed (2 leaked per call)
//   iget() scans all 50 slots, finds none with ref==0, returns 0
//   namei() returns 0 → open() returns -1 → all file opens fail</code></pre>

<p>The symptom: after enough symlink traversals, all file operations that require an inode — including opening non-symlink files — start returning -1, with "iget: no inodes" printed to the kernel console.</p>

</div>
</div>

---

## Hard

<div class="var-question">
<div class="var-qnum">Q9 <span class="diff diff-hard">Hard</span></div>

Write the complete corrected <code>sys_open()</code> else branch (the non-O_CREATE path) as it should look after the lab, with the symlink loop included. Annotate every line explaining why it must appear in exactly that position.

<div class="answer-content">

<pre><code class="language-c">} else {
    // ── Step 1: Look up the path ──
    if((ip = namei(path)) == 0){
        end_op();
        return -1;
    }
    // namei() calls iget() which increments ip->ref.
    // If namei returns 0, the path doesn't exist — clean up and return.

    // ── Step 2: Lock the inode ──
    ilock(ip);
    // MUST lock before reading ip->type or calling readi().
    // Without the lock: another process could free the inode between namei() and here.
    // The lock also ensures ip->valid=1 (ilock reads from disk if not yet loaded).

    // ── Step 3: Symlink-following loop ──
    int depth = 0;
    while(ip->type == T_SYMLINK &amp;&amp; !(omode &amp; O_NOFOLLOW)){
        // MUST be after ilock() — reading ip->type requires the lock.
        // MUST be before the T_DIR check — the T_DIR check should apply
        // to the final resolved target, not the symlink itself.

        if(depth > 10){
            // Depth guard: prevents infinite loops on circular symlinks (A→B→A).
            // Check BEFORE doing any work on this iteration.
            iunlockput(ip);   // release and decrement ref — we're aborting
            end_op();
            return -1;        // ELOOP equivalent
        }
        depth++;
        // Increment AFTER the check so depth=0 is the initial state
        // and depth=11 triggers the error on the 12th iteration.

        char target[MAXPATH];
        int n = readi(ip, 0, (uint64)target, 0, MAXPATH);
        // readi() requires the inode lock (held above).
        // Reads the stored target path string from the symlink's data block.
        if(n &lt;= 0){
            iunlockput(ip);   // release on error path
            end_op();
            return -1;        // empty or unreadable symlink
        }
        target[n] = 0;
        // readi() does NOT add a null terminator. MUST add manually.
        // Without this: namei(target) reads garbage bytes as part of the path.

        iunlockput(ip);
        // MUST call iunlockput (not just iunlock) to decrement ip->ref.
        // MUST release BEFORE calling namei(target):
        //   namei() acquires locks on directory inodes it traverses.
        //   If the target path leads back to the symlink's own inode,
        //   trying to ilock() while already holding it → sleeplock deadlock.

        ip = namei(target);
        // Resolve the target path to its inode.
        // namei() increments ref on the returned inode.
        if(ip == 0){
            // Target path doesn't exist → dangling symlink
            end_op();
            return -1;        // no iunlockput needed: ip=0, nothing to release
        }

        ilock(ip);
        // Lock the new inode before the while condition is re-evaluated.
        // The condition reads ip->type — requires the lock.
        // After this ilock(), loop back to check if the new inode is also a symlink.
    }
    // On loop exit: ip holds a non-symlink inode, locked.

    // ── Step 4: Directory check ──
    if(ip->type == T_DIR &amp;&amp; omode != O_RDONLY){
        // MUST be after symlink resolution: a symlink can point to a directory.
        // The check must apply to the final resolved target, not the symlink itself.
        // Prevents write-opening a directory (would corrupt dirent records).
        iunlockput(ip);
        end_op();
        return -1;
    }
}
// Remainder of sys_open() (permission check, filealloc, fdalloc) continues here...</code></pre>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q10 <span class="diff diff-hard">Hard</span></div>

<code>symlinktest</code> includes a <em>concurrent symlinks</em> test that forks multiple child processes each creating and opening symlinks simultaneously. What race conditions could exist in a buggy <code>sys_symlink()</code> implementation and how does wrapping the operation in <code>begin_op()</code>/<code>end_op()</code> and using <code>create()</code> prevent them?

<div class="answer-content">

<p><strong>The concurrent test scenario:</strong></p>
<p>Multiple processes simultaneously call:</p>
<ol>
  <li><code>symlink("target_N", "link_N")</code> — create a symlink for each process</li>
  <li><code>open("link_N", O_RDONLY)</code> — open the symlink through the following path</li>
</ol>

<p><strong>Race conditions a buggy implementation could have:</strong></p>

<p><strong>Race 1 — Double allocation of the same inode slot:</strong></p>
<pre><code class="language-c">// Buggy: manually scanning inode table without locking
for(int inum = 1; inum &lt; NINODES; inum++){
    // read dinode for inum -- NOT atomic
    if(dip->type == 0){       // found free slot
        // context switch here: another process also finds this slot free!
        dip->type = T_SYMLINK; // both processes write type to same inum
        // Two files now share one inode — both see each other's data
    }
}</code></pre>

<p><strong>How create() prevents it:</strong></p>
<p><code>create()</code> calls <code>ialloc()</code>, which reads and writes the inode block under the buffer cache lock (<code>bcache.lock</code> inside <code>bget()</code>). The read-modify-write of the type field is atomic with respect to other <code>ialloc()</code> calls — no two processes can both see the same inode as free and both claim it.</p>

<p><strong>Race 2 — Partial symlink visible to another thread:</strong></p>
<pre><code class="language-c">// Without begin_op/end_op:
// Process A: create() allocates inode — type=T_SYMLINK written to disk
// CRASH / context switch
// Process B: calls namei("link_A") — finds T_SYMLINK inode
//            tries to read target path: data block not yet written (writei not called)
//            readi() reads zeros → target = "" → namei("") returns 0 → open fails
// Process A: writei() writes "target_A" — too late, process B already failed</code></pre>

<p><strong>How begin_op/end_op prevents it:</strong></p>
<pre><code class="language-c">begin_op();
ip = create(path, T_SYMLINK, 0, 0);  // inode allocation + directory entry
writei(ip, ...target...);             // target path written to data block
iunlockput(ip);
end_op();                              // ALL of the above committed atomically
// Either the complete symlink (inode + directory entry + data) appears on disk,
// or nothing does. No process can see a half-created symlink.</code></pre>

<p><strong>Race 3 — Directory entry and inode inconsistency:</strong></p>
<p><code>create()</code> atomically: allocates an inode, writes the inode type to disk, and adds the directory entry (name → inum). Since all three operations are inside the same log transaction, they are committed together. A concurrent <code>namei()</code> call either finds no directory entry (transaction not yet committed) or finds the entry pointing to a fully initialised inode — never a half-state where the directory entry exists but the inode is still zeroed.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q11 <span class="diff diff-hard">Hard</span></div>

A student correctly implements everything but accidentally writes <code>if(depth >= 10)</code> instead of <code>if(depth > 10)</code>. What is the maximum symlink chain length that now works, and construct a test case that passes with the correct implementation but fails with the buggy one.

<div class="answer-content">

<p><strong>Comparing the two conditions:</strong></p>
<pre><code class="language-c">// Correct:   if(depth > 10)   → triggers when depth = 11
// Buggy:     if(depth >= 10)  → triggers when depth = 10

// Correct loop trace:
// Iteration 1: depth=0, 0>10? No, depth++ → 1, follow link 1
// Iteration 2: depth=1, 1>10? No, depth++ → 2, follow link 2
// ...
// Iteration 10: depth=9, 9>10? No, depth++ → 10, follow link 10
// Iteration 11: depth=10, 10>10? No, depth++ → 11, follow link 11
// Iteration 12: depth=11, 11>10? YES → return -1
// Maximum chain: 10 symlinks followed successfully → reaches the 11th target

// Buggy loop trace:
// Iteration 1: depth=0, 0>=10? No, depth++ → 1, follow link 1
// ...
// Iteration 9: depth=8, 8>=10? No, depth++ → 9, follow link 9
// Iteration 10: depth=9, 9>=10? No, depth++ → 10, follow link 10
// Iteration 11: depth=10, 10>=10? YES → return -1
// Maximum chain: 9 symlinks followed successfully → reaches the 10th target</code></pre>

<p><strong>Maximum chain length:</strong></p>

<table>
  <tr><th>Implementation</th><th>Max successful hops</th><th>Rejects at depth</th></tr>
  <tr><td>Correct: <code>depth > 10</code></td><td>10 hops (reaches 11th inode)</td><td>11</td></tr>
  <tr><td>Buggy: <code>depth >= 10</code></td><td>9 hops (reaches 10th inode)</td><td>10</td></tr>
</table>

<p><strong>Test case that distinguishes them — a 10-hop chain:</strong></p>
<pre><code class="language-c">// Create 10 symlinks in a chain: L1→L2→L3→...→L10→realfile

// Setup:
int fd = open("realfile", O_CREATE|O_RDWR);
write(fd, "found", 5);
close(fd);

symlink("realfile", "L10");   // L10 → realfile
symlink("L10",      "L9");
symlink("L9",       "L8");
// ... and so on ...
symlink("L2",       "L1");    // L1 → L2 → ... → L10 → realfile (10 hops total)

// Test: open the start of the chain
fd = open("L1", O_RDONLY);

// ── With correct implementation (depth > 10): ──
// Hops: L1→L2→L3→L4→L5→L6→L7→L8→L9→L10→realfile (10 hops)
// At iteration 11: ip = realfile (T_FILE) → loop exits normally
// fd > 0: read(fd) returns "found" ✓

// ── With buggy implementation (depth >= 10): ──
// Hops: L1→L2→L3→L4→L5→L6→L7→L8→L9→L10 (9 hops to L10, 10th hop starts)
// Iteration 10: depth=9, depth++ → 10, follows L10 to realfile
// Actually: depth++ runs before the NEXT iteration's check.
// Let's retrace:
// Iter 1-9: follow L1..L9 (depth ends at 9 after iter 9)
// Iter 10: depth=9, 9>=10? No. depth++=10. readi(L10)="realfile". follow.
// ip = realfile (T_FILE). Loop condition: T_FILE != T_SYMLINK → EXIT. Success!
// Wait — this still works for a 10-hop chain?

// Retrace more carefully with depth >= 10:
// Iter 10: depth before check = 9. 9 >= 10? No. depth++ = 10. Follow L10 → realfile.
// Iter 11: depth = 10. ip = realfile, T_FILE ≠ T_SYMLINK → exit loop. Success.
// Hmm — still succeeds. Need an 11-hop chain to distinguish.

// CORRECT 10-hop distinction: need 11 symlinks in a chain.
// With depth > 10: follows all 11 hops → success
// With depth >= 10: on hop 11, depth=10, 10>=10 → return -1 → FAIL

// Add one more symlink:
symlink("L1", "L0");   // L0→L1→...→L10→realfile = 11 hops

fd = open("L0", O_RDONLY);
// Correct (>10): follows all 11 hops, returns fd to realfile ✓
// Buggy (>=10):  on 11th iteration depth=10, 10>=10 → return -1 ✗</code></pre>

<p><strong>Summary:</strong> A chain of exactly 11 symlinks distinguishes the two implementations. The correct code opens it successfully; the buggy code returns -1 on the 11th hop. <code>symlinktest</code>'s concurrent test uses shorter chains (typically 1–3 hops) and does not trigger this specific boundary — making it easy for this bug to pass the grader while still being logically incorrect.</p>

</div>
</div>
