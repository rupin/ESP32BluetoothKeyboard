# ⌨️ ESP32 Bluetooth Keyboard (MicroPython)

A robust, battle-tested MicroPython library for turning an ESP32 into a Bluetooth Low Energy (BLE) HID Keyboard. 

Unlike standard BLE libraries that suffer from infinite disconnect/reconnect loops on Windows 11, this library has been specifically engineered to survive strict OS security handshakes, aggressive Bluetooth caching, and the quirks of the MicroPython `NimBLE` stack.

## ✨ Features
* **Full QWERTY Support:** Type strings, press individual keys, or use modifiers (Ctrl, Alt, Shift, Win).
* **Touch-Capacitive Integration:** Includes examples for mapping ESP32 touch pins directly to keystrokes.
* **Persistent Auto-Reconnect:** Saves BLE bonding keys so you don't have to re-pair after every reboot.
* **Cross-Platform:** Works on Windows 10/11, macOS, Android, and iOS.

---

## ⚠️ The Elephant in the Room: The "First-Time Crash" (Dirty Save)

If you run this code and pair it to Windows for the first time, your ESP32 will likely throw a terrifying wall of red text in your IDE console:
`Guru Meditation Error: Core 1 panic'ed (LoadProhibited). Exception was unhandled.`

**Do not panic. This is an intended feature.**

### Why does it crash?
Windows 11 strictly requires BLE keyboards to exchange encrypted passwords (bonding keys). In MicroPython, these keys are handed to the ESP32 inside a hardware interrupt (`_IRQ_SET_SECRET`). 

Hardware interrupts are highly restricted memory states. When our code attempts to save these keys to a `ble_secrets.json` file inside the interrupt, the ESP32's safety watchdog realizes we are doing "slow" flash-memory writing during a critical system pause, and it intentionally panics and crashes the chip.

### Why is this good? (The "Dirty Save")
Because flash memory is incredibly fast, the ESP32 actually finishes writing the `ble_secrets.json` file *fractions of a millisecond before it dies*. 
1. You pair the device.
2. The ESP32 writes the JSON file and instantly crashes/reboots.
3. Upon rebooting, the keys are securely loaded from the file.
4. **Result:** The ESP32 seamlessly auto-reconnects to Windows forever without ever crashing again. 

It effectively turns a kernel panic into an automated "One-Time Setup Reboot"!

---

## 🚀 Installation

1. Flash your ESP32 with **MicroPython v1.20 or newer** (v1.27+ recommended).
2. Using your preferred IDE (Thonny is highly recommended for beginners), upload the `ble_keyboard.py` library file to the root directory of your ESP32.
3. Once the library is saved to the board, you can open `main.py` in your IDE, paste any of the example codes below, and click "Run" to execute it directly on the board.

---

## 📖 Usage Guide

### 🟢 Level 1: Beginner (Basic Keystrokes)
To use the library, instantiate the `BLEKeyboard` object and wait for a connection.

```python
from ble_keyboard import BLEKeyboard
import time

# IMPORTANT: Keep the name under 15 characters!
kb = BLEKeyboard("ESP32_MyKb")

while True:
    if kb.is_connected():
        print("Typing hello...")
        kb.type_text("Hello World!\n")
        time.sleep(5)
```

### 🟡 Level 2: Intermediate (Touch Pins & Modifiers)
You can map the built-in capacitive touch pins on the ESP32 to specific actions. This requires basic debouncing (waiting for the user to lift their finger) so it doesn't spam the computer with 500 keystrokes a second.

```python
from ble_keyboard import BLEKeyboard
from machine import Pin, TouchPad
import time

kb = BLEKeyboard("Touch_Keyboard")
touch_left = TouchPad(Pin(4))
THRESHOLD = 300 

connection_ready = False

while True:
    if kb.is_connected():
        # Add a small delay on first connection for OS security handshake
        if not connection_ready:
            time.sleep(2)
            connection_ready = True

        try:
            if touch_left.read() < THRESHOLD:
                # Copy text using modifier
                kb.ctrl_c() 

                # Wait for finger release (Debounce)
                while touch_left.read() < THRESHOLD: 
                    time.sleep_ms(20)
        except ValueError:
            pass
    else:
        connection_ready = False

    time.sleep_ms(50)
```

