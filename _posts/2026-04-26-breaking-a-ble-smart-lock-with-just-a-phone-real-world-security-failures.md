---
layout: post
title: "Breaking a BLE Smart Lock with Just a Phone: Real-World Security Failures"
date: 2026-04-25
categories: [IoT Security, Pentesting]
tags: [BLE, Bluetooth Low Energy, IoT Security, Smart Lock, GATT, Replay Attack, Hardcoded Credentials, nRF Connect]
author: harsh
pin: false
image:
  path: https://github.com/user-attachments/assets/4ca76d1a-1656-4c22-952e-39b5f475b3f6
  alt: BLE Smart Lock Pentesting
---

I built an intentionally vulnerable BLE smart lock on an ESP32 and then broke into it using nothing but my phone.

No custom scripts. No Linux machine. No Flipper Zero. Just **nRF Connect** — a free app you can grab from the Play Store right now — and about ten minutes of poking around.

This post walks through exactly what I did, why it worked, and what it tells us about how BLE security actually fails in the real world.

---

## First, a Quick Primer on How BLE Works

Before we get into the fun part, let us understand what we are dealing with.

BLE devices communicate through something called **GATT** — the Generic Attribute Profile. Think of it as a structured list of data slots called **characteristics**. Each characteristic holds a piece of information or accepts a command. Some are readable, some are writable, some fire off notifications when something changes.

Every BLE peripheral also broadcasts **advertisement packets** constantly — even when no one is connected. These are tiny radio frames that say "hey, I exist, here is my name, here are my services." Your phone picks these up when you open nRF Connect and tap scan.

That advertisement phase, before any connection, is where our story begins.

---

## The Target: VulnLock-SmartDoor-v1

