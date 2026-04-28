# DTMF Secure Access System

A real-time DTMF tone detection and passcode validation system built on the STM32 Nucleo-L476RG microcontroller. The system listens for DTMF tones via an analog MEMS microphone, analyzes them using FFT, and grants or denies access based on a configurable 4-digit passcode — with LED and audio feedback.

---

## Features

- Real-time 20 kHz audio sampling via ADC + DMA
- FFT-based DTMF frequency detection using the ARM CMSIS DSP library
- Supports the full 16-key DTMF matrix (`0–9`, `A–D`, `*`, `#`)
- 4-digit configurable passcode
- Visual feedback via the on-board LED (solid on = correct, 5 Hz blink = wrong)
- Audio feedback through a DAC-driven speaker (1 kHz beep / 300 Hz buzz)
- UART debug output over ST-LINK virtual COM port
- Software button debouncing

---

## Hardware

| Component | Details |
|---|---|
| Microcontroller | STM32 Nucleo-L476RG (64-pin) |
| Microphone | ICS-40180 analog MEMS breakout (analog out → PA0 / ADC1_IN5) |
| Amplifier | PAM8302 Class-D audio amplifier board |
| Speaker | 8Ω mono enclosed speaker |
| Interface | USB Type-A/C to Mini-B (ST-LINK) |

### Wiring Summary

```
Microphone  VCC  → Nucleo 3.3V
Microphone  GND  → Nucleo GND
Microphone  OUT  → PA0 (ADC1 Channel 5)

PAM8302     AUD IN  → PA4 (DAC1 Channel 1)
PAM8302     GND     → Nucleo GND
PAM8302     VCC     → Nucleo 3.3V
PAM8302     OUT+/-  → Speaker terminals
```

---

## Software Architecture

```
Button Press (EXTI PC13)
        │
        ▼
ADC DMA Capture (2048 samples @ 20 kHz)
        │
        ▼
Normalise to [-1.0, +1.0]
        │
        ▼
Real FFT (ARM CMSIS DSP arm_rfft_fast_f32)
        │
        ▼
Magnitude Spectrum Analysis
    ├── Find dominant row freq (650–960 Hz)
    └── Find dominant col freq (1170–1660 Hz)
        │
        ▼
Map to DTMF key character
        │
        ▼
Accumulate digits (state machine)
        │
        ▼
Passcode Validation → LED + Audio Feedback
```

---

## Configuration

All tunable parameters are `#define` constants at the top of `dtmf_secure_access.c`:

| Constant | Default | Description |
|---|---|---|
| `FFT_FULL_SIZE` | 2048 | FFT buffer length (must be power of 2) |
| `SAMPLE_RATE_HZ` | 20000 | ADC sampling rate in Hz |
| `PASSCODE_LENGTH` | 4 | Number of digits in the passcode |
| `FREQ_TOLERANCE_HZ` | 50.0 | ±Hz tolerance for DTMF matching |
| `MIN_MAGNITUDE` | 10.0 | Minimum FFT magnitude to accept a tone |
| `BEEP_FREQ_HZ` | 1000 | Success feedback tone (Hz) |
| `BUZZ_FREQ_HZ` | 300 | Error feedback tone (Hz) |
| `DEBOUNCE_MS` | 200 | Button debounce window (ms) |
| `DEBUG_UART_ENABLED` | 1 | Set to 0 to disable UART output |

### Changing the Passcode

Edit the `CORRECT_PASSCODE` array in `dtmf_secure_access.c`:

```c
static const char CORRECT_PASSCODE[PASSCODE_LENGTH] = { '1', '2', '3', '4' };
```

---

## Build & Flash

### Requirements

- [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html)
- CMSIS DSP library (included in STM32Cube firmware package)
- STM32 HAL drivers for L476RG

### Steps

1. Create a new STM32CubeIDE project targeting the Nucleo-L476RG.
2. In CubeMX, configure:
   - **ADC1** – continuous DMA mode, triggered by TIM6, 12-bit resolution
   - **TIM6** – period set to yield a 20 kHz trigger (ARR = `(SystemClock / 20000) - 1`)
   - **DAC1** – channel 1 with DMA request enabled
   - **EXTI** – PC13 (user button) falling edge interrupt
   - **USART2** – 115200 baud (ST-LINK virtual COM)
3. Enable the **CMSIS DSP** middleware library in the project settings.
4. Copy `dtmf_secure_access.c` into the `Core/Src/` directory.
5. Build and flash via **Run → Debug** or the flash button.

---

## Usage

1. Open a serial terminal at **115200 8N1** on the ST-LINK COM port to see debug output.
2. Open a DTMF tone generator (e.g., [onlinetonegenerator.com/dtmf.html](https://onlinetonegenerator.com/dtmf.html)) on a phone or laptop.
3. Hold a DTMF key on the generator, then press the blue button on the board.
4. Release the button after ~0.5 s. The system captures and analyzes the tone.
5. Repeat for each of the 4 digits.
6. After the 4th digit:
   - **Correct passcode** → 1 kHz beep + LED solid on for 2 s
   - **Wrong passcode** → 300 Hz buzz + LED blinks 5× at 5 Hz

---

## DTMF Frequency Reference

| | 1209 Hz | 1336 Hz | 1477 Hz | 1633 Hz |
|---|---|---|---|---|
| **697 Hz** | 1 | 2 | 3 | A |
| **770 Hz** | 4 | 5 | 6 | B |
| **852 Hz** | 7 | 8 | 9 | C |
| **941 Hz** | * | 0 | # | D |

---

## Project Structure

```
├── Core/
│   ├── Inc/
│   │   └── main.h
│   └── Src/
│       ├── main.c                  ← CubeMX-generated init stubs
│       ├── dtmf_secure_access.c    ← This project's logic
│       └── stm32l4xx_it.c
├── Drivers/
│   ├── CMSIS/
│   └── STM32L4xx_HAL_Driver/
└── README.md
```

