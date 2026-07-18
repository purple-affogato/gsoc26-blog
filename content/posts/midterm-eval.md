+++
title = 'Midterm Evaluation'
date = 2026-07-15T22:31:54-07:00
draft = false
+++
## Summary of My Deliverables

For the first half of GSoC, my goal is to implement MPCP into RTEMS and open an MR with the completed work. The work that needs to be done for this can be split into 3 parts: **implementation, testing, and documentation**.

### Implementation of MPCP

As previously mentioned, the MPCP implementation was initially created by the research group (Junjie Shi et al.) behind the paper, "Supporting Multiprocessor Resource Synchronization Protocols in RTEMS." More specifically, Jan Pham is credited for writing the initial implementations for MPCP & DPCP.

My main focus was to improve this implementation to 1) better follow the protocol definition, and 2) align with current RTEMS' coding conventions & internal structuring.

For the first point, Kuan, my primary mentor and an author of the paper, helped me a lot with what to target since this is my first time properly developing for SMP systems. The main modifications I ended up making were for `_MPCP_Wait_for_ownership()`, `_MPCP_Surrender()`, and `_MPCP_Initialize()`.

The major flaw in `_MPCP_Wait_for_ownership()` was that it didn't return the actual wait status of an enqueued thread and almost always returns a successful status instead. I fixed this by changing the return statement at the end of the routine from `return STATUS_SUCCESSFUL` to `return _Thread_Wait_get_status( executing );`.

`_MPCP_Surrender()` saw more of an upgrade rather than having to fix an issue. We decided to use `_Thread_queue_Surrender_priority_ceiling()` instead of `_Thread_queue_Surrender()` on the MPCP wait queue. The main advantage of using `_Thread_queue_Surrender_priority_ceiling()` is that it adds the ceiling priority node to the next thread in the wait queue based on priority order. It also removes the ceiling priority node from the executing thread that is surrendering the MPCP control. This greatly simplifies the MPCP implementation, but more crucially, raises the priority of the waiting thread to the ceiling priority before it gets unblocked. This means by the time the next thread is ready and unblocked, it'll be protected by the ceiling priority node and is guaranteed to enter the ready queue with a boosted priority, preventing unsavory race conditions! This also leads to a change in `_MPCP_Wait_for_ownership()` where it's no longer necessary for the routine to use `_MPCP_Raise_priority()` and `_MPCP_Replace_priority()`. Because of this, I added some code at the start of `_MPCP_Wait_for_ownership()` to check that the thread's priority was not higher than the ceiling priority of the MPCP control.

```c
static inline Status_Control _MPCP_Wait_for_ownership(
  MPCP_Control         *mpcp,
  Thread_Control       *executing,
  Thread_queue_Context *queue_context
)
{
  ISR_lock_Context         lock_context;
  Priority_Control         ceiling_priority;
  const Scheduler_Control *scheduler;
  Scheduler_Node          *scheduler_node;

  _Thread_Wait_acquire_default_critical( executing, &lock_context );

  scheduler = _Thread_Scheduler_get_home( executing );
  scheduler_node = _Thread_Scheduler_get_home_node( executing );
  ceiling_priority = _MPCP_Get_priority( mpcp, scheduler );

  if (
    ceiling_priority > _Priority_Get_priority( &scheduler_node->Wait.Priority )
  ) {
    _Thread_Wait_release_default_critical( executing, &lock_context );
    _MPCP_Release( mpcp, queue_context );
    return STATUS_MUTEX_CEILING_VIOLATED;
  }

  _Thread_Wait_release_default_critical( executing, &lock_context );

  _Thread_queue_Context_set_thread_state(
    queue_context,
    STATES_WAITING_FOR_SEMAPHORE
  );

  _Thread_queue_Context_set_deadlock_callout(
    queue_context,
    _Thread_queue_Deadlock_status
  );

  _Thread_queue_Enqueue(
    &mpcp->Wait_queue.Queue,
    MPCP_TQ_OPERATIONS,
    executing,
    queue_context
  );

  return _Thread_Wait_get_status( executing );
}
```

