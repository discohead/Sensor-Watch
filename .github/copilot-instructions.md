# GitHub Copilot Instructions for Sensor Watch

This file provides context to GitHub Copilot for the Sensor Watch project.

## Project Overview

Sensor Watch is a board replacement for the Casio F-91W wristwatch powered by a Microchip SAM L22 (ARM Cortex M0+) microcontroller. The project consists of:
- **watch-library**: Low-level hardware abstraction layer for the SAM L22
- **movement**: Watch face framework and collection of user-facing applications
- **apps**: Standalone test applications and utilities

## Hardware Variants

The build system requires specifying a `COLOR` variable for different board revisions:
- `RED`: Sensor Watch Lite (OSO-SWAT-B1)
- `GREEN`: Standard Sensor Watch (OSO-SWAT-A1-05)
- `BLUE`: Alternative variant with inverted LED polarity
- `PRO`: Sensor Watch Pro (OSO-SWAT-C1-00)

## Build Commands

### Building Movement (Main Firmware)

```bash
cd movement/make
make COLOR=GREEN              # Build for hardware
make install COLOR=GREEN      # Flash to device via UF2 bootloader
make clean COLOR=GREEN        # Clean build artifacts
```

### Building Standalone Apps

```bash
cd apps/<app-name>/make       # e.g., apps/buzzer-test/make
make COLOR=GREEN
make install COLOR=GREEN
```

### Emulator Builds

