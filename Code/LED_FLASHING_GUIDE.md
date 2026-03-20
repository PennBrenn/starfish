# Starfish LED Flashing Guide

## Overview
The onboard LED now provides visual feedback about system status through different flashing patterns. The LED is connected to GPIO10 and can display 7 distinct patterns.

---

## LED Flashing Patterns

### 🟢 **LED_OFF**
- **Appearance**: LED is always OFF
- **Meaning**: System shutdown or critical failure
- **Period**: N/A

### 🔴 **LED_SOLID** 
- **Appearance**: LED is always ON
- **Meaning**: Active operation (driving or auto-cycle)
- **Period**: N/A

### 🟡 **LED_NORMAL**
- **Appearance**: Slow blink (1 second on, 1 second off)
- **Meaning**: Normal idle state, system healthy
- **Period**: 1000ms (500ms on, 500ms off)

### 🟠 **LED_WARN**
- **Appearance**: Fast blink (200ms on, 200ms off)
- **Meaning**: Thermal warning detected
- **Period**: 200ms (100ms on, 100ms off)

### 🔴 **LED_ERROR**
- **Appearance**: Very fast blink (100ms on, 100ms off)
- **Meaning**: Critical error (IMU communication failure)
- **Period**: 100ms (50ms on, 50ms off)

### 🔵 **LED_IMU_ERROR**
- **Appearance**: Medium blink (500ms on, 500ms off)
- **Meaning**: IMU not initialized or communication issues
- **Period**: 500ms (250ms on, 250ms off)

### 🟣 **LED_WIFI_CONNECTING**
- **Appearance**: Rapid blink (150ms on, 150ms off)
- **Meaning**: WiFi AP mode starting up
- **Period**: 150ms (75ms on, 75ms off)

---

## Priority System

The LED pattern follows a priority system where higher priority conditions override lower ones:

1. **Highest Priority**: Thermal Warning (LED_WARN)
2. **High Priority**: IMU Not Initialized (LED_IMU_ERROR)
3. **Medium Priority**: IMU Communication Errors (LED_ERROR)
4. **Normal Priority**: Mode-based patterns
   - Safe Idle → LED_NORMAL
   - Active Drive/Auto-Cycle → LED_SOLID
   - Other modes → LED_NORMAL

---

## State-Based Behavior

### During Startup
- **Power On**: LED_OFF → LED_WIFI_CONNECTING (WiFi AP starting)
- **WiFi Ready**: Pattern changes based on IMU status

### During Normal Operation
- **Safe Idle Mode**: LED_NORMAL (slow blink)
- **Active Drive Mode**: LED_SOLID (always on)
- **Auto-Cycle Mode**: LED_SOLID (always on)
- **Stabilize/Hill Hold**: LED_NORMAL (slow blink)

### During Error Conditions
- **IMU Failure**: LED_IMU_ERROR (medium blink)
- **Thermal Warning**: LED_WARN (fast blink) - overrides all others
- **Communication Errors**: LED_ERROR (very fast blink)

---

## Implementation Details

### Constants
```cpp
#define LED_FLASH_NORMAL_MS    1000   // Normal operation blink
#define LED_FLASH_WARN_MS      200    // Warning state fast blink
#define LED_FLASH_ERROR_MS     100    // Error state very fast blink
#define LED_FLASH_IMU_ERR_MS   500    // IMU error specific pattern
#define LED_FLASH_WIFI_MS      150    // WiFi connecting blink
```

### Functions
- **`led_set_pattern(pattern)`**: Changes LED flashing pattern
- **`led_update()`**: Called every 50ms to handle timing
- **`led_get_period_ms(pattern)`**: Returns blink period for pattern

### Thread Safety
- LED updates happen in both control task (50Hz) and main loop (20Hz)
- Pattern changes are atomic and thread-safe
- No mutex needed for LED operations (digitalWrite is atomic)

---

## Visual Reference

| Pattern | Visual | Meaning | Priority |
|---------|--------|---------|----------|
| LED_OFF | ⚫ | System off | - |
| LED_SOLID | 🔴 | Active operation | 4 |
| LED_NORMAL | 🟡⚫🟡⚫ | Normal idle | 4 |
| LED_WARN | 🟠🟠🟠🟠 | Thermal warning | 1 |
| LED_ERROR | 🔴🔴🔴🔴🔴 | IMU error | 3 |
| LED_IMU_ERROR | 🔵⚫🔵⚫ | IMU not ready | 2 |
| LED_WIFI_CONNECTING | 🟣🟣🟣🟣🟣 | WiFi starting | 0 |

---

## Troubleshooting

### LED Not Blinking
- Check GPIO10 connection
- Verify LED polarity (ESP32-C3 uses 3.3V logic)
- Check for short circuits

### Unexpected Pattern
- Check system status via Serial output
- Verify IMU initialization in logs
- Check thermal warnings in telemetry

### LED Stuck Solid
- System likely in active drive mode
- Check WebSocket commands
- Verify mode state in telemetry

---

## Integration with Web Interface

The LED status can be monitored via WebSocket telemetry:
```json
{
  "thermal_warn": true/false,
  "mode": "safe_idle|active_drive|stabilize|hill_hold|auto_cycle"
}
```

---

## Power Consumption

LED patterns have minimal power impact:
- LED_OFF: 0mA
- LED_SOLID: ~2mA
- Flashing patterns: ~1mA average

The LED uses GPIO10 which can source up to 40mA safely.

---

## Future Enhancements

Possible additions:
- Custom patterns for specific events
- Brightness control via PWM
- Morse code status messages
- RGB LED support (hardware modification)

---

**Note**: The LED system is designed to be intuitive - faster blinking indicates more urgent conditions, while solid light indicates active operation.
