#### `README.md`

**1. Team Information**

- Vishwas Gowda S -- PES1UG24CS539
- Vikas Vittal Aihole -- PES1UG24CS534

**2. Build, Load, and Run Instructions**

- Step-by-step commands to build the project, load the kernel module, and start the supervisor
- How to launch containers, run workloads, and use the CLI
- How to unload the module and clean up
- These must be complete enough that we can reproduce your setup from scratch on a fresh Ubuntu 22.04/24.04 VM

The following is a reference run sequence you can use as a starting point:

```bash
# Build
make

# Load kernel module
sudo insmod monitor.ko

# Verify control device
ls -l /dev/container_monitor

# Start supervisor
sudo ./engine supervisor ./rootfs-base

# Create per-container writable rootfs copies
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta

# In another terminal: start two containers
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta ./rootfs-beta /bin/sh --soft-mib 64 --hard-mib 96

# List tracked containers
sudo ./engine ps

# Inspect one container
sudo ./engine logs alpha

# Run memory test inside a container
# (copy the test program into rootfs before launch if needed)

# Run scheduling experiment workloads
# and compare observed behavior

# Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta

# Stop supervisor if your design keeps it separate

# Inspect kernel logs
dmesg | tail

# Unload module
sudo rmmod monitor
```

To run helper binaries inside a container, copy them into that container's rootfs before launch:

```bash
cp workload_binary ./rootfs-alpha/
```

### `Reset Before Item 7 and Item 8`

Run these once before capturing the new Item 7 and Item 8 screenshots.

```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate

# Stop any stale supervisor process and clear stale socket
sudo pkill -f "./engine supervisor ./rootfs-base" || true
sudo rm -f /tmp/mini_runtime.sock

# Reset monitor module state
sudo rmmod monitor 2>/dev/null || true
sudo insmod monitor.ko

# Rebuild workloads and ensure they are runnable in container rootfs trees
make cpu_hog io_pulse
cp ./cpu_hog ./rootfs-alpha/cpu_hog
cp ./cpu_hog ./rootfs-beta/cpu_hog
cp ./io_pulse ./rootfs-alpha/io_pulse
cp ./io_pulse ./rootfs-beta/io_pulse
chmod +x ./rootfs-alpha/cpu_hog ./rootfs-beta/cpu_hog
chmod +x ./rootfs-alpha/io_pulse ./rootfs-beta/io_pulse
```




### `TASK 1`

**1. Running the supervisor**
![Task 1 Screenshot Supervisior](assets/supervisor_task1.png)

**2. Running/Stopping Alpha and Beta**
![Task 1 Screenshot Alpha Beta](assets/alpha_beta_task1.png)

### `TASK 2`

**1. Running the supervisor**
![Task 2 Screenshot Supervisor](assets/supervisor_task2.png)

**2. Running/Stopping Alpha and Beta**
![Task 2 Screenshot Alpha Beta](assets/alpha_beta_task2.png)

### `TASK 3 - Bounded-Buffer Logging and IPC Design`

This task is implemented in the supervisor logging path in `engine.c`.

#### Logging IPC path (Path A)

- For each started container, the supervisor creates a pipe using `pipe(pipefd)`.
- The child process redirects both `stdout` and `stderr` to the pipe write-end via:
	- `dup2(cfg->log_write_fd, STDOUT_FILENO)`
	- `dup2(cfg->log_write_fd, STDERR_FILENO)`
- A per-container producer thread reads from the pipe read-end and pushes log chunks into a shared bounded buffer.

#### Producer-consumer pipeline

- Producer side:
	- `log_producer_thread()` reads bytes from the container pipe.
	- Each read chunk is wrapped into a `log_item_t` and enqueued with `bounded_buffer_push()`.
- Consumer side:
	- `logging_thread()` dequeues with `bounded_buffer_pop()`.
	- It resolves the container log path and appends data to `logs/<container_id>.log`.

#### Bounded shared buffer

- Data structure: fixed-size ring buffer (`LOG_BUFFER_CAPACITY`) with `head`, `tail`, and `count`.
- Synchronization primitives:
	- `pthread_mutex_t` protects all ring-buffer state transitions.
	- `pthread_cond_t not_full` blocks producers when full.
	- `pthread_cond_t not_empty` blocks consumer when empty.
- This prevents data races and ensures bounded memory usage under high output rates.

#### Why these synchronization primitives

