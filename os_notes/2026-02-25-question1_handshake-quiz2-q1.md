---
layout: page
title: "Batch 2026-02-25 — Question1 Handshake, Quiz2 Q1"
date: 2026-02-25
---

# Batch 2026-02-25 — Question1 Handshake, Quiz2 Q1

## File 1: Question1_handshake.pdf

> **💡 Hint:** This is a fill-in-the-blanks task. Only `user/handshake.c` and `Makefile` need to be modified.

### Complete Solution: `user/handshake.c`

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
  int p2c[2];
  int c2p[2];

  char buf[4] = {0};  // 4-byte word, zero-initialized

  if(argc < 2)
  {
    fprintf(2, "Usage: handshake <4-char-string>\n");
    exit(1);
  }

  for(int i = 0; i < 4; i++){
    if(argv[1][i] == '\0')
      break;
    buf[i] = argv[1][i];
  }

  if(pipe(p2c) < 0 || pipe(c2p) < 0){
    fprintf(2, "handshake: pipe failed\n");
    exit(1);
  }

  int pid = fork();
  if(pid < 0){
    fprintf(2, "handshake: fork failed\n");
    exit(1);
  }

  if(pid == 0){
    // CHILD PROCESS
    close(p2c[1]);   // child reads from p2c[0]
    close(c2p[0]);   // child writes to c2p[1]

    if(read(p2c[0], buf, 4) != 4)
    {
      fprintf(2, "handshake: child read failed\n");
      exit(1);
    }

    printf("%d: child received %.4s from parent\n", getpid(), buf);

    if(write(c2p[1], buf, 4) != 4) {
      fprintf(2, "handshake: child write failed\n");
      exit(1);
    }

    close(p2c[0]);
    close(c2p[1]);
    exit(0);
  }

  // PARENT PROCESS
  close(p2c[0]);   // parent writes to p2c[1]
  close(c2p[1]);   // parent reads from c2p[0]

  if(write(p2c[1], buf, 4) != 4)
  {
    fprintf(2, "handshake: parent write failed\n");
    exit(1);
  }

  if(read(c2p[0], buf, 4) != 4)
  {
    fprintf(2, "handshake: parent read failed\n");
    exit(1);
  }

  printf("%d: parent received %.4s from child\n", getpid(), buf);

  close(p2c[1]);
  close(c2p[0]);
  wait(0);
  exit(0);
}
```

**Key fill-ins explained:**
- `char buf[4] = {0}` — declares a 4-byte buffer zero-filled (handles padding for short strings and truncation for long strings via the copy loop)
- `argv[1][i]` — reads from the first argument
- `read(p2c[0], buf, 4) != 4` — child reads exactly 4 bytes from the read end of p2c
- `printf("%d: child received %.4s from parent\n", getpid(), buf)` — `%.4s` prints up to 4 chars (handles non-null-terminated buf safely)
- `write(c2p[1], buf, 4) != 4` — child writes 4 bytes back via c2p write end
- `write(p2c[1], buf, 4) != 4` — parent writes 4 bytes via p2c write end
- `read(c2p[0], buf, 4) != 4` — parent reads 4 bytes from c2p read end
- `printf("%d: parent received %.4s from child\n", getpid(), buf)` — same format

---

### Makefile Change

Add `$U/_handshake` to the `UPROGS` list in `Makefile`:

```makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_handshake\
```

> **💡 Hint:** Just find the `UPROGS` block in the Makefile and append `$U/_handshake\` to the list.

---

### How It Works

| Scenario | Behaviour |
|---|---|
| `handshake 1234` | buf = `{'1','2','3','4'}` — full 4 bytes |
| `handshake 12` | buf = `{'1','2',0,0}` — padded with zeros |
| `handshake 12345` | buf = `{'1','2','3','4'}` — loop stops at i=4, '5' ignored |

The two pipes form a **bidirectional channel**:
- `p2c` (parent→child): parent writes on `p2c[1]`, child reads on `p2c[0]`
- `c2p` (child→parent): child writes on `c2p[1]`, parent reads on `c2p[0]`

Both processes close their **unused pipe ends** immediately after `fork()` to prevent blocking on reads when the writer is gone.

> **⚠️ Warning:** If you forget to close unused pipe ends (e.g., parent keeps `p2c[0]` open), the child's `read()` will block forever waiting for EOF because the write end is never fully closed.

---

## File 2: quiz2-q1.zip

> **💡 Hint:** The zip contains the skeleton/infrastructure files. The only file you need to write/modify is `user/handshake.c` (shown above) and the `Makefile`. All kernel files are provided and unchanged.

### Key Files Summary

| File | Purpose |
|---|---|
| `user/handshake.c` | **Your implementation** (solution above) |
| `Makefile` | Add `$U/_handshake` to `UPROGS` |
| `gradelib.py` | Auto-grading framework — do not modify |
| All `kernel/` files | xv6 kernel — do not modify |
| `user/user.h` | Declares system calls available to user programs |

### System Calls Used in This Task

```c
pipe(int fds[2])    // creates a pipe; fds[0]=read end, fds[1]=write end
fork()              // returns 0 in child, child's PID in parent
read(fd, buf, n)    // reads n bytes; returns bytes actually read
write(fd, buf, n)   // writes n bytes; returns bytes actually written  
close(fd)           // closes a file descriptor
wait(0)             // parent waits for any child to exit
getpid()            // returns current process's PID
exit(status)        // terminates process with given status
```

### Testing Commands

```bash
# Clean build and run in QEMU
make clean && make qemu

# In QEMU shell:
$ handshake 1012
4: child received 1012 from parent
3: parent received 1012 from child

# Run grade script
$ ./grade-quiz handshake

# Full grade
$ make clean && make grade
```

> **⚠️ Warning:** Always use `make zipball` to create the submission zip — do NOT use Windows Zip or macOS Files App, as they create nested directory structures that break the autograder.
