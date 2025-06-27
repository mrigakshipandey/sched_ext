
|Resource  | Link                                                                                             |
|----------|--------------------------------------------------------------------------------------------------|
|Docs      |[sched-ext.com](https://sched-ext.com/docs/OVERVIEW)                                              |
|Videos    |[youtube.com](https://www.youtube.com/watch?v=MXejs4KGAro&list=PLLLT4NxU7U1TnhgFH6k57iKjRu6CXJ3yB)|

# sched_ext

## What is a CPU scheduler?
* Manages the finite resource of CPU between all of the execution contexts on the system
* Decide who gets to run next, where they run, and for how long
* Does context switching

## CFS: The Completely Fair Scheduler
CFS is a “fair, weighted, virtual time scheduler”;
Threads are allocated a proportional share of the CPU, based on their weight and load.
Experimenting with CFS directly or implementing a new sched_class from scratch is, of course, possible, but is often difficult and time-consuming.

## Introducing: sched_ext
sched-ext is a new scheduling class introduced in the Linux kernel that provides a mechanism to implement scheduling policies as BPF (Berkeley Packet Filter) programs.
We now have the possibility to easily and quickly implement and test scheduling policies, making it an effective tool for easy experimentation.

Sched_ext is specifically designed to provide significant value in:

  1. **Ease of experimentation and exploration:** Enabling rapid iteration of new scheduling policies.
  2. **Customisation:** Building application-specific schedulers which implement policies that do not apply to general-purpose schedulers.
  3. **Rapid scheduler deployments:** Non-disruptive swap-outs of scheduling policies in production environments.


![image](https://github.com/user-attachments/assets/d7b9107d-dd6e-4d92-8253-60f70d33a656)


---

## Dispatch Queues (DSQs)
Dispatch Queues (DSQs) are the basic building block of scheduler policies. 

Sitting as a layer of abstraction between the scheduler core and the BPF scheduler, sched_ext uses DSQs (dispatch queues), which can operate as both a FIFO and a priority queue. 
By default, there is one global FIFO `SCX_DSQ_GLOBAL`, and one local dsq per CPU `SCX_DSQ_LOCAL`. 
The BPF scheduler can manage an arbitrary number of dsq's using `scx_bpf_create_dsq()` and `scx_bpf_destroy_dsq()`.

A CPU always executes a task from its local DSQ. 
A task is "dispatched" to a DSQ. A non-local DSQ is "consumed" to transfer a task to the consuming CPU's local DSQ.
When a CPU is looking for the next task to run, if the local DSQ is not empty, the first task is picked. 
Otherwise, the CPU tries to consume the global DSQ. 
If that doesn't yield a runnable task either, `ops.dispatch()` is invoked.

DSQs can be FIFO or priority

## DSQ Operations: dispatch and consume
### Dispatch
Placing a task into a DSQ.
It can be done use the follwing callbacks:
* `ops.enqueue()` used to enqueue a task in the BPF scheduler.
* `ops.dispatch()` invoked when the core is going to go idle if a task is not found.

### Consume
Take a task from the DSQ to run on the calling CPU

---

## Global FIFO Scheduler
The BPF side of the scheduler is mostly implemented as a set of callbacks to be invoked via an operations structure, each of which informs the BPF code of an event or a decision that needs to be made. 
The list is long; the full set can be found in [include/sched/ext.h](https://github.com/htejun/sched_ext/blob/sched_ext-v2/include/linux/sched/ext.h#L165) in the SCHED_EXT repository branch.

![image](https://github.com/user-attachments/assets/5517b5d5-ee9a-4dd7-87b4-862206c97376)

To make this happen, we need a global DSQ. Whenever a task wakes up, it is enqueued in the DSQ.

```
 s32 BPF_STRUCT_OPS_SLEEPABLE(mysched_init)
 {
 scx_bpf_create_dsq(/* queue id = */ 0, -1);
 }

void BPF_STRUCT_OPS(mysched_enqueue, struct task_struct *p, u64 enq_flags)
 {
 scx_bpf_dispatch(p, /* queue id = */ 0, SCX_SLICE_DFL, enq_flags);
 }
```
If the core is idle, it will consume a task from the Global DSQ and put it on its Local DSQ.

```
 void BPF_STRUCT_OPS(mysched_dispatch, s32 cpu, struct task_struct *prev)
 {
 scx_bpf_consume(/* queue id = */ 0);
 }

```
