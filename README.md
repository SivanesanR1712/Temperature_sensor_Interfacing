# DHT22 ↔ STM32F411 (Register-Level) – README
This project demonstrates bare-metal one-wire style communication between an STM32 (STM32F411) and a DHT22/AM2302 temperature-humidity sensor. The DHT22 uses a timing-based digital protocol (no I²C/SPI/UART), so the STM32 must generate an accurate start pulse and then decode 40 bits by measuring microsecond-level HIGH pulse widths (≈24 µs for ‘0’, ≈65 µs for ‘1’).

Key highlights:
Uses a hardware timer (TIM2) for µs timing and edge/timeout handling.
Implements the full frame decode: humidity + temperature + checksum validation.
Shows practical embedded lessons: GPIO open-drain, pull-up behavior, pin selection pitfalls (e.g., avoiding pins tied to onboard UART), and robust timeout-based polling.
You can use it as a reference for sensor interfacing, precise timing, and protocol decoding on STM32 without HAL drivers.

## Tools & Hardware Used
### Development / Debug
- **STM32CubeIDE**  
  Build, flash, and debug firmware (register-level development and variable watch).
### Signal Observation
- **PulseView** (Sigrok)  
  Verify the DHT22 waveform and confirm pulse widths (0/1 bit timing) against captured timer values.
### Target Hardware
- **STM32F411E Discovery board**  
  Used for GPIO control, timer input capture, and interrupt handling.
### Sensor
- **DHT22 (AM2302) temperature & humidity sensor**  
  Single-wire digital sensor producing a 40-bit data frame.
  
## 1) Hardware Needed

- STM32F411E Discovery board
- DHT22 sensor (module or bare sensor)
- Jumper wires
- **Pull-up resistor (recommended): 4.7kΩ–10kΩ** from DATA to VCC

### Wiring (3.3V logic)
| DHT22 Pin | Connect to STM32 | Notes                        |
| VCC       |       3.3V       |DHT22 typically works at 3.3V |
| GND       | GND              | Common ground is mandatory |
| DATA      | **PA0**          | Use TIM2_CH1 on PA0 (AF1) |

 If your DHT22 is a breakout module, it may already include a pull-up resistor. If not sure, add an external pull-up.

---

## 2) How DHT22 Communication Works

DHT22 uses a **single-wire bidirectional** data line (open-drain style).

### Transaction Overview
1. **MCU Start Signal**
   - Drive DATA **LOW for ≥1 ms**
   - Release DATA (let it go **HIGH**) for **20–40 µs**
2. **Sensor Response**
   - Sensor pulls LOW ~80 µs
   - Sensor pulls HIGH ~80 µs
3. **40 Data Bits**
   Each bit:
   - LOW ~50 µs
   - HIGH pulse width encodes the bit:
     - **~26–28 µs → bit 0**
     - **~70 µs → bit 1**
4. **End of frame** and line returns HIGH.

---

## 3) Why Timer Input Capture (Best Practice)

A naive loop-count method can work, but it is sensitive to:
- compiler optimization
- interrupt jitter
- instruction timing

**TIM input capture** timestamps edges in hardware, giving stable microsecond-level pulse width measurement.

### What you measure
For each bit you care about the **HIGH pulse width**:
- Capture **rising edge** timestamp
- Capture **falling edge** timestamp
- Width = t_fall - t_rise
---

## 4) MCU Configuration Summary (Concept)

### A) GPIO for Start Signal (Open-Drain Output)
During start request:
- Configure PA0 as **GPIO output**
- Set **open-drain** mode
- Drive low ≥1 ms
- Then write “1” (release line high while still OD) for 20–40 µs

### B) Switch PA0 to Timer Alternate Function
After start:
- Configure PA0 as **Alternate Function**
- Select **AF1** for **TIM2_CH1**
- Enable pull-up (internal optional; external recommended)

### C) TIM2 Input Capture (Channel 1)
- **TIM2_CH1** mapped to **TI1** (CC1S = 01)
- Capture on **both edges** (rising & falling) or switch edge polarity in software
- Enable **CC1 interrupt** (CC1IE)
- Ensure timer runs with known tick rate (ideally **1 µs per tick**)

---

## 5) Timer Tick Rate (Very Important)

 **Target:** 1 tick = 1 µs

Compute TIM2 prescaler:
-PSC = (TIM2CLK / 1_000_000) - 1

> On STM32F4, TIM2CLK depends on APB1 prescaler:
> - If APB1 prescaler = 1 → TIM2CLK = PCLK1
> - Else → TIM2CLK = 2 × PCLK1

