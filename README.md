# TTK4147 - Real-time Systems

# Exercise - BRTT and PREEMPT_RT

In this exercise we will do the same things as in the previous exercise, just
that we will be using PREEMPT_RT (<https://en.wikipedia.org/wiki/PREEMPT_RT>) in
the Linux kernel, which improves upon the real-time properties by making the
kernel preemptable. On September 20, 2024, PREEMPT_RT was fully merged and
enabled in mainline Linux on the supported architectures x86, x86_64, RISC-V and
ARM64.

## Compiling the kernel

The process generally involves cross-compiling the kernel, configuring boot
files, and replacing the existing kernel image on your Raspberry Pi.

In order to build the new kernel, you need to remove the micro-SD card from the
RPi and insert it into the SD-card adapter, which you connect to the PC. Start
Ubuntu on the desktop PC. The username and password are the same as previously,
`student` and `Sanntid15` respectively. If everything goes well, you will not
lose any files on the RPi.

Open a terminal in Ubuntu, and follow the steps below:

1. Install the required toolchain for cross-compilation:

```shell
sudo apt install gcc make gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu flex bison libssl-dev
```

2. Get the correct kernel source:

```shell
git clone https://github.com/raspberrypi/linux.git -b rpi-6.12.y --depth=1 linux-rpi4
```

3. Move into the folder with the kernel:

```shell
cd linux-rpi4
```

4. Set the variables for compilation:

```shell
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
```

5. Configure the kernel:

Use the Raspberry Pi default configuration for Raspberry Pi 4 (bcm2711).

```shell
make bcm2711_defconfig
```

6. Enable Real-Time Preemption:

```shell
make menuconfig
```

In the menu, navigate to: **General setup** -> **Preemption model** -> Change it to: **Fully Preemptible Kernel (Real-Time)**

1. Build the target image:

```shell
make -j$(nproc)
```

8. Mount the SD-card:

```shell
mkdir -p mnt/boot
sudo mount /dev/sda1 mnt/boot
```

If you get an error you can check where the SD-card is mounted with the command
`sudo fdisk -l` and look for the 16 GB USB disk.

9. Copy built files to the created directories:

```shell
sudo cp arch/arm64/boot/Image mnt/boot/kernel8.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb mnt/boot
sudo cp arch/arm64/boot/dts/overlays/*.dtb* mnt/boot/overlays/
sudo cp arch/arm64/boot/dts/overlays/README mnt/boot/overlays/
sync
```

10. Unmount directories:

```shell
sudo umount mnt/boot
```

Once the above process is done, you can eject the SD-card and insert the
micro-SD card back into the RPi. Once you turn the RPi on, it should boot as
normal. Since it might be difficult to spot that the PREEMPT_RT is active, we
recommend that you check the kernel version with the following command:
`uname -a`. Look for the RT tag in the kernel version string, indicating that
the Real-Time features are enabled.

## PREEMPT_RT performance

To fully utilize the PREEMPT_RT capabilities in your C code, you’ll need to make
some adjustments to ensure that your code is compatible with real-time
requirements. Here are key elements to consider:

1. Use Real-Time Scheduling Policies

If your application is multithreaded, use POSIX threads with real-time
attributes.

```c
pthread_t thread_task;
pthread_attr_t attr_task;
struct sched_param param_task;

pthread_attr_init(&attr_task);
pthread_attr_setschedpolicy(&attr_task, SCHED_FIFO);

param_task.sched_priority = 80;
pthread_attr_setschedparam(&attr_task, &param_task);
pthread_attr_setinheritsched(&attr_task, PTHREAD_EXPLICIT_SCHED);

if (pthread_create(&thread_task, &attr_task, task, NULL) != 0)
{
    perror("Failed to create real-time thread");
}
```

Here we are using `SCHED_FIFO`, which is appropriate for time-sensitive tasks,
but ensure you have some sleep or yield (for example, `pthread_yield()`) to
avoid monopolizing the CPU. We can also use `SCHED_RR`, which ensures each
thread gets a fair share of the CPU in a round-robin manner.

Here you can find a description of all scheduling policies: <https://wiki.linuxfoundation.org/realtime/documentation/technical_basics/sched_policy_prio/start>.

2. Use `mlockall()` to Avoid Swapping

Swapping can cause delays in real-time processes. Use `mlockall()` to lock your process's memory to prevent it from being swapped out.

```c
#include <sys/mman.h>

void lock_memory() {
    if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) {
        perror("Failed to lock memory");
    }
}
```

3. Avoid Blocking System Calls

Blocking system calls (e.g., `sleep()`, `read()`, and `write()` on certain
devices) can introduce delays in real-time tasks. Where possible:

- Use non-blocking I/O.
- For timing, use high-resolution timers with `clock_nanosleep()`.

```c
#include <time.h>

void sleep_realtime(long nanoseconds) {
    struct timespec ts;
    ts.tv_sec = nanoseconds / 1e9;
    ts.tv_nsec = nanoseconds % (long)1e9;
    clock_nanosleep(CLOCK_MONOTONIC, 0, &ts, NULL);
}
```

### With and without PREEMPT_RT

You can apply many of these techniques without the PREEMPT_RT, but your system
won't achieve the same level of real-time performance as with PREEMPT_RT
enabled. Here's how some techniques behave on a standard kernel compared to a
PREEMPT_RT kernel:

1. Real-Time Scheduling Policies

- Without PREEMPT_RT: The standard Linux kernel supports SCHED_FIFO and
  SCHED_RR, but preemption can still be delayed by high-priority kernel tasks
  and interrupts.
- With PREEMPT_RT: The kernel allows almost full preemption, so user-space
  real-time threads have lower latency and better response times.

2. CPU Affinity

- Without PREEMPT_RT: You can set CPU affinity to bind threads to specific
  cores, which helps reduce interference from other processes. This works
  similarly on both standard and real-time kernels.
- With PREEMPT_RT: CPU affinity works the same, but the benefits of reduced
  latency are enhanced due to lower overall system jitter.

Using these techniques on a standard kernel can improve responsiveness and
latency somewhat, but without PREEMPT_RT, there are limitations due to
unavoidable kernel latencies and non-preemptible kernel code paths. For best
real-time performance, especially in applications with strict timing
requirements, PREEMPT_RT is highly recommended.

If upgrading to PREEMPT_RT is not possible, however, these techniques can still
offer a degree of improved timing and predictability, particularly with careful
system tuning (like isolating CPUs, reducing system load, and controlling
interrupts).

## Tasks

If you run the code from the previous exercise for each task, you will most
likely get approximately the same results. The default scheduler in Linux is the
Completely Fair Scheduler (CFS), associated with the `SCHED_OTHER` policy. Here
we will try to improve our tasks performance by using real-time scheduling
policies.

Real-time tasks, using `SCHED_FIFO` or `SCHED_RR`, will still take precedence
over normal tasks scheduled by CFS. This means that if real-time tasks are ready
to run, they will preempt non-real-time tasks, but CFS will continue to manage
the execution of non-real-time tasks in a fair manner.

### Task A

In this task, you will modify the multithreaded C code to take advantage of
real-time scheduling policies with the PREEMPT_RT kernel. We should use only one
CPU core to demonstrate this behaviour.

- Modify the task threads to use `SCHED_RR` with a high priority (80) to ensure
  real-time behavior for these threads.
- Modify the disturbance threads to use `SCHED_RR`, giving them a lower priority
  (40).
- Verify that task threads are not interrupted by disturbance threads.

### Task B

We can observe that with the current setup disturbance threads would never get
their turn (unless task threads sleep for a certain amount of time).

- Modify the code so that disturbance threads run, but without significantly
  degrading our response time measured with BRTT. Note down the different ways
  you have tried.

### Task C

In this task, you will enhance your real-time program to prevent memory swapping
and ensure memory availability for your real-time tasks.

- Use `mlockall()` to lock memory and prevent the system from swapping out your
  task's memory.
- Observe how memory locking and the elimination of blocking calls affect the
  timing of your tasks.
- Introduce some modification that will help demonstrate the effects of locking
  memory versus not locking it. (Reserving a memory block in your code and then
  `stress --vm 1 --vm-bytes 7700M --timeout 100`).
