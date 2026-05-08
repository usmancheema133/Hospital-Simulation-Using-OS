# Hospital Patient Triage & Bed Allocator

**CL2006 – Operating Systems Lab | Spring 2026 | FAST-NUCES, CFD Campus**

---

## Overview

A fully functional system-level C program that simulates a hospital emergency room by applying core Operating Systems concepts. Patients arrive, get triaged, and are assigned to beds based on priority. The system handles concurrent arrivals, enforces priority-based scheduling, prevents race conditions on shared resources, and manages bed memory using configurable allocation strategies.

This project was built as part of the CL2006 Operating Systems Lab semester project, covering:

- **Process Management** — `fork()` / `execv()` per patient, `SIGCHLD` zombie reaping
- **IPC** — Anonymous pipes, Named FIFO (`/tmp/discharge_fifo`), Shared Memory (`shmget` / `shmat`)
- **CPU Scheduling** — Priority Queue, FCFS, SJF, Priority Scheduling, Round Robin
- **Synchronization** — Mutex, Condition Variables, Counting Semaphores, Producer-Consumer
- **Memory Management** — Best-Fit / First-Fit / Worst-Fit, Coalescing, Fragmentation Reporting, Paging Simulation

---

## Project Structure

```
hospital_project/
├── src/
│   ├── hospital.h              # Shared structs, constants, enums (PatientRecord, BedPartition)
│   ├── admissions.c            # Central manager: threads, IPC, scheduling, bed allocation
│   ├── patient_simulator.c     # Child process per patient: simulates treatment, sends discharge
│   ├── bed_allocator.c         # Best/First/Worst-Fit allocator, coalescing, paging, fragmentation
│   ├── bed_allocator.h
│   ├── scheduler.c             # Priority queue + FCFS, SJF, Priority, Round Robin simulations
│   └── scheduler.h
├── scripts/
│   ├── triage.sh               # Validates input, computes triage priority, pipes to admissions
│   ├── start_hospital.sh       # Initializes IPC resources, launches admissions in background
│   ├── stop_hospital.sh        # Sends SIGTERM, cleans up all IPC, prints final summary
│   └── stress_test.sh          # Spawns 20 rapid patient arrivals for load testing
├── logs/                       # Auto-created at runtime
│   ├── schedule_log.txt        # Gantt charts + metrics for all 4 scheduling algorithms
│   └── memory_log.txt          # Timestamped fragmentation stats per allocation/deallocation
├── Makefile
└── README.md
```

---

## Prerequisites

Ubuntu 22.04 / 24.04 (or any Linux with GCC and POSIX support):

```bash
sudo apt update
sudo apt install gcc make valgrind -y
```

No external libraries are required. Only standard POSIX/glibc is used.

---

## Build

```bash
cd hospital_project
make all
```

Expected output:

```
gcc -Wall -Wextra -pthread -g -o admissions src/admissions.c src/bed_allocator.c src/scheduler.c -lpthread
✔  admissions built
gcc -Wall -Wextra -pthread -g -o patient_simulator src/patient_simulator.c -lpthread
✔  patient_simulator built
✔  Build complete.
```

---

## How to Run

### Option 1 — Interactive Mode (Best-Fit, recommended for demo)

```bash
make run
```

Then type one patient per line in the format `<name> <age> <severity>` and press Enter:

```
Alice 30 2
Bob 45 9
Carol 22 5
quit
```

### Option 2 — Scripted 5-Patient Test

```bash
make test
```

Automatically pipes Alice, Bob, Carol, David, and Eve into the system and prints the schedule and memory logs.

### Option 3 — 20-Patient Stress Test

```bash
make stress
```

Runs `scripts/stress_test.sh` which floods the system with 20 rapid arrivals to test concurrent process handling and semaphore blocking.

### Option 4 — Choose Allocation Strategy at Runtime

```bash
./admissions --strategy best     # Best-Fit  (default)
./admissions --strategy first    # First-Fit
./admissions --strategy worst    # Worst-Fit
```

### Option 5 — Using triage.sh (with full input validation)

```bash
chmod +x scripts/triage.sh
./scripts/triage.sh Alice 30 2 | ./admissions --strategy best
```

### Option 6 — Valgrind Memory Check

```bash
make valgrind
```

Report is saved to `logs/valgrind_output.txt`.

---

## Input Format

```
<name> <age> <severity>
```

| Field    | Type    | Description                         |
|----------|---------|-------------------------------------|
| name     | string  | Patient name (no spaces)            |
| age      | integer | Range: 0 – 150                      |
| severity | integer | Range: 1 – 10 (10 = most critical)  |

### Severity to Priority Mapping

| Severity | Priority | Label       | Bed Assigned |
|----------|----------|-------------|--------------|
| 1 – 2    | 1        | CRITICAL    | ICU          |
| 3 – 4    | 2        | URGENT      | ICU          |
| 5 – 6    | 3        | MODERATE    | Isolation    |
| 7 – 8    | 4        | MINOR       | General Ward |
| 9 – 10   | 5        | NON-URGENT  | General Ward |

---

## Ward Configuration

| Bed Type     | Count | Care Units Each | Total Units |
|--------------|-------|-----------------|-------------|
| ICU          | 4     | 3               | 12          |
| Isolation    | 4     | 2               | 8           |
| General Ward | 12    | 1               | 12          |
| **TOTAL**    | **20**| —               | **32**      |

---

## Sample Output

