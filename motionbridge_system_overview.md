# MotionBridge System Overview

## How MotionBridge Works

MotionBridge acts as a bridge between games/simulators and motion platforms, processing telemetry data through a plugin-based architecture.

---

## System Flow

```
[Game Plugin] → [MotionBridge Core] → [Motion Interface Plugin] → [Motion Platform]
     ↓                    ↓                        ↓
  Telemetry          Processing              UDP Output
   Capture       (Filters/Scaling)          
```

---

## Application Lifecycle

### 1. Plugin Discovery & Loading

**On Application Start:**
- MotionBridge scans designated plugin directories
- Validates each plugin by checking for:
  - `[Plugin]` attribute with required metadata
  - Inheritance from `GamePluginAPI<T>` or `MotionInterfaceAPI<T>`
- Loads only plugins that meet validation requirements
- Invalid plugins are silently rejected

**Default Selection:**
- First valid game plugin is selected as active
- First valid motion interface is selected as output
- Users can change selection via UI

---

### 2. Telemetry Startup Sequence

**When User Starts Telemetry:**

1. **Configuration Patching** (if required by plugin)
   - Calls `PatchConfigFile()` on the active game plugin
   - Enables telemetry output in game configuration
   - Creates backups before modification

2. **Game Launch** (if auto-start enabled)
   - Launches game executable if `ExclusionFromProccessStart` is not set
   - Games marked with `ExclusionFromProccessStart` must be started manually

3. **Connection Establishment**
   - Calls `Initialize()` on game plugin
   - Sets up UDP listeners or API connections (e.g., SimConnect)
   - Calls `Start()` when ready to receive data

4. **Telemetry Processing Loop** (begins automatically)
   - See detailed flow below

---

### 3. Real-Time Telemetry Processing

**Continuous Loop (high frequency):**

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Poll Game Plugin                                         │
│    └─> AttemptTelemetryFrame() - Get new data from game    │
│                                                              │
│ 2. Apply Processing (MotionBridge Core)                     │
│    ├─> Axis Inversion (flip axes if needed)                │
│    ├─> Gain Scaling (amplify/reduce motion intensity)      │
│    └─> Limits/Clamping (prevent excessive movement)        │
│                                                              │
│ 3. Send to Motion Platform                                  │
│    └─> SendTelemetry() - Transmit to configured IP/port    │
└─────────────────────────────────────────────────────────────┘
```

**Flow Details:**

- **Game Plugin** provides raw telemetry data
- **MotionBridge Core** applies user-configured adjustments:
  - **Axis Inversion:** Reverse axis direction (e.g., pitch up becomes pitch down)
  - **Gains:** Multiply axis values to increase/decrease intensity
  - **Limits:** Clamp values to prevent hardware overextension
- **Motion Interface** sends processed data to configured target IP

---

### 4. Shutdown Sequence

**When User Stops Telemetry:**

1. Stop data collection from game
2. Call `Shutdown()` on game plugin
   - Close connections
   - Stop background threads
   - Clean up resources
3. Call `Shutdown()` on motion interface
   - Close UDP connections
   - Final cleanup

---

## Plugin Responsibilities

### Game Plugins Must Provide:

- ✅ **Data Capture:** Extract telemetry from game
- ✅ **Format Conversion:** Convert to `TelemetryFrame` structure
- ✅ **Connection Management:** Handle game API lifecycle
- ❌ **Processing/Filtering:** NOT responsible for gains, limits, or scaling

### MotionBridge Core Handles:

- ✅ **Plugin Discovery:** Find and validate plugins
- ✅ **Data Processing:** Apply inversion, gains, limits
- ✅ **User Configuration:** Store and apply user settings
- ✅ **Routing:** Move data from game plugin to motion interface

### Motion Interface Plugins Must Provide:

- ✅ **Data Transmission:** Send to motion platform
- ✅ **Format Conversion:** Convert `TelemetryFrame` to platform-specific format
- ✅ **Axis Mapping:** Reorder axes if platform requires specific layout
- ❌ **Processing:** NOT responsible for gains or limits (already applied)

---

## Configuration Flow

### User-Adjustable Settings (Applied by Core):

**Per-Axis Configuration:**
- **Inversion:** Toggle to reverse axis direction
- **Gain:** Multiplier for motion intensity (0.1x to 5.0x typical)
- **Limits:** Maximum/minimum values to prevent overtravel

**Network Configuration:**
- **Output IP:** Motion platform target address
- **Output Port:** Motion platform target port

**Plugin Configuration:**
- **Active Game Plugin:** Which game to capture from
- **Active Motion Interface:** Which platform to send to
- **Plugin-Specific Settings:** Paths, ports, etc. from plugin's settings class

---

## Data Processing Pipeline Example

**Raw Data from Game:**
```
Pitch: 15.0°
Roll: -8.0°
Heave: 2.5 m/s²
```

**After User Configuration:**
- Pitch: Gain = 1.5x, Limit = ±20°
- Roll: Inverted, Gain = 2.0x, Limit = ±25°
- Heave: Gain = 0.8x, Limit = ±5 m/s²

**Processed Output:**
```
Pitch: 15.0 × 1.5 = 22.5° → Clamped to 20.0°
Roll: -8.0 × -1 × 2.0 = 16.0° (inverted)
Heave: 2.5 × 0.8 = 2.0 m/s²
```

**Sent to Motion Platform:**
- Motion Interface transmits processed values
- Platform applies values to actuators

---

## Plugin Developer Guidelines

### What Plugins Should NOT Do:

❌ **Don't apply filtering** - MotionBridge core handles this
❌ **Don't scale values** - User-configured gains are applied by core
❌ **Don't clamp values** - Limits are applied by core
❌ **Don't invert axes** - Core handles inversion configuration

### What Plugins SHOULD Do:

✅ **Provide raw, accurate data** - Let core do the processing
✅ **Convert units to standard format** - Use SI units in `TelemetryFrame`
✅ **Return quickly when no data available** - Don't block `AttemptTelemetryFrame()`
✅ **Handle connection lifecycle properly** - Initialize/Start/Shutdown

---

## Architecture Benefits

### Separation of Concerns:

- **Plugins:** Focus only on data capture/transmission
- **Core:** Handles all processing and user configuration
- **Result:** Plugins are simple and reusable

### User Control:

- All processing parameters adjustable without plugin changes
- Settings persist across sessions
- Easy to tune for different motion platforms

### Extensibility:

- New game plugins only need to extract data
- New motion interfaces only need to send data
- No complex processing logic required in plugins

---

## Quick Reference

### For Plugin Developers:

**Game Plugin Checklist:**
1. Extract data from game
2. Convert to `TelemetryFrame` (use SI units)
3. Return `true` only when new data available
4. Return `false` quickly when no data
5. Don't apply processing - core handles it

**Motion Interface Checklist:**
1. Accept `TelemetryFrame` data (already processed)
2. Convert to platform-specific format
3. Transmit to configured IP/port
4. Handle transmission errors gracefully

### For Users:

**Typical Workflow:**
1. Launch MotionBridge
2. Select game plugin
3. Select motion interface  
4. Configure output IP/port
5. Start telemetry (game launches automatically if supported)
6. Adjust gains/limits/inversion as needed
7. Enjoy motion simulation!

---

## Support

For questions about the system architecture or plugin development, contact the MotionBridge development team.

**Version:** 1.0.0  
**Last Updated:** December 2024