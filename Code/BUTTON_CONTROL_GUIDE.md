# Starfish Button & Status LED Control Guide

## Hardware Overview

### **GPIO2 - Push Button**
- **Type**: Momentary push button (normally open)
- **Wiring**: Button between GPIO2 and GND
- **Pull-up**: Internal pull-up resistor enabled
- **Active State**: LOW (pressed), HIGH (released)

### **GPIO3 - Status LED**
- **Type**: Standard LED with current-limiting resistor
- **Wiring**: LED anode to GPIO3, cathode to GND (via resistor)
- **Recommended Resistor**: 220-330Ω for 3.3V
- **Current**: ~10mA typical

---

## Button Functions

### 🔴 **Single Press** - Emergency Stop
**Action**: Press and release quickly (< 400ms)

**What it does**:
- Immediately releases all solenoids
- Switches to SAFE_IDLE mode
- Saves current mode for later resume
- Status LED switches to fast blink (emergency stop pattern)

**Use case**: Instant stop during unexpected behavior

---

### 🟢 **Double Click** - Mode Toggle
**Action**: Press twice within 400ms

**What it does**:
- **If in emergency stop**: Releases emergency stop, returns to safe idle
- **If in safe idle**: Resumes previous active mode
- **If in active mode**: Switches to safe idle

**Use case**: Quick mode switching without web interface

---

### 🟡 **Long Press (2 seconds)** - Reset Parameters
**Action**: Hold button for 2 seconds

**What it does**:
- Resets adaptive parameters to factory defaults:
  - Kick duty: 255
  - Hold duty: 77
  - Kick duration: 50ms
  - Hold frequency: 100Hz
- Saves to NVS (non-volatile storage)
- Shows double-blink confirmation on status LED
- Returns to previous LED pattern

**Use case**: Reset tuning after experimentation

---

### 🔴 **Very Long Press (5 seconds)** - System Reboot
**Action**: Hold button for 5 seconds

**What it does**:
- Shows triple-blink pattern on status LED
- Waits 3 seconds (countdown)
- Performs ESP32 system restart

**Use case**: Full system reset without power cycle

---

## Status LED Patterns (GPIO3)

### **Solid Green** 🟢
- **Pattern**: Always ON
- **Meaning**: System armed and ready for operation
- **Modes**: Active Drive, Auto-Cycle, Stabilize, Hill Hold

### **Slow Blink** 🟡
- **Pattern**: 1 second ON, 1 second OFF
- **Meaning**: Safe idle mode
- **Modes**: Safe Idle (normal operation)

### **Fast Blink** 🔴
- **Pattern**: 200ms ON, 200ms OFF
- **Meaning**: Emergency stop active
- **Action**: All solenoids disabled, system locked

### **Double Blink** 💚
- **Pattern**: Quick double flash, then pause
- **Meaning**: Parameter reset confirmation
- **Duration**: 2 blinks then returns to normal pattern

### **Triple Blink** 🔵
- **Pattern**: Quick triple flash, then pause
- **Meaning**: System reboot countdown
- **Duration**: 3 blinks then system restarts

### **OFF** ⚫
- **Pattern**: No light
- **Meaning**: System not initialized or critical error

---

## Button Timing Reference

| Action | Timing | Detection Window |
|--------|--------|------------------|
| Single Press | < 400ms | After release |
| Double Click | 2 presses within 400ms | After 2nd release |
| Long Press | 2000ms held | While holding |
| Very Long Press | 5000ms held | While holding |
| Debounce | 50ms | On state change |

---

## State Machine Diagram

```
IDLE
  ├─ Single Press → EMERGENCY_STOP
  ├─ Double Click → MODE_TOGGLE
  ├─ Long Press (2s) → RESET_PARAMS
  └─ Very Long Press (5s) → REBOOT

EMERGENCY_STOP
  ├─ Single Press → (ignored, already stopped)
  └─ Double Click → RELEASE_ESTOP → SAFE_IDLE

SAFE_IDLE
  ├─ Single Press → EMERGENCY_STOP
  └─ Double Click → RESUME_MODE

ACTIVE_MODE
  ├─ Single Press → EMERGENCY_STOP
  └─ Double Click → SAFE_IDLE
```

---

## Integration with Web Interface

The button provides **physical override** capabilities:
- Works even if WiFi connection is lost
- Faster response than web commands
- Emergency stop has hardware-level priority
- Status LED provides visual feedback without screen

### Telemetry Updates
Button actions are logged via Serial:
```
EMERGENCY STOP ACTIVATED
Emergency stop released - system in safe idle
Long press detected - resetting parameters
Adaptive parameters reset to defaults
Very long press - REBOOTING in 3 seconds...
```

---

## Safety Features

### **Debouncing**
- 50ms hardware debounce prevents false triggers
- Stable state required before action