```
╔══════════════════════════════════════════╗
║   Hospital Patient Triage & Bed Alloc   ║
║   Allocation Strategy: Best-Fit         ║
╚══════════════════════════════════════════╝

[WARD] Initialized: 4 ICU, 4 Isolation, 12 General beds (32 total care units)
[RECEPTIONIST] Patient 1: Alice | Age:30 | Sev:2 → Priority:1 (CRITICAL)
[PQ] Enqueued patient 1 (Alice) priority=1 (CRITICAL) | Queue size: 1
[SCHEDULER] Waiting for ICU semaphore for patient 1...
[ALLOC] Patient 1 (Alice) → ICU bed (partition 0, start=0, size=3)
[FRAG] Total free: 29 units | Largest block: 3 | External frag: 89.7%
[ADMISSIONS] Spawned patient_simulator PID=12345 for patient 1
[PATIENT 1] ► ARRIVED   | Triage P1 | Bed 0 (ICU) | PID=12345
[PATIENT 1] ► TREATMENT START | Duration: 9 s
[PATIENT 1] ► DISCHARGED | Treatment complete (9 s)
[DISCHARGE] Patient 1 discharged — freeing bed
[COALESCE] Before: Partition 0 [ICU] start=0 size=3 (OCCUPIED by P1)
[COALESCE] After: ward freed for patient 1
[FRAG] Total free: 32 units | Largest block: 12 | External frag: 0.0%
```

### Schedule Log Sample (`logs/schedule_log.txt`)

```
=== FCFS SCHEDULING SIMULATION ===
ID    Name       Burst   Start   Waiting   Turnaround
1     Alice      13      0       0.0       13.0
  Gantt: |P1-------------| [0-13]
4     David      10      13      13.0      23.0
  Gantt: |P4----------| [13-23]
Avg Waiting Time   : 18.80 s
Avg Turnaround Time: 26.20 s

=== SJF SCHEDULING SIMULATION ===
...
=== ROUND ROBIN SCHEDULING (quantum=3) ===
|P1---|P4---|P3---|P5---|P2---|P1---|P4---|...
```

### Memory Log Sample (`logs/memory_log.txt`)

```
[2026-05-08 12:24:56] Free=29 Largest=3 ExtFrag=89.7%
[2026-05-08 12:24:56] Free=26 Largest=3 ExtFrag=88.5%
[2026-05-08 12:24:58] Free=23 Largest=3 ExtFrag=87.0%
```

---

## OS Concepts Demonstrated

| Concept                         | Location in Code                                  |
|---------------------------------|---------------------------------------------------|
| `fork()` + `execv()`            | `admissions.c` → `spawn_patient_process()`        |
| `SIGCHLD` + `waitpid(WNOHANG)`  | `admissions.c` → `sigchld_handler()`              |
| Anonymous Pipe                  | `triage.sh` stdout → `admissions` stdin           |
| Named FIFO                      | `patient_simulator.c` → `discharge_listener` thread |
| Shared Memory (`shmget/shmat`)  | `admissions.c` → `SharedMemory` struct            |
| Priority Queue (min-heap)       | `scheduler.c` → `pq_enqueue()` / `pq_dequeue()`  |
| POSIX Threads (6 threads)       | Receptionist, Scheduler, Nurse×3, Discharge Listener |
| `pthread_mutex_t`               | `g_bed_mutex`, `g_pq_mutex`                       |
| `pthread_cond_t`                | `g_bed_freed`, `g_pq_cond`                        |
| Counting Semaphores             | ICU/Isolation capacity limits + producer-consumer queue |
| Best-Fit Allocator              | `bed_allocator.c` → `allocate_bed()`              |
| Coalescing                      | `bed_allocator.c` → `free_bed()`                  |
| Fragmentation Reporting         | `bed_allocator.c` → `log_fragmentation()`         |
| Paging Simulation               | `bed_allocator.c` → page table in `SharedMemory`  |

---

## Makefile Targets

| Target     | Description                                         |
|------------|-----------------------------------------------------|
| `make all` | Compile all binaries with `-Wall -Wextra -pthread`  |
| `make run` | Build and launch interactive Best-Fit session       |
| `make test`| Run scripted 5-patient test and print logs          |
| `make stress` | Run 20-patient rapid stress test                 |
| `make valgrind` | Run with Valgrind full leak check             |
| `make clean` | Remove binaries, logs, and FIFO                  |

---

## Clean Up

```bash
make clean
```

Or to stop a running instance:

```bash
./scripts/stop_hospital.sh
```

This sends SIGTERM to the admissions process, kills all patient simulators, removes shared memory, unlinks semaphores, and deletes the discharge FIFO.

---

## Personal Contribution

**[Usman Zafar] (23F-0699):**
- Implemented Phase 1: triage.sh, start_hospital.sh, stop_hospital.sh, Makefile
- Implemented Phase 2A: fork/execv spawning, SIGCHLD handler, patient_simulator lifecycle
- Implemented Phase 4: Best-Fit/First-Fit/Worst-Fit allocator, coalescing, fragmentation logging, paging simulation

**[Ali Afzal] (23F-0594):**
- Implemented Phase 2B: Anonymous pipe, Named FIFO, Shared Memory IPC
- Implemented Phase 2C: Priority queue, FCFS/SJF/Priority/RR scheduling simulations
- Implemented Phase 3: All 6 POSIX threads, mutexes, condition variables, semaphores

---

## Academic Integrity

All code in this repository was written by the group members listed above. AI tools were used only to understand concepts, not to generate entire functions or modules.
