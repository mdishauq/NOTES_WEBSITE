# FreeRTOS Stack Management — Field Guide

---

## What a Task Stack Actually Is

Every task gets its **own private block of RAM** carved from the FreeRTOS heap at creation.  
It is **not** shared. It is **not** dynamic. It is a fixed slice — and when it runs out, things die quietly.

It stores:
- Local variables declared inside the task function
- Return addresses for every function call
- CPU register saves during every context switch
- Nested function call frames (HAL, printf, your own helpers)

```c
// This number is in WORDS, not bytes
// On STM32 (32-bit): 1 word = 4 bytes
// So 128 words = 512 bytes only
xTaskCreate(MyTask, "MyTask", 128, NULL, 1, NULL);
//                             ^^^
//                             Stack size in WORDS
```

> **Quick conversion:** `words × 4 = bytes` on any 32-bit MCU.

---

## Stack Size Reference by Task Type

| Task Type | Stack (words) | Stack (bytes) | Notes |
|-----------|:---:|:---:|-------|
| Simple blink / GPIO toggle | `64` | 256 B | Absolute minimum, no calls out |
| Basic logic, no I/O | `128` | 512 B | Safe floor for simple tasks |
| UART transmit (polling) | `192` | 768 B | |
| I2C / SPI with HAL | `256` | 1 KB | HAL call depth adds up fast |
| `sprintf` / `snprintf` | `256` | 1 KB | String formatting is stack-heavy |
| `printf` (any use) | `512` | 2 KB | printf alone can eat 300–500 bytes |
| `printf` + HAL + logic | `512–1024` | 2–4 KB | Typical real-world task |
| FatFS / SD card | `512–1024` | 2–4 KB | File system is deeply nested |
| USB / network stack | `1024+` | 4 KB+ | Never undersize these |

> **Rule of thumb:** Measure first with `uxTaskGetStackHighWaterMark()`, then trim.  
> Always keep a **30–50% headroom** above your measured peak — context switches and ISRs consume extra stack silently.

---

## How Stack Corruption Actually Kills You

FreeRTOS stacks grow **downward** in memory. When a task writes past its bottom boundary, it silently overwrites whatever sits below it in RAM.

```
High Address ┌──────────────────────┐
             │    Task A Stack      │
             │    (allocated)       │
             │          ↓          │  ← Stack grows downward
             │   [overflow zone]    │  ← Task A wrote past here
             ├──────────────────────┤  ← Stack boundary (no guard)
             │    Task B Stack      │  ← NOW SILENTLY CORRUPTED
             │    or Heap           │
Low Address  └──────────────────────┘
```

### Why the crash happens somewhere else

The overflow corrupts a **neighboring region** — another task's stack, a heap block, or a global variable. That region keeps running normally until it uses the corrupted value. So:

1. Task A overflows at **time T**
2. Task B reads corrupted data at **time T + 500ms**
3. System crashes at **time T + 500ms** inside **Task B's** code
4. You debug Task B for hours — the bug was always in Task A

This is why stack overflows are the hardest class of RTOS bug to diagnose without the right tools enabled.

### What corruption looks like in practice

| Symptom | Likely Cause |
|---------|-------------|
| Hard fault in unrelated code | Adjacent task stack corrupted |
| Variable randomly changes value | Global/static variable overwritten |
| Task suddenly stops running | Its TCB (task control block) corrupted |
| System works fine at low load, crashes under load | Stack fills deeper under interrupt pressure |
| Crash only happens after specific call sequence | Deep call chain finally pushes stack over edge |

---

## What Eats Your Stack (The Real Culprits)

### 1. Large local arrays — the #1 killer

```c
// BAD — 512 bytes disappear from your stack immediately
void MyTask(void *pvParameters) {
    uint8_t rxBuffer[512];
    char    logStr[128];
    // Your 512-byte (128-word) stack is already gone before a single line runs
    for(;;) {}
}

// GOOD — move large buffers out of the task's stack frame
static uint8_t rxBuffer[512];   // Lives in .bss, not on the stack
static char    logStr[128];

void MyTask(void *pvParameters) {
    for(;;) {}
}
```

### 2. `printf` and `sprintf`

These are secretly enormous. They pull in floating point formatting, internal buffers, and deep call chains.

- `printf` alone: **300–500 bytes** of stack
- `sprintf` with floats: can exceed **600 bytes**
- **Never** use `printf` inside a task with less than `512` words of stack

### 3. HAL function call depth

A single `HAL_I2C_Master_Transmit()` call does not stay shallow:

```
MyTask()
  → HAL_I2C_Master_Transmit()
    → HAL_I2C_WaitOnFlagUntilTimeout()
      → HAL_GetTick()
```

Each frame pushes registers and return addresses. HAL alone adds **200–400 bytes** to your peak.

### 4. Recursive or deeply nested functions

Every function call adds a frame. A chain of 5–6 helper functions called from within a task can silently consume hundreds of bytes. Keep task logic **flat**, not deeply nested.

