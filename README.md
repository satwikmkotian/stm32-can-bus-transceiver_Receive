# STM32 CAN Bus Transceiver — Transmit & Receive

A full-duplex CAN bus node built on the **STM32L452RE-P Nucleo** board using a **TJA1050 CAN transceiver**, verified end-to-end against a **PEAK PCAN-USB** adapter running **PCAN-View**.

The board both transmits CAN frames on a timer and receives frames from the bus via interrupt, confirming full two-way CAN communication in hardware.

---

## 🔧 Hardware

| Component | Details |
|---|---|
| MCU | STM32L452RE (Nucleo-64, "-P" variant) |
| CAN Transceiver | TJA1050 |
| Bus Analyzer | PEAK PCAN-USB + PCAN-View software |
| Bitrate | 500 kbit/s |

### Wiring

| TJA1050 pin | Nucleo pin | Notes |
|---|---|---|
| TX (CTX) | PA12 | CAN1_TX, AF9 |
| RX (CRX) | PA11 | CAN1_RX, AF9 |
| VCC | 5V | |
| GND | GND | shared with PCAN-USB ground |
| CANH / CANL | → bus / PCAN-USB | 120Ω termination at both ends of the bus |

> CAN1 on the STM32L452 only maps to **PA11 (RX)** / **PA12 (TX)** via AF9 — no other pins carry the CAN1 alternate function.

---

## 🖥️ Firmware overview

Built with **STM32CubeIDE** using the STM32 HAL library (`STM32L4xx_HAL_Driver`).

**CAN1 configuration:**
- Mode: Normal (not loopback/silent)
- Bitrate: 500 kbit/s → Prescaler = 10, TimeSeg1 = 13TQ, TimeSeg2 = 2TQ, SJW = 1TQ (assumes PCLK1 = 80 MHz)
- Accept-all filter configured on FIFO0 (bxCAN drops every frame until at least one filter bank is active)

**Behavior:**
- **Transmit**: sends an 8-byte data frame (ID `0x123`) every 1 second in the main loop
- **Receive**: `HAL_CAN_RxFifo0MsgPendingCallback()` fires on any incoming frame via the CAN1_RX0 NVIC interrupt, toggling the onboard LED (LD4) on every received message

Main logic lives in [`main.c`](main.c).

---

## ✅ Verification

Tested live against a PEAK PCAN-USB adapter running PCAN-View at 500 kbit/s.

- **STM32 → bus**: PCAN-View's Receive pane showed ID `123h`, 8 bytes, arriving every ~1 second — confirmed the board's transmit path.
- **PCAN-View → STM32**: sending a manual/cyclic frame from PCAN-View's Transmit pane visibly toggled the onboard LED — confirmed the board's receive/interrupt path.
- Bus status held steady at `OK` throughout.

<!-- Add screenshots here, e.g.:
![PCAN-View showing received frames](images/pcan_view_receive.png)
![Hardware setup](images/hardware_setup.jpg)
-->

---

## 🐛 Debugging notes / lessons learned

A few real issues hit and fixed during development, kept here for reference:

- **Wrong RX pin (PB12 instead of PA11)**: PB12 has no CAN alternate function on this MCU — the peripheral was configured correctly but physically listening to nothing. Fixed by wiring RX to PA11.
- **No accept filter configured**: the bxCAN peripheral silently discards every incoming frame at the hardware level unless at least one filter bank is explicitly enabled — independent of whether interrupts are turned on.
- **Missing NVIC interrupt**: `HAL_CAN_ActivateNotification()` alone isn't enough — the `CAN1 RX0` interrupt also has to be enabled in NVIC (CubeMX `.ioc` → System Core → NVIC) for the receive callback to ever fire.
- **`BUSHEAVY` status on PCAN-View**: turned out to be a wiring/continuity issue introduced while moving the RX wire, not a firmware bug — isolated by re-flashing a known-working transmit-only build onto the same wiring.

---

## 📂 Files

- `main.c` — application logic (CAN init, filter, TX, RX callback)
- `main.h` — pin/peripheral definitions
- `stm32l4xx_it.c` — interrupt handlers (auto-generated, CAN1_RX0_IRQHandler included)
- `stm32l4xx_hal_conf.h` — HAL module configuration

---

## 📝 License

MIT
