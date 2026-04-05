# FreeRTOS Native API — Cheat Sheet

> Every file using FreeRTOS **must** start with:
> ```c
> #include "FreeRTOS.h"  // Mandatory core definitions
> ```

---

## Naming Convention

| Prefix | Return Type | Example |
|--------|-------------|---------|
| `x`    | `BaseType_t` or handle | `xTaskCreate()` |
| `v`    | `void` | `vTaskDelay()` |
| `ux`   | `UBaseType_t` | `uxQueueMessagesWaiting()` |
| `p`    | pointer | `pvTimerGetTimerID()` |

---

## 1. Task Management
**Header:** `#include "task.h"`

| Function | Purpose |
|----------|---------|
| `xTaskCreate()` | Allocate memory and register a task with the scheduler |
| `vTaskDelay(ticks)` | Block the task for N ticks |
| `vTaskDelete(handle)` | Remove a task from the scheduler |

```c
void MyTaskFunction(void *pvParameters) {
    for(;;) {
        // Do work
        vTaskDelay(pdMS_TO_TICKS(100)); // Delay 100 ms
    }
}

// In main()
xTaskCreate(MyTaskFunction, "TaskName", 128, NULL, 1, NULL);
//           ^function       ^name       ^stack ^params ^priority ^handle
```

---

## 2. Semaphores & Mutexes
**Header:** `#include "semphr.h"`

| Function | Purpose |
|----------|---------|
| `xSemaphoreCreateBinary()` | Simple 0/1 flag for signaling |
| `xSemaphoreCreateMutex()` | Token for resource locking (e.g. I2C bus) |
| `xSemaphoreTake(sem, timeout)` | **Wait** for semaphore — P operation |
| `xSemaphoreGive(sem)` | **Release** semaphore — V operation |

```c
// Waiting on a signal (blocks forever until received)
if (xSemaphoreTake(mySem, portMAX_DELAY) == pdPASS) {
    // Got the signal — process data
}
```

---

## 3. Queues
**Header:** `#include "queue.h"`

| Function | Purpose |
|----------|---------|
| `xQueueCreate(len, itemSize)` | Define queue capacity and element size |
| `xQueueSend(queue, &item, timeout)` | Push to the back of the queue |
| `xQueueReceive(queue, &buf, timeout)` | Pull from the front (blocks if empty) |

```c
uint8_t data = 0xAA;
xQueueSend(myQueue, &data, 0);         // Send without waiting
xQueueReceive(myQueue, &data, portMAX_DELAY); // Block until data arrives
```

---

## 4. Software Timers
**Header:** `#include "timers.h"`

| Function | Purpose |
|----------|---------|
| `xTimerCreate()` | Define period and callback function |
| `xTimerStart(timer, timeout)` | Start the countdown |
| `xTimerStop(timer, timeout)` | Stop the timer |
| `xTimerReset(timer, timeout)` | Restart from zero |

```c
TimerHandle_t myTimer = xTimerCreate(
    "MyTimer",              // Name
    pdMS_TO_TICKS(500),     // Period (500 ms)
    pdTRUE,                 // Auto-reload
    NULL,                   // Timer ID
    MyTimerCallback         // Callback function
);
xTimerStart(myTimer, 0);
```

---

## 5. Event Groups
**Header:** `#include "event_groups.h"`

Use when you need to wait on **multiple signals at once**.

| Function | Purpose |
|----------|---------|
| `xEventGroupCreate()` | Create an event group |
| `xEventGroupSetBits()` | Signal one or more bits |
| `xEventGroupWaitBits()` | Block until required bits are set |

---

## ⚠️ The `FromISR` Rule — Critical for STM32

**Inside any ISR** (e.g. `HAL_I2C_MasterRxCpltCallback`), you **cannot** use standard FreeRTOS functions. Use the `FromISR` variants instead.

| Standard (task context) | ISR context |
|-------------------------|-------------|
| `xSemaphoreGive()` | `xSemaphoreGiveFromISR()` |
| `xSemaphoreTake()` | ❌ Never take inside ISR |
| `xQueueSend()` | `xQueueSendFromISR()` |
| `xTimerStart()` | `xTimerStartFromISR()` |

```c
// Inside HAL_I2C_MasterRxCpltCallback (an ISR)
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
xSemaphoreGiveFromISR(mySem, &xHigherPriorityTaskWoken);
portYIELD_FROM_ISR(xHigherPriorityTaskWoken); // Yield if a higher-priority task woke up
```

> **Why?** ISRs cannot block or wait. `FromISR` functions execute instantly and signal the scheduler to immediately wake a higher-priority task if needed.

---

## Quick Reference Table

| Topic | Header | Common Prefix | Purpose |
|-------|--------|---------------|---------|
| Tasks | `task.h` | `vTask...` / `xTask...` | Running code in parallel |
| Semaphores | `semphr.h` | `xSemaphore...` | Signaling and locking |
| Queues | `queue.h` | `xQueue...` | Moving data between tasks |
| Timers | `timers.h` | `xTimer...` | Timed callbacks |
| Events | `event_groups.h` | `xEventGroup...` | Complex multi-signal logic |

---

## Useful Macros

| Macro | Purpose |
|-------|---------|
| `pdMS_TO_TICKS(ms)` | Convert milliseconds → ticks |
| `portMAX_DELAY` | Block indefinitely |
| `pdPASS` / `pdFAIL` | Check return values |
| `pdTRUE` / `pdFALSE` | Boolean constants |
| `configTICK_RATE_HZ` | Tick frequency (usually 1000) |
