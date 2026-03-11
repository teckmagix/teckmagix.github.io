---
layout: page
title: "Lab Predictions - lab-predictions-ru0ob"
lab: lab-predictions-ru0ob
description: "Lab predictions - lab-predictions-ru0ob"
---

{% raw %}
```c
#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static int counterX = 0;
static int counterY = 0;

static long total_count = 0;
static int finished = 0;

static pthread_mutex_t mutexX = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t mutexY = PTHREAD_MUTEX_INITIALIZER;

#define ITER_LIMIT 20000

typedef struct {
    int tid;
} worker_params_t;

typedef struct {
    int tid;
    long count_done;
    int snapX;
    int snapY;
    long snapTotal;
    int snapFinished;
} worker_output_t;

static void small_pause(void)
{
    usleep(1);
}

static worker_output_t* build_output(int id, long cnt)
{
    worker_output_t* out = malloc(sizeof(worker_output_t));
    if (!out) { perror("malloc"); exit(1); }

    out->tid = id;
    out->count_done = cnt;

    pthread_mutex_lock(&mutexX);
    out->snapX = counterX;
    pthread_mutex_unlock(&mutexX);

    pthread_mutex_lock(&mutexY);
    out->snapY = counterY;
    pthread_mutex_unlock(&mutexY);

    pthread_mutex_lock(&mutexX);
    out->snapTotal = total_count;
    pthread_mutex_unlock(&mutexX);

    pthread_mutex_lock(&mutexX);
    out->snapFinished = finished;
    pthread_mutex_unlock(&mutexX);

    return out;
}

void* task_XY(void* arg)
{
    worker_params_t* params = (worker_params_t*)arg;
    long cnt = 0;

    for (int i = 0; i < ITER_LIMIT; i++) {
        pthread_mutex_lock(&mutexX);
        int chk;
        chk = finished;
        pthread_mutex_unlock(&mutexX);
        if (chk) break;

        pthread_mutex_lock(&mutexX);
        counterX++;
        total_count++;
        pthread_mutex_unlock(&mutexX);

        pthread_mutex_lock(&mutexY);
        counterY++;
        pthread_mutex_unlock(&mutexY);

        pthread_mutex_lock(&mutexX);
        total_count++;
        pthread_mutex_unlock(&mutexX);

        cnt += 2;

        small_pause();
    }

    return build_output(params->tid, cnt);
}

void* task_YX(void* arg)
{
    worker_params_t* params = (worker_params_t*)arg;
    long cnt = 0;

    for (int i = 0; i < ITER_LIMIT; i++) {
        pthread_mutex_lock(&mutexX);
        int chk;
        chk = finished;
        pthread_mutex_unlock(&mutexX);
        if (chk) break;

        pthread_mutex_lock(&mutexY);
        counterY += 2;
        pthread_mutex_unlock(&mutexY);

        pthread_mutex_lock(&mutexX);
        counterX += 2;
        total_count += 2;
        pthread_mutex_unlock(&mutexX);

        cnt += 2;

        small_pause();
    }

    return build_output(params->tid, cnt);
}

void* task_watcher(void* arg)
{
    worker_params_t* params = (worker_params_t*)arg;
    long cnt = 0;

    while (1) {
        pthread_mutex_lock(&mutexX);
        int chk = finished;
        pthread_mutex_unlock(&mutexX);
        if (chk) break;

        pthread_mutex_lock(&mutexX);
        int ax = counterX;
        long tc = total_count;
        int df = finished;
        pthread_mutex_unlock(&mutexX);

        pthread_mutex_lock(&mutexY);
        int by = counterY;
        pthread_mutex_unlock(&mutexY);

        if ((tc % 20000) == 0) {
            printf("[monitor] A=%d B=%d total_ops=%ld done=%d\n", ax, by, tc, df);
        }

        small_pause();
    }

    return build_output(params->tid, cnt);
}

int main(void)
{
    pthread_t th1, th2, th3;

    worker_params_t p1 = { 1 };
    worker_params_t p2 = { 2 };
    worker_params_t p3 = { 3 };

    pthread_create(&th1, NULL, task_XY, &p1);
    pthread_create(&th2, NULL, task_YX, &p2);
    pthread_create(&th3, NULL, task_watcher, &p3);

    worker_output_t *out1 = NULL, *out2 = NULL, *out3 = NULL;

    pthread_join(th1, (void**)&out1);
    pthread_join(th2, (void**)&out2);

    pthread_mutex_lock(&mutexX);
    finished = 1;
    pthread_mutex_unlock(&mutexX);

    pthread_join(th3, (void**)&out3);

    const long expected_total = 4L * ITER_LIMIT;

    printf("\n--- Results (returned to main) ---\n");
    printf("Thread %d ops=%ld lastA=%d lastB=%d lastTotalOps=%ld lastDone=%d\n",
           out1->tid, out1->count_done, out1->snapX, out1->snapY, out1->snapTotal, out1->snapFinished);
    printf("Thread %d ops=%ld lastA=%d lastB=%d lastTotalOps=%ld lastDone=%d\n",
           out2->tid, out2->count_done, out2->snapX, out2->snapY, out2->snapTotal, out2->snapFinished);
    printf("Thread %d ops=%ld lastA=%d lastB=%d lastTotalOps=%ld lastDone=%d\n",
           out3->tid, out3->count_done, out3->snapX, out3->snapY, out3->snapTotal, out3->snapFinished);

    printf("\nFinal Shared: A=%d B=%d total_ops=%ld done=%d\n",
           counterX, counterY, total_count, finished);
    printf("Expected: total_ops=%ld done=1\n", expected_total);

    free(out1);
    free(out2);
    free(out3);

    return (total_count == expected_total && finished == 1) ? 0 : 1;
}
```
{% endraw %}
