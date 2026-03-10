---
layout: page
title: "lab-predictions-3r88p"
date: 2026-03-11
---

**user/handshake.c**
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int p2c[2], c2p[2];
  pipe(p2c);
  pipe(c2p);

  int child_pid = fork();
  if(child_pid == 0){
    close(p2c[1]);
    close(c2p[0]);

    char received_byte;
    read(p2c[0], &received_byte, 1);
    printf("%d: received %d from parent\n", getpid(), received_byte);
    write(c2p[1], &received_byte, 1);

    close(p2c[0]);
    close(c2p[1]);
    exit(0);
  } else {
    close(p2c[0]);
    close(c2p[1]);

    char send_byte = atoi(argv[1]);
    write(p2c[1], &send_byte, 1);

    char echo_byte;
    read(c2p[0], &echo_byte, 1);
    printf("%d: received %d from child\n", getpid(), echo_byte);

    close(p2c[1]);
    close(c2p[0]);
    wait(0);
    exit(0);
  }
}
```

**user/sniffer.c**
```c
#include "kernel/types.h"
#include "kernel/fcntl.h"
#include "user/user.h"
#include "kernel/riscv.h"

int
main(int argc, char *argv[])
{
  char *heap_start = sbrk(0);
  sbrk(4096 * 100);
  char *heap_end = sbrk(0);

  char *scan_ptr;
  for(scan_ptr = heap_start; scan_ptr < heap_end - 16; scan_ptr++){
    if(scan_ptr[0] == 'T' && scan_ptr[1] == 'h' &&
       scan_ptr[2] == 'i' && scan_ptr[3] == 's'){
      char *secret_loc = scan_ptr + 16;
      if(secret_loc[0] != '\0'){
        printf("%s\n", secret_loc);
        exit(0);
      }
    }
  }

  exit(1);
}
```

**user/monitor.c**
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  if(argc < 3){
    fprintf(2, "Usage: monitor <mask> <command> [args...]\n");
    exit(1);
  }

  int trace_mask = atoi(argv[1]);
  monitor(trace_mask);

  if(exec(argv[2], argv+2) < 0){
    fprintf(2, "monitor: exec %s failed\n", argv[2]);
    exit(1);
  }

  exit(0);
}
```

**kernel/syscall.h** (add to existing)
```c
#define SYS_monitor 22
```

**user/user.h** (add prototype)
```c
int monitor(int);
```

**user/usys.pl** (add entry)
```perl
entry("monitor");
```

**kernel/proc.h** (add to struct proc)
```c
uint32 monitor_mask;
```

**kernel/sysproc.c** (add function)
```c
uint64
sys_monitor(void)
{
  int tracking_mask;
  argint(0, &tracking_mask);
  struct proc *cur = myproc();
  cur->monitor_mask = (uint32)tracking_mask;
  return 0;
}
```

**kernel/proc.c** (in kfork, add after np->sz = p->sz)
```c
np->monitor_mask = p->monitor_mask;
```

**kernel/syscall.c** (modified)
```c
extern uint64 sys_monitor(void);

static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_pause]   sys_pause,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_monitor] sys_monitor,
};

static const char *syscall_names[] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_pause]   "pause",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_monitor] "monitor",
};

void
syscall(void)
{
  int num;
  struct proc *cur_proc = myproc();

  num = cur_proc->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    cur_proc->trapframe->a0 = syscalls[num]();
    if((cur_proc->monitor_mask >> num) & 1){
      printf("%d: syscall %s -> %d\n", cur_proc->pid,
             syscall_names[num], cur_proc->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            cur_proc->pid, cur_proc->name, num);
    cur_proc->trapframe->a0 = -1;
  }
}
```