### **Emergency Stop Priority**
- Emergency stop cannot be overridden by web commands
- Must be explicitly released via double-click
- All solenoids immediately disabled

### **Long Press Protection**
- Long press actions only trigger once per hold
- Release required before next action
- Visual feedback during countdown

### **Mode Memory**
- System remembers mode before emergency stop
- Can resume previous operation safely
- Prevents accidental mode changes

---

## Troubleshooting

### Button Not Responding
1. **Check wiring**: GPIO2 to one side of button, GND to other
2. **Verify pull-up**: Internal pull-up should read HIGH when not pressed
3. **Test with multimeter**: Should read 0V when pressed, 3.3V when released
4. **Check Serial output**: Should show "Button initialized on GPIO2"

### Status LED Not Working
1. **Check polarity**: Anode (long leg) to GPIO3, cathode to GND
2. **Verify resistor**: 220-330Ω in series with LED
3. **Test LED**: Should light when GPIO3 is HIGH
4. **Check Serial output**: Should show "Status LED initialized on GPIO3"

### False Triggers
- **Increase debounce**: Modify `BUTTON_DEBOUNCE_MS` (default 50ms)
- **Check button quality**: Mechanical bounce can cause issues
- **Add external capacitor**: 0.1µF across button terminals

### LED Too Bright/Dim
- **Adjust resistor**: Lower value = brighter (min 150Ω)
- **Change LED**: Different colors have different forward voltages
- **Check current**: Should be 10-20mA max

---

## Example Usage Scenarios

### **Scenario 1: Quick Stop During Testing**
1. Robot starts rolling unexpectedly
2. **Single press** button → Emergency stop
3. Status LED fast blinks (red pattern)
4. Investigate issue
5. **Double click** → Release and return to safe idle

### **Scenario 2: Parameter Tuning Session**
1. Test various PWM settings via web interface
2. Robot becomes unstable
3. **Long press (2s)** → Reset to factory defaults
4. Status LED double blinks (confirmation)
5. Start tuning from known-good baseline

### **Scenario 3: System Hang Recovery**
1. Web interface becomes unresponsive
2. **Very long press (5s)** → Initiate reboot
3. Status LED triple blinks
4. System restarts after 3 seconds
5. WiFi AP comes back online

### **Scenario 4: Mode Switching Without Web**
1. Currently in auto-cycle mode
2. **Double click** → Switch to safe idle
3. Status LED changes to slow blink
4. Make adjustments
5. **Double click** → Resume auto-cycle
6. Status LED solid green

---

## Advanced Configuration

### Modify Button Timings
Edit in `src/main.cpp`:
```cpp
#define BUTTON_DEBOUNCE_MS     50     // Debounce time
#define BUTTON_LONG_PRESS_MS   2000   // Long press threshold
#define BUTTON_VLONG_PRESS_MS  5000   // Very long press (reboot)
#define BUTTON_DOUBLE_CLICK_MS 400    // Double click window
```

### Modify LED Patterns
Edit in `src/main.cpp`:
```cpp
#define STATUS_LED_ARMED_MS    0      // Solid on when armed
#define STATUS_LED_IDLE_MS     1000   // Slow blink in safe idle
#define STATUS_LED_ESTOP_MS    200    // Fast blink on emergency stop
#define STATUS_LED_CONFIRM_MS  150    // Double blink pattern
```

---

## Power Consumption

| Component | Current Draw | Notes |
|-----------|--------------|-------|
| Button (idle) | 0 µA | Pull-up only |
| Button (pressed) | ~1 µA | Negligible |
| Status LED (ON) | 10-15 mA | Depends on resistor |
| Status LED (blinking) | 5-7 mA avg | 50% duty cycle |

**Total Impact**: < 15mA additional current draw

---

## Future Enhancements

Possible additions:
- **Triple click**: Cycle through modes
- **Button combinations**: Multiple buttons for complex actions
- **RGB status LED**: Color-coded system states
- **Haptic feedback**: Vibration motor confirmation
- **Button hold patterns**: Different actions for different hold durations

---

## Technical Details

### Button Debouncing Algorithm
```cpp
1. Read current button state
2. If state changed:
   a. Wait DEBOUNCE_MS (50ms)
   b. Read state again
   c. If still changed, accept new state
3. Track press duration for long press detection
4. Track release timing for double-click detection
```

### Status LED State Machine
```cpp
1. Pattern set → Initialize timing
2. Every loop iteration:
   a. Check elapsed time
   b. If period/2 elapsed, toggle LED
   c. Increment blink counter
   d. If max blinks reached, stop pattern
```

### Thread Safety
- Button state read in main loop (core 0)
- Mode changes protected by `state_mutex`
- Emergency stop has atomic flag
- No race conditions between button and WebSocket

---

**The button and status LED provide essential physical control and feedback for safe, reliable Starfish operation!** 🌟