- `mutex` gives mutual exclusion for ring buffer indices and count.
- `condition variables` provide blocking wait/signal semantics without busy-waiting.
- Separate lock domains are used:
	- `log_buffer.mutex` only for queue operations.
	- `metadata_lock` only for container metadata/list lookups.
- Keeping these separate avoids lock contention and lock-order deadlocks between log traffic and control-plane metadata updates.

#### Race conditions without synchronization

- Without a queue mutex:
	- concurrent producers/consumer can corrupt `head/tail/count`.
	- entries can be overwritten or lost.
- Without `not_full` and `not_empty`:
	- producers would spin or overwrite when full.
	- consumer would spin aggressively when empty.
- Without separate metadata synchronization:
	- logger could read partially-updated container metadata/log-path values while control threads modify records.

#### Correctness and shutdown behavior

- Persistent per-container logs:
	- log files are created before container launch (`logs/<id>.log`), then opened in append mode by the consumer.
- No log loss on abrupt container exit:
	- producer keeps reading pipe until EOF and enqueues every chunk before exit.
	- supervisor joins producer threads after container reaping.
- Full-buffer behavior without deadlock:
	- producer blocks on `not_full`; consumer drains and signals `not_full`.
	- no circular wait because queue lock and metadata lock are not held together in opposite order.
- Clean logger shutdown:
	- `join_remaining_producers()` waits for producer exit.
	- `bounded_buffer_begin_shutdown()` broadcasts termination to queue waiters.
	- consumer flushes remaining items, exits only when queue is empty and shutdown is set.
	- logger thread is joinable and joined during supervisor shutdown.

#### Quick validation commands

Run in one terminal:

```bash
sudo ./engine supervisor ./rootfs-base
```

Run in another terminal:

```bash
sudo ./engine start alpha ./rootfs-alpha "sh -c 'echo out; echo err 1>&2; sleep 1'"
sudo ./engine start beta ./rootfs-beta "sh -c 'for i in 1 2 3 4 5; do echo beta-$i; done'"
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine logs beta
```

Stress test for bounded buffer behavior:

```bash
sudo ./engine run gamma ./rootfs-alpha "sh -c 'i=0; while [ $i -lt 20000 ]; do echo line-$i; i=$((i+1)); done'"
sudo ./engine logs gamma
```

Expected observations:

- both `stdout` and `stderr` lines appear in per-container files.
- logs are available under `logs/*.log`.
- supervisor remains responsive while containers log concurrently.
- on stop/exit, producer and logger threads terminate cleanly.

![Task 3 Screenshot supervisor](assets/supervisor_task3.png)

![Task 3 Screenshot Alpha Beta](assets/alpha_beta_task3.png)

### `TASK 4 - Kernel Memory Monitoring with Soft and Hard Limits`

This task is implemented in the kernel module `monitor.c` and integrated with the supervisor in `engine.c`.

#### What is implemented

- Control device at `/dev/container_monitor` via character-device registration.
- PID registration from supervisor using ioctl:
	- `MONITOR_REGISTER`
	- `MONITOR_UNREGISTER`
- Kernel tracked-process table as a linked list (`struct monitored_entry`).
- Lock-protected shared access to list state (spinlock).
- Periodic RSS checks from timer callback.
- Two-threshold policy:
	- soft limit warning (first crossing only)
	- hard limit kill (SIGKILL)
- Automatic stale-entry cleanup when a process exits.

#### Soft-limit and hard-limit policy

- Soft limit:
	- on first RSS crossing, kernel logs warning event:
		- `[container_monitor] SOFT LIMIT ...`
	- warning is emitted once per tracked entry (`soft_limit_reported` flag).
- Hard limit:
	- if RSS exceeds hard limit, kernel sends SIGKILL.
	- hard-limit event is logged:
		- `[container_monitor] HARD LIMIT ...`
	- entry is removed from tracking list after enforcement.

#### Integration with supervisor metadata

- Supervisor registers host PID and limits after successful container launch.
- Supervisor unregisters on exit/reap path and explicit cleanup paths.
- Termination attribution in metadata follows grading rule:
	- `stop_requested = 1` before supervisor stop flow signals container.
	- if process exits by signal and `stop_requested` is set: state becomes `stopped`.
	- if process exits with `SIGKILL` and `stop_requested` is not set: state becomes `hard_limit_killed`.
	- otherwise signal-based exit is `killed`.
- `ps` output shows final state in `STATE` column (`exited`, `stopped`, `killed`, `hard_limit_killed`).

#### Why spinlock was used

