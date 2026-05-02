# Mini Task Scheduler
### Computer Organization & Assembly Language — Course Project

**Authors:** Juwairiya · Muskan · Parwin · Maryam  
**Language:** x86 Assembly (MASM syntax)  
**Assembler:** JWasm + JWlink via MASM Runner (VS Code)  
**Library:** Irvine32  

---

## Project Overview

Mini Task Scheduler is an x86 assembly program that simulates how an operating system manages multiple processes using a **round-robin CPU scheduling algorithm**. The program maintains four independent tasks, gives each one a turn to execute, tracks their state using a Task Control Block (TCB) array, and displays a live color-coded dashboard showing the status of every task in real time.

The project demonstrates core OS and assembly concepts including process scheduling, Task Control Blocks, stack memory management, status-driven execution, software timer simulation, and low-level register manipulation — all implemented from scratch in assembly language without any high-level language abstractions.

---

## How to Run

1. Open `scheduler_base.asm` in VS Code
2. Make sure the **MASM Runner** extension is installed
3. Click inside the file to make it the active tab
4. Press `Ctrl+Alt+R` to assemble and run
5. The program runs automatically — no user input required
6. Press `Ctrl+C` to stop early, or let it run until all tasks complete

---

## Program Flow

```
Launch
  │
  ▼
Print "Initializing tasks..."
  │
  ▼
INIT_TASKS — fills TCB array with stack pointers and READY status
  │
  ▼
Print "Starting scheduler..."
  │
  ▼
START_SCHEDULER — round-robin loop begins
  │
  ├─► Task 0 runs → display updates → timer delay → next task
  ├─► Task 1 runs → display updates → timer delay → next task
  ├─► Task 2 runs → display updates → timer delay → next task
  ├─► Task 3 runs → display updates → timer delay → next task
  └─► (repeat until all tasks reach MAX_ITER iterations)
  │
  ▼
All tasks INACTIVE — final values screen shown
  │
  ▼
Press any key to exit
```

---

## The Four Tasks

Each task runs independently and performs a different computation every turn it receives from the scheduler.

| Task | Procedure | What it does | Variable |
|------|-----------|--------------|----------|
| Task 0 | `DO_TASK0` | Increments a counter by 1 each turn | `counter0` |
| Task 1 | `DO_TASK1` | Increments a counter by 2 each turn | `counter1` |
| Task 2 | `DO_TASK2` | Multiplies `counter0` by 3 each turn | `result2` |
| Task 3 | `DO_TASK3` | Cycles through letters A to Z each turn | `current_char` |

Each task runs for exactly `MAX_ITER` (30) iterations before marking itself as INACTIVE and exiting the rotation permanently.

### Expected Final Values (with MAX_ITER = 30)

| Task | Final Value | Explanation |
|------|-------------|-------------|
| Task 0 | 30 | 1 × 30 iterations |
| Task 1 | 60 | 2 × 30 iterations |
| Task 2 | 90 | counter0 (30) × 3 |
| Task 3 | E | 30 letters from A wraps through Z back to E |

---

## Task Control Block (TCB)

The TCB is the most important data structure in this project. Each task has a 20-byte entry in the `TCB_ARRAY` that stores its complete state. This is exactly how real operating systems track process information.

```
TCB Entry Layout (20 bytes per task):
┌──────────┬──────────┬──────────┬──────────┬──────────┬─────────┐
│ EIP (4B) │ ESP (4B) │ EAX (4B) │ EBX (4B) │ STATUS(1)│ PAD(3B) │
│ offset 0 │ offset 4 │ offset 8 │ offset 12│ offset 16│         │
└──────────┴──────────┴──────────┴──────────┴──────────┴─────────┘
```

### TCB Field Offsets

| Constant | Offset | Size | Purpose |
|----------|--------|------|---------|
| `TCB_EIP` | 0 | 4 bytes | Saved instruction pointer |
| `TCB_ESP` | 4 | 4 bytes | Saved stack pointer |
| `TCB_EAX` | 8 | 4 bytes | Saved EAX register |
| `TCB_EBX` | 12 | 4 bytes | Saved EBX register |
| `TCB_STATUS` | 16 | 1 byte | Current task state |

