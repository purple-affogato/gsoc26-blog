+++
title = 'Final-Submission'
date = 2026-09-11T10:27:23-07:00
draft = false
+++

## Project Summary

My project is titled "Add MPCP & DPCP Support to RTEMS", and involves implementing two locking protocols, MPCP & DPCP, to RTEMS. The work is based on [Supporting Multiprocessor Resource Synchronization Protocols in RTEMS](https://arxiv.org/pdf/2104.06366) by Junjie Shi et al. The two locking protocols in question, Multiprocessor Priority Ceiling Protocol (MPCP) and Distributed Priority Ceiling Protocol (DPCP), are both based off of the uniprocessor Priority Ceiling Protocol.

A short explanation of MPCP and DPCP: Both protocols are meant for symmetric multiprocessor (SMP) configurations and work with binary semaphores.

MPCP is meant for partitioned systems where tasks are statically bound to their host processors. A task obtaining a MPCP semaphore has its priority raised to the MPCP semaphore's ceiling priority, which prevents priority inversion. The idea (as with PCP) is that a task's critical section should be of higher priority compared to the rest of the task. Upon releasing a MPCP semaphore, the task's priority is lowered to its previous priority. Tasks that are waiting to obtain a MPCP semaphore are blocked.

DPCP is similar to MPCP & PCP in that it uses the same priority ceiling mechanism. DPCP is instead meant for semi-partitioned systems, where tasks typically run their assigned application processors and can only migrate to other processors in specific situations. A task obtaining a DPCP semaphore is migrated to the DPCP semaphore's assigned synchronization processor. Only global critical sections of tasks run on synchronization processors.

## Goals

My main goals for this project were:

1. Improve the original MPCP implementation.
2. Create a testsuite for MPCP in smptests.
4. Improve the original DPCP implementation.
5. Create a testsuite for DPCP in smptests.
6. Write documentation for MPCP & DPCP in RTEMS Classic API Guide.

I had split up this project into two halves. For the first half of the project period (up to the midterm evaluation), I worked on the MPCP portion, while in the later half, I worked on DPCP.

## What was accomplished

Out of my goals, I was able to complete all of them except the DPCP documentation.

As for specifics on the main changes I made: The original implementation for MPCP and DPCP were originally written for RTEMS 4-5.3, meaning that it would need some renovations to be fit for the current RTEMS 7.

The main flaws of the original MPCP implementation was its block-unblock sequence. To wait for a MPCP semaphore, a thread would enqueue to the MPCP semaphore's wait queue which would then block the thread. However, the thread's wait status was not checked afterward to see if the thread successfully became the owner of MPCP semaphore. To fix this, it is easy enough to check `_Thread_Wait_get_status()`. Furthermore, the next owner of the MPCP semaphore needs to have its priority raised before it is unblocked to prevent race conditions. This was fixed by changing the helper function used to surrender the thread queue from `_Thread_queue_Surrender()` to `_Thread_queue_Surrender_priority_ceiling()`. More specifics can be found [in this blog post](https://purple-affogato.github.io/gsoc26-blog/posts/midterm-eval/).

The tests I wrote for the MPCP implementation can be found in smptests/smpmpcp01. It tests for errors related to nested obtain, deadlock, flush, ceiling violation, timeout, wrong owner release, and task migration attempts. It also looks out for common sequences such as obtain & release, block & unblock, obtain multiple MPCP semaphores, initially-locked MPCP semaphore, and priority-based waiter selection.

As for DPCP, it also had similar issues to MPCP with blocking & unblocking, as well as having issues with its migration mechanism. More specifically, the original implementation added `_Scheduler_Migrate_to()` and `_Scheduler_Migrate_back()` for migrating threads to and back from the synchronization processor, which I found did not work based on the DPCP testsuite I wrote (more on that in the next paragraph). To figure out what was not going correctly, I looked into how other migration mechanisms worked in RTEMS. I eventually decided to scrap `_Scheduler_Migrate_to()` & `_Scheduler_Migrate_back()`, and use `_Scheduler_Set()` instead. The basic idea was to store the DPCP owner's original scheduler and priority in the DPCP block and then change the owner's home scheduler to the scheduler of the synchronization processor. The rest of the work that came after was making sure specific locks and critical sections were acquired and released in the correct order, since changing a thread's scheduler state requires different critical sections than the critical section of the DPCP thread wait queue. More on that in this blog post (TBA).

The tests I wrote for the DPCP implementation can be found in smptets/smpdpcp01. It tests for the following errors: nested obtain, flush, deadlock, timeout, ceiling violation, and migration failure. The common sequences it tests are task migration, block & unblock, and obtain multiple DPCP semaphores (of the same synchronization processor).

## My contributions

- [Add MPCP Support to RTEMS PR](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/1358)
- [Add MPCP documentation to Classic API Guide PR](https://gitlab.rtems.org/rtems/docs/rtems-docs/-/merge_requests/247)
- Add DPCP Support to RTEMS PR (TBA)

## Lessons I learned

I gained a lot of technical knowledge to do with RTEMS, locking protocols, and RTOS in general, but also with how to work with an open-source project. One hurdle was figuring out how to integrate the new locking protocols to RTEMS. Originally, each locking protocol had its own bit in the Classic API attributes bit-mask. However, if RTEMS wants to be open to adding new locking protocols, it's good to be careful about how many attribute bits (of 32) are given to just locking protocols. I opened [this post](https://users.rtems.org/t/updating-the-rtems-classic-attributes-to-support-a-semaphore-kind-mask/884/4) on the RTEMS Discourse to create discussion on the topic to eventually come to a conclusion.

Additionally, this was my first time working on the source code of a RTOS. There were a lot of nitty-gritty details I had to become familiar with in both RTEMS and the locking protocols themselves. I also spent a decent amount of time reading and taking notes on research papers behind MPCP & DPCP, and other materials related to resource synchronization protocols.

## Potential Future Work

The most immediate future work on my part would be to see through the MPCP & DPCP PRs.

Outside of that though, the research paper this project was based on also proposes other locking protocols that can be added to RTEMS, especially if they are more dissimilar to MrsP/DPCP/MPCP (all based on PCP/ICPP).

The testsuites for MPCP & DPCP can also be expanded to cover more cases, especially compared to smpmrsp01.

## Final Thoughts

I think I did well on learning many new concepts quickly enough given the time period and translating what I knew about the locking protocols to the implementation itself. What I could've done better on was managing my time, since I had a 9-to-5 during the daytime. There were some days where I was too tired to work on my GSoC project + I was moving in early August, and I could've taken that into consideration more when planning which times of my week I chose to work on RTEMS. In the end, I was able to finish the important parts in time, though this blog became inactive later in the working period.

Overall, I learned a lot from working on this project and I would like to contribute to RTEMS more in the future. ദ്ദി・ᴗ・)✧
