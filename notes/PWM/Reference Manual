# ⚡ PWM & STM32 HAL — Complete Reference Guide

> **Pulse Width Modulation** — The art of making digital hardware speak analog.  
> A field engineer's reference for STM32 timer-based PWM using HAL and direct register access.

---

## 📋 Table of Contents

1. [What is PWM?](#1-what-is-pwm)
2. [The Anatomy of a PWM Signal](#2-the-anatomy-of-a-pwm-signal)
3. [Key Parameters Explained](#3-key-parameters-explained)
4. [STM32 Timer Architecture](#4-stm32-timer-architecture)
5. [HAL API — The Big Three Macros](#5-hal-api--the-big-three-macros)
6. [Starting & Stopping PWM](#6-starting--stopping-pwm)
7. [Interrupts & Callbacks](#7-interrupts--callbacks)
8. [Input Capture — Reading PWM](#8-input-capture--reading-pwm)
9. [Direct Register Access](#9-direct-register-access)
10. [Preload Enable — Glitch-Free Updates](#10-preload-enable--glitch-free-updates)
11. [Potentiometer → PWM Complete Example](#11-potentiometer--pwm-complete-example)
12. [Common Pitfalls & Gotchas](#12-common-pitfalls--gotchas)
13. [Quick Reference Cheat Sheet](#13-quick-reference-cheat-sheet)

---

## 1. What is PWM?

**Pulse Width Modulation** is a technique where a digital signal rapidly switches between HIGH (`3.3V` / `5V`) and LOW (`0V`). By controlling **how long** the signal stays HIGH versus LOW, we can simulate analog voltage levels, control motor speeds, dim LEDs, and generate audio waveforms — all using purely digital hardware.

```
  Real Analog Signal        PWM Equivalent (simulated analog)
  ───────────────────       ──────────────────────────────────

  3.3V ┤  ╭───────╮         3.3V ┤ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
       │ ╱         ╲              │ │ │ │ │ │ │ │ │ │ │ │ │
  1.6V ┤╱           ╲        1.6V┤ │ │ │ │ │ │ │ │ │ │ │ │  ← Average ≈ 1.65V
       │             ╲           │ │ │ │ │ │ │ │ │ │ │ │ │
  0V   ┤              ╰───   0V  ┤─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └
       └──────────────────       └───────────────────────────
         Smooth waveform            ~50% Duty Cycle PWM
```

PWM is the workhorse behind:

| Application       | How PWM is Used                              |
|-------------------|----------------------------------------------|
| 💡 LED Dimming    | Short ON time = dim, Long ON time = bright   |
| 🔧 Servo Motors   | Pulse width determines angle (1ms–2ms)       |
| 🔊 Audio / Buzzer | Frequency of PWM = pitch of sound            |
| ⚙️ DC Motor Speed | Duty cycle controls average power delivered  |
| 🌡️ Heater Control | On/Off ratio controls temperature setpoint   |

---

## 2. The Anatomy of a PWM Signal

```
                    ◄──────── Period (T) ─────────►
                    ◄── ON ──►◄──── OFF ────────────►

  Voltage                                              
    │                                                  
HIGH┤  ┌────────┐              ┌────────┐              ┌──
    │  │        │              │        │              │  
    │  │        │              │        │              │  
LOW ┤──┘        └──────────────┘        └──────────────┘  
    │                                                  
    └──┬────────┬──────────────┬────────┬──────────────┬──
       t₀      t₁             t₂      t₃             t₄
                                                         
       │◄ Ton ►│◄─── Toff ───►│                         
       │◄──────────── T ──────────────►│                 

  ┌─────────────────────────────────────────────────────┐
  │  Duty Cycle  =  (Ton / T) × 100%                   │
  │  Frequency   =  1 / T   (in Hz)                    │
  │  Period      =  T       (in seconds)               │
  └─────────────────────────────────────────────────────┘
```

### Duty Cycle Spectrum

```
  0% Duty                                           100% Duty
  ──────                                            ─────────

  0%    ┌┐┌┐┌┐┌┐     Barely on, almost always LOW
  LOW ──┘└┘└┘└┘└────────────────────────────────────

  25%   ┌──┐   ┌──┐   ┌──┐   ┌──┐
  ──────┘  └───┘  └───┘  └───┘  └───────────────────

  50%   ┌────┐  ┌────┐  ┌────┐  ┌────┐
  ──────┘    └──┘    └──┘    └──┘    └───────────────

  75%   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
  ──────┘      └─┘      └─┘      └─┘      └──────────

  100%  ────────────────────────────────────────────
  HIGH  Always ON, always HIGH
```

---

## 3. Key Parameters Explained

### 🔢 The STM32 Timer Math

Every STM32 PWM signal is driven by a hardware **timer** with three critical registers:

```
  CPU Clock (e.g. 72 MHz)
       │
       ▼
  ┌────────────┐      ┌─────────────────────────────────────────────┐
  │ Prescaler  │──────► Timer Clock = CPU_Clock / (PSC + 1)        │
  │  (PSC)     │      │ e.g. PSC=71 → 72MHz/(71+1) = 1 MHz        │
  └────────────┘      └─────────────────────────────────────────────┘
       │
       ▼
  ┌────────────┐      ┌─────────────────────────────────────────────┐
  │  Counter   │      │ Counts 0 → ARR, then resets ("overflow")   │
  │  (CNT)     │      │ Frequency = Timer_Clock / (ARR + 1)        │
  └────────────┘      │ e.g. ARR=999 → 1MHz/1000 = 1 kHz          │
       │              └─────────────────────────────────────────────┘
       ▼
  ┌────────────┐      ┌─────────────────────────────────────────────┐
  │  Compare   │      │ When CNT < CCR → PIN = HIGH                │
  │  (CCR)     │      │ When CNT ≥ CCR → PIN = LOW                 │
  └────────────┘      │ Duty = CCR / (ARR+1) × 100%               │
                      └─────────────────────────────────────────────┘
```

### 📐 Visual Timer Count vs PWM Output

```
  Counter   ARR=999
  Value     ↑
            │
  999 ┤     │              ╭─────╮              ╭─────
      │     │             ╱       ╲            ╱
  CCR ┤ ····················· ···············································
  600 │     │           ╱           ╲         ╱
      │     │         ╱               ╲     ╱
    0 ┤─────╯───────╯                   ╰───╯                    ──► time
  
  PWM  HIGH ┤  ┌─────────────────────┐     ┌─────────────────────┐
  Pin       │  │                     │     │                     │
  Output LOW┤──┘                     └─────┘                     └──
  
             ◄──── CCR/ARR = 60% ON ────►◄── 40% OFF ──►
```

### 🧮 Formula Reference Table

| Parameter   | Register | Formula                                        | Example                        |
|-------------|----------|------------------------------------------------|--------------------------------|
| Timer Clock | —        | `f_tim = f_cpu / (PSC + 1)`                   | 72MHz / (71+1) = **1 MHz**    |
| Frequency   | ARR      | `f_pwm = f_tim / (ARR + 1)`                   | 1MHz / (999+1) = **1 kHz**    |
| Duty Cycle  | CCR      | `duty% = CCR / (ARR + 1) × 100`              | 600/1000 × 100 = **60%**      |
| CCR Value   | CCR      | `CCR = (duty% / 100) × (ARR + 1)`            | (75/100) × 1000 = **750**     |

---

## 4. STM32 Timer Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    STM32 Timer Block                            │
  │                                                                 │
  │  AHB/APB Bus                                                    │
  │      │                                                          │
  │  ┌───▼────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │
  │  │  PSC   │───►│   CNT    │───►│  CCR1    │───►│  CH1 OUT │──►│── GPIO Pin
  │  │(÷clock)│    │(0→ARR)   │    │(compare) │    │(PWM Mode)│   │
  │  └────────┘    └────┬─────┘    └──────────┘    └──────────┘   │
  │                     │          ┌──────────┐    ┌──────────┐   │
  │                     │─────────►│  CCR2    │───►│  CH2 OUT │──►│── GPIO Pin
  │                     │          └──────────┘    └──────────┘   │
  │                  ┌──▼─────┐    ┌──────────┐    ┌──────────┐   │
  │                  │  ARR   │    │  CCR3    │───►│  CH3 OUT │──►│── GPIO Pin
  │                  │(reload)│    └──────────┘    └──────────┘   │
  │                  └────────┘    ┌──────────┐    ┌──────────┐   │
  │                     │─────────►│  CCR4    │───►│  CH4 OUT │──►│── GPIO Pin
  │                     │          └──────────┘    └──────────┘   │
  │                  Update IRQ ──────────────────────────────────►│── CPU IRQ
  └─────────────────────────────────────────────────────────────────┘

  One timer → Up to 4 independent PWM channels, all sharing the same
  frequency (ARR/PSC) but each with its own duty cycle (CCRx).
```

### STM32 Timer Types

| Timer Type     | Examples           | Resolution | Channels | Special Features                        |
|----------------|--------------------|------------|----------|-----------------------------------------|
| **Advanced**   | TIM1, TIM8         | 16-bit     | 4        | Complementary outputs, dead-time, break |
| **General GP** | TIM2–TIM5          | 16/32-bit  | 4        | Input capture, encoder mode             |
| **Basic**      | TIM6, TIM7         | 16-bit     | 0        | DMA trigger only, no PWM output         |
| **LP Timer**   | LPTIM1             | 16-bit     | 1        | Runs in low-power stop mode             |

---

## 5. HAL API — The Big Three Macros

These three macros let you change PWM parameters **on the fly**, without stopping or re-initializing the timer.

### 🎯 `__HAL_TIM_SET_COMPARE` — Change Duty Cycle

```c
// Syntax:
__HAL_TIM_SET_COMPARE(&htimX, TIM_CHANNEL_Y, value);

// Example: Set TIM1 Channel 1 to a CCR value of 750 (= 75% if ARR=999)
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 750);

// This is equivalent to direct register access:
TIM1->CCR1 = 750;
```

```
  Before:  CCR=250  →  25% duty
  ┌──┐        ┌──┐        ┌──
  ┘  └────────┘  └────────┘

  After:   CCR=750  →  75% duty
  ┌──────┐  ┌──────┐  ┌──────
  ┘      └──┘      └──┘
```

### 📏 `__HAL_TIM_SET_AUTORELOAD` — Change Frequency

```c
// Syntax:
__HAL_TIM_SET_AUTORELOAD(&htimX, value);

// Example: Change period count to 499 → doubles the frequency
__HAL_TIM_SET_AUTORELOAD(&htim1, 499);  // f = f_tim / (499+1) = 2× the old rate

// Equivalent to:
TIM1->ARR = 499;
```

> ⚠️ **Warning:** If your new ARR value is smaller than the current `CNT`, the counter overshoots  
> and must wrap around a full 16-bit cycle (65535) before resetting. Use with care.

### ⚙️ `__HAL_TIM_SET_PRESCALER` — Change Clock Divider

```c
// Syntax:
__HAL_TIM_SET_PRESCALER(&htimX, value);

// Example: Slow the timer clock by 10×
__HAL_TIM_SET_PRESCALER(&htim1, 719);  // 72MHz / 720 = 100 kHz timer clock

// Equivalent to:
TIM1->PSC = 719;
```

> ⏳ **Note:** PSC changes take effect **only after the next update event** (when CNT resets to 0),  
> unlike CCR which can take effect immediately (or at next cycle with preload enabled).

---

## 6. Starting & Stopping PWM

### Standard Mode

```c
// ✅ Start PWM output on a pin
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);

// ⛔ Stop PWM output (pin goes LOW)
HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_1);
```

### Interrupt Mode

```c
// Start PWM + enable period-elapsed interrupt
HAL_TIM_PWM_Start_IT(&htim1, TIM_CHANNEL_1);

// Stop PWM + disable interrupt
HAL_TIM_PWM_Stop_IT(&htim1, TIM_CHANNEL_1);
```

### DMA Mode — The Pro Choice ⚡

```c
// Buffer of CCR values — hardware will cycle through them automatically
uint32_t pwm_buffer[] = {0, 200, 400, 600, 800, 999, 800, 600, 400, 200};
uint32_t buffer_len   = sizeof(pwm_buffer) / sizeof(pwm_buffer[0]);

// Start PWM with DMA — CPU does ZERO work after this call
HAL_TIM_PWM_Start_DMA(&htim1, TIM_CHANNEL_1, pwm_buffer, buffer_len);

// Stop DMA-driven PWM
HAL_TIM_PWM_Stop_DMA(&htim1, TIM_CHANNEL_1);
```

```
  DMA Mode signal (smooth LED breathing effect):

  Brightness
  100% ┤                 ╭───╮
       │              ╭──╯   ╰──╮
   75% ┤           ╭──╯         ╰──╮
       │        ╭──╯               ╰──╮
   50% ┤     ╭──╯                     ╰──╮
       │  ╭──╯                           ╰──╮
   25% ┤──╯                                 ╰──╮
    0% ┤                                        ╰──────
       └──────────────────────────────────────────────►
                    DMA cycles through buffer                time
```

### Comparison: HAL Modes

| Mode         | CPU Usage | Use Case                              | Smoothness    |
|--------------|-----------|---------------------------------------|---------------|
| Standard     | High      | Simple LED, servo angle set-and-forget | Manual update |
| IT (Interrupt)| Medium   | Timed reactions, sensor sync           | Per-period    |
| **DMA**      | **~0%**   | Audio, motor ramps, LED animations     | **Seamless**  |

---

## 7. Interrupts & Callbacks

### Period Elapsed Callback

Fires every time the timer counter hits the **ARR** value (end of each PWM cycle):

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM1) {
        // ✅ Safe moment to read sensors or update the next CCR value
        // This runs at the exact PWM frequency — e.g., every 1ms at 1kHz
        my_sensor_reading = HAL_ADC_GetValue(&hadc1);
    }
    if (htim->Instance == TIM3) {
        // Multiple timers can share the same callback
        toggle_debug_pin();
    }
}
```

```
  PWM Signal    ┌─────┐     ┌─────┐     ┌─────┐     ┌─────┐
                │     │     │     │     │     │     │     │
  ──────────────┘     └─────┘     └─────┘     └─────┘     └──
  
  IRQ fires →               ↑           ↑           ↑
                       (end of cycle, CNT resets to 0)
                       
  Your code runs →          │           │           │
                       Read ADC    Update LED   Log data
```

### DMA Half/Complete Callbacks (for large buffers)

```c
// Called when DMA reaches the MIDPOINT of the buffer
void HAL_TIM_PWM_PulseFinishedHalfCpltCallback(TIM_HandleTypeDef *htim) {
    // Refill the FIRST half of the buffer here
}

// Called when DMA reaches the END of the buffer
void HAL_TIM_PWM_PulseFinishedCallback(TIM_HandleTypeDef *htim) {
    // Refill the SECOND half of the buffer here
}
```

> 💡 This is called **"Double Buffering"** — it lets you stream infinite waveforms  
> without ever stopping the DMA or the PWM output.

---

## 8. Input Capture — Reading PWM

Use this when your STM32 needs to **measure** an incoming PWM signal (e.g., reading a receiver signal or another controller's output).

```c
// Start input capture with interrupt on TIM2 Channel 1
HAL_TIM_IC_Start_IT(&htim2, TIM_CHANNEL_1);

// Callback fires on every rising/falling edge
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM2 && htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1) {
        static uint32_t rising_edge  = 0;
        static uint32_t falling_edge = 0;

        if (GPIO_PIN_SET == HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0)) {
            rising_edge = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1);
        } else {
            falling_edge  = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1);
            uint32_t pulse_width = falling_edge - rising_edge;
            // pulse_width in timer ticks → convert to µs using timer clock
        }
    }
}
```

```
  Incoming      ┌──────────┐              ┌──────────┐
  PWM Signal    │          │              │          │
  ──────────────┘          └──────────────┘          └──

  IRQ on        ↑          ↑              ↑          ↑
  every edge    │          │              │          │
                │          │              │          │
  Capture     CNT=100   CNT=600       CNT=1100   CNT=1600
  values:
  
                Ton = 600-100 = 500 ticks
                T   = 1100-100 = 1000 ticks
                Duty = 500/1000 = 50%
```

---

## 9. Direct Register Access

When you need maximum speed and minimal overhead, skip the HAL and write directly to peripheral registers.

### Core PWM Registers

```c
// ══════════════════════════════════════════════
//  DUTY CYCLE  — Update immediately or on next
//               update event (if preload ON)
// ══════════════════════════════════════════════
TIM1->CCR1 = 750;   // Channel 1 → 75% (if ARR=999)
TIM1->CCR2 = 500;   // Channel 2 → 50%
TIM1->CCR3 = 250;   // Channel 3 → 25%
TIM1->CCR4 = 100;   // Channel 4 → 10%

// ══════════════════════════════════════════════
//  FREQUENCY  — Change the period
// ══════════════════════════════════════════════
TIM1->ARR = 1999;   // New period → f = f_tim / 2000

// ══════════════════════════════════════════════
//  PRESCALER  — Change clock divider
//  (effective after next update event)
// ══════════════════════════════════════════════
TIM1->PSC = 71;     // 72MHz / 72 = 1MHz timer clock

// ══════════════════════════════════════════════
//  FORCE UPDATE EVENT  — Apply PSC/ARR now
// ══════════════════════════════════════════════
TIM1->EGR = TIM_EGR_UG;  // Trigger update generation
```

### HAL Macro vs Direct Register

```c
// These two lines do EXACTLY the same thing:

// HAL Macro (readable, safe):
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 750);

// Direct Register (fast, terse):
TIM1->CCR1 = 750;

// The HAL macro expands to:
// (htim)->Instance->CCR1 = (value)   ← literally one register write
```

| Approach        | Code Size | Speed     | Safety    | Readability |
|-----------------|-----------|-----------|-----------|-------------|
| HAL Functions   | Largest   | Slowest   | ✅ High   | ✅ High    |
| HAL Macros      | Medium    | Fast      | ✅ Good   | ✅ Good    |
| Direct Register | Smallest  | ⚡ Fastest | ⚠️ Manual  | ⚠️ Cryptic |

---

## 10. Preload Enable — Glitch-Free Updates

This is the most important concept for **real-time PWM control**.

### The Problem

```
  Timer counts: 0 ──────────────────────────────► ARR=999

  Old CCR = 800 (80% duty)
  ┌──────────────────────────────────────────┐   ┌────
  │              HIGH (800 ticks)            │   │
  ┘                                          └───┘

  While timer is at CNT=500, you write CCR = 200:

  ┌──────────────────────────────────────────────────── ???
  │              HIGH ─── wait, CCR<CNT, go LOW! GLITCH!
  ┘
  
  ⚡ GLITCH: The output immediately goes LOW mid-cycle because
     CNT (500) is now greater than the new CCR (200).
```

### The Solution — Preload (Shadow Register)

```
  With PRELOAD ENABLED (OC1PE bit set in TIM_CCMR1):

  Timer CNT: 0 ─────────────────── 999 │ 0 ─────────────── 999

  You write CCR = 200 here ─────────► │
                                        │
  New CCR takes effect HERE ────────────┘ (at next update event)

  ┌──────────────────────────────────────┐ ┌──┐
  │         80% (old CCR=800)            │ │  │ ← 20% (new CCR=200)
  ┘                                      └─┘  └────────────────
  
  ✅ Clean, glitch-free transition at the cycle boundary!
```

### How to Enable Preload in CubeMX

In STM32CubeMX, under `Timer → Parameter Settings`:
- **CH Polarity**: Rising
- **OCPreload**: `TIM_OCPRELOAD_ENABLE` ← **Enable this!**

Or in code after `HAL_TIM_PWM_Init()`:

```c
TIM_OC_InitTypeDef sConfig = {0};
sConfig.OCMode     = TIM_OCMODE_PWM1;
sConfig.OCPolarity = TIM_OCPOLARITY_HIGH;
sConfig.OCFastMode = TIM_OCFAST_DISABLE;

// ✅ This line enables the shadow/preload register
sConfig.OCPreload  = TIM_OCPRELOAD_ENABLE;

HAL_TIM_PWM_ConfigChannel(&htim1, &sConfig, TIM_CHANNEL_1);
```

And ensure the ARR preload is also set:

```c
// In TIM_Base_InitTypeDef:
htim1.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;
```

---

## 11. Potentiometer → PWM Complete Example

A real-world implementation: reading an ADC potentiometer and smoothly controlling PWM duty cycle.

### System Overview

```
  ┌────────────┐   Voltage    ┌──────────────┐  12-bit   ┌──────────────┐
  │   10kΩ     │  0V – 3.3V  │   ADC1       │  0–4095   │  Scaling     │
  │   Pot      │────────────►│   (PA0)      │──────────►│  Formula     │
  └────────────┘             └──────────────┘           └──────┬───────┘
                                                                │ CCR value
                                                                ▼ (0–ARR)
                                                        ┌──────────────┐
                                                        │  TIM1 CCR1   │
                                                        │  (PWM out)   │
                                                        └──────┬───────┘
                                                               │
                                                        ┌──────▼───────┐
                                                        │   PA8 pin    │
                                                        │  0%–100% PWM │
                                                        └──────────────┘
```

### The Scaling Formula

```
  ADC Range:  0 ──────────────────────────── 4095   (12-bit)
  CCR Range:  0 ──────────────────────────── ARR    (e.g., 2399)

  CCR = (adc_val × ARR) / 4095
  CCR = (adc_val × 2399) / 4095

  ADC=0    → CCR=0     →  0% duty  (LED off, motor stopped)
  ADC=2047 → CCR=1199  → 50% duty  (half speed/brightness)
  ADC=4095 → CCR=2399  → 100% duty (full speed/brightness)
```

### Complete Implementation

```c
/* USER CODE BEGIN Includes */
#include "main.h"
/* USER CODE END Includes */

/* Defines for clarity */
#define PWM_TIMER       htim1
#define PWM_CHANNEL     TIM_CHANNEL_1
#define ADC_MAX         4095U
#define PWM_ARR         2399U    // Must match your CubeMX ARR setting

/* USER CODE BEGIN 2 */
// Start ADC in continuous mode (configured in CubeMX)
HAL_ADC_Start(&hadc1);

// Start PWM output on TIM1 CH1 → PA8 pin
HAL_TIM_PWM_Start(&PWM_TIMER, PWM_CHANNEL);
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1) {
    // 1. Read the latest ADC conversion result
    uint32_t adc_val = HAL_ADC_GetValue(&hadc1);

    // 2. Scale ADC range (0–4095) to CCR range (0–ARR)
    uint32_t ccr_val = (adc_val * PWM_ARR) / ADC_MAX;

    // 3. Apply new duty cycle — takes effect at next cycle boundary
    //    (safe because we enabled OCPreload in CubeMX)
    __HAL_TIM_SET_COMPARE(&PWM_TIMER, PWM_CHANNEL, ccr_val);

    // 4. Small delay to avoid hammering the register every CPU cycle
    //    (optional, but good practice)
    HAL_Delay(1);
}
/* USER CODE END WHILE */
```

### Smooth Version (with software low-pass filter)

```c
// Prevents flickering from ADC noise using exponential moving average
#define ALPHA   8U   // Filter strength (higher = smoother, slower response)

uint32_t filtered_ccr = 0;

while (1) {
    uint32_t adc_val = HAL_ADC_GetValue(&hadc1);
    uint32_t new_ccr = (adc_val * PWM_ARR) / ADC_MAX;

    // Exponential Moving Average: filtered = (old × (ALPHA-1) + new) / ALPHA
    filtered_ccr = ((filtered_ccr * (ALPHA - 1)) + new_ccr) / ALPHA;

    __HAL_TIM_SET_COMPARE(&PWM_TIMER, PWM_CHANNEL, filtered_ccr);
    HAL_Delay(1);
}
```

---

## 12. Common Pitfalls & Gotchas

### ❌ Mistake 1: Forgetting `HAL_TIM_PWM_Start()`

```c
// ❌ WRONG — timer is initialized but no signal on pin
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 500);  // Nothing happens!

// ✅ CORRECT — start the PWM first
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);            // Pin now outputs signal
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 500);   // Now this works
```

### ❌ Mistake 2: Setting CCR > ARR

```c
// ARR = 999 (period count)

// ❌ WRONG — CCR > ARR means pin is ALWAYS HIGH (100% but uncontrolled)
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 1500);  // 1500 > 999 = always HIGH

// ✅ CORRECT — clamp CCR to ARR maximum
uint32_t ccr = MIN(calculated_ccr, htim1.Init.Period);
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, ccr);
```

### ❌ Mistake 3: Wrong Channel Number

```c
// ❌ WRONG — mismatched instance and channel
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, 500);  // htim1 CH2 is a DIFFERENT pin!

// ✅ CORRECT — verify your CubeMX pin assignment
// PA8 = TIM1_CH1 → use TIM_CHANNEL_1
// PA9 = TIM1_CH2 → use TIM_CHANNEL_2
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 500);
```

### ❌ Mistake 4: ADC Not Started

```c
// ❌ WRONG — HAL_ADC_GetValue() returns stale/garbage data
uint32_t val = HAL_ADC_GetValue(&hadc1);  // ADC was never started!

// ✅ CORRECT — start ADC before the while loop
HAL_ADC_Start(&hadc1);       // Single mode: start before each read
// OR:
HAL_ADC_Start_DMA(&hadc1, &adc_buffer, 1);  // Continuous DMA mode
```

### ❌ Mistake 5: Glitch from Missing Preload

```
  Without Preload:  CCR can change IMMEDIATELY mid-cycle → glitchy output
  With Preload:     CCR shadow register loads on next update → clean output

  Check in CubeMX:  Timer → PWM Generation CHx → "CH Polarity": OCPreload = ENABLE
  Check in code:    sConfig.OCPreload = TIM_OCPRELOAD_ENABLE;
```

---

## 13. Quick Reference Cheat Sheet

```
╔═══════════════════════════════════════════════════════════════════════╗
║              STM32 PWM — QUICK REFERENCE CHEAT SHEET                 ║
╠═══════════════════════════════════════════════════════════════════════╣
║  FORMULAS                                                             ║
║  ─────────────────────────────────────────────────────────────────── ║
║  f_timer  = f_cpu / (PSC + 1)                                        ║
║  f_pwm    = f_timer / (ARR + 1)                                       ║
║  duty %   = CCR / (ARR + 1) × 100                                    ║
║  CCR      = (duty% / 100) × (ARR + 1)                                ║
╠═══════════════════════════════════════════════════════════════════════╣
║  MACROS (on-the-fly, timer keeps running)                             ║
║  ─────────────────────────────────────────────────────────────────── ║
║  __HAL_TIM_SET_COMPARE(&htimX, TIM_CHANNEL_Y, ccr);    // duty      ║
║  __HAL_TIM_SET_AUTORELOAD(&htimX, arr);                 // frequency ║
║  __HAL_TIM_SET_PRESCALER(&htimX, psc);                  // clk div   ║
╠═══════════════════════════════════════════════════════════════════════╣
║  START / STOP                                                         ║
║  ─────────────────────────────────────────────────────────────────── ║
║  HAL_TIM_PWM_Start(&htimX, TIM_CHANNEL_Y);              // standard  ║
║  HAL_TIM_PWM_Start_IT(&htimX, TIM_CHANNEL_Y);           // +IRQ      ║
║  HAL_TIM_PWM_Start_DMA(&htimX, CH, buf, len);           // +DMA      ║
║  HAL_TIM_PWM_Stop(&htimX, TIM_CHANNEL_Y);               // stop      ║
╠═══════════════════════════════════════════════════════════════════════╣
║  DIRECT REGISTER ACCESS (fastest)                                     ║
║  ─────────────────────────────────────────────────────────────────── ║
║  TIM1->CCR1 = value;    // duty cycle ch1                            ║
║  TIM1->ARR  = value;    // period/frequency                          ║
║  TIM1->PSC  = value;    // prescaler                                 ║
║  TIM1->EGR  = TIM_EGR_UG;  // force update event                    ║
╠═══════════════════════════════════════════════════════════════════════╣
║  CALLBACKS                                                            ║
║  ─────────────────────────────────────────────────────────────────── ║
║  HAL_TIM_PeriodElapsedCallback()      // fires at end of each period ║
║  HAL_TIM_IC_CaptureCallback()         // fires on each edge (IC mode)║
║  HAL_TIM_PWM_PulseFinishedCallback()  // fires at DMA buffer end     ║
╠═══════════════════════════════════════════════════════════════════════╣
║  PRELOAD  (always enable for glitch-free real-time updates)          ║
║  ─────────────────────────────────────────────────────────────────── ║
║  sConfig.OCPreload = TIM_OCPRELOAD_ENABLE;                           ║
║  htim1.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Frequency Reference Table

> Assuming 72 MHz CPU clock (STM32F103 / STM32F401 @ max speed)

| PSC  | Timer Clock | ARR   | PWM Frequency | Use Case              |
|------|-------------|-------|---------------|-----------------------|
| 0    | 72 MHz      | 719   | 100 kHz       | High-speed SMPS       |
| 71   | 1 MHz       | 999   | 1 kHz         | LED dimming, general  |
| 71   | 1 MHz       | 19999 | 50 Hz         | RC servo (1ms–2ms)    |
| 719  | 100 kHz     | 99    | 1 kHz         | Audio PWM buzzer      |
| 7199 | 10 kHz      | 99    | 100 Hz        | Slow motor control    |

---

*Generated as a living reference document — update CCR values, ARR, and PSC to match your specific STM32 variant and clock configuration.*

> 📌 **Tip:** Always check the **Reference Manual** for your specific STM32 family  
> (e.g., RM0008 for STM32F1, RM0383 for STM32F4) when dealing with advanced timer  
> features like complementary outputs, dead-time, and break inputs.