```c
static inline Status_Control _MPCP_Surrender(
  MPCP_Control         *mpcp,
  Thread_Control       *executing,
  Thread_queue_Context *queue_context
)
{
  if ( _MPCP_Get_owner( mpcp ) != executing ) {
    _ISR_lock_ISR_enable( &queue_context->Lock_context.Lock_context );
    return STATUS_NOT_OWNER;
  }

  _MPCP_Acquire_critical( mpcp, queue_context );

  return _Thread_queue_Surrender_priority_ceiling(
    &mpcp->Wait_queue.Queue,
    executing,
    &mpcp->Ceiling_priority,
    queue_context,
    MPCP_TQ_OPERATIONS
  );
}
```

Finally, in `_MPCP_Initialize()`, I removed the usage of workspace allocation and changed the `ceiling_priorities` pointer in the MPCP control to a zero-length array. After looking at the MrsP implementation, I noticed it went through some updates since Junjie Shi et al. wrote the initial MPCP code. The main ones are allowing for initially locked MrsP semaphore and the removal of workspace allocation on the MrsP control. Since initializing the MPCP control is pretty much one-to-one with initializing the MrsP control since neither has any unique traits at this stage, this was another simple change.

### The Test Cases

The main way I tested the MPCP implementation is using the smpmpcp01 testsuite that I wrote up as part of this GSoC project. I've written about smpmpcp01 before on this blog before, but since then it's gone through some additions. Here's the full list of test cases:

- test mpcp obtain and release
- test mpcp nested obtain error
- test mpcp deadlock error
- test mpcp flush error
- test mpcp multiple obtain
- test mpcp initially locked
- test mpcp delete with waiter
- test mpcp ceiling violation
- test mpcp task migrate error
- test mpcp block and unblock
- test mpcp timeout
- test mpcp wrong owner release
- test mpcp waiter selection

Without using too many words, these test cases are meant to ensure that the MPCP implementation actually follows the definition of MPCP stated in the protocol's original paper, Real-time synchronization protocols for shared memory multiprocessors by R. Rajkumar.

A good portion of the test cases check for the error statuses that have to be thrown in undefined scenarios as per the protocol. Good examples of this include test_mpcp_nested_obtain(), test_mpcp_deadlock_error(), test_mpcp_delete_with_waiter(), etc.

Tests like test_mpcp_block_and_unblock(), test_mpcp_waiter_selection(), test_mpcp_multiple_obtain(), etc. are positive tests that ensure that the MPCP implementation works correctly when MPCP semaphores are created and used as intended. This includes asserting task priority after obtaining and releasing a MPCP semaphore, ensuring a blocked task goes to sleep & isn't in the ready queue, checking for priority-based waiter selection, etc.

### MPCP Documentation

This part was relatively quick compared to the above parts. The main documentation I contributed for MPCP is 1) in the Doxygen comments within the source code and 2) in the RTEMS Classic API Guide.

This part mainly consisted of writing up a nice description for MPCP and adding the necessary mentions of MPCP in the Semaphore Manager documentation.

## Challenges Along the Way

Out of the 3 parts of the first half of my GSoC project, creating the testsuite easily took the most time. I took a bit of inspiration from smpmrsp01 to develop smpmpcp01, although I took a simpler approach to the test context structure as I didn't intend to include a high load test. The test context itself mainly contains the IDs of the different worker tasks & semaphores, as well as volatile booleans used to check for which tasks have started running in a couple test cases. At the start, I wasn't super clear how many semaphores & tasks I would need to write all the tasks, so I had to adjust the test context a few times along the way.

Beyond that, I ended up writing many more test cases that I originally wrote about in my project proposal. Looking back, I had previously planned for 5-6 test cases, but ended up writing 13 test cases (-‿-"). I also ended up *not* using the protocol tests from Junjie Shi et al. to better the style of how I was developing smpmpcp01 as a whole.

Additionally, several of the new test cases were suggested by my mentor Kuan a month into working on the project. Going forward, I'll have to keep in mind asking for feedback sooner to give myself more flexibility and have a better pace while working on the project.

## Next Steps

The second half of my GSoC project would be to add support for DPCP (Distributed Priority Ceiling Protocol) to RTEMS. I'm fairly excited for this since DPCP is a much more unique protocol compared to MPCP and I have a feeling the test cases will be also be a bit more complex. ദ്ദി・ᴗ・)✧