- The monitor list is accessed from:
	- ioctl context (register/unregister)
	- timer callback context (periodic RSS scan)
- Timer callback must remain non-blocking and cannot use sleeping lock paths.
- A spinlock protects short critical sections for list insert/remove/iteration safely across both contexts.

#### Validation commands

Build and load:

```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate
make monitor.ko memory_hog
lsmod | grep '^monitor' || sudo insmod monitor.ko
ls -l /dev/container_monitor

# Ensure memory workload is available inside container rootfs trees
cp ./memory_hog ./rootfs-alpha/memory_hog
cp ./memory_hog ./rootfs-beta/memory_hog
chmod +x ./rootfs-alpha/memory_hog ./rootfs-beta/memory_hog
```

Start supervisor:

```bash
sudo ./engine supervisor ./rootfs-base
```

In another terminal, capture Item 5 (soft-limit warning):

```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate

SOFT_ID="soft_$(date +%H%M%S)"
sudo ./engine start "$SOFT_ID" ./rootfs-alpha "/memory_hog 8 300" --soft-mib 24 --hard-mib 1024

# Capture kernel warning and metadata for screenshot
sudo dmesg -T | grep -E "container_monitor|SOFT LIMIT" | tail -n 60
sudo ./engine ps | grep "$SOFT_ID"

# Cleanup the soft-limit test container after capture
sudo ./engine stop "$SOFT_ID"
```

Capture Item 6 (hard-limit enforcement):


```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate

HARD_ID="hard_$(date +%H%M%S)"
sudo ./engine run "$HARD_ID" ./rootfs-beta "/memory_hog 8 200" --soft-mib 24 --hard-mib 40

# Capture hard-limit kernel event and supervisor final state for screenshot
sudo dmesg -T | grep -E "container_monitor|SOFT LIMIT|HARD LIMIT" | tail -n 80
sudo ./engine ps | grep "$HARD_ID"
```

Expected observations:

- Item 5 screenshot: kernel log includes `SOFT LIMIT` for `SOFT_ID`, and metadata shows container tracked/running before manual stop.
- Item 6 screenshot: kernel log includes `HARD LIMIT` for `HARD_ID`, and `engine ps` shows `STATE=hard_limit_killed`.

Manual stop attribution check:

```bash
sudo ./engine start stop1 ./rootfs-beta "sleep 120" --soft-mib 32 --hard-mib 48
sudo ./engine stop stop1
sudo ./engine ps
```

Expected:

- state is `stopped` (not `hard_limit_killed`) because stop flow sets `stop_requested` before signaling.

**Screenshot 5 Caption:** Terminal capture for the soft-limit test workflow, including `dmesg` filter output and container metadata lookup.

![Item 5 - Soft-limit warning](assets/task4_soft_limit_terminal.png)

**Screenshot 6 Caption:** Terminal capture for the hard-limit test workflow and supervisor metadata lookup for the test container.

![Item 6 - Hard-limit enforcement](assets/task4_hard_limit_terminal.png)


Cleanup:

```bash
sudo rmmod monitor
```

Optional cleanup screenshot (not part of Item 5/6 proof):

![Task 4 Cleanup](assets/cleanup_task4.png)

### `TASK 5 - Scheduler Experiments and Analysis`

This task uses the runtime as an experiment platform to observe Linux scheduling behavior under concurrent container workloads.

#### Workloads used

- CPU-bound: `/cpu_hog 20`
- I/O-bound: `/io_pulse 80 100`

#### Scheduling configurations used

- Configuration A: same priority (`--nice 0`) for concurrent CPU-bound and I/O-bound containers.
- Configuration B: different priorities for concurrent CPU-bound containers:
	- high priority: `--nice -5`
	- low priority: `--nice 10`
- To force visible contention, the supervisor is pinned to CPU 0 during the experiment run.

#### How to run 

Terminal 1 (keep this running):

```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate
sudo ./engine supervisor ./rootfs-base
```

Terminal 2: setup for experiments

```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate
sudo -v

# Build workloads first
make cpu_hog io_pulse

# Copy workload binaries into both container rootfs trees
cp ./cpu_hog ./rootfs-alpha/cpu_hog
cp ./cpu_hog ./rootfs-beta/cpu_hog
cp ./io_pulse ./rootfs-alpha/io_pulse
cp ./io_pulse ./rootfs-beta/io_pulse
chmod +x ./rootfs-alpha/cpu_hog ./rootfs-beta/cpu_hog
chmod +x ./rootfs-alpha/io_pulse ./rootfs-beta/io_pulse

# Optional: pin supervisor to one core to make contention effects clearer
SUP_PID=$(pgrep -fo "./engine supervisor ./rootfs-base")
sudo taskset -pc 0 "$SUP_PID"
```

