---
layout: page
title: "Lab Predictions - lab-predictions-rvlhs"
lab: lab-predictions-rvlhs
description: "Lab predictions - lab-predictions-rvlhs"
---

{% raw %}
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
  int pipe_down[2];
  int pipe_up[2];

  char msg[4] = {0};

  if(argc < 2)
  {
    fprintf(2, "Usage: handshake <4-char-string>\n");
    exit(1);
  }

  for(int i = 0; i < 4; i++){
    if(argv[1][i] != '\0')
      msg[i] = argv[1][i];
    else
      msg[i] = 0;
  }

  if(pipe(pipe_down) < 0 || pipe(pipe_up) < 0){
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
    close(pipe_down[1]);
    close(pipe_up[0]);

    if(read(pipe_down[0], msg, 4) != 4)
    {
      fprintf(2, "handshake: child read failed\n");
      exit(1);
    }

    printf("%d: child received %s from parent\n", getpid(), msg);

    if(write(pipe_up[1], msg, 4) != 4){
      fprintf(2, "handshake: child write failed\n");
      exit(1);
    }

    close(pipe_down[0]);
    close(pipe_up[1]);
    exit(0);
  }

  // PARENT PROCESS
  close(pipe_down[0]);
  close(pipe_up[1]);

  if(write(pipe_down[1], msg, 4) != 4)
  {
    fprintf(2, "handshake: parent write failed\n");
    exit(1);
  }

  if(read(pipe_up[0], msg, 4) != 4)
  {
    fprintf(2, "handshake: parent read failed\n");
    exit(1);
  }

  printf("%d: parent received %s from child\n", getpid(), msg);
  close(pipe_down[1]);
  close(pipe_up[0]);
  wait(0);
  exit(0);
}
```
{% endraw %}
