# Module 1 — FreeRTOS Setup and First Program

## What You Learned
Bring up FreeRTOS on STM32F446RE and run two tasks at the same time.

---

## CubeMX Setup (Do This Once)

| Setting | Value | Why |
|---|---|---|
| HAL Timebase | TIM6 | SysTick belongs to FreeRTOS |
| Tick Rate | 1000 Hz | 1 ms resolution |
| HEAP Size | 15360 bytes | Enough for ~6 tasks |
| Stack Overflow Check | Option 2 | Catches crashes during dev |
| USART2 IRQ Priority | 5 | Minimum safe for FreeRTOS API |

---

## First Two Tasks You Coded

```c
// LED Task — toggles every 500ms
void LED_Task(void *argument)
{
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        vTaskDelay(pdMS_TO_TICKS(500));   // ✅ NOT HAL_Delay()
    }
}

// Print Task — sends counter to UART every 1s
void Print_Task(void *argument)
{
    uint32_t counter = 0;
    char msg[64];
    for (;;)
    {
        int len = snprintf(msg, sizeof(msg), "Count: %lu\r\n", counter++);
        if (len > 0)
            HAL_UART_Transmit(&huart2, (uint8_t *)msg, len, 100);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

## How to Create Tasks (in main.c USER CODE BEGIN 2)

```c
// Create AFTER osKernelInitialize(), BEFORE osKernelStart()
xTaskCreate(LED_Task,   "LED",   128, NULL, 2, NULL);
xTaskCreate(Print_Task, "PRINT", 256, NULL, 1, NULL);
osKernelStart();
```

---

## Key Rules

- Every task is an infinite `for(;;)` loop — never return.
- Use `vTaskDelay()` inside tasks, never `HAL_Delay()`.
- Always check `xTaskCreate()` return value (`pdPASS` = success).
- Stack size is in **WORDS** (×4 = bytes). 128 words = 512 bytes.
- `printf`/`snprintf` tasks need at least 256 words stack.

---

## Interrupt Priority Rule

```
Priority 0–4  → FreeRTOS API FORBIDDEN here
Priority 5–15 → Safe to call xSemaphoreGiveFromISR() etc.
USART2 = priority 5 ✅
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| `HAL_Delay()` in task | Use `vTaskDelay(pdMS_TO_TICKS(x))` |
| Task function returns | Wrap body in `for(;;)` |
| SysTick conflict | Set HAL timebase to TIM6 in CubeMX |
| Stack too small | Start with 256 words for any printf task |

---

## Outcome
Two tasks running independently. LED blinks. UART prints. FreeRTOS scheduler is running.
