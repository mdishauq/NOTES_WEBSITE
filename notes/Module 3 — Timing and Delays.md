# Module 3 — Timing and Delays

## What You Learned
Difference between vTaskDelay and vTaskDelayUntil. Proved it with your own output on serial.

---

## The Two Delay Functions

### vTaskDelay — Relative (Drifts)

```c
void vTaskDelay_Task(void *pvParameters)
{
    int loop_count = 0;
    char msg[100];

    for (;;)
    {
        if ((loop_count % 10) == 0)
        {
            TickType_t now = xTaskGetTickCount();   // ← call with ()
            int len = snprintf(msg, sizeof(msg),
                               "From TaskDelay --> counter=%d tick=%lu\r\n",
                               loop_count, (unsigned long)now);
            if (len > 0) UART_Print(msg);
        }
        vTaskDelay(pdMS_TO_TICKS(100));   // sleep 100ms from NOW
        loop_count++;
    }
}
```

Period = work time + 100ms → drifts over time.

---

### vTaskDelayUntil — Absolute (Stays Fixed)

```c
void vTaskDelayUntill_Task(void *pvParameters)
{
    int loop_count = 0;
    TickType_t lastWake = xTaskGetTickCount();  // ← MUST init before loop
    char msg[100];

    for (;;)
    {
        if ((loop_count % 10) == 0)
        {
            TickType_t now = xTaskGetTickCount();
            int len = snprintf(msg, sizeof(msg),
                               "From TaskDelayUntil --> counter=%d tick=%lu\r\n",
                               loop_count, (unsigned long)now);
            if (len > 0) UART_Print(msg);
        }
        vTaskDelayUntil(&lastWake, pdMS_TO_TICKS(100));  // wake at lastWake + 100ms
        loop_count++;
    }
}
```

Period = exactly 100ms. Sleep time adjusts to compensate for work time.

---

## Your Actual Output (Proof)

```
From TaskDelayUntil --> counter=10   tick=1001   ← ~1000 as expected
From TaskDelayUntil --> counter=20   tick=2001   ← exactly +1000
From TaskDelayUntil --> counter=30   tick=3001   ← exactly +1000
From TaskDelayUntil --> counter=100  tick=10001  ← no drift after 100 loops
```

vTaskDelay task only printed once at tick=0 — because both tasks shared UART without protection, causing a **race condition**. This leads directly into Module 4 (Mutexes).

---

## When to Use Which

| Situation | Use |
|---|---|
| LED blink, display update | `vTaskDelay` — drift doesn't matter |
| Sensor sampling, PID loop | `vTaskDelayUntil` — fixed period required |
| Something needs to run exactly every N ms | `vTaskDelayUntil` |

---

## Software Timers (Covered — Use When Needed)

```c
// Auto-reload (periodic)
TimerHandle_t t = xTimerCreate("LED", pdMS_TO_TICKS(500), pdTRUE, NULL, Callback);
xTimerStart(t, 0);

// One-shot (fires once after delay)
TimerHandle_t t = xTimerCreate("Timeout", pdMS_TO_TICKS(3000), pdFALSE, NULL, Callback);
xTimerStart(t, 0);
```

**Timer callback rules:**
- Must be SHORT (< 1ms ideally)
- Must NOT block
- Do NOT call `HAL_Delay` or `vTaskDelay` inside a callback
- For heavy work: set a flag or give a semaphore, let a task do the work

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| `xTaskGetTickCount` without `()` | Always write `xTaskGetTickCount()` |
| `%p` for tick value | Use `%lu` and cast to `(unsigned long)` |
| `lastWake` not initialized before loop | Init with `xTaskGetTickCount()` once before `for(;;)` |
| Blocking inside timer callback | Only set flag or give semaphore |
| Using `vTaskDelay` for PID/sensor | Use `vTaskDelayUntil` |

---

## Bug You Found (Important)

Two tasks printing to UART at the same time = corrupted output.  
This is a **Race Condition** — fix is a **Mutex** (Module 4).

---

## Outcome
You proved timing drift with real serial output. You know when each delay function fits. You discovered the shared-resource problem on your own.
