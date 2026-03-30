
# Freezer

<div align='center'>
    <img src='./cs4118-freezer.png'/><br/>
    Why's this one called freezer? ...Oh
</div>

## Submission

As with previous assignments, we wil be using GitHub to distribute skeleton code
and collect submissions. Please refer to our [Git Workflow][git-workflow] guide
for more details. **Note for this homework, you do not need to commit and push tags for any of the part. We will simply grade your last submission before the deadline**.

For students on arm64 computers (e.g. M1/M2 machines): if you want your
submission to be built/tested for ARM, you must create and submit a file called
`.armpls` in the top-level directory of your repo; feel free to use the
following one-liner:

```sh
cd "$(git rev-parse --show-toplevel)" && touch .armpls && git add -f .armpls && git commit .armpls -m "ARM pls"
```

**You should do this first so that this file is present in all parts.**

[git-workflow]: https://columbia-os.github.io/dev-guides/git-workflow

## Code Style

There is a script in the skeleton code named `run_checkpatch.sh`. It is a
wrapper over `linux/scripts/checkpatch.pl`, which is a Perl script that comes
with the Linux kernel that checks if your code conforms to the kernel coding
style.

Execute `run_checkpatch.sh` to see if your code conforms to the kernel style –
it'll let you know what changes you should make. You must make these changes
before pushing a tag. Passing `run_checkpatch.sh` with no warnings and no errors
**is required** for this assignment.

## Part 0: Before Start

In this assignment, we will work on the Linux kernel's scheduler, 
specifically modifying and analyzing scheduling policies. To help you understand 
the impact of different schedulers, we will introduce a simulated workload using 
a Fibonacci calculator. 

The `user/taskset1.txt` and `user/taskset2.txt` provide two task set workloads, 
each consisting of a list of jobs, with each line representing the  N-th 
Fibonacci number to compute.

Start by taking a look at `user/fibonacci.c` and `user/run_tasks.sh`. Play around with 
the scripts to get familiar with how tasks are executed and scheduled.

**Note:** The Fibonacci calculator has been provided as `user/fibonacci.c`. It uses an 
inefficient algorithm by design to produce differing job lengths. You can modify the file, 
but **do not modify the `fib` function**.

## Part 1: Measure scheduler performance with eBPF

To see how well a scheduler performs, it is necessary to first have a way to measure performance,specifically completion time of tasks. In this part, we are looking for a way to trace scheduling events and determine how well a scheduler functions by measuring tasks' run queue latency and total duration from start to finish.