#### Experiment 1: CPU-bound + I/O-bound at same priority

```bash
ID_E1_CPU="e1cpu_$(date +%H%M%S)"
ID_E1_IO="e1io_$(date +%H%M%S)"

sudo /usr/bin/time -f "elapsed_seconds=%e" -o "${ID_E1_CPU}.time" \
	./engine run "$ID_E1_CPU" ./rootfs-alpha "/cpu_hog 20" --nice 0 > "${ID_E1_CPU}.out" 2>&1 &
P1=$!

sudo /usr/bin/time -f "elapsed_seconds=%e" -o "${ID_E1_IO}.time" \
	./engine run "$ID_E1_IO" ./rootfs-beta "/io_pulse 80 100" --nice 0 > "${ID_E1_IO}.out" 2>&1 &
P2=$!

wait "$P1" "$P2"

sudo ./engine logs "$ID_E1_CPU" > "${ID_E1_CPU}.log" 2>&1
sudo ./engine logs "$ID_E1_IO" > "${ID_E1_IO}.log" 2>&1
```

#### Experiment 2: CPU-bound + CPU-bound with different priorities

```bash
ID_E2_HI="e2hi_$(date +%H%M%S)"
ID_E2_LO="e2lo_$(date +%H%M%S)"

sudo /usr/bin/time -f "elapsed_seconds=%e" -o "${ID_E2_HI}.time" \
	./engine run "$ID_E2_HI" ./rootfs-alpha "/cpu_hog 20" --nice -5 > "${ID_E2_HI}.out" 2>&1 &
P3=$!

sudo /usr/bin/time -f "elapsed_seconds=%e" -o "${ID_E2_LO}.time" \
	./engine run "$ID_E2_LO" ./rootfs-beta "/cpu_hog 20" --nice 10 > "${ID_E2_LO}.out" 2>&1 &
P4=$!

wait "$P3" "$P4"

sudo ./engine logs "$ID_E2_HI" > "${ID_E2_HI}.log" 2>&1
sudo ./engine logs "$ID_E2_LO" > "${ID_E2_LO}.log" 2>&1
```

#### Extract measurements

```bash
echo "Experiment 1 times (seconds):"
awk -F= '/elapsed_seconds=/{print FILENAME": "$2}' "${ID_E1_CPU}.time" "${ID_E1_IO}.time"

echo "Experiment 2 times (seconds):"
awk -F= '/elapsed_seconds=/{print FILENAME": "$2}' "${ID_E2_HI}.time" "${ID_E2_LO}.time"

echo "Experiment 2 CPU accumulators:"
grep -Eo 'accumulator=[0-9]+' "${ID_E2_HI}.log" | tail -n1
grep -Eo 'accumulator=[0-9]+' "${ID_E2_LO}.log" | tail -n1

echo "Final run status lines:"
grep -E 'run complete:' "${ID_E1_CPU}.out" "${ID_E1_IO}.out" "${ID_E2_HI}.out" "${ID_E2_LO}.out"
```

#### Measured outcomes

- Completion time per container (seconds).
- Final runtime status reported by `engine run`.
- CPU-share proxy for CPU-bound tasks: final `accumulator=` value from `cpu_hog` logs.


**Screenshot 7 Caption:** Scheduler experiment setup and command execution capture (workload preparation and experiment launch).

![Item 7 - Scheduling experiment measurements](assets/task5_setup_and_launch.png)

![Item 7 - Scheduling experiment status output](assets/task5_results_summary.png)

**Screenshot 7 (Results) Caption:** Elapsed-time, accumulator extraction, and run-status summary output captured after both experiments.

#### Short scheduler analysis

- In Configuration A (CPU + I/O, same nice), the I/O-bound task frequently sleeps and wakes; under CFS it tends to stay responsive because waking tasks have smaller virtual runtime than continuously running CPU hogs.
- In Configuration B (CPU + CPU, different nice), the higher-priority (`nice -5`) container should receive a larger CPU share than the lower-priority (`nice 10`) container. In fixed wall time, this appears as a significantly larger `accumulator` value for the higher-priority workload.
- Even with priority differences, both tasks continue to make progress, matching CFS fairness semantics (weighted, not strict starvation).


