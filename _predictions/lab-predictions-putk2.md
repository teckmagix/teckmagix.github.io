---
layout: page
title: "Lab Predictions - lab-predictions-putk2"
lab: lab-predictions-putk2
description: "Lab predictions - lab-predictions-putk2"
---

{% raw %}
```c
#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static int gValX = 0;
static int gValY = 0;

static long gCounter = 0;
static int gFinished = 0;

static pthread_mutex_t muxP = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t muxQ = PTHREAD_MUTEX_INITIALIZER;

#define ITER_MAX 20000

typedef struct {
    int tid;
} targs_s;

typedef struct {
    int tid;
    long completed;
    int snapX;
    int snapY;
    long snapCounter;
    int snapFinished;
} tres_s;

static void pause_briefly(void)
{
    usleep(1);
}

static tres_s* build_result(int xid, long xops)
{
    tres_s* rv = malloc(sizeof(tres_s));
    if (!rv) { perror("malloc"); exit(1); }

    rv->tid = xid;
    rv->completed = xops;

    pthread_mutex_lock(&muxP);
    pthread_mutex_lock(&muxQ);
    rv->snapX = gValX;
    rv->snapY = gValY;
    rv->snapCounter = gCounter;
    rv->snapFinished = gFinished;
    pthread_mutex_unlock(&muxQ);
    pthread_mutex_unlock(&muxP);

    return rv;
}

void* funcXY(void* varg)
{
    targs_s* qa = (targs_s*)varg;
    long lcount = 0;

    for (int k = 0; k < ITER_MAX && !gFinished; k++) {

        pthread_mutex_lock(&muxP);
        pthread_mutex_lock(&muxQ);

        gValX++;
        gValY++;
        gCounter += 2;
        lcount += 2;

        pthread_mutex_unlock(&muxQ);
        pthread_mutex_unlock(&muxP);

        pause_briefly();
    }

    return build_result(qa->tid, lcount);
}

void* funcYX(void* varg)
{
    targs_s* qa = (targs_s*)varg;
    long lcount = 0;

    for (int k = 0; k < ITER_MAX && !gFinished; k++) {

        pthread_mutex_lock(&muxP);
        pthread_mutex_lock(&muxQ);

        gValY += 2;
        gValX += 2;
        gCounter += 2;
        lcount += 2;

        pthread_mutex_unlock(&muxQ);
        pthread_mutex_unlock(&muxP);

        pause_briefly();
    }

    return build_result(qa->tid, lcount);
}

void* funcWatch(void* varg)
{
    targs_s* qa = (targs_s*)varg;
    long lcount = 0;

    while (!gFinished) {
        pthread_mutex_lock(&muxP);
        pthread_mutex_lock(&muxQ);

        int cx = gValX;
        int cy = gValY;
        long ct = gCounter;
        int cd = gFinished;

        pthread_mutex_unlock(&muxQ);
        pthread_mutex_unlock(&muxP);

        if ((ct % 20000) == 0) {
            printf("[monitor] A=%d B=%d total_ops=%ld done=%d\n", cx, cy, ct, cd);
        }

        pause_briefly();
    }

    return build_result(qa->tid, lcount);
}

int main(void)
{
    pthread_t hd1, hd2, hd3;

    targs_s arg1 = { 1 };
    targs_s arg2 = { 2 };
    targs_s arg3 = { 3 };

    pthread_create(&hd1, NULL, funcXY, &arg1);
    pthread_create(&hd2, NULL, funcYX, &arg2);
    pthread_create(&hd3, NULL, funcWatch, &arg3);

    tres_s *res1 = NULL, *res2 = NULL, *res3 = NULL;

    pthread_join(hd1, (void**)&res1);
    pthread_join(hd2, (void**)&res2);

    pthread_mutex_lock(&muxP);
    pthread_mutex_lock(&muxQ);
    gFinished = 1;
    pthread_mutex_unlock(&muxQ);
    pthread_mutex_unlock(&muxP);

    pthread_join(hd3, (void**)&res3);

    const long exp_total = 4L * ITER_MAX;

    printf("\n--- Results (returned to main) ---\n");
    printf("Thread %d ops=%ld lastA=%d lastB=%d lastTotalOps=%ld lastDone=%d\n",
           res1->tid, res1->completed, res1->snapX, res1->snapY, res1->snapCounter, res1->snapFinished);
    printf("Thread %d ops=%ld lastA=%d lastB=%d lastTotalOps=%ld lastDone=%d\n",
           res2->tid, res2->completed, res2->snapX, res2->snapY, res2->snapCounter, res2->snapFinished);
    printf("Thread %d ops=%ld lastA=%d lastB=%d lastTotalOps=%ld lastDone=%d\n",
           res3->tid, res3->completed, res3->snapX, res3->snapY, res3->snapCounter, res3->snapFinished);

    printf("\nFinal Shared: A=%d B=%d total_ops=%ld done=%d\n",
           gValX, gValY, gCounter, gFinished);
    printf("Expected: total_ops=%ld done=1\n", exp_total);

    free(res1);
    free(res2);
    free(res3);

    return (gCounter == exp_total && gFinished == 1) ? 0 : 1;
}
```
{% endraw %}
