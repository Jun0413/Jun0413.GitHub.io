---
title: "(In)correctness in Semaphore Implementation"
layout: post
date: 2020-03-28 23:59
image: /assets/images/semaphore.jpg
headerImage: true
tag:
- OS
category: blog
author: junhao
description: N.A.
---

## Intro
Synchronization is an important topic in OS. Conceptual-wise, it prevents race condition by making processes wait till the resource become available. This is also called mutual exclusion in that only one process can be active in a critical section. To achieve mutual exclusion, Dijkstra invented the semaphore concept.

A semaphore variable is an integer whose value may cause processes to block or unblock. There are two operations associated with the variable, namely **wait(s)** and **signal(s)**.

I tend to imagine semaphore in such way. There is a room that some people want to get in. The room has a capacity *C*. So, to avoid exceeding the capacity, one may design a system that only allows personnels who carry a token to enter the room. Initially, there are *C* tokens available. Upon exiting the room, the person must return back the token so that if anyone waiting they can get the token and enter the room. Here, the room is the critical section; each person is a process; capacity is the initial semaphore value. wait(s) operation corresponds to taking the token or simply waiting if there are no tokens. signal(s) then corresponds to returning the token.

## Two Different Implementations
With the anology above, we can implement the two operations straightforwardly:

{% highlight raw %}
wait(sem s)
{
  s.val = s.val – 1;
  if (s.val < 0) add calling process to s.queue and block;
}

signal(sem s) {
  s.val = s.val + 1;
  if (s.queue !empty) remove/unblock a process from s.queue;
}
{% endhighlight %}

More specifically, in kernel code C implementation, we can write as follows.

{% highlight C %}
// semaphores are kept in a table called sem_tab
// with information including semaphore ID and a queue of PIDs

void wait(int sid) // sid: semaphore ID
{
  sem_tab[sid].val--;
  if (sem_tab[sid].val < 0) {
    int pid = get_proc_id();
    enqueue(&(sem_tab[sid].queue), pid);
    block();
  }
}

void signal(int sid)
{
  sem_tab[sid].val++;
  if (!queue_empty(&(sem_tab[sid].queue))) {
    int pid = dequeue(&(sem_tab[sid].queue));
    unblock(pid);
  }
}
{% endhighlight %}

Alternatively, they can be implemented as follows:
{% highlight raw %}
wait(sem s)
{
  if (s.val == 0) add calling process to s.queue and block;
  else s.val = s.val – 1;
}

signal(sem s) {
  if (s.queue !empty) remove/unblock a process from s.queue;
  else s.val = s.val + 1;
}
{% endhighlight %}

Why would this also work?

Because the only difference compared to the 1st implementation is that it ensures the semaphore value never goes down to negative. When new processes arrive at the critical section that is filled up by other processes, they simply block and do not further decrement the semaphore value. Correspondingly, in signal(s) operation, processes shall not increment the semaphore if there are any blocking processes, but just unblock them.

Based on the 1st implementation, one may try a 3rd implementation as follows:
{% highlight raw %}
wait(sem s)
{
  // breakpoint 1
  if s.val <= 0 {
    addProc(me, s.queue);
    block(me);
  }
  // breakpoint 2
  s.val = s.val – 1;
}

signal(sem s) {
  s.val = s.val + 1;
  if (s.queue !empty) remove/unblock a process from s.queue;
}
{% endhighlight %}

This is logically correct. But it is **wrong**. The tricky part here is to account for context switching.

Assume that this is a binary semaphore and the CPU can only execute one process at a time. Initially, process A enters into the critical section, so newly arrived process B has to wait and it is blocked at breakpoint 2. Now process C finishes and calls signal(s). Semaphore gets incremented and process B is unblocked. But before process B continues from breakpoint 2, a new process C arrives, which sees semaphore equal to 1 at breakpoint 1, thus will also enter into the critical section. As such, mutual exclusion of B and C fail.

We may wonder, if context switching can happen during wait(s) and signal(s) operations, which are supposed to be atomic, wouldn't the 1st implementation also fails?

First, it will not fail. If we apply the same scenario, we will realize that in wait(s) the semaphore value is always tested after it gets decremented, and will bar process C from entering.

Second, the 1st implementation guarantees the *effective atomicity* of wait(s) and signal(s). The two operations do not have to be absolutely atomic. It suffices as long as the interruption does not affect the result.