Requires [emscripten](https://emscripten.org/) installed:

```bash
cd movement/make
emmake make COLOR=GREEN
python3 -m http.server -d build-sim
# Visit http://localhost:8000/watch.html
```

The emulator compiles to WebAssembly and runs in the browser using the `watch-library/simulator/` implementation.

## Architecture

### Watch Library Structure

Located in `watch-library/`:
- `hardware/`: HAL, peripheral drivers, and startup code for SAM L22
  - `hardware/watch/`: Watch-specific hardware abstractions (RTC, LCD, buttons, buzzer, LED, ADC, I2C, SPI, UART)
  - `hardware/hal/`: Microchip HAL (Hardware Abstraction Layer)
  - `hardware/hpl/`: Microchip HPL (Hardware Peripheral Library)
  - `hardware/linker/`: Linker scripts for SAM L22
- `shared/`: Platform-independent code (drivers, utilities)
  - `shared/driver/`: Sensor/peripheral drivers (thermistor, LIS2DW accelerometer, OPT3001 light sensor, SPI flash)
  - `shared/watch/`: Display utilities and watch utility functions
- `simulator/`: Emscripten-based browser simulator implementation

### Movement Framework

Located in `movement/`:
- `movement.c/h`: Core framework managing watch faces, events, and power modes
- `movement_config.h`: **User-editable list of watch faces to include**
- `watch_faces/`: Collection of watch face implementations organized by category:
  - `clock/`: Time display faces
  - `settings/`: Configuration and system faces
  - `complication/`: Feature faces (stopwatch, timer, calculator, games, etc.)
  - `sensor/`: Sensor reading and logging faces
  - `demo/`: Test and demonstration faces
- `lib/`: Third-party libraries (TOTP, astronomical calculations, chess engine, etc.)
- `template/`: Template for creating new watch faces
- `filesystem.c/h`: LittleFS integration for persistent storage
- `shell.c/h`: USB CDC serial shell for debugging

### Watch Face Pattern

Each watch face implements 4 required functions:
- `setup`: Initialize context, allocate memory, called at boot and wake from sleep
- `activate`: Called when face enters foreground, enable peripherals
- `loop`: Event-driven update function (tick events, button presses), returns boolean for sleep eligibility
- `resign`: Called before leaving foreground, disable peripherals

See `movement/README.md` for complete watch face development guide.

### Build System

- `make.mk`: Defines compiler flags, includes, sources, board configurations
  - Hardware builds use `arm-none-eabi-gcc`
  - Simulator builds use emscripten
  - Auto-detects OS for parallel builds (`-j` flag)
- `rules.mk`: Build rules and targets (`.elf`, `.hex`, `.bin`, `.uf2`)
- Each app/firmware has a `Makefile` that includes these common files

### UF2 Bootloader

The watch uses a UF2 bootloader activated by double-tapping the reset button:
1. Double-tap reset → red LED pulses
2. "WATCHBOOT" drive appears
3. Copy `.uf2` file to drive
4. Watch reboots with new firmware

## Key Implementation Details

### Movement Configuration

Edit `movement/movement_config.h` to customize:
- `watch_faces[]`: Array of watch faces in navigation order
- `MOVEMENT_SECONDARY_FACE_INDEX`: Faces accessible via long Mode press (excluded from normal rotation)
- Default settings: 12/24h mode, LED colors, timeouts, button sounds

### Watch Library API Overview

Watch library provides clean abstractions in `watch-library/shared/watch/`. Key APIs:

**Real-Time Clock (`watch_rtc.h`)**
- `watch_rtc_set_date_time()`, `watch_rtc_get_date_time()`: Core date/time management
- `watch_rtc_register_tick_callback()`: 1Hz tick for updates (works in STANDBY)
- `watch_rtc_register_periodic_callback()`: 1-128Hz callbacks (power of 2 only)
- Year stored as offset from 2020 (0=2020, 63=2083) - handle offset in display logic
- Fast ticks (>8Hz) increase power consumption significantly

**Segment LCD (`watch_slcd.h`)**
- `watch_display_string(str, pos)`: 10 positions (0-1=weekday, 2-3=day, 4-9=time)
- Does NOT clear display automatically - manage state yourself
- Hardware blink only works in position 7 (6 of 7 segments, excludes segment B)
- 5 indicators: SIGNAL, BELL, PM, 24H, LAP
- Power: ~3µA idle, ~4µA all segments on

**Buttons & Interrupts (`watch_extint.h`)**
- `watch_register_interrupt_callback()`: Configure button/pin interrupts
- Buttons wake from STANDBY only (not BACKUP mode)
- External pins A2/A4 support deep sleep wake via `watch_register_extwake_callback()`

**LED Control (`watch_led.h`)**
- `watch_set_led_color(red, green)`: PWM control 0-255
- Red LED: ~4.5mA, Green: ~0.44mA (12x chip power!)
- PWM requires app to stay awake (return false from loop)
- Shares TCC peripheral with buzzer

**Analog Input (`watch_adc.h`)**
- `watch_get_analog_pin_level(pin)`: Returns 16-bit value (default: 16 samples averaged)
- `watch_get_vcc_voltage()`: Battery voltage in millivolts
- High impedance inputs: increase sampling cycles (1-64) for ~2MΩ sources
- Reference voltages: VCC, VCC/1.6, VCC/2, or 1.024V internal

**Flash Storage (`watch_storage.h`)**
- 8KB EEPROM emulation: 32 rows × 256 bytes
- MUST erase row (sets to 0xFF) before writing
- Write in 64-byte page-aligned chunks
- Persistent across power loss

**Utility Functions (`watch_utility.h`)**
- `watch_utility_convert_to_unix_time()`: Date → UNIX timestamp
- `watch_utility_convert_to_12_hour()`: WARNING: Overwrites date_time, returns PM bool
- `watch_utility_thermistor_temperature()`: Handles beta equation for thermistor sensors
- Calendar helpers: weekday strings, ISO week numbers, leap year checks

### Power Management

Movement implements two power modes:
- **Standby**: Default between ticks, peripherals disabled, wakes quickly
- **Low Energy Mode**: After timeout (1 hour - 7 days), minimal power, 1 update/minute
  - Watch faces receive `EVENT_LOW_ENERGY_UPDATE` but **must not wake peripherals**

### Sensor Board Interface

9-pin connector provides:
- 3V power, I²C (with pull-ups), 5 GPIO pins
- GPIOs configurable as: analog inputs, digital I/O, SPI, UART, PWM, external wake inputs
- See README.md pin table for SERCOM/peripheral multiplexing

## Development Workflow

### Adding a New Watch Face

1. Create `movement/watch_faces/<category>/<name>_face.c` and `.h`
2. Implement the 4 required functions
3. Add `#define <name>_face` macro in header
4. Include header in `movement/movement_faces.h`
5. Add source file to `movement/make/Makefile` SRCS
6. Add face to `movement_config.h` watch_faces array
7. Build with `make COLOR=GREEN`

### Testing

Use emulator for rapid iteration:
- No hardware required
- Fast compile/test cycles
- Browser-based debugging

For hardware testing:
- Use test apps in `apps/` directory for peripheral validation
- Movement provides `shell.c` USB serial interface for debugging

### Toolchain Setup

**macOS:**
```bash
brew install --cask gcc-arm-embedded
```

**Debian/Ubuntu:**
```bash
apt install gcc-arm-none-eabi
```

**Emulator (all platforms):**
```bash
# macOS
brew install emscripten

# Linux - see https://emscripten.org/docs/getting_started/downloads.html
```

**Verify Installation:**
```bash
arm-none-eabi-gcc --version  # Should show 14.3.1 or compatible
emcc --version                # Should show emscripten version
```

The project has been tested with:
- ARM toolchain: `9-2019-q4-major`, `10.3-2021.07`, and `14.3.1`
- Emscripten: `4.0.18`

**First Build:**

On first build, git submodules will be automatically initialized:
```bash
cd movement/make
make COLOR=GREEN              # Auto-runs: git submodule update --init
```

Build output includes:
- `build/watch.uf2` - Flashable firmware for hardware
- `build-sim/watch.html` - Emulator entry point
- `build-sim/watch.js` + `watch.wasm` - WebAssembly emulator

**Expected Build Results:**
- ROM usage: ~118KB / 240KB (48-50%)
- RAM usage: ~15KB / 32KB (45%)
- Build time: ~30-60s (hardware), ~45-90s (emulator)

### Documentation

Build API documentation locally using Doxygen:
```bash
doxygen Doxyfile
# Opens docs/index.html in your browser
open docs/index.html
```

Online documentation: https://joeycastillo.github.io/Sensor-Watch/

The Doxyfile generates docs for the watch library API located in `watch-library/shared/watch/`.

## Common Development Patterns

**Power-Conscious Display Updates**
```c
// Only update when content changes
static char last_display[11] = "";
char current_display[11];
sprintf(current_display, "%2d%2d%02d%02d", ...);
if (strcmp(last_display, current_display) != 0) {
    watch_display_string(current_display, 0);
    strcpy(last_display, current_display);
}
```

**Typical Watch Face Update Loop**
```c
// 1. Register callback in setup
watch_rtc_register_tick_callback(my_tick_callback);

// 2. Update in callback
void my_tick_callback() {
    watch_date_time now = watch_rtc_get_date_time();
    // Update display based on 'now'
}

// 3. Return true from loop to enable STANDBY sleep
bool my_face_loop(movement_event_t event, ...) {
    // ... handle events ...
    return true; // Allow sleep between ticks
}
```

**Button Handling with Debouncing**
```c
// Set flag in interrupt, process in main loop
volatile bool button_pressed = false;

void button_callback(uint8_t pin) {
    button_pressed = true;
}

bool my_face_loop(...) {
    if (button_pressed) {
        button_pressed = false;
        // Handle button press safely in main context
    }
    return true;
}
```

**Non-Volatile Settings Storage**
```c
// 1. Read on boot
typedef struct { uint8_t setting1; uint16_t setting2; } settings_t;
settings_t settings;
watch_storage_read(0, 0, (uint8_t*)&settings, sizeof(settings));

// 2. Modify in RAM during operation
settings.setting1 = new_value;

// 3. Save when changed
watch_storage_erase(0);
watch_storage_write(0, 0, (uint8_t*)&settings, sizeof(settings));
watch_storage_sync();
```

## Important Notes

- All builds require `COLOR` environment variable set (RED/GREEN/BLUE/PRO)
- Build artifacts go to `build/` (hardware) or `build-sim/` (emulator)
- The `.uf2` file is the final flashable binary
- `tinyusb` and `littlefs` are git submodules - run `git submodule update --init` if missing
- Movement manages watch face lifecycle - faces should not directly manipulate system state beyond their own peripherals
- Watch library API documentation provides full function signatures and detailed parameter descriptions

## Coding Guidelines for GitHub Copilot

When suggesting code for this project:

1. **Watch Face Development**: Always implement all 4 required functions (setup, activate, loop, resign)
2. **Power Efficiency**: Consider power consumption in all suggestions - prefer STANDBY-compatible operations
3. **Display Management**: Remember that `watch_display_string()` does NOT clear the display automatically
4. **Year Handling**: Account for year offset from 2020 when working with RTC functions
5. **Memory Safety**: Use appropriate bounds checking for arrays and string operations
6. **Peripheral Conflicts**: Be aware that LED and buzzer share TCC peripheral
7. **Build System**: Always specify COLOR variable when suggesting build commands
8. **Storage Safety**: Always erase flash rows before writing new data
9. **Event-Driven Design**: Watch face loop functions should be event-driven and return quickly
10. **Platform Abstraction**: Use watch library APIs instead of direct hardware access
