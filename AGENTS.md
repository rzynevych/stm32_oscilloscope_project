# Project instructions

## Overview

This project is a USB oscilloscope built from two Git submodules:

- `stm32scope/`: STM32F411 firmware in C using STM32 HAL and FreeRTOS.
- `stm32scopeUI/`: Qt/C++ desktop application for device control, waveform display, and measurements.

The root repository tracks the submodule revisions. Component changes belong in their respective submodule repositories.

## Architecture and data flow

Firmware samples ADC1 channel 8 from TIM2 TRGO into circular DMA. DMA callbacks copy samples into a ring buffer; a FreeRTOS task searches for a trigger, packages 1,000 samples, and transmits them through USB CDC. A second task handles host commands.

The desktop opens the CDC serial port with `QSerialPort`. `SamplingDevice` parses messages and sends commands; `OscilloscopeCore` converts samples to voltage/time points and calculates measurements; `MainWindow` handles controls and QCustomPlot rendering.

Host/device protocol definitions are duplicated in `stm32scope/App/oscilloscope_protocol.h` and `stm32scopeUI/SamplingDeviceProtocol.h`. Keep message types, sizes, enum values, byte order, and CRC behavior synchronized when changing the protocol. Data messages carry a 1-byte sequence, 2-byte data length, 4-byte trigger-search time, 1,000 16-bit samples, and a 2-byte footer. Host commands use a length byte and CRC-8; firmware setting responses do not use a footer or CRC.

## Important constraints

- Target MCU: STM32F411; board HSE is 25 MHz. ADC input range is 0 to 3.3 V.
- The analog frontend attenuates and offsets the input around a 1.65 V virtual ground, clamps overvoltage, and buffers the ADC input with an LM358. The UI currently maps samples to approximately +/-4 V; treat frontend scaling as hardware-dependent.
- Capture size is fixed at 1,000 samples. Current rate choices are 2 MSa/s, 1 MSa/s, 100 kSa/s, and 10 kSa/s. The 2 MSa/s rate is hardware-measured; USB throughput is sufficient for the current version.
- The firmware command receiver uses a fixed buffer and expects a command in a received CDC packet. Consider packetization and buffer limits when changing command framing.
- Firmware responses and host commands are not framed identically; use the implementations, not the README summary, as the protocol reference.

## General development guidelines

- Read the component `AGENTS.md` before changing that component.
- Keep changes focused and preserve STM32CubeMX-generated boundaries and `USER CODE` regions where applicable.
- When changing shared behavior or protocol fields, inspect and update both submodules in the same change set.
- Do not assume displayed voltage, trigger voltage, or sampling-period units are interchangeable; follow the conversion code at each boundary.
- No automated test suite is currently evident. For behavior changes, state what build, hardware check, or manual verification was performed; do not claim unrun checks.

## Git workflow

- `stm32scope` and `stm32scopeUI` are independent Git repositories/submodules. Component changes belong in the relevant submodule repository.
- Check `git status` in the root and in each affected submodule. Avoid accidentally committing generated build output or unrelated changes.
- Coordinate protocol changes across both repositories so the checked-out revisions remain compatible.
- Do not commit changes unless explicitly requested by the user.
- When the user requests commits, commit changes in all affected component repositories first.
- Update and commit the parent repository's submodule pointers only when the user explicitly requests committing changes to the root repository as well.
- Never include unrelated changes in commits.
- Before committing, inspect the status and diff of every affected repository.
- Before updating a submodule pointer, verify that it points to the intended component commit.