### 5. Context switch register saves

Every time the scheduler preempts your task, it saves the **full CPU register state** onto that task's stack. On Cortex-M with FPU enabled, this is **136 bytes** per switch. If your FPU is enabled in `FreeRTOSConfig.h`, account for this.

```c
// In FreeRTOSConfig.h — if this is 1, add ~136 bytes to every task's minimum
#define configUSE_TASK_FPU_SUPPORT    1
```

---

## Detection & Measurement Toolkit

### Enable overflow detection — do this before anything else

In `FreeRTOSConfig.h`:

```c
#define configCHECK_FOR_STACK_OVERFLOW    2
// Method 1 = checks last few bytes only (fast, less reliable)
// Method 2 = paints entire stack with 0xA5, checks pattern (thorough — always use this)
```

Implement the hook — without this, overflow kills silently:

```c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    // pcTaskName tells you exactly which task overflowed
    __BKPT(0);    // Breaks into debugger on STM32
    for(;;);      // Hang so the state is preserved for inspection
}
```

### Measure real stack usage with high water mark

```c
void MyTask(void *pvParameters) {
    for(;;) {
        UBaseType_t hwm = uxTaskGetStackHighWaterMark(NULL);
        // hwm = minimum FREE words ever recorded since task started
        // Healthy: hwm should never drop below 20–30 words
        // Danger:  single digits = increase stack immediately
        // Zero:    overflow has already occurred
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

### Snapshot all tasks at once (for a full system audit)

```c
void PrintAllTaskStacks(void) {
    TaskStatus_t taskArray[10];
    UBaseType_t taskCount = uxTaskGetSystemState(taskArray, 10, NULL);

    for(UBaseType_t i = 0; i < taskCount; i++) {
        // usStackHighWaterMark = minimum free words ever, for each task
        printf("Task: %-16s | Free Stack: %u words\n",
            taskArray[i].pcTaskName,
            taskArray[i].usStackHighWaterMark);
    }
}
```

Call this periodically during development to catch any task drifting toward zero.

---

## Heap vs Stack — Know the Difference

These are two separate memory pools. They fail in different ways.

| | Stack | Heap |
|---|---|---|
| **What it is** | Per-task private RAM slice | Shared FreeRTOS memory pool |
| **Allocated by** | `xTaskCreate()` at task creation | `xTaskCreate()`, `xQueueCreate()`, etc. |
| **Configured in** | `xTaskCreate()` size argument | `configTOTAL_HEAP_SIZE` |
| **Overflow symptom** | Corrupts neighboring memory silently | `xTaskCreate()` returns `pdFAIL` |
| **Detection tool** | `uxTaskGetStackHighWaterMark()` | `xPortGetFreeHeapSize()` |

```c
// Always check xTaskCreate return value — pdFAIL means heap is exhausted
BaseType_t result = xTaskCreate(MyTask, "MyTask", 256, NULL, 1, &myHandle);
if (result != pdPASS) {
    // Heap is full — increase configTOTAL_HEAP_SIZE in FreeRTOSConfig.h
    Error_Handler();
}

// Check remaining heap at runtime
size_t freeHeap = xPortGetFreeHeapSize();        // Current free bytes
size_t minHeap  = xPortGetMinimumEverFreeHeapSize(); // Lowest point ever reached
```

---

## Pre-Flight Checklist Before Flashing

- [ ] `configCHECK_FOR_STACK_OVERFLOW` set to `2` in `FreeRTOSConfig.h`
- [ ] `vApplicationStackOverflowHook()` implemented with a breakpoint or `Error_Handler()`
- [ ] Every `xTaskCreate()` return value is checked against `pdPASS`
- [ ] No large arrays (`> 64 bytes`) declared as local variables inside task functions
- [ ] `printf` / `sprintf` tasks allocated at least `512` words
- [ ] Tasks using HAL I2C / SPI allocated at least `256` words
- [ ] `uxTaskGetStackHighWaterMark()` logged during a full test run
- [ ] All high water marks are above **20 words** minimum
- [ ] `xPortGetMinimumEverFreeHeapSize()` checked — not near zero
- [ ] FPU usage accounted for if `configUSE_TASK_FPU_SUPPORT` is enabled

---

## Golden Rules

| Rule | Reason |
|------|--------|
| Stack size is in **words**, not bytes | Easy to undersize by 4× without realizing |
| Large local arrays go `static` or global | Local arrays live on the stack — they will overflow it |
| `printf` needs at minimum `512` words | It is far larger than it looks |
| Always implement `vApplicationStackOverflowHook` | Silent overflow is undebuggable |
| Measure with high water mark in every new project | You cannot guess stack depth accurately |
| Leave 30–50% headroom above measured peak | Interrupts and context switches take extra stack you cannot see |
| Check `xTaskCreate()` return value every time | Heap exhaustion gives no warning otherwise |
| Never call blocking FreeRTOS functions from an ISR | Use `FromISR` variants — standard calls will corrupt the scheduler |