**Sanity check:** In debugger, TIM2->CNT should increment at ~1 count per µs.

---

## 6) Interrupt Handling (Edge Differentiation)

When capturing both edges:
- On interrupt, check the pin level (GPIOA->IDR bit0):
  - If pin is HIGH → **rising edge capture**
  - If pin is LOW → **falling edge capture**
- Store t_rise` on rising edges
- On falling edge, compute width = t_fall - t_rise

### About CC1IF clearing
In input capture mode, **CC1IF** can be cleared by:
- reading `CCR1`, and/or
- writing to clear the SR flag (device behavior dependent)

**Debugging tip:** Watching `CCR1` live in CubeIDE can clear CC1IF due to debugger reads.

---

## 7) Data Buffering Strategy

You will typically store:
- 1 “response high” width (~80 µs)
- then 40 widths for 40 bits

Example buffers:
- time[]: raw high pulse widths (microseconds)
- bits[]: decoded 0/1 bits

### When to start decoding
Do **not** use a fixed delay.
Decode when enough pulses have been captured:
- Wait until count >= 41 (1 sync + 40 bits), then decode.

## 8) Decode Bits → Bytes → Physical Values

### Bit decoding threshold
A robust method:
- if HIGH width < ~45–50 µs → **0**
- else → **1**

### Bitstream format (40 bits)
Bytes (MSB-first per byte):
1. Humidity high byte
2. Humidity low byte
3. Temperature high byte
4. Temperature low byte
5. Checksum

### Convert to physical values (DHT22)
- RH_raw = (Hh << 8) | Hl
- RH_% = RH_raw / 10.0

Temperature:
- T_raw = (Th << 8) | Tl
- If sign bit set, temperature is negative
- T_C = magnitude / 10.0

Checksum:
- sum = (Hh + Hl + Th + Tl) & 0xFF
- Must equal checksum



## 9) Common Problems & Fixes

### A) Pulses visible in PulseView but no ISR
Check:
- PA0 MODER is AF mode
- PA0 AFRL[3:0] = AF1
- TIM2 CC1S = TI1
- CC1E enabled
- CC1IE enabled
- NVIC enabled for TIM2
- Global interrupts enabled (PRIMASK = 0)

### B) ISR fires during GPIO configuration
Cause: changing PUPDR/MODER can cause edges.
Fix: configure GPIO first, then clear:
- TIM2 SR flags
- NVIC pending IRQ
Then enable IRQ.




## 10) Debugging Tips (STM32CubeIDE + PulseView)

### Use a clean GPIO (PA0) for DHT22 instead of UART pins, then switch it to TIM2_CH1 input capture to decode the waveform reliably.
PA0 is shared between:
- **GPIO mode** (used to send the DHT22 start pulse)
- **TIM2_CH1 AF mode** (used to capture sensor pulses)

To keep behavior consistent while debugging:
- Keep the start phase short (drive low, release, then immediately switch to AF).
- Avoid pausing the CPU during the active 40-bit burst (breakpoints can cause missed captures / CC1OF).
- If you need a breakpoint, place it **after** the burst (after count reaches the expected value).

### Practical debugging workflow
- Watch count first.
- Only inspect arrays (time[], bits[]) after the CPU is halted.
- Avoid live-viewing TIM2->CCR1 during capture; debugger reads can clear CC1IF.

### “Target not available” in watch window
- This usually happens when the target is running. **Suspend/Halt** first, then expand arrays.

### Confirm timing against PulseView
- Compare time[] HIGH widths with the PulseView measurement:
  - bit 0 cluster: ~20–35 µs
  - bit 1 cluster: ~60–80 µs
- If clusters drift, verify timer tick (PSC) and check pull-up strength.


## 11) Expected Signal Signature

In time[] (HIGH widths):
- One early pulse around ~70–90 µs (sensor response HIGH)
- Then 40 values in two clusters:
  - ~20–35 µs (bit 0)
  - ~60–80 µs (bit 1)
  - exmaple outputs:
  - Name : bits
	     Details:{0, 0, 0, 0, 0, 0, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 1, 0, 1, 1, 1, 0, 1, 0, 0, 0}
  -	Name : time
      Details:{72, 24, 24, 24, 23, 24, 23, 66, 23, 66, 66, 24, 66, 66, 24, 66, 66, 24, 24, 24, 24, 24, 24, 24, 65, 24, 23, 24, 23, 66, 23, 66, 24, 65, 65, 66, 23, 65, 23, 23, 23}
  - Name: humd
    decimal:731
  -Name:temp
     decimal:266	

- debugging with a logic analyzer (PulseView)

  


