<!-- vim: set spell spelllang=en_us: -->

# Writing Real-Time Software on Linux

As the entire purpose of embedded Linux in many cases such as industrial
control, automotive control, and other similarly time-sensitive areas often is
to have real-time capable control with an OS-level software stack, it follows,
naturally, that one would want to write real-time software on Linux.

Before I start, note that the main *external* reference for this section is The
Linux Foundation's wiki. However, Linux documentation should always be
referenced in all cases, and since the aforementioned wiki is rather lacking in
some information, a lot of this comes from my own testing and experience.

As with all real-time systems, you should experiment, do your homework, and test
what works best for you rather than blindly following what someone online said
will work. This worked for me, and should work in most or all other cases, but
may not be appropriate for your use case.

## Table of Contents
<!-- vim-markdown-toc GFM -->

* [Real-Time vs. Real Fast](#real-time-vs-real-fast)
* [RT Tasks](#rt-tasks)
  * [Scheduling Options](#scheduling-options)
  * [Memory Locking](#memory-locking)
  * [Creating Tasks](#creating-tasks)
  * [EDF Scheduling](#edf-scheduling)
  * [Task Timing](#task-timing)
  * [Task Synchronization](#task-synchronization)
    * [Mutexes](#mutexes)
    * [Condition Variables](#condition-variables)

<!-- vim-markdown-toc -->

## Real-Time vs. Real Fast

I encourage you to read [this excellent
paper](https://www.kernel.org/doc/ols/2008/ols2008v2-pages-57-66.pdf) written by
Pale E McKenney of the IBM Linux Technology Center. This paper explains how to
choose whether or not you want a real-time kernel (using the PREEMPT RT patches
and the like) or a standard "real fast" kernel (the sort an average desktop user
might use). There are benchmarks, code samples, and hard data explaining when to
choose each.

Here's a few quick details:
- If your system needs to meet precise timing requirements and be extremely
  deterministic, then you want real-time
- If your system is just going to be used for basic user-space things (such as
  building the Linux kernel, as shown in the paper), you should use real fast
- If your system needs to be on time, use real-time; if your system needs to
  process a bunch of data, use real fast

Now that you should be fully aware of which to choose (or at least have some
inkling that you should choose one or the other, but may need to do more
testing), we can get on with the real-time stuff.

## RT Tasks

As always, the [Linux manual pages](https://www.kernel.org/doc/man-pages/) are a
great source of information for all things Linux. I will refer to them as we go,
so you will want to be cross-referencing with them as well.

### Scheduling Options

Linux has a whole bunch of different scheduling types. The real-time scheduling
options are FIFO, RR, and most recently, DEADLINE. These are explained in great
detail in [sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html), so
I will not go into a lot of detail here. The basic concepts are that for
a real-time system without EDF (earliest-deadline-first) scheduling, you will
want to use FIFO: first in, first out. The real-time priorities on Linux are
given priorities 0-99, with the exception of DEADLINE, which is higher priority
than the others and can (and will) preempt them as necessary to meet the task's
deadline.

The most straightforward way of doing real-time scheduling (and likely the most
common) on Linux is to use FIFO. Your real-time tasks should avoid doing
extensive I/O when possible and should spend as little time as possible doing
blocking operations when you can avoid it, such that other threads with lower
priority can do their work without being preempted all the time. Additionally,
it's commonly recommended not to give a task priority 99, as this priority will
compete with interrupt handlers and the like, causing a lot of slowdowns.
Instead, max your tasks out at 98 when using FIFO or RR scheduling.

Again, in accordance with best practices, you should avoid memory allocation in
hot paths of real-time tasks. You should use statically allocated memory when
possible, or if you must use dynamic allocation, you should allocate it ahead of
time as much as possible. At the very least, you should allocate a pool of
memory to use later on if you must.

### Memory Locking

When memory is swapped out and must be retrieved (among other common
operations), a page fault is generated. These can be either minor or major page
faults. There are plenty of details online about how these work, and the word
"fault" here does not actually mean it's an error. However, these do take some
computational time away from your program, so you should always lock your
program's memory space such that it doesn't get swapped out. See
[mlock(2)](https://man7.org/linux/man-pages/man2/mlock.2.html) for more details
on this process.

```c
#include <stdio.h>
#include <stdlib.h>

#include <sys/mman.h>

int main(void) {
  if (mlockall(MCL_CURRENT | MCL_FUTURE) < 0) {
    perror("mlockall");
    exit(1);
  }
  
  return 0;
}
```

### Creating Tasks

If you want to create a task (aside from main, which will simply wait for the
task to finish) which has FIFO scheduling and priority 98, you will write
something like this:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <pthread.h>
#include <sched.h>
#include <sys/mman.h>

// this function will be run by the task we create, so any RT task stuff should
// happen here
void* rt_func(void* data) {
  // (do RT stuff)
  return NULL;
}

int main(void) {
  struct sched_param sp;
  pthread_attr_t attr;
  pthread_t thread;
  int ret;

  if (mlockall(MCL_CURRENT | MCL_FUTURE) < 0) {
    perror("mlockall");
    exit(1);
  }

  ret = pthread_attr_init(&attr);
  if (ret < 0) {
    perror("pthread_attr_init");
    exit(1);
  }

  // this step is optional, as this should be the default value
  ret = pthread_attr_setstacksize(&attr, PTHREAD_STACK_MIN);
  if (ret < 0) {
    perror("pthread_attr_setstacksize");
    exit(1);
  }

  ret = pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
  if (ret < 0) {
    perror("pthread_attr_setschedpolicy");
    exit(1);
  }

  memset(&sp, 0, sizeof(sp));
  // since technically the max priority is implementation-defined, we will query
  // it and then subtract 1
  sp.sched_priority = sched_get_priority_max(SCHED_FIFO) - 1;

  ret = pthread_attr_setschedparam(&attr, &sp);
  if (ret < 0) {
    perror("pthread_attr_setschedparam");
    exit(1);
  }

  // by default, threads will inherit the parent's scheduling, so we will tell
  // this thread to use the sched params we just gave it instead of inheriting
  ret = pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
  if (ret < 0) {
    perror("pthread_attr_setinheritsched");
    exit(1);
  }

  // create our thread
  ret = pthread_create(&thread, &attr, rt_func, NULL);
  if (ret < 0) {
    perror("pthread_create");
    exit(1);
  }

  // wait for the thread to return (we could also proceed to do other things
  // here as well, but for this example we will just wait)
  ret = pthread_join(thread, NULL);
  if (ret < 0) {
    perror("pthread_join");
    exit(1);
  }

  return 0;
}
```

You could, of course, create more tasks with other priorities (or even the same;
see sched(7) for details) to do other things. These would be preempted by the
higher priority ones.

### EDF Scheduling

Earliest-deadline-first scheduling is quite new to Linux. I have tested a little
and it seems to work quite well and yield excellent performance. However, as it
requires a little more work than FIFO at the moment (glibc currently does not
have wrappers for some of the syscalls and structures used), it's best to simply
refer to the kernel documentation for this one: [Deadline Task Scheduling](https://www.kernel.org/doc/html/latest/scheduler/sched-deadline.html)

### Task Timing

When writing a real-time application, it is important that your tasks perform
their duties at a specific interval and sleep during the unused time so that
other lower priority tasks can do their work. There are a handful of ways to
accomplish this, but I have adopted a very simple and reliable way based on the
solution found [here](https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/cyclic).
This can easily be wrapped in a C++ class (which is, in fact, what I did) to
make it more convenient to use. That is left as an exercise to the reader.
Instead, here is an example implementation that causes a task to wakeup every
1ms, do some work, and then go back to sleep for the remainder of the period.
This is base on the link above, but also includes error handling.

```c
#include <errno.h>
#include <time.h>

struct tasktimer {
  struct timespec next_period;
  long period_ns;
};

inline int tasktimer_init(struct tasktimer* tmr, long period_ns) {
  tmr->period_ns = period_ns;

  return clock_gettime(CLOCK_MONOTONIC, &tmr->next_period);
}

inline void tasktimer_inc(struct tasktimer* tmr) {
  tmr->next_period.tv_nsec += tmr->period_ns;

  while (tmr->next_period.tv_nsev >= 1000000000L) {
    ++tmr->next_period.tv_sec;
    tmr->next_period.tv_nsec -= 1000000000L;
  }
}

inline int tasktimer_wait(struct tasktimer* tmr) {
  int ret;

  tasktimer_inc(tmr);

  do {
    ret = clock_nanosleep(CLOCK_MONOTONIC,
                          TIMER_ABSTIME,
                          &tmr->next_period,
                          NULL);
  } while (ret < 0 && errno == EINTR);

  return ret;
}

void* rt_task(void* data) {
  struct tasktimer timer;
  int ret;

  ret = tasktimer_init(&timer);
  if (ret < 0) {
    perror("tasktimer_init");
    // handle error however fits your program the best
  }

  while (1) {
    // do RT work here...
    ret = tasktimer_wait(&timer);
    if (ret < 0) {
      perror("tasktimer_wait");
      // handle error however fits your program the best
    }
  }

  return NULL;
}
```

This should allow other tasks to fill in the gaps while this task is sleeping.
It also results in extremely precise timing on RT-patched systems.

### Task Synchronization

Synchronizing data between real-time tasks has some quirks compared to a normal
non-real-time program. You must take care to ensure your application employs
proper [priority inversion](https://wiki.linuxfoundation.org/realtime/documentation/technical_basics/pi),
else you will run into deadlocks between tasks when using mutexes and other
synchronization types!

#### Mutexes

Securing shared data is done with a pthread mutex, like normal, but with one
major difference: you *must* set the mutex to inherit the priority of the tasks
waiting on it. This allows for proper [priority inversion](https://wiki.linuxfoundation.org/realtime/documentation/technical_basics/pi)
when tasks are waiting on other tasks to finish with the lock. When creating
your mutex, you should do the following (the initialization of the mutex has
been left out for brevity, so see [pthread_mutex_init(3p)](https://man7.org/linux/man-pages/man3/pthread_mutex_init.3p.html)
for more info). Error handling has been omitted for brevity.

```c
pthread_mutexattr_t attr;

pthread_mutexattr_init(&attr);

// enable priority inversion
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);

// set to at least the highest priority of the threads which will lock it
pthread_mutexattr_setprioceiling(&attr, sched_get_priority_max(SCHED_FIFO));
```

#### Condition Variables

I cannot stress this enough: **DO NOT USE** glibc's pthread condition variables!
They do NOT properly implement priority inheritance and will cause unbounded
priority inversion. I cannot seem to find this documented anywhere, but [the
issue](https://sourceware.org/bugzilla/show_bug.cgi?id=11588) has been open
since 2010 and doesn't appear to have any real solution in sight. If you need
something like a condition variable, you should manually implement it. This is
a fair bit of boilerplate, but it is necessary to properly achieve this
functionality with glibc. I don't know about other C libraries, but for glibc
you must absolutely avoid pthread condition variables at all costs!

I found this out after using them for quite some time. It turns out that by
removing them from my program, the CPU utilization of the system went down by
nearly 40%. It would be quite nice if GNU decided to document this somewhere,
but they didn't, so here we are.