### 🔴 Level 3: Advanced (The "Escape Hatch" Pairing Button)
Windows 11 Bluetooth caching is notoriously buggy. If a student maxes out their Bluetooth device limit, or the Windows NimBLE cache gets permanently corrupted, the ESP32 will fall into an infinite "Connected -> Disconnected" loop.

You can build a hardware "Escape Hatch" using the ESP32's `BOOT` button (Pin 0). If held during power-on, it generates a random device name, tricking Windows into treating it as a brand-new device without having to dig into OS settings.

```python
from ble_keyboard import BLEKeyboard
from machine import Pin
import random

pair_btn = Pin(0, Pin.IN, Pin.PULL_UP)

if pair_btn.value() == 0:
    # Button held: Bypass Windows cache with new name
    device_name = f"ESP_KB_{random.randint(100, 999)}"
else:
    # Normal boot
    device_name = "ESP32_KB_Main"

kb = BLEKeyboard(device_name)
# ... continue with main loop
```

---

## 🧠 Deep Dive: The Technical Vagaries

Building this library exposed several obscure bugs in the intersection between Windows 11, macOS, and MicroPython. If you are modifying this library, keep these documented vagaries in mind:

### 1. The 15-Character Name Limit (The 31-Byte Payload Bug)
If you try to name your device something long like `BLEKeyboard("My_Awesome_ESP32_Keyboard")`, **the ESP32 will crash with an `OSError: [Errno 12] ENOMEM` or simply fail to advertise.**

This happens because the total size of a Bluetooth Legacy Advertising Packet is strictly limited to **31 bytes** by the BLE specification. 
In our `adv.extend()` packet:
* 3 bytes are used for standard BLE Flags.
* 4 bytes are used for the Keyboard Appearance Icon (so it shows up as a keyboard in Windows).
* 6 bytes are used for the Service UUIDs.
* 2 bytes are used for the Name Header.

This leaves exactly **16 bytes remaining for your device name string.** To be safe, keep your `name=` parameter to **15 characters or less**.

### 2. The Missing MicroPython Constants
To force Windows 11 to initiate a bonding sequence, the BLE Characteristics must be flagged as encrypted. However, MicroPython 1.27 forgot to expose the text variables for these flags! 
In `ble_keyboard.py`, you will notice we hardcoded the raw hex values (e.g., `0x0200` for `FLAG_READ_ENCRYPTED`). Do not change these, or Windows will refuse to pair.

### 3. Memoryviews vs. JSON serialization
When MicroPython triggers `_IRQ_SET_SECRET` to hand over the Bluetooth passwords, it provides them as `memoryview` objects. The standard Python `json.dump()` function will crash with a `TypeError` if you try to save a memoryview or a `bytes` object. We bypass this complexity entirely by relying on the "dirty save" crash, but if you attempt to rewrite this using standard JSON saves, you must convert keys to `.hex()` strings first.

### 4. Mac Name Caching vs. Windows Caching
* **Windows 11** enforces a strict paired-device limit (usually 7-8 devices). If you use a random name generator on *every* boot, Windows will quickly hit this limit and permanently block the ESP32.
* **macOS** ignores random names if the hardware MAC address remains the same. It aggressively caches the MAC address and will force the name back to whatever it was called the first time it paired. 

---

## 🛠️ Modifying the Keyboard Map
If you need to add special keys (like Media keys or F1-F12), you can modify the `_K` dictionary and the `_HID_MAP` descriptor array in `ble_keyboard.py`. 
The `_K` dictionary uses tuples: `(HID_Keycode, Modifier_Hex)`. 
* Example: `'a': (4, 0)` is a standard lowercase 'a'.
* Example: `'A': (4, 2)` is an uppercase 'A' (sending the keycode 4, along with modifier 2, which is Left Shift).

---

## License
Feel free to use, modify, and distribute this library for educational and maker projects. Designed specifically for experiential learning and hardware classrooms.
