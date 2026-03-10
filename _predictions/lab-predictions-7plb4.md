---
layout: page
title: "lab-predictions-7plb4"
date: 2026-03-11
permalink: /predictions/lab-predictions-7plb4/
---

{% raw %}
**user/handshake.c**
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int pipe_to_child[2];
  int pipe_to_parent[2];
  int child_pid;
  char byte_val;

  if(argc != 2){
    fprintf(2, "Usage: handshake <byte>\n");
    exit(1);
  }

  byte_val = atoi(argv[1]);

  pipe(pipe_to_child);
  pipe(pipe_to_parent);

  child_pid = fork();
  if(child_pid == 0){
    char received_byte;
    close(pipe_to_child[1]);
    close(pipe_to_parent[0]);

    read(pipe_to_child[0], &received_byte, 1);
    printf("%d: received %d from parent\n", getpid(), received_byte);

    write(pipe_to_parent[1], &received_byte, 1);

    close(pipe_to_child[0]);
    close(pipe_to_parent[1]);
    exit(0);
  } else {
    char received_byte;
    close(pipe_to_child[0]);
    close(pipe_to_parent[1]);

    write(pipe_to_child[1], &byte_val, 1);
    read(pipe_to_parent[0], &received_byte, 1);
    printf("%d: received %d from child\n", getpid(), received_byte);

    close(pipe_to_child[1]);
    close(pipe_to_parent[0]);
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
  char *mem_region;
  int page_idx;
  int byte_idx;

  mem_region = sbrk(32 * 4096);

  for(page_idx = 0; page_idx < 32; page_idx++){
    char *cur_page = mem_region + page_idx * 4096;
    for(byte_idx = 0; byte_idx < 4096 - 16; byte_idx++){
      if(cur_page[byte_idx] == 'T' &&
         cur_page[byte_idx+1] == 'h' &&
         cur_page[byte_idx+2] == 'i' &&
         cur_page[byte_idx+3] == 's' &&
         cur_page[byte_idx+4] == ' '){
        char *secret_start = cur_page + byte_idx + 16;
        printf("%s\n", secret_start);
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

  int monitor_mask = atoi(argv[1]);

  if(monitor(monitor_mask) < 0){
    fprintf(2, "monitor: monitor call failed\n");
    exit(1);
  }

  exec(argv[2], argv + 2);
  fprintf(2, "monitor: exec %s failed\n", argv[2]);
  exit(0);
}
```

**user/user.h** (add prototype):
```c
int monitor(int);
```

**user/usys.pl** (add entry):
```perl
entry("monitor");
```

**kernel/syscall.h** (add):
```c
#define SYS_monitor 22
```

**kernel/proc.h** (add to struct proc):
```c
uint32 monitor_mask;
```

**kernel/sysproc.c** (add function):
```c
uint64
sys_monitor(void)
{
  int sys_mask;
  argint(0, &sys_mask);
  struct proc *cur_proc = myproc();
  cur_proc->monitor_mask = (uint32)sys_mask;
  return 0;
}
```

**kernel/syscall.c** (modifications):
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
  int call_num;
  struct proc *cur_proc = myproc();

  call_num = cur_proc->trapframe->a7;
  if(call_num > 0 && call_num < NELEM(syscalls) && syscalls[call_num]) {
    cur_proc->trapframe->a0 = syscalls[call_num]();
    if((cur_proc->monitor_mask >> call_num) & 1){
      printf("%d: syscall %s -> %d\n",
             cur_proc->pid,
             syscall_names[call_num],
             cur_proc->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            cur_proc->pid, cur_proc->name, call_num);
    cur_proc->trapframe->a0 = -1;
  }
}
```

**kernel/proc.c** (in kfork, add after copying trapframe):
```c
np->monitor_mask = p->monitor_mask;
```
{% endraw %}
