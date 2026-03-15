# ⌨️ ESP32 Bluetooth Keyboard (MicroPython)

A robust, battle-tested MicroPython library for turning an ESP32 into a Bluetooth Low Energy (BLE) HID Keyboard. 

Unlike standard BLE libraries that suffer from infinite disconnect/reconnect loops on Windows 11, this library has been specifically engineered to survive strict OS security handshakes, aggressive Bluetooth caching, and the quirks of the MicroPython `NimBLE` stack.

## ✨ Features
* **Full QWERTY Support:** Type strings, press individual keys, or use modifiers (Ctrl, Alt, Shift, Win).
* **Touch-Capacitive Integration:** Easily map ESP32 touch pins directly to keystrokes.
* **Persistent Auto-Reconnect:** Saves BLE bonding keys so you don't have to re-pair after every reboot.
* **Cross-Platform:** Works seamlessly on Windows 10/11, macOS, Android, and iOS.

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

## 🚀 Installation & Getting Started

1. Flash your ESP32 with **MicroPython v1.20 or newer** (v1.27+ recommended).
2. Using your preferred IDE (Thonny is highly recommended for beginners), upload the `ble_keyboard.py` library file to the root directory of your ESP32.
3. Upload `example_kb.py` to the board and run it.

### Running the Example (`example_kb.py`)
The provided example script turns your ESP32 into a two-button touch keyboard. 

**Hardware Connections:**
* Take two male-to-male jumper wires (or alligator clips).
* Plug one wire into **GPIO 4** (Pin 4).
* Plug the second wire into **GPIO 13** (Pin 13).
* Leave the other ends of the wires loose.

**What to do:**
1. Run `example_kb.py` in your IDE.
2. Open your computer's Bluetooth settings and pair to **"ESP32_KB_Main"**.
3. Once connected, physically tap the exposed metal end of the wire connected to **Pin 4**. It will act as the **Left Arrow** key.
4. Tap the wire connected to **Pin 13**. It will act as the **Right Arrow** key.

---

## 💡 Project Inspiration & Build Guides

Because the ESP32 acts exactly like a standard USB keyboard, you can use it to interact with almost any software. Here are 5 hardware projects you can build right now.

### 🏎️ 1. Cardboard Racing Controller (For Asphalt or Need for Speed)
Turn a piece of cardboard into a wireless steering wheel!
* **The Build:** Cut a steering wheel out of cardboard. Tape a patch of aluminum foil to the left grip, and another patch to the right grip. Connect Pin 4 to the left foil, and Pin 13 to the right foil.
* **How it works:** When you hold the foil on the left, the ESP32 holds down the left arrow. 
* **The Code:**
```python
from ble_keyboard import BLEKeyboard
from machine import Pin, TouchPad
import time

kb = BLEKeyboard("ESP_Wheel")
left_pad = TouchPad(Pin(4))
right_pad = TouchPad(Pin(13))
THRESHOLD = 300

while True:
    if kb.is_connected():
        if left_pad.read() < THRESHOLD:
            kb.arrow_left()
        elif right_pad.read() < THRESHOLD:
            kb.arrow_right()
    time.sleep_ms(50)
```

### 🍌 2. Capacitive-Touch Banana Piano
Play an online synthesizer using fruit.
* **The Build:** Get 5 bananas. Push a jumper wire into each one. Connect the other ends to ESP32 Touch Pins (e.g., 4, 13, 14, 27, 33). 
* **How it works:** Open a browser-based piano. Touching a banana triggers a specific letter key to play a musical note.
* **The Code:**
```python
from ble_keyboard import BLEKeyboard
from machine import Pin, TouchPad
import time

kb = BLEKeyboard("Banana_Piano")
keys = {
    'a': TouchPad(Pin(4)),
    's': TouchPad(Pin(13)),
    'd': TouchPad(Pin(14)),
    'f': TouchPad(Pin(27)),
    'g': TouchPad(Pin(33))
}

while True:
    if kb.is_connected():
        for letter, pad in keys.items():
            if pad.read() < 300:
                kb.type_text(letter)
                # Wait for you to let go of the banana!
                while pad.read() < 300: 
                    time.sleep_ms(20)
    time.sleep_ms(20)
```

### 🚨 3. The Ultimate "Boss Key"
A massive physical button hidden under your desk that instantly hides all your windows when someone walks into the room.
* **The Build:** Get a physical arcade button or push-switch. Wire one leg to `GND` and the other leg to Pin 15. 
* **How it works:** Instead of touch, this uses a physical digital switch. When pressed, it sends `Windows Key + D` (or `Cmd + F3` on Mac) to minimize everything instantly.
* **The Code:**
```python
from ble_keyboard import BLEKeyboard
from machine import Pin
import time

kb = BLEKeyboard("Boss_Key")
# Use internal pull-up resistor. Button press connects to Ground (0)
btn = Pin(15, Pin.IN, Pin.PULL_UP) 

while True:
    if kb.is_connected():
        if btn.value() == 0: # Button pressed
            print("Hiding windows!")
            kb.win_d() # Sends Win+D
            # Wait for button release
            while btn.value() == 0: 
                time.sleep_ms(20)
    time.sleep_ms(50)
```

### 📱 4. Wearable TikTok / YouTube Shorts Scroller
A tiny remote so you can swipe to the next video from across the room without touching your phone.
* **The Build:** Tape a small jumper wire to your index finger (connected to Pin 4). Connect a battery to your ESP32. 
* **How it works:** Pair the ESP32 to your iOS/Android phone. Simply tap your thumb and index finger together to trigger the "Down Arrow", which scrolls to the next video.
* **The Code:**
```python
from ble_keyboard import BLEKeyboard
from machine import Pin, TouchPad
import time

kb = BLEKeyboard("Tok_Scroller")
thumb_tap = TouchPad(Pin(4))

while True:
    if kb.is_connected():
        if thumb_tap.read() < 300:
            kb.arrow_down()
            while thumb_tap.read() < 300:
                time.sleep_ms(20)
    time.sleep_ms(50)
```

### 🎼 5. Musician's Page Turner Foot Pedal
Reading sheet music on an iPad while playing guitar or piano? Build a custom foot pedal to turn the pages.
* **The Build:** Create a simple cardboard hinge pedal. Put foil on the top and bottom flaps so they touch when you step on it. Connect one foil to GND, and the other to Pin 15.
* **How it works:** Stepping on the pedal sends a single "Right Arrow" press to flip the PDF page. It includes a long delay so you don't accidentally skip two pages!
* **The Code:**
```python
from ble_keyboard import BLEKeyboard
from machine import Pin
import time

kb = BLEKeyboard("Page_Turner")
pedal = Pin(15, Pin.IN, Pin.PULL_UP) 

while True:
    if kb.is_connected():
        if pedal.value() == 0: # Pedal pressed
            kb.arrow_right()
            # 1-second delay so you don't accidentally double-flip!
            time.sleep(1) 
    time.sleep_ms(50)
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
When MicroPython triggers `_IRQ_SET_SECRET` to hand over the Bluetooth passwords, it provides them as `memoryview` objects. The standard Python `json.dump()` function will crash with a `TypeError` if you try to save a memoryview or a `bytes` object. We bypass this complexity entirely by relying on the "dirty save" crash.

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
