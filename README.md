# sched_ext
Sched_ext is a new scheduling class, at a lower priority than CFS, that enables scheduling policies to be be implemented in eBPF programs.
The eBPF program must implement a set of callbacks:
* Task wakeup
* Task enqueue/dequeue
* Task state change
* CPU needs tasks
* Cgroup Integration
* etc...

The program must also configure the scheduler with the following:
* Maximum number of tasks that can be dispatched
* Timeout threshold in ms (<= 30ms)
* Name of the scheduler

## Simple Scheduler Example
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