### `TASK 6 - Resource Cleanup Verification`

This task validates that cleanup paths from Tasks 1-4 work end-to-end in both user and kernel space.

#### Teardown verification flow

Terminal 1 (supervisor):

```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate
sudo ./engine supervisor ./rootfs-base
```

Terminal 2 (generate container activity):

```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate

# Ensure workload binaries exist and are container-runnable
make cpu_hog io_pulse

# Copy workloads into container rootfs trees
cp ./cpu_hog ./rootfs-alpha/cpu_hog
cp ./cpu_hog ./rootfs-beta/cpu_hog
cp ./io_pulse ./rootfs-alpha/io_pulse
cp ./io_pulse ./rootfs-beta/io_pulse
chmod +x ./rootfs-alpha/cpu_hog ./rootfs-beta/cpu_hog
chmod +x ./rootfs-alpha/io_pulse ./rootfs-beta/io_pulse

# Use unique container IDs on each run to avoid "container id already exists"
TS=$(date +%H%M%S)
T6CPU_ID="t6cpu_${TS}"
T6IO_ID="t6io_${TS}"
T6STOP_ID="t6stop_${TS}"

# Run short-lived workloads to exercise start/run/stop/reap and logging paths
sudo ./engine run "$T6CPU_ID" ./rootfs-alpha "/cpu_hog 10" --nice 0
sudo ./engine run "$T6IO_ID"  ./rootfs-beta  "/io_pulse 30 100" --nice 0

# Start one long task and stop it manually (tests stop_requested path + reap)
sudo ./engine start "$T6STOP_ID" ./rootfs-beta "sleep 30"
sudo ./engine stop "$T6STOP_ID"

# Read logs to confirm logger consumed queued entries
sudo ./engine logs "$T6CPU_ID"
sudo ./engine logs "$T6IO_ID"

# Inspect container states before supervisor shutdown
sudo ./engine ps
```

Stop supervisor cleanly in Terminal 1 using `Ctrl+C`.

For screenshot evidence, keep Terminal 1 visible after pressing `Ctrl+C` and capture supervisor shutdown output.

#### Post-shutdown cleanup checks

Run in Terminal 2:

```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate

echo "[1] No lingering supervisor process"
pgrep -af "engine supervisor" || echo "ok: no supervisor process"

echo "[2] No lingering zombies"
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/ {print}' || true

echo "[3] No stale control socket user"
ss -xl | grep mini_runtime.sock || echo "ok: no active mini_runtime.sock listener"

echo "[4] No stale monitor device users"
sudo lsof /dev/container_monitor || echo "ok: no open /dev/container_monitor handles"

echo "[5] Module unload/reload succeeds (kernel list cleanup check)"
sudo rmmod monitor
lsmod | grep '^monitor' || echo "ok: monitor unloaded"
sudo insmod monitor.ko
lsmod | grep '^monitor'
```

Run this extra check to print an explicit zombie summary line in the same terminal:

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/ {print; found=1} END {if(!found) print "ok: no zombies"}'
```

#### Stale-metadata check

After supervisor shutdown and restart, tracked runtime metadata should not persist from the previous run:

```bash
sudo ./engine supervisor ./rootfs-base
```

In another terminal:

```bash
cd /home/maddox/Desktop/OS-Jackfruit/boilerplate
sudo ./engine ps
```

Expected: `No containers tracked.`

**Screenshot 8 Caption:** Teardown workflow capture showing pre-shutdown container state, post-shutdown cleanup diagnostics, and final no-zombie/no-container verification.

![Item 8 - Teardown checks](assets/task6_activity_and_ps.png)

![Item 8 - Post-shutdown verification](assets/task6_cleanup_checks.png)

![Item 8 - Fresh restart metadata cleared](assets/task6_no_zombies_and_empty_ps.png)

#### Mapping to required properties

- Child process reap in supervisor: validated by completed run/stop operations and no zombie entries after shutdown.
- Logging threads exit/join: validated by successful supervisor termination after log-heavy runs and readable final logs.
- File descriptors closed on all paths: validated by no active listeners/holders for `mini_runtime.sock` and `/dev/container_monitor` after teardown.
- User-space heap resources released: validated operationally by repeated clean shutdown/start cycles without process residue.
- Kernel list entries freed on module unload: validated by successful `rmmod` followed by `insmod` without stale in-use errors.
- No lingering zombie processes or stale metadata: validated by zombie check + fresh `ps` output after restart.


