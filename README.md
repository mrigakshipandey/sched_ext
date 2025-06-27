# sched_ext
|Resource  | Link                                                                                             |
|----------|--------------------------------------------------------------------------------------------------|
|Docs      |[sched-ext.com](https://sched-ext.com/docs/OVERVIEW)                                              |
|Videos    |[youtube.com](https://www.youtube.com/watch?v=MXejs4KGAro&list=PLLLT4NxU7U1TnhgFH6k57iKjRu6CXJ3yB)|

sched_ext is a Linux kernel feature that enables the implementation and dynamic loading of safe kernel thread schedulers in BPF.

Sched_ext is specifically designed to provide significant value in:

  1. **Ease of experimentation and exploration:** Enabling rapid iteration of new scheduling policies.
  2. **Customisation:** Building application-specific schedulers which implement policies that do not apply to general-purpose schedulers.
  3. **Rapid scheduler deployments:** Non-disruptive swap-outs of scheduling policies in production environments.

Experimenting with CFS directly or implementing a new sched_class from scratch is of course, possible, but is often difficult and time-consuming. 

## Simple Scheduler Example
sched_ext is a new sched_class (at a lower priority than CFS) which allows scheduling policies to be implemented in BPF programs.

sched_ext leverages BPF's struct_ops feature to define a structure which exports function callbacks and flags to BPF programs that wish to implement scheduling policies. 
The struct_ops structure exported by sched_ext is struct sched_ext_ops, and is conceptually similar to struct sched_class. 
The role of sched_ext is to map the complex sched_class callbacks to the simpler and ergonomic struct sched_ext_ops callbacks.

Callbacks:
* Task wakeup
* Task enqueue/dequeue
* Task state change
* CPU needs tasks
* Cgroup Integration
* etc...

The only struct_ops field that is required to be specified by a scheduler is the 'name' field.

```
const volatile bool switch_partial: /*can be set by user space before loading the program*/

s32 BPF_STRUCT_OPS(simple_init){
  if(!switch_partial) /*if set, tasks will be individually configured to SCHED_EXT class*/
    scx_bpf_switch_all(): /* switch all CFS tasks to sched_ext*/
  return 0:
}

// Enqueue a task to the shared DSQ, dispatching it with a time slice
int BPF_STRUCT_OPS(simple_enqueue, struct task_struct *p, u64 enq_flags) {
  if(enq_flags & SCX_ENQ_LOCAL) // if current CPU has no tasks to run
    scx_bpf_dispatch(p, SHARED_DSQ_LOCAL, slice, enq_flags); // dispatch to local FIFO
  else
    scx_bpf_dispatch(p, SHARED_DSQ_GLOBAL, slice, enq_flags); // dispatch to global FIFO
}

void BPF_STRUCT_OPS(simple_exit, struct simple_exit_info *ei) {
  bpf_printk("Exited");
}

SEC(".struct_ops")
struct sched_ext_ops simple_ops = {
    .enqueue   = (void *)simple_enqueue,
    .init      = (void *)simple_init,
    .exit      = (void *)simple_exit,
    .name      = "simple",
};
```
---

## Dispatch Queues (DSQs)
Sitting as a layer of abstraction between the scheduler core and the BPF scheduler, sched_ext uses DSQs (dispatch queues), which can operate as both a FIFO and a priority queue. 
By default, there is one global FIFO `SCX_DSQ_GLOBAL`, and one local dsq per CPU `SCX_DSQ_LOCAL`. 
The BPF scheduler can manage an arbitrary number of dsq's using scx_bpf_create_dsq() and scx_bpf_destroy_dsq().

A CPU always executes a task from its local DSQ. 
A task is "dispatched" to a DSQ. A non-local DSQ is "consumed" to transfer a task to the consuming CPU's local DSQ.
When a CPU is looking for the next task to run, if the local DSQ is not empty, the first task is picked. 
Otherwise, the CPU tries to consume the global DSQ. 
If that doesn't yield a runnable task either, ops.dispatch() is invoked.

---
## DSQ Operations: dispatch and consume
### Dispatch
Placing a task into a DSQ.
It can be done use the follwing callbacks:
* `ops.enqueue()` used to enqueue a task in the BPF scheduler.
* `ops.dispatch()` invoked when the core is going to go idle if a task is not found.

### Consume
Take a task from the DSQ to run on the calling CPU
