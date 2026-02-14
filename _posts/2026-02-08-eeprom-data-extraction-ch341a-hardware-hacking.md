---
layout: post
title: "EEPROM Memory Reading of a Printer using CH341A "
date: 2026-02-08
categories: [Hardware Security, IoT Pentesting, Hardware Hacking]
tags: [CH341A, EEPROM, 24C64, SPI Flash, Hardware Hacking, Reverse Engineering, Printer Security, EPROM Extraction, Harsh Srivastava, harshsecurity, Harsh]
author: harsh
pin: false
image:
  path: https://github.com/user-attachments/assets/c4e1d11f-43c7-4427-93f1-a28b5a883a79
  alt: EEPROM Memory Reading of a Printer using CH341A
---
Hardware security assessments often go beyond software and firmware into **direct chip-level analysis**. In this blog, I demonstrate a **real-world EEPROM extraction and analysis** performed on an **HP DeskJet series printer**, using a **CH341A USB programmer** and an **SOIC-8 test clip**.

This write-up is intended for **hardware security researchers, IoT pentesters, and reverse engineers** who want a clear, practical, and accurate reference.

> ⚠️ **Disclaimer:** This research was performed on personally owned hardware for educational purposes only. Do not attempt hardware extraction on devices you do not own or have permission to test.

---

## 🎯 Objective

- Identify memory components on a printer PCB
- Understand differences between **24xx EEPROM** and **25xx SPI Flash**
- Correctly wire **CH341A for 24C64 EEPROM**
- Extract EEPROM contents safely
- Perform basic data analysis on the dumped memory

---

## 🖨️ Device Overview

- **Device:** HP DeskJet 2130 Printer
- **Category:** Consumer embedded device
- **Focus:** Non‑volatile memory extraction

---

## 🔍 PCB Inspection (Front Side)

![PCB Front](https://github.com/user-attachments/assets/ae072473-4b82-4e85-90df-027fb9397483){: .shadow }

After opening the printer, the main PCB was exposed. A visual inspection was performed to locate components responsible for storage and firmware.

---

## 🔬 PCB Back Side – EEPROM Location

![PCB Back EEPROM](https://github.com/user-attachments/assets/2bcbc468-d01a-44d5-a1b2-dc6fe42b00eb){: .shadow }

On the rear side of the PCB, the **24C64 EEPROM** and **SPI Flash Memory** was identified.

---

## 🧠 Memory Types Identified

### 1️⃣ EEPROM (Configuration Memory)

- **Part Number**: `24C64WP`
- **Type**: I²C EEPROM
- **Capacity**: 64 Kbit
- **Purpose**:
  - Device configuration
  - Calibration values
  - Counters & flags
  - Model‑specific parameters

This was the **primary target** for this exercise.

### 2️⃣ SPI Flash Memory (Firmware Storage)

- **Part Number**: `MXIC MX25L1606E`
- **Type**: CMOS Serial Flash (SPI)
- **Capacity**: 16 Mbit
- **Purpose**:
  - Firmware storage
  - Larger data sections

> ℹ️ Note: Although SPI flash is equally interesting, this blog focuses specifically on the **24C64 EEPROM**.

---

## 🔌 CH341A Pin Usage (Important)

![CH341A Pin Mapping](https://github.com/user-attachments/assets/4d222fc7-a23f-4ed6-ad70-e48c15d8bc41){: .shadow }

If we take the USB connector as reference, the first 8 pins (1–8) of the CH341A ZIF socket are for 25xxx SPI flash, and the next 8 pins (9–16) are for 24xxx I²C EEPROM

### Rule of Thumb

- **24xx I2C EEPROM** → Use the 24xx/I2C socket block (**Pin 9–16**)
- **25xx SPI Flash** → Use the 24xx/I2C socket block (**Pin 1-8**)

Incorrect pin usage will result in detection failure or corrupted reads.

---

## 🔗 SOIC-8 Clip Clip Placement

When using a SOIC-8 test clip with the CH341A programmer, no manual wiring is required. The clip cable is internally mapped for EEPROM/FLASH communication, so you only need to ensure correct alignment and socket placement. For 24xxx (I²C) EEPROMs, the SOIC-8 clip must be plugged into the 24xx section of the CH341A ZIF socket (pins 9–16). When viewed from the USB connector side, the right half of the socket is reserved for 24xxx devices, and Pin-1 of the EEPROM must be aligned with the 24xx block Pin-1, which corresponds to physical Pin-16 of the ZIF socket. To avoid confusion, always refer to the above image for correct orientation and placement.

🔴 **Red wire on SOIC clip indicates Pin-1 of the EEPROM/FLASH memory**

---

---

## 🧭 Placement Steps

- **1. Locate Pin-1 on the EEPROM chip**
  - Identified by a dot, notch, or bevel mark on the IC.
- **2. Locate the red wire on the SOIC clip**
  - The red wire indicates Pin-1 of the clip.
- **3. Attach the clip to the chip**
  - Ensure the red wire aligns with Pin-1 of the EEPROM.
- **4. Insert the clip cable into CH341A**
  - Place it in the 24xx / I²C socket block (Pins 9–16)
  - Place the pin 1 (Red Wire) of SOIC clip on the 24xx block pin 1 (refer to the  above image for correct orientation and placement) 
  - Do NOT insert it in the 25xx block.

---

## ⚠️ Voltage Safety Warning

- Many CH341A boards default to **5V** ❌
- **24C64 EEPROM requires 3.3V** ✅

> 🔥 Applying 5V can permanently damage the chip.

**Recommendation:**
- Use a 3.3V‑modded CH341A
- Or external 3.3V regulator

---

## ⚙️ EEPROM Reading Using CH341A

![CH341A Software Read](https://github.com/user-attachments/assets/ecfee5c2-7a51-46b4-8b98-8ee014de3378){: .shadow }

The CH341A programming software was configured with:

- Type: **24 EEPROM**
- Manu: **ST**
- Name: **ST24C64**

Click on **Read** and wait until the process reaches 100%. Once completed, the EEPROM contents are successfully dumped into a binary (.bin) file.

---

## 🧪 Data Analysis Using `strings`

![Strings Output](https://github.com/user-attachments/assets/ec97b0ca-cf45-43f9-95ab-6eab2a209366){: .shadow }

To analyze the extracted EEPROM data, the **Sysinternals `strings.exe`** utility was used:

```bash
strings.exe EEPROM_READ.bin
```

This helped identify:
- Human-readable strings
- Debug markers
- Configuration-related values

---

## 🔐 Security Observations

- EEPROM data is often stored **unencrypted**
- Physical access enables **direct data extraction**
- No authentication is required for EEPROM reads
- In-circuit access remains a common attack surface

---

## 🎥 Reference Video (Demonstration)

For a demonstration of this process, refer to my LinkedIn video:

🔗 [EEPROM Memory Reading using CH341A – Live Demo](https://www.linkedin.com/posts/harsh01200_hardwaresecurity-firmwareanalysis-reverseengineering-activity-7398271586342096898-t_2l)

---

## ✅ Conclusion

This case study demonstrates how **EEPROM memory from a printer** can be safely extracted using a **CH341A programmer**, provided the correct **pin mapping, voltage, and protocol** are used.

For **IoT pentesters and hardware security researchers**, EEPROM and flash memory analysis is a foundational skill that highlights why **hardware-level protections** are essential.

---

*Break responsibly. Secure deeply.* 🔐