### Task Status Values

| Constant | Value | Meaning |
|----------|-------|---------|
| `INACTIVE` | 0 | Task has finished or does not exist |
| `READY` | 1 | Task is waiting for its turn |
| `RUNNING` | 2 | Task is currently executing |

### Full TCB Array in Memory

```
TCB_ARRAY (80 bytes total):
  [0  .. 19] = Task 0 TCB
  [20 .. 39] = Task 1 TCB
  [40 .. 59] = Task 2 TCB
  [60 .. 79] = Task 3 TCB
```

---

## Memory Layout

```
Data Segment:
┌─────────────────────────────┐
│  TCB_ARRAY (80 bytes)       │  ← 4 × 20-byte TCB entries
├─────────────────────────────┤
│  TASK0_STACK (128 bytes)    │  ← private stack for Task 0
│  TASK1_STACK (128 bytes)    │  ← private stack for Task 1
│  TASK2_STACK (128 bytes)    │  ← private stack for Task 2
│  TASK3_STACK (128 bytes)    │  ← private stack for Task 3
├─────────────────────────────┤
│  CURRENT_TASK (4 bytes)     │  ← index of running task (0-3)
│  TASK_COUNT   (4 bytes)     │  ← total number of tasks
│  switch_count (4 bytes)     │  ← total context switches done
├─────────────────────────────┤
│  Display strings            │
│  Task variables             │
│  Iteration counters         │
└─────────────────────────────┘
```

Each task's stack grows **downward**, so the initial stack pointer is set to the **end** (highest address) of each stack block.

---

## Procedures

### `main` — Juwairiya
Entry point. Clears the screen, prints startup messages, calls `INIT_TASKS`, then calls `START_SCHEDULER`. After the scheduler returns, the program exits.

### `INIT_TASKS` — Juwairiya
Initializes all four TCB entries before scheduling begins. Sets each task's stack pointer to the top of its private stack block and sets status to `READY`. Called once at startup.

### `START_SCHEDULER` — Muskan
The core scheduling loop. Implements round-robin scheduling:
1. Checks if all four tasks are `INACTIVE` — if yes, shows final results and exits
2. For each task in order (0 → 1 → 2 → 3), checks if it is still active
3. Sets the task to `RUNNING`, increments `switch_count`, calls `DISPLAY_STATUS`, then calls the task's procedure
4. After the task returns, checks if it just went `INACTIVE` — if not, sets it back to `READY`
5. Repeats forever until all tasks are done

### `TIMER_DELAY` — Muskan
Software timer simulation. Burns 5,000,000 CPU cycles in a countdown loop, creating a visible delay between task executions. This simulates the timer tick that would trigger task switching in a real OS.

### `DO_TASK0` — Parwin
Counter task. Each turn, checks if `iter0` has reached `MAX_ITER`. If yes, sets TCB status to `INACTIVE` and returns. Otherwise, increments `counter0` by 1, increments `iter0`, calls `TIMER_DELAY`, and returns.

### `DO_TASK1` — Parwin
Counter task. Same structure as `DO_TASK0` but increments `counter1` by 2 each turn.

### `DO_TASK2` — Parwin
Arithmetic task. Each turn multiplies `counter0` by 3 and stores the result in `result2`. Demonstrates a dependent computation — its output is directly tied to another task's data.

### `DO_TASK3` — Parwin
Letter cycling task. Each turn advances `current_char` by one ASCII value. When it passes `Z` (5Bh), it resets to `A` (41h), creating a continuous A→Z cycle.

### `DISPLAY_STATUS` — Maryam
Live dashboard procedure. Called before each task executes. Clears the screen and prints:
- Header with project title and team names
- Live context switch counter
- For each task: name, color-coded status, progress bar, and current value
- Footer separator

Color coding: **green** = RUNNING, **white** = READY, **gray** = INACTIVE.

---

## Live Display

The scheduler refreshes the screen before each task turn, showing a dashboard like this:

