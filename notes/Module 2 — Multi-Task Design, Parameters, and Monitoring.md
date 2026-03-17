# Module 2 — Multi-Task Design, Parameters, and Monitoring

## What You Learned
Build a real 4-task system. Understand priorities, task states, and how to pass data into tasks.

---

## The 4-Task System You Built

```c
// Your actual task setup from Module 2 exercise
void ButtonTask(void *pvParameters)  { /* Priority 3 — fastest response */ }
void SensorTask(void *pvParameters)  { /* Priority 2 — periodic read     */ }
void LEDTask(void *pvParameters)     { /* Priority 2 — status blink       */ }
void DisplayTask(void *pvParameters) { /* Priority 1 — UART output        */ }

// Creation
xTaskCreate(ButtonTask,  "BTN",  256, NULL, 3, &buttonTaskHandle);
xTaskCreate(SensorTask,  "SENS", 256, NULL, 2, &sensorTaskHandle);
xTaskCreate(LEDTask,     "LED",  128, NULL, 2, &ledTaskHandle);
xTaskCreate(DisplayTask, "DISP", 256, NULL, 1, &displayTaskHandle);
```

---

## Task States

```
Created → READY → RUNNING → BLOCKED (vTaskDelay / queue wait)
                          → SUSPENDED (vTaskSuspend)
```

Only ONE task can be RUNNING at a time.  
Scheduler always picks the **highest priority READY** task.

---

## Suspend and Resume (Used in ButtonTask)

```c
// Every 5th press: suspend sensor for 3 seconds
vTaskSuspend(sensorTaskHandle);
vTaskDelay(pdMS_TO_TICKS(3000));
vTaskResume(sensorTaskHandle);
```

---

## Passing Parameters to Tasks

```c
// Safe: static struct — lives forever
static LED_Config_t led1 = { GPIOA, GPIO_PIN_5, 200 };
xTaskCreate(LED_Task, "LED1", 128, &led1, 2, NULL);

// Safe: cast integer directly
xTaskCreate(Worker, "W1", 256, (void *)1, 2, NULL);

// ❌ UNSAFE: local variable — gets destroyed when function returns
LED_Config_t cfg = { ... };           // local!
xTaskCreate(LED_Task, "X", 128, &cfg, 2, NULL);  // pointer goes bad
```

---

## Check Stack Usage at Runtime

```c
// Put this inside any task to check how close to overflow you are
UBaseType_t freeWords = uxTaskGetStackHighWaterMark(NULL);
// NULL = check current task
// If result < 20 words → increase stack size!
```

---

## Monitor Task (Add During Development)

```c
void MonitorTask(void *pvParameters)
{
    char buf[512];
    for (;;)
    {
        vTaskList(buf);           // prints all tasks: name, state, stack free
        UART_Print(buf);
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
// Stack: 512 words minimum (vTaskList needs it)
// Priority: 1 (lowest — only runs when others block)
```

---

## Priority Rules

| Priority | Task Type |
|---|---|
| 3–4 | Button, safety, fast response |
| 2 | Sensor read, control |
| 1 | Display, logging |
| 0 | Idle task (FreeRTOS auto-creates, never use this) |

**If a task never calls `vTaskDelay` or blocks → it starves all lower-priority tasks.**

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| All tasks same priority | Assign by time-criticality |
| Task never calls delay | Add `vTaskDelay` or block on queue/semaphore |
| Pass local variable pointer | Use static or global struct |
| Not checking `xTaskCreate` return | Always check `if (ret != pdPASS)` |
| Delete tasks frequently | Use suspend/resume instead |

---

## Outcome
Multi-task system running. Tasks have correct priorities. Stack and heap are monitored. Suspend/resume demonstrated.