![BLE smart lock on an ESP32 ](https://github.com/user-attachments/assets/ec10904e-8b98-48fa-bea8-e0ded90e350b){: .shadow }

The device I built is an ESP32 running a BLE smart lock firmware, connected to a TM1637 four-digit display that gives real-time visual feedback — `LOCK`, `OPEN`, `CONN`, and so on. It is intentionally designed with real vulnerabilities that mirror what shows up in actual commercial smart lock security research.

The goal: approach it exactly as a stranger would. No source code, no documentation, no credentials. Just the device sitting there, broadcasting, and nRF Connect on my phone.

---

## Step 1: Scanning — The Device Tells You Everything Before You Even Connect

Open nRF Connect, tap **SCAN**, and do nothing.

Within a few seconds the device shows up in the list:

![nRF Connect scan list showing VulnLock-SmartDoor-v1 with RSSI and advertisement details](https://github.com/user-attachments/assets/bd9d3a03-5bb7-46c2-84ff-bfe6ab80fed7){: .shadow style="max-height: 500px; width: auto; display:block; margin:auto;"}

Look at what is sitting right there in the advertisement packet — the complete device name: `VulnLock-SmartDoor-v1`. Product line, model, version number, all broadcast to anyone within 50 metres. The service UUID `0000AA00` is also visible before I have connected to anything. The MAC address is static — same value every time the device boots, no rotation, no privacy mode.

This is already a problem. A real attacker now knows exactly what device this is, can look it up, find known vulnerabilities, and target it specifically — without having made a single connection. This is passive. The device cannot even detect that someone is looking.

The lesson here is simple: **your BLE device's advertisement is a public announcement**. Whatever you put in it, assume the whole world can read it.

---

## Step 2: Connecting and Seeing the Full Attack Surface

Now we actually connect. Tap the device in nRF Connect, tap **CONNECT**, and wait for service discovery to complete.

The TM1637 display on the device flickers to `CONN` — it knows someone connected. And nRF Connect shows us the full GATT structure:

![nRF Connect showing the expanded service 0000AA00 with all six characteristics listed](https://github.com/user-attachments/assets/f9e26f06-e10d-4155-ba03-6d05db978bff){: .shadow style="max-height: 500px; width: auto; display:block; margin:auto;"}

Six characteristics, all sitting there. Here is the important part: **at no point during this connection did any authentication dialog appear**. No passkey. No pairing confirmation. No PIN prompt. I connected as a total stranger and the device showed me everything it has.

This is what "no GATT-level authentication" looks like in practice. In a properly secured BLE device, the server would refuse to return characteristic data until you have completed LE Secure Connections pairing — proving you have the right key. This device skips that entirely. The attack surface is open to anyone.

Now I start poking the characteristics one by one.

---

## Exploit 1: Reading the Admin Password Out of the Device

The first characteristic — `0000AA01` — has a READ property. I tap the little download arrow in nRF Connect to read it, switch the value display to UTF-8, and get this:

![nRF Connect showing the UTF-8 coded value of CHAR_DEVICE_INFO](https://github.com/user-attachments/assets/eb1c4f48-b74c-4ef1-b1d2-4398e651058e){: .shadow style="max-height: 500px; width: auto; display:block; margin:auto;"}

![Decoding the UTF-8 coded value of CHAR_DEVICE_INFO using Python that is contains the credentials](https://github.com/user-attachments/assets/01ed5b68-9ffe-44b8-a5a7-b7c88f132575){: .shadow }

```
ADMIN_USER:admin|ADMIN_PASS:admin@123|API_KEY:sk-lock-PROD-9f3a2c|FW:v1.0.0|BUILD:DEBUG
```

Admin username, Admin password, production API key, Firmware version and — very interestingly — `BUILD:DEBUG`, which tells me the device is running a debug build in production.

I did not crack anything. I did not brute force anything. I just read a characteristic.

This happens more often than you would think in real embedded devices. A developer stores device info in a characteristic during testing — hardware version, build tag, maybe some config — and the credentials end up in there because it is convenient during development. The device ships, the characteristic ships with it, and now the admin password is one tap away from anyone with a phone.

The `BUILD:DEBUG` flag is a hint I tuck away for later.

---

## Exploit 2: Unlocking the Door Without a Password

Characteristic `0000AA02` has a WRITE property. Based on the name `CHAR_LOCK_CMD` and the context, this is clearly how you send lock and unlock commands.

I tap the upload arrow in nRF Connect to open the write dialog, set encoding to UTF-8, type `UNLOCK`, and tap **WRITE**.

![nRF Connect write dialog with UNLOCK typed into the value field](https://github.com/user-attachments/assets/be95a10f-48fe-40de-a8c1-f842dee7203e){: .shadow style="max-height: 500px; width: auto; display:block; margin:auto;"}

The TM1637 display on the lock goes from `LOCK` to `OPEN`.

![TM1637 display showing OPEN](https://github.com/user-attachments/assets/327fa06c-7bfa-41ae-b370-aa8ad6e1d449){: .shadow }

That is it. The lock is open.

No password. No pairing. No challenge. No verification that I am who I say I am. The device received the string `UNLOCK` over BLE and executed it immediately. I write `LOCK`, it locks. I write `UNLOCK`, it opens. Over and over, as fast as I can tap.

What should be happening instead: when I connect, the device should generate a random number — a **nonce** — and send it to me. I should have to prove I know the secret by signing that nonce with it (HMAC). The device verifies my signature before executing any command. Without that, the command is just a string anyone can send.

---

## Exploit 3: Guessing the PIN — No One Stops You

The device also has a PIN-based unlock path on characteristic `0000AA03`. Four digits, so 10,000 possible combinations.

I write `0000` to it. Wrong. The TM1637 shows `0001` — a live attempt counter. I write `0001`. Wrong. `0002`. `0003`. `0004`. `0005`. Still wrong, still no lockout, still no delay.

![TM1637 display counting PIN attempts with no lockout](https://github.com/user-attachments/assets/ac07d454-41ec-483a-8ea1-cc7136afb709){: .shadow }

The characteristic accepts writes at full speed indefinitely. There is no "you have 3 attempts" logic anywhere. I skip ahead and write `1234` — the correct PIN — and the lock opens.

![Correct PIN entered](https://github.com/user-attachments/assets/7c959198-4d4a-40aa-b298-eb6689a3a9bb){: .shadow style="max-height: 500px; width: auto; display:block; margin:auto;"}

![TM1637 showing OPEN after correct PIN entered](https://github.com/user-attachments/assets/41861613-e0b6-41d8-abb7-f13d5b492de0){: .shadow }

A physical keypad at least forces an attacker to stand visibly at the door and tap buttons someone might notice. Over BLE, this brute force happens silently from across the hallway. There is nothing in the device logs to distinguish an attacker trying 9,999 wrong PINs from someone who just forgot their code.

The fix is straightforward: lock the characteristic after 10 failures, make it need a physical reset to unlock again, and add a growing delay after each wrong attempt. Three lines of firmware logic. Somehow it is not there.

---

## Exploit 4: Just Ask the Device for the PIN

Here is my favourite one — because it makes the brute force completely unnecessary.

Remember that `BUILD:DEBUG` flag from Exploit 1? Time to use it.

Characteristic `0000AA04` is `CHAR_DEBUG` — it has both READ and WRITE properties. I write `DEBUG_ON` to it in UTF-8. The TM1637 display starts blinking. I then write `DUMP_STATE` to the same characteristic and tap the **read icon**:

![Writing 0xAA04 DUMP_STATE](https://github.com/user-attachments/assets/cc0e281f-f38c-4313-8083-f827c3f24bbd){: .shadow style="max-height: 500px; width: auto; display:block; margin:auto;"}


![nRF Connect showing DUMP_STATE read response in UTF-8 with PIN visible](https://github.com/user-attachments/assets/7ca36c2a-0936-4c69-ba5a-5b8fdbca3a8c){: .shadow style="max-height: 500px; width: auto; display:block; margin:auto;"}

```
LOCKED=1,PIN=1234,ATT=0,CMD=DEBUG_ON,DBG=1,CONN=1
```

`PIN=1234`. Right there. In the response. No brute force. No math. Just read.

This is what happens when developer tooling ships in a production binary. During development it makes sense — you want to be able to dump device state without attaching a serial monitor. But that debug characteristic is just as readable by an attacker as it is by a developer. The GATT structure is public. Anyone who connects can see that `0000AA04` exists and has a READ property. The name `CHAR_DEBUG` is a free hint.

The only correct fix is a compile-time `#ifdef DEBUG` guard that removes the characteristic entirely from production builds. Not hidden behind a flag, not "disabled by default" — simply not present in the binary.

---

## What This All Adds Up To

Five exploits. One phone. Zero specialist hardware. Here is what the attack chain looks like end to end:

1. Scan passively → device name and service UUID handed over for free
2. Connect → full GATT structure accessible, no authentication required
3. Read one characteristic → admin credentials and API key in plaintext
4. Write `UNLOCK` to another characteristic → door opens immediately
5. Disconnect and reconnect → same write still opens the door, forever
6. Can't be bothered with replay? → brute force the PIN in a few minutes
7. Still too much effort? → read the debug characteristic and get the PIN directly

None of these steps required anything beyond what is built into a free phone app. The vulnerabilities are not subtle. They are foundational — missing authentication, plaintext secrets, no replay protection, no rate limiting. These are decisions made (or not made) during firmware design, and they make every other security measure irrelevant.

---

## The Bigger Picture

Every commercial smart lock I have read a security paper on has at least one of these issues. Replay attacks on static BLE commands have been publicly documented since 2016. Hardcoded credentials in BLE characteristics showed up in CVE databases years ago. Debug interfaces in production firmware are a recurring finding in every major IoT security research report.

And yet here we are.

The reason is not that developers do not know better. It is that BLE security requires deliberate, upfront design decisions — LE Secure Connections pairing, per-session nonces, application-layer authentication — and those decisions add complexity during a phase when the priority is getting the device to work at all. Security gets deferred. Then it ships.

If you are building a BLE device — even a hobby project — the minimum bar should be: authenticated pairing before any characteristic access, no secrets in readable payloads, and compile-time removal of all debug interfaces before the binary leaves your machine.

Everything else is a matter of time before someone with nRF Connect on their phone figures it out.

---

*Built on ESP32 DevKit V1 with TM1637 display. All testing performed on a self-owned lab device. Do not try this on hardware you do not own.*