```
==============================
      MINI TASK SCHEDULER
  Juwairiya Muskan Parwin Maryam
  Context switches: 47
==============================
  Task 0 (Counter  x1): INACTIVE  [##############################]  val=30
  Task 1 (Counter  x2): RUNNING   [###############---------------]  val=32
  Task 2 (Multiply x3): READY     [##############----------------]  val=87
  Task 3 (A-Z Cycle  ): READY     [##############----------------]  val=O
==============================
```

The progress bar fills with `#` symbols as iterations complete and empties with `-` for remaining slots, giving an immediate visual indication of each task's progress toward completion.

---

## Final Output Screen

When all four tasks reach `MAX_ITER` iterations, the scheduler exits and displays final values:

```
==============================
      All tasks completed!
      Scheduler exiting.
==============================
  Task 0 final counter  : 30
  Task 1 final counter  : 60
  Task 2 final result   : 90
  Task 3 final letter   : E
==============================
Press any key to continue...
```

---

## Constants Reference

| Constant | Value | Purpose |
|----------|-------|---------|
| `NUM_TASKS` | 4 | Total number of tasks |
| `TCB_SIZE` | 20 | Bytes per TCB entry |
| `STACK_SIZE` | 128 | Bytes per task private stack |
| `MAX_ITER` | 30 | Iterations before a task terminates |

### Tuning the Demo Speed

To make the scheduler run faster or slower, adjust these two values:

```asm
MAX_ITER  EQU  30        ; increase = more iterations, longer run
```
```asm
mov  ecx, 5000000        ; in TIMER_DELAY — increase = slower switching
```

---

## Key Concepts Demonstrated

**Round-robin scheduling** — each task gets exactly one turn per cycle, in fixed order (0 → 1 → 2 → 3 → 0 → ...), which is the simplest and most common CPU scheduling algorithm.

**Task Control Block** — a fixed-size data structure per task storing its execution state. This is the same concept used in real operating system process tables.

**Task lifecycle** — each task moves through three states during its lifetime: `READY` (waiting) → `RUNNING` (executing) → `INACTIVE` (finished). The scheduler respects these states and skips inactive tasks automatically.

**Software timer simulation** — a countdown loop of 5,000,000 iterations simulates the hardware timer interrupt that triggers task switching in a real preemptive OS.

**Private stacks** — each task has its own 128-byte stack block in the data segment. Stack pointers are initialized to the top (highest address) of each block since x86 stacks grow downward.

**Stack operations** — `DISPLAY_STATUS` uses `pushad`/`popad` to save and restore all general-purpose registers, ensuring the scheduler's register state is preserved across display calls.

**Modular design** — the project is split across four team members with clean procedure boundaries, shared constants, and a single data segment that all modules read from and write to.

---

## Project Structure

```
scheduler_base.asm
│
├── Constants & EQU definitions
├── .data segment
│   ├── TCB_ARRAY
│   ├── Task stacks (×4)
│   ├── Scheduler state variables
│   ├── Display strings
│   └── Task variables & iteration counters
│
└── .code segment
    ├── main                (Juwairiya)
    ├── INIT_TASKS          (Juwairiya)
    ├── START_SCHEDULER     (Muskan)
    ├── TIMER_DELAY         (Muskan)
    ├── DO_TASK0            (Parwin)
    ├── DO_TASK1            (Parwin)
    ├── DO_TASK2            (Parwin)
    ├── DO_TASK3            (Parwin)
    └── DISPLAY_STATUS      (Maryam)
```

---

## Team Contributions

| Member | Role | Procedures |
|--------|------|------------|
| Juwairiya | System architecture, memory layout, integration | `main`, `INIT_TASKS` |
| Muskan | Scheduling algorithm, timer simulation | `START_SCHEDULER`, `TIMER_DELAY` |
| Parwin | Task creation, execution routines | `DO_TASK0`, `DO_TASK1`, `DO_TASK2`, `DO_TASK3` |
| Maryam | Display, I/O interfacing, testing | `DISPLAY_STATUS` |