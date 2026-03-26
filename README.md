# STM32 Bare-Metal Hardware Interrupts & Analog Watchdog

## Overview
This repository contains a Safety-Critical, bare-metal implementation of a hardware-supervised safety system. It demonstrates a progression from basic CPU polling to a highly optimized zero-polling architecture using the STM32F401RE microcontroller, completely bypassing the STM32 HAL( Hardware Abstraction Layer).

## Development & Testing Progression
This project was built in sequential testing phases to empirically prove the efficiency of hardware-level event handling over CPU polling:

* **Phase 1: Bare-Metal Polling (The Baseline)**
  * Implemented a standard `while(1)` loop to continuously poll the `PC13` GPIO Input Data Register (`IDR`). 
  * *Result:* CPU was locked at 100% utilization just checking for button states.
* **Phase 2: EXTI Hardware Interrupts**
  * Configured the `SYSCFG` and `NVIC` to route the physical `PC13` button press directly to the Cortex-M4 core via `EXTI15_10_IRQHandler`.
  * *Result:* Eliminated button polling, freeing CPU cycles.
* **Phase 3: ADC Single Conversion & Polling**
  * Built a bare-metal ADC1 driver to read an analog potentiometer (`PA0`) using 8-bit resolution. 
  * *Result:* Successfully mapped physical voltage to a 0-255 digital scale, but still required CPU polling (`ADC_SR`) to verify conversion completion.
* **Phase 4: Analog Watchdog & Dual-Interrupt Harmony (Final Architecture)**
  * Configured the ADC Continuous Conversion engine and programmed the hardware Analog Watchdog (AWD) window comparator.
  * *Result:* Achieved a zero-polling system-critical safe state. The system autonomously monitors voltage and fires `ADC_IRQHandler` to trigger an alarm LED, requiring an EXTI button press to acknowledge and clear.

## Hardware Architecture
* **Microcontroller:** STM32F401RE (Nucleo-64)
* **Inputs:** * Rotary Potentiometer (`PA0` - ADC1 Channel 0)
  * User Push Button (`PC13` - EXTI13)
* **Outputs:** Onboard Warning LED (`PA5`)

## Hardware Limitations & Silicon Discoveries
* **The 8-Bit Truncation Quirk:** The STM32 Reference Manual states the Analog Watchdog compares voltages "before alignment" (implying a 12-bit static comparator). However, physical testing proved that when `ADC_CR1` is set to 8-bit mode, the silicon dynamically truncates the AWD comparator. 
* **Resolution:** Thresholds (`ADC_HTR`, `ADC_LTR`) must not be zero-padded to 12-bit (e.g., `200U << 4`), as this causes a permanent false-positive breach. They must be fed absolute 8-bit integer values (e.g., `200U`, `50U`).

## Future Improvements (Tier 2 Integration)
* **Safe State Fallbacks (ISO 14971):** To be implemented when transitioning to Unity/Ceedling testing to ensure the system detects if a physical sensor wire is pulled out or disconnected.
* **Low Power States:** Integration of `__asm volatile ("wfi")` (Wait For Interrupt) to halt the CPU clock during