Extended Berkeley Packet Filter ([eBPF](https://ebpf.io/)) is a powerful feature of the Linux kernel that allows programs to inject code into the kernel at runtime to gather metrics and observe kernel state. You will use eBPF to trace scheduler events. Tracepoints for these scheduler events are already available in the scheduler (for example, you can search in `kernel/sched/core.c` for `trace_sched_*`), so your job is to write code that will be injected into the kernel to use them. You should not need to modify any kernel source code files for this first part of the assignment.

bpftrace is a Linux tool that allows using eBPF with a high level, script-like language. Install it on your VM with:

```sh
sudo apt install bpftrace
```


Instead of searching through source code, you can run `sudo bpftrace -l` to see what available tracepoints there are in the kernel.

Write a bpftrace script named trace.bt to trace how much time a task spends on the run queue, and how much time has elapsed until the task completed. Your profiler should run until stopped with Ctrl-C. While tracing, your script should, at tasks' exits, print out a table of the task name, task PID, milliseconds spent on the run queue (not including time spent running), and milliseconds until task completion. Note that task PID here is the actual pid field in a `task_struct`, not what is returned by getpid. The output should be comma-delimited and follows the format:

```
COMM,PID,RUNQ_MS,TOTAL_MS
myprogram1,5000,2,452
myprogram2,5001,0,15
...
```


Ensure that the output of your eBPF script is synchronous. That is, it should print each line immediately after a task completes, and not when Ctrl-C is sent to the eBPF script. You should also only print traces from processes that have started after your script begins running. You should trace all such process and perform no additional filtering.

Test your script by running sudo bpftrace trace.bt in one terminal. In a separate terminal, run a few commands and observe the trace metrics in your first terminal. Experiment with different task sizes. Submit your eBPF script as `user/trace.bt`.

Now that you have an eBPF measurement tool, use it to measure the completion times for the the two task set workloads when running on the default CFS scheduler in Linux. What is the resulting average completion time for each workload? Write the average completion times of both workloads in your README. You may find `user/run_tasks.sh` helpful for running your tasks. You should perform your measurements for two different VM configurations, a single CPU VM and a four CPU VM. We are using the term CPU here to mean a core, so if your hardware has a single CPU but four cores, that should be sufficient for running a four CPU VM. If your hardware does not support four cores, you may instead run with a two CPU VM. Specify in your README the number of CPUs used with your VM for your measurements.

**Hint:** You may find it useful to reference the bpftrace [GitHub project](https://github.com/iovisor/bpftrace), which contains a manual and a list of sample scripts. A useful example script to start with is [runqlat.bt](https://github.com/bpftrace/bpftrace/blob/aa041d9d85f9ec11235c39fdcb5833412ec27083/tools/runqlat.bt).

**Note:** Since your profiler will run on the same machine as the test workloads, we will need to ensure that the profiler gets sufficient CPU time to record data. Consider why running `chrt --fifo 90` is helpful in this context in `user/run_tasks.sh`. You may find [`chrt`](https://man7.org/linux/man-pages/man1/chrt.1.html) helpful.

### Deliverables

- trace.bt file in the `user/` folder

- average completion times of both workloads in your README

## Freezer/Heater Overview
In this assignment, we will add two new Linux scheduler classes called **Freezer** and **Heater(optional, 50 extra credits)**. **Freezer** will be default to Linux scheduler in the end, while **Heater** does not.

**Freezer** will have the following features and characteristics:

-   Implements a simple round-robin scheduling algorithm.

-   Supports SMP.

-   Every task has the same time slice of 100 milliseconds.

-   When deciding which CPU a task should be assigned to, it should be assigned
    to the CPU with the fewest Freezer tasks. In addition, when a CPU becomes
    idle, it will steal a task from another CPU.

-   The Freezer scheduling policy, `SCHED_FREEZER`, sits right in between
    `SCHED_NORMAL` and the real-time scheduling policies. That is,
    `SCHED_FREEZER` takes priority over `SCHED_NORMAL`, but not over `SCHED_RR`
    or `SCHED_FIFO`.

**Heater(optional, 50 extra credits)** will have the following features and characteristics:

-   Implements a global-queue scheduling algorithm.

-   Each CPU will fetch tasks from the global queue as needed for running.

-   The Heater scheduling policy, `SCHED_HEATER`, sits right below
    `SCHED_NORMAL`. That is,
    `SCHED_NORMAL` takes priority over `SCHED_HEATER`.


Parts 2-7 are structured to guide you in developing new schedulers. Each part
represents a milestone towards a working scheduler implementation. **This forces
you to develop incrementally. This also allows us to award you partial credit if
you cannot get the whole scheduler working at the end.**

In part 2, you will code up all the necessary data structures for **Freezer** and
add them to the right places, but they will remain unused by the rest of the
kernel.

In part 3, you will flesh out the implementation of **Freezer** and enable it in the
kernel. But you will keep the old CFS scheduler as the default, so that the
system will boot even if your Freezer implementation is still unstable.

In part 4(optional), you will code up the necessary data structures for **Heater**, flesh out
the implementation of Heater and enable it in the kernel. Still, you will keep
the old CFS scheduler as the default.

In part 5, you will compare the performance of Fibonacci tasks across different
scheduling policies, including FIFO, **Freezer** and **Heater**.

In part 6, you will make **Freezer** the default scheduler of your kernel. All
normal processes and kernel threads will run on **Freezer** (unless they are
explicitly assigned to a specific scheduling policy).

In part 7, you will add idle load balancing to **Freezer**. This allows processes
to make better use of available CPUs.

## Part 2: Freezer, unplugged

### Reading assignment

-   Please make sure you have done the OSTEP and LKD reading assignments on
    scheduling and reviewed the lecture slides on Linux scheduling classes.

-   A section from another book, *Professional Linux Kernel Architecture*, by
    Wolfgang Mauerer

    -   Section 2.5, Implementation of the Scheduler (pages 83 - 105)

### Tasks

-   Implement the following data structures and definitions. For grading
    purposes, please use the **EXACT** names.

    ```c
    /* SCHED_FREEZER must be defined to be 7 */
    #define  SCHED_FREEZER  7

    /*
     * define FREEZER_TIMESLICE so that every task receives
     * a time slice of 100 milliseconds
     */
    #define  FREEZER_TIMESLICE  <the-right-timeslice-value>

    /* freezer run queue and entity */
    struct freezer_rq;
    struct sched_freezer_entity;

    /*
     * freezer scheduling policy implementation;
     * put the code in kernel/sched/freezer.c
     */
    struct sched_class freezer_sched_class;
    ```

-   The `freezer_sched_class` methods can be left incomplete at this point because
    no one is going to call them. Maybe have the struct and functions defined,
    but leave most of them empty. Maybe put some `TODO` comments. Your objective
    in this part is to get all your data structures in the right place and get
    them compiled.

    -   See `idle.c` for an example of a very minimal `sched_class` object.

-   Learn from the man page of the `ps` command what the following command shows
    you:

        ps -e --forest -o sched,policy,psr,pcpu,c,pid,user,cmd

-   Use the `ps` command above before and after this part, to verify that your
    addition to the kernel had no effect.

### Deliverables

-   Points for this part will be based on your design and placement of these
    data structures.

-   Please put `freezer_sched_class` in `kernel/sched/freezer.c` in your Linux
    source tree.


## Part 3: Turn on the Freezer

Now we turn on the freezer, and hook it up to the rest of the kernel. You will
have to modify various parts of the kernel to enable the new `SCHED_FREEZER`
scheduling policy. However, in this part, `SCHED_NORMAL` will remain as the
default policy.

### Tasks

-   Begin to implement the Freezer scheduler according to the specifications in
    the overview, such that you should be able to set a process to
    `SCHED_FREEZER` and see it run.

    -   Make sure that CFS is still the default scheduler. Use the `ps` command
        from part 3 to verify that everything is still the same when you booted
        into your new Freezer-enabled kernel.

-   Recall that `fibonacci.c` attempts to change its scheduling policy to 
    `SCHED_FREEZER` to compute the Fibonacci number, and `run_tasks.sh` will 
    take a file as input and launch a lot of Fibonacci tasks.

-   Your objective in this part is to have your kernel, `fibonacci.c` and 
    `run_tasks.sh` program to work together successfully, so you can change 
    a process's scheduling policy to Freezer.

-   The `fibonacci.c` will now have no problem switching to the `SCHED_FREEZER` 
    with the call `sched_setscheduler(0, 7, &params)`

-   **This is extremely important!** Make sure that your Freezer task is actually 
    using `freezer_sched_class`. For example, you can confirm that your functions 
    are being called by adding a `pr_info()` to a function like `freezer_enqueue_task()`, 
    and verify that it appears in `dmesg` output upon running your Freezer program. 
    You should not submit with these debugging prints; these can break your Part 6 if
    invoked too frequently.

### Tips

-   Some relevant source files: `core.c`, `rt.c`, `fair.c`

    -   There is a lot of code in these files, for sure. An important goal here
        is for you to understand how to abstract the scheduler code so that you
        learn in detail the parts that matter, and ignore the parts that don't.

-   This is going to be really hard to debug. You might want to invest a little
    bit of time up front and learn how to use `trace_printk()`, and the `ftrace`
    tracing framework in general. Googling turns up many resources.

-   You might also find that your kernel will crash before you can see your
    debug messages show up in the log buffer (ie. before you can run `dmesg`).
    Instead of printing to the kernel log buffer, you can "redirect" your
    messages to a file on your host machine by setting up a serial port. See the
    Serial Console section of our [Linux Kernel Debugging guide][serial-console]
    for details.

    [serial-console]: https://columbia-os.github.io/dev-guides/kernel-debugging.html

### Deliverables

-   Modifications to the Linux source tree implementing the Freezer scheduler.

    -   Please implement your `freezer_sched_class` in `kernel/sched/freezer.c`
        in your Linux source tree.

    -   You may also need to modify other source files in the Linux kernel.

## Part 4: Turn on the Heater(optional, 50 extra credits)

Now we will implement the heater, and hook it up to the rest of the kernel. Again, 
You will have to modify various parts of the kernel to enable the new `SCHED_HEATER`
scheduling policy. `SCHED_HEATER` could sit either above or below the `SCHED_FREEZER`
at this time. Still, `SCHED_NORMAL` will remain as the default policy.

### Tasks

- Here are some suggested data structures and definitions you may find useful for implementation. Feel free to use or modify them as needed.

    ```c
    /* SCHED_HEATER must be defined to be 8, and sit below SCHED_NORMAL in priority */
    #define  SCHED_HEATER  8

    /* generic implementation of a run queue */
    struct heater_rq;
    
    /*
    * A global run queue that is shared by all CPUs.
    * CPUs will pull from the head of this global run queue to decide which task to execute next.
    */
    struct heater_rq global_rq;

    /* a spin lock for the global run queue; each cpu will need to acquire a spin lock to modify the global run queue */
    DEFINE_RAW_SPINLOCK(global_rq_lock);

    /* heater entity */
    struct sched_heater_entity;

    ```
- The size of each cpu's run queue should be limited to a maximium of 1 task at any time. Any overflow should go to the global run queue.
- Newly enqueued task should be placed at the end of the global run queue.
- When a CPU needs a new task (for example during `dequeue_task` or `pick_next_task`), the CPU should pull a task from the head of the global run queue.
- Heater is non-preemptive. When a heater task is scheduled to run, it continues executing until it voluntarily yields or exits. It should not be preempted by another heater task.
    
### Tips:
-   Refer to the tasks and tips in Parts 2 and 3. This part follows the same process 
    as the previous two parts, where you implemented `FREEZER` scheduler.

-   The `fibonacci.c` will now have no problem switching to the `SCHED_HEATER` 
    with the call `sched_setscheduler(0, 8, &params)`.

-   **This is extremely important!** Make sure that your Heater task is actually 
    using `heater_sched_class`. For example, you can confirm that your functions 
    are being called by adding a `pr_info()` to a function like `heater_enqueue_task()`, 
    and verify that it appears in `dmesg` output upon running your Heater program. 
    You should not submit with these debugging prints; these can break your Part 6 if
    invoked too frequently.

### Deliverables(optional)

-   Modifications to the Linux source tree implementing the Heater scheduler.

    -   Please implement your `heater_sched_class` in `kernel/sched/heater.c`
        in your Linux source tree.

    -   You may also need to modify other source files in the Linux kernel.


## Part 5: Freezer or Heater? or FIFO?

The **Heater** measurement is only required if you chose to implement it.

Recall your average completion time measurements from Part 1, which used the default CFS scheduler. 
Now, evaluate the performance of the **Freezer** and **Heater** you implemented in the previous parts.
Additionally, compare these results with **FIFO**, a real-time scheduling policy already implemented in Linux.  

Modify the `user/fibonacci.c` or `user/tun_tasks.sh` to measure the performance for **Freezer** 
and **Heater**. Again,  conduct these measurements for two different VM configurations, 
a single CPU VM and a four CPU VM. Record the average completion times of both workloads in your 
README, and specify the number of CPUs used with your VM for your measurements.  

Next, assess the performance of FIFO. Note that Linux’s built-in FIFO scheduling policy includes 
load balancing and task migration support. To ensure a fair comparison with Freezer and Heater, 
you should disable these features. One approach to achieving this is by setting the CPU affinity 
of tasks to restrict them to a specific core, preventing unintended migrations. You may find the 
[`taskset`](https://man7.org/linux/man-pages/man1/taskset.1.html) command helpful for this purpose.
Once again, Record the average completion times of both workloads in your README.  

After completing your measurements, compare the results of the three scheduling policies. Which 
scheduler performed best in terms of average completion time? Which one was the worst? What are
the possible reasons?

Another important metric to consider is tail completion time, which is the maximum completion time 
across 99% of all jobs. Tail completion time focuses on ensuring that the completion time of most 
jobs is no worse than some amount. Using tail completion time as the performance metric, repeat the
measurement process described above. Record the tail completion times for both workloads across all 
three scheduling policies in your README. Finally, revisit the comparison and analysis: Which 
scheduler performed best? Which performed the worst? What factors contributed to these results?

### Deliverables

-   Recorded data(average and tail completion time) for three schedulers and answers to the questions 
    in the README.

-   Describe what changes you made to disable the load balancing and task migration for FIFO.


## Part 6: Freeze everything

Now let's make Freezer the default scheduling policy. This means that all system
and user tasks will have the `SCHED_FREEZER` policy by default. You need to set
the Freezer scheduling policy for the `init_task`, which will be inherited by
all its descendants. You also need to set it for all kernel threads, which is
done in `kernel/kthread.c`.

### Tasks

- Modify the Linux kernel source to use `SCHED_FREEZER` as the default
    scheduling policy.

- Verify that the `ps` output shows that most tasks, including kernel threads,
    are running under Freezer. The SCH column will show `7` and the POL column
    will show `#7`.

- Verify that new tasks get the Freezer policy.

- Verify that the tasks are assigned to CPUs as specified.

- Verify that the tasks run in round-robin fashion.

- Verify that your scheduler is stable and can handle running multiple
    CPU-intensive tasks. You may write your own stress-test program.

- **This is extremely important!** Make sure that tasks are actually 
    using `freezer_sched_class` by default. Adding a `pr_info()` to a 
    function like `freezer_enqueue_task()` might print too much things to
    the `dmesg` output. Instead, you can check the task's name using an 
    if condition, so that `pr_info()` prints a message to dmesg only when a 
    specific task is enqueued, preventing excessive logging.

Here is a part of a `ps` command output that shows 20 CPU-bound processes (not
counting the four parent processes that forked them) running across 4 CPUs:

    $ ps -e --forest -o sched,policy,psr,pcpu,c,pid,user,cmd

    SCH POL PSR %CPU  C   PID USER     CMD

      7 #7    3  0.0  0     2 root     [kthreadd]
      7 #7    0  0.0  0     3 root      \_ [rcu_gp]
      7 #7    0  0.0  0     4 root      \_ [rcu_par_gp]
      7 #7    0  0.0  0     6 root      \_ [kworker/0:0H-kblockd]
      7 #7    1  0.0  0     7 root      \_ [kworker/u8:0-events_unbound]
      7 #7    0  0.0  0     8 root      \_ [mm_percpu_wq]
      7 #7    0  0.0  0     9 root      \_ [ksoftirqd/0]
      7 #7    2  0.0  0    10 root      \_ [rcu_sched]
      7 #7    0  0.0  0    11 root      \_ [rcu_bh]
      1 FF    0  0.0  0    12 root      \_ [migration/0]
      7 #7    0  0.0  0    13 root      \_ [kworker/0:1-events]
      7 #7    0  0.0  0    14 root      \_ [cpuhp/0]
      7 #7    1  0.0  0    15 root      \_ [cpuhp/1]
      1 FF    1  0.0  0    16 root      \_ [migration/1]

                                      ...

      7 #7    3  0.1  0     1 root     /sbin/init
      7 #7    3  0.0  0   221 root     /lib/systemd/systemd-journald
      7 #7    3  0.0  0   241 root     /lib/systemd/systemd-udevd
      7 #7    1  0.0  0   454 root     /usr/sbin/ModemManager --filter-policy=strict
      7 #7    2  0.0  0   455 root     /usr/sbin/cron -f
      7 #7    0  0.0  0   456 avahi    avahi-daemon: running [porygon.local]
      7 #7    2  0.0  0   509 avahi     \_ avahi-daemon: chroot helper

                                      ...
        
      7 #7    3  0.0  0   465 root     /lib/systemd/systemd-logind
      7 #7    1  0.0  0   510 root     /usr/lib/policykit-1/polkitd --no-debug
      7 #7    1  0.0  0   511 root     /usr/sbin/alsactl -E HOME=/run/alsa -s -n 19 -c rdaemon
      7 #7    0  0.0  0   535 root     /usr/sbin/sshd -D
      7 #7    0  0.0  0   761 root      \_ sshd: hans [priv]
      7 #7    2  0.0  0   779 hans          \_ sshd: hans@pts/0
      7 #7    0  0.0  0   780 hans              \_ -bash
      7 #7    1  0.0  0   808 hans                  \_ tmux
            
                                      ...

      7 #7    2  0.0  0   810 hans     tmux
      7 #7    2  0.0  0   811 hans      \_ -bash
      7 #7    0  0.0  0  1001 hans      |   \_ ps -e --forest -o sched,policy,psr,pcpu,c,pid,user,cmd
      7 #7    0  0.0  0   822 hans      \_ -bash
      7 #7    0  0.0  0   970 hans          \_ ./myprogram
      7 #7    0 19.9 19   972 hans          |   \_ ./myprogram
      7 #7    0 19.8 19   973 hans          |   \_ ./myprogram
      7 #7    0 19.8 19   974 hans          |   \_ ./myprogram
      7 #7    0 19.9 19   975 hans          |   \_ ./myprogram
      7 #7    0 19.9 19   976 hans          |   \_ ./myprogram
      7 #7    1  0.0  0   977 hans          \_ ./myprogram
      7 #7    1 19.8 19   979 hans          |   \_ ./myprogram
      7 #7    1 19.8 19   980 hans          |   \_ ./myprogram
      7 #7    1 19.9 19   981 hans          |   \_ ./myprogram
      7 #7    1 19.9 19   982 hans          |   \_ ./myprogram
      7 #7    1 19.8 19   983 hans          |   \_ ./myprogram
      7 #7    2  0.0  0   985 hans          \_ ./myprogram
      7 #7    2 20.0 20   987 hans          |   \_ ./myprogram
      7 #7    2 20.0 20   988 hans          |   \_ ./myprogram
      7 #7    2 19.9 19   989 hans          |   \_ ./myprogram
      7 #7    2 19.8 19   990 hans          |   \_ ./myprogram
      7 #7    2 19.8 19   991 hans          |   \_ ./myprogram
      7 #7    3  0.0  0   992 hans          \_ ./myprogram
      7 #7    3 19.9 19   994 hans              \_ ./myprogram
      7 #7    3 20.0 20   995 hans              \_ ./myprogram
      7 #7    3 19.9 19   996 hans              \_ ./myprogram
      7 #7    3 19.9 19   997 hans              \_ ./myprogram
      7 #7    3 19.9 19   998 hans              \_ ./myprogram

### Tips

- Focus on making a stable system that can handle running multiple CPU-bound
    processes, distributed to all available CPUs.

- Refer to the debugging tips in Part 3.

- Note that the `ps` output above shows balanced usage stats in the "%CPU"
    column, as expected. If you can get the stats to display correctly, awesome.
    If not, and yours shows nonsensical numbers, don't worry about it.

### Deliverables

- Modifications to the Linux source tree setting the Freezer scheduler to be the
    default scheduler.

- Documentation of any testing you did to verify that your Freezer scheduler is
    indeed the default, is stable, and schedules processes according to
    specification. Write this in `README.txt`.


## Part 7: Freezer Load Balancing

Finally, let's add idle load balancing to Freezer. If a CPU is about to start
running the `idle_task` because there are no other tasks running on it, try to
move a task from another CPU to the current CPU. This helps us utilize as many
CPUs as possible at all times, speeding up performance.

Our load balancing policy is as follows:

-   Try to find the CPU with the highest number of running Freezer tasks and
    manually move a task from that CPU to the current CPU.

-   There are a few exceptions to the above rule:

    -   Do not steal a task from CPUs with fewer than 2 tasks.
    
    -   Make sure to respect the CPU affinity of a given task.

    -   Make sure to respect per-CPU kthreads. These should remain on their
        specified CPUs.

    -   Do not move tasks that are currently running on a CPU (obviously).

-   If there is no movable task on the busiest CPU based on the above criteria,
    then it's ok to not idle balance. Just give up.

### Tips

-   The relevant `sched_class` function for load balancing is `.balance`.
    Investigate how this is implemented and used in the CFS, RT, and DL
    schedulers.

-   You may find referencing the following function helpful:
    `can_migrate_task()`.

-   Avoid deadlocks! To move a task from one `rq` to another, you may need to
    manually grab a `rq` lock. Make sure you do this in a deadlock-free manner.

-   It's okay if the busiest CPU you found is no longer the busiest CPU when you
    are actually moving the task.

### Deliverables

- Modifications to the Linux source tree implementing load balancing for the Freezer 
    scheduler.

Good luck!

--------

### *Acknowledgments*

Sankalp Singayapally, a TA for COMS W4118 Operating Systems I, Spring 2015,
Columbia University, wrote the solution code for this assignment, based on his
work in Fall 2014 in collaboration with Nicolas Mesa and Di Ruan.

Mitchell Gouzenko and Akira Baruah, TAs in Spring 2016, updated the code for the
4.1.18 kernel.

Benjamin Hanser and Emily Meng, TAs in Spring 2017, updated the code for the
4.4.50 kernel.

John Hui, TA in Spring 2018, updated the code for the 64-bit 4.9.81 kernel, and
updated the BFS portion of the assignment to MuQSS.

Jonas Guan, TA in Spring 2019, updated the code for the 4.9.153 kernel.

Dave Dirnfeld and Hans Montero, TAs in Spring 2020, updated the assignment for
the 4.19.50 kernel.

Kent Hall, John Hui, Xijiao Li, Hans Montero, Akhil Ravipati, Maÿlis Whetsel,
and Tal Zussman, TAs in Fall 2021, updated the assignment for the 5.10.57
kernel.

Andy Cheng, Helen Chu, Phoebe Lu, Cynthia Zhang, and Tal Zussman, TAs in Spring
2023, updated the assignment for the 5.10.158 kernel and implemented idle load
balancing.

Brennan McManus, TA in Spring 2024, updated the code for the 5.10.205 kernel.

Ruizhe Fu and Nicholas Yap, TAs in Spring 2025, created and changed part 1, 4, 5, and updated code and 
solution to the 6.8 kernel.

--------
*Last updated: 2025-03-22*
