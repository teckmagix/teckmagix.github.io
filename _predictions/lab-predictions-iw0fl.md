---
layout: page
title: "lab-predictions-iw0fl"
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
  int pipe_to_child[2];
  int pipe_to_parent[2];
  int child_pid;
  char byte_val;

  if(argc < 2){
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
  char *region;
  char *scan_ptr;
  int region_size = 32 * 4096;

  region = sbrk(region_size);

  for(scan_ptr = region; scan_ptr < region + region_size; scan_ptr += 16){
    if(scan_ptr[0] != 0){
      printf("%s\n", scan_ptr);
      exit(0);
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
  int monitor_mask;
  char *exec_args[32];
  int arg_idx;

  if(argc < 3){
    fprintf(2, "Usage: monitor mask command\n");
    exit(1);
  }

  monitor_mask = atoi(argv[1]);
  monitor(monitor_mask);

  for(arg_idx = 2; arg_idx < argc; arg_idx++){
    exec_args[arg_idx - 2] = argv[arg_idx];
  }
  exec_args[argc - 2] = 0;

  exec(exec_args[0], exec_args);
  fprintf(2, "monitor: exec failed\n");
  exit(1);
}
```

**kernel/syscall.h** (add at end)
```c
#define SYS_monitor 22
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
  int input_mask;
  argint(0, &input_mask);
  struct proc *cur_proc = myproc();
  cur_proc->monitor_mask = (uint32)input_mask;
  return 0;
}
```

**kernel/syscall.c** (modified)
```c
#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "riscv.h"
#include "spinlock.h"
#include "proc.h"
#include "syscall.h"
#include "defs.h"

int
fetchaddr(uint64 addr, uint64 *ip)
{
  struct proc *p = myproc();
  if(addr >= p->sz || addr+sizeof(uint64) > p->sz)
    return -1;
  if(copyin(p->pagetable, (char *)ip, addr, sizeof(*ip)) != 0)
    return -1;
  return 0;
}

int
fetchstr(uint64 addr, char *buf, int max)
{
  struct proc *p = myproc();
  if(copyinstr(p->pagetable, buf, addr, max) < 0)
    return -1;
  return strlen(buf);
}

static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
  panic("argraw");
  return -1;
}

void
argint(int n, int *ip)
{
  *ip = argraw(n);
}

void
argaddr(int n, uint64 *ip)
{
  *ip = argraw(n);
}

int
argstr(int n, char *buf, int max)
{
  uint64 addr;
  argaddr(n, &addr);
  return fetchstr(addr, buf, max);
}

extern uint64 sys_fork(void);
extern uint64 sys_exit(void);
extern uint64 sys_wait(void);
extern uint64 sys_pipe(void);
extern uint64 sys_read(void);
extern uint64 sys_kill(void);
extern uint64 sys_exec(void);
extern uint64 sys_fstat(void);
extern uint64 sys_chdir(void);
extern uint64 sys_dup(void);
extern uint64 sys_getpid(void);
extern uint64 sys_sbrk(void);
extern uint64 sys_pause(void);
extern uint64 sys_uptime(void);
extern uint64 sys_open(void);
extern uint64 sys_write(void);
extern uint64 sys_mknod(void);
extern uint64 sys_unlink(void);
extern uint64 sys_link(void);
extern uint64 sys_mkdir(void);
extern uint64 sys_close(void);
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
  int syscall_num;
  struct proc *cur_proc = myproc();

  syscall_num = cur_proc->trapframe->a7;
  if(syscall_num > 0 && syscall_num < NELEM(syscalls) && syscalls[syscall_num]) {
    cur_proc->trapframe->a0 = syscalls[syscall_num]();
    if((cur_proc->monitor_mask >> syscall_num) & 1){
      printf("%d: syscall %s -> %ld\n", cur_proc->pid,
             syscall_names[syscall_num], cur_proc->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            cur_proc->pid, cur_proc->name, syscall_num);
    cur_proc->trapframe->a0 = -1;
  }
}
```

**user/user.h** (add prototype)
```c
int monitor(int);
```

**user/usys.pl** (add entry)
```
entry("monitor");
```

**kernel/proc.c** (in kfork, add after np->sz = p->sz)
```c
np->monitor_mask = p->monitor_mask;
```
