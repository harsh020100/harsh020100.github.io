---
layout: post
title: "TP-Link Router Hardware Pentesting: Exploiting an Unauthenticated UART Console"
date: 2026-08-17
categories: [IoT Security, Hardware Pentesting]
tags: [UART, Serial Console, Hardware Hacking, TP-Link, Router Security, picocom, Multimeter, Pinout, USB-TTL, Firmware]
author: harsh
pin: false
image:
  path: https://github.com/user-attachments/assets/524fd21b-1950-400e-854c-f760e335ba41
  alt: TP-Link Router UART Hardware Hacking
---

Every embedded device has a debug interface hiding somewhere on the board. Manufacturers need it to flash firmware and run diagnostics on the production line — and more often than you'd expect, that same interface ships live in the product sitting in your living room.

This post is a full walkthrough of how I identified an unmarked, unpopulated **UART header** on a **TP-Link TL-WR845N** router using nothing but a multimeter, wired it up to a USB-TTL adapter, and dropped straight into a **root shell with zero authentication** — which then handed me the router's live Wi-Fi password in plaintext.

No exploit code. No firmware vulnerability. Just four pads and a bit of patience.

---

## What You Need

- The target device, opened up, with the PCB exposed
- A digital multimeter (continuity + DC voltage modes)
- A 3.3V USB-to-TTL / USB-to-Serial adapter (**not** a 5V one — most embedded SoCs run their logic at 3.3V, and 5V on the wrong pin can kill the chip)
- Jumper wires / male-to-female dupont cables
- A serial terminal — I used `picocom` on Kali Linux
- Patience, a steady hand, and a device you own or are authorized to test
![TP-Link TL-WR845N router opened up, showing the PCB with the main SoC, flash chip, and antenna connectors exposed](https://github.com/user-attachments/assets/8d284e40-bff4-428b-acf0-6ef7147ae2d7){: .shadow }

---

## Step 1: Spotting the Candidate Header

Once the case is open, the first thing to look for is any row of **3–5 unpopulated through-holes or pads** near the main SoC, flash chip, or edge of the board — often with silkscreen markings like `J1`, `UART`, `TP1`–`TP4`, or sometimes no markings at all. On the TL-WR845N, I found exactly this: a 4-pin header with no silkscreen labels, sitting quietly near the main chipset.

![Unpopulated 4-pin header identified as the likely UART interface, marked for reference](https://github.com/user-attachments/assets/c2d788b9-9ad0-4769-b24f-df7ab3ec7f07){: .shadow }

A 4-pin unmarked header next to the SoC is the classic signature of `VCC`, `GND`, `TX`, `RX` — the standard UART quartet. The only question left is **which pin is which**, and that's where the multimeter comes in.

---

## Step 2: Identifying Each Pin with a Multimeter

This is the part most tutorials rush past, so here's exactly how I worked through it, pin by pin.

### Finding GND — Continuity Mode

With the router **fully powered off**, I switched the multimeter to continuity mode (the setting that beeps when two points are electrically connected) and touched one probe to a known ground point — the shielding can over the RF section, or the negative terminal of the power barrel jack — and walked the other probe across each of the four pins.

Exactly one pin beeped, confirming continuity to chassis ground. That's **GND**.

### Finding VCC — DC Voltage Mode, Powered On

Next, with the router powered on, I switched to DC voltage mode and measured each remaining pin against GND.

- One pin read a **steady 3.3V**, unchanging — that's **VCC**, the logic-level supply rail for the SoC's serial interface.

I noted this pin but **did not connect it** to my adapter — most USB-TTL adapters don't need to supply or receive power on this line, and back-feeding voltage into a live board is one of the fastest ways to damage it. Leave VCC disconnected unless you specifically need to power the board from the adapter.

### Finding TX — Watching for Activity at Boot

With two pins left, I measured each against GND again — but this time while power-cycling the router and watching the meter during the boot sequence (a scope makes this easier, but a multimeter set to a fast-updating voltage read works too).

- One pin showed **rapid, erratic voltage fluctuation** during the first few seconds after power-on — that's the router actively transmitting boot log data. This is **TX** (from the router's perspective — it's the pin that talks).

### Finding RX — By Elimination

The one pin left, with the flattest, quietest voltage profile during boot, is **RX** — the router's receive line, which stays idle until something is sent to it.

### Summary Table

| Pin # | Signal | How Identified |
|---|---|---|
| 1 | TX (router → you) | Active/fluctuating voltage during boot |
| 2 | RX (you → router) | Idle, flat voltage — identified by elimination |
| 3 | GND | Continuity to chassis ground (meter beeps) |
| 4 | VCC (3.3V) | Steady 3.3V, powered on, unchanging |


---

## Step 3: Wiring the USB-TTL Adapter to the Router

This is the step that trips people up the most, because it's counter-intuitive: **TX connects to RX, and RX connects to TX** — never TX-to-TX. You're crossing the lines, the same way a null-modem cable works.

| Router Pin | USB-TTL Adapter Pin |
|---|---|
| GND | GND |
| TX  | RXD |
| RX  | TXD |
| VCC | *(leave unconnected)* |

![USB-TTL adapter wired to the router's UART header — GND-GND, router TX to adapter RXD, router RX to adapter TXD, VCC left floating](https://github.com/user-attachments/assets/7badb60a-303e-402a-9143-aab84dc4efb0){: .shadow }

A quick sanity check before powering anything on: confirm your adapter is set to **3.3V logic level**, not 5V — many adapters (like the common CP2102 and FTDI boards) have a physical jumper or switch for this. Running 5V into a 3.3V-rated SoC pin risks permanently damaging the router's UART transceiver.

With the wiring confirmed, I plugged the adapter into my Kali machine and confirmed the device enumerated:

```bash
ls /dev/ttyUSB*
```

---

## Step 4: Connecting with picocom

With everything wired up, I opened a serial session at the standard baud rate for this SoC family:

```bash
picocom -b 115200 /dev/ttyUSB0
```

![picocom session connecting to the router's UART header at 115200 baud, 8N1](https://github.com/user-attachments/assets/b163eeb0-1107-485a-a041-ca947e6c9a5c){: .shadow }

115200 8N1 (8 data bits, no parity, 1 stop bit) is the near-universal default for MediaTek/Ralink-based SoCs like the one in this router, and it worked immediately.

The moment I power-cycled the router, the full boot log started streaming into my terminal in real time — bootloader messages, flash partition reads, filesystem mounts, service startup — everything the device normally keeps to itself.

I never had to touch a key to interrupt the bootloader, break into U-Boot, or race a boot countdown — none of that was necessary. The board simply booted straight through, and Linux userspace came up and handed me a prompt directly on the serial line:

```
~ #
```

No login prompt. No username. No password. No boot interruption of any kind. Just power the router on, sit on the serial line, and wait a few seconds — a live, interactive **root shell**, handed over by the device itself as part of its normal boot process.

---

## Step 5: Confirming Root and Pulling the Wi-Fi Password

A quick directory listing confirmed this was a genuine root shell on the router's embedded Linux filesystem, not just a bootloader menu:

![Root shell output of ls showing standard embedded Linux directories](https://github.com/user-attachments/assets/dd19f70b-07a4-4c70-860e-1626a3453b43){: .shadow }

```
~ # ls
web    usr    sbin   mnt    lib    dev
var    sys    proc   linuxrc etc   bin
```

From there, reading the live wireless configuration file was a single command:

```bash
cat /var/Wireless/RT2860AP/RT2860AP.dat
```

Which returned the router's entire radio and security configuration — SSID, encryption mode, and the live pre-shared key — in plaintext:

```
SSID1=TP-Link_Testing
AuthMode=WPA2PSK;OPEN
EncrypType=AES;NONE
WPAPSK1=harshsecurity.com/password
```
![Wifi Password](https://github.com/user-attachments/assets/224283eb-51e7-42b5-b6d3-937156b4aad8){: .shadow }

![Terminal output confirming the WPAPSK1 field — the live Wi-Fi password — read directly from the router's config file over the serial console](https://github.com/user-attachments/assets/90ad3e29-b728-4c0f-9aaf-aead3d76c33b){: .shadow }

`WPAPSK1` is the field that matters: the active WPA2 pre-shared key for the router's primary network, retrieved without a single dictionary word tried, without touching `aircrack-ng`, and without ever interacting with the wireless interface at all. Physical UART access made the entire WPA2 handshake-cracking problem irrelevant.

---

## Why This Was Possible

This isn't a firmware exploit — there's no memory corruption or logic flaw to patch. It's an **access control decision that was never made**:

- The UART console shipped active in retail firmware with no login prompt (`getty`/inittab not configured to require authentication on this port)
- No console lockout, timeout, or physical tamper detection
- The Wi-Fi PSK and other sensitive config values are stored in plaintext flat files rather than encrypted or access-controlled storage

Once a root shell is available, the device's entire security model — WPA2, admin panel auth, everything — is moot. You're not attacking any of those mechanisms; you're just reading the answers off disk.

---

## Recommendations for Manufacturers

- Require authentication (or disable the console entirely) on production UART interfaces — `#ifdef DEBUG` it out of retail firmware builds, don't just "not document" it
- If a serial console must ship for RMA/support reasons, gate it behind a login and audit access
- Never store PSKs, admin credentials, or API keys in plaintext on flash — use hashed or encrypted storage where the platform allows it
- Consider physically removing or epoxy-covering debug headers on retail boards where a console isn't needed post-manufacturing

---

*All testing performed on a self-owned lab device. Do not attempt UART/JTAG access on hardware you do not own or are not explicitly authorized to test.*