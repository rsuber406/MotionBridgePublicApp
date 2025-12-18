# MotionBridge Plugin API Documentation

## Overview

The MotionBridge Plugin API provides a framework for creating game plugins and motion platform interfaces. It uses a templated settings system with attributes to define plugin metadata and configuration requirements.

---

## Core Architecture

### Two Plugin Types

1. **Game Plugins** (`GamePluginAPI<TSettings>`) - Extract telemetry data from games/simulators
2. **Motion Interface Plugins** (`MotionInterfaceAPI<TSettings>`) - Send telemetry data to motion platforms

Both use generic type parameters (`TSettings`) for strongly-typed configuration.

---

## Data Structures

### `TelemetryFrame`

The core data structure for motion telemetry. All values are in SI units unless otherwise specified.

```csharp
public struct TelemetryFrame
{
    // Rotation (in degrees)
    public float pitch;    // Nose up/down
    public float yaw;      // Nose left/right
    public float roll;     // Wing tilt left/right

    // Translation (in meters)
    public float heave;    // Up/down movement
    public float surge;    // Forward/backward movement
    public float sway;     // Left/right movement

    // Velocity (in meters/second)
    public float velocityX;
    public float velocityY;
    public float velocityZ;

    // Position (in meters)
    public float posX;
    public float posY;
    public float posZ;

    // Timing
    public double timestamp;  // Unix timestamp or game time
}
```

---

## Attributes System

### `[Plugin]` Attribute

Declares plugin metadata. Applied to the plugin class.

> **⚠️ REQUIRED:** The `[Plugin]` attribute is **mandatory**. MotionBridge will **reject and not load** any plugin class that does not have this attribute. Your plugin will be silently discarded during the loading process if this attribute is missing.

```csharp
[Plugin(
    pluginAuthor = "Your Name",
    gameName = "Game Title",
    pluginName = "Plugin Name",
    pluginVersion = "1.0.0"
)]
public class MyPlugin : GamePluginAPI<MySettings>
{
    // ...
}
```

**Properties:**
- `pluginAuthor` - Developer/organization name **(required)**
- `gameName` - Target game/simulator name **(required)**
- `pluginName` - Display name for the plugin **(required)**
- `pluginVersion` - Semantic version string **(required)**

**All four properties must be provided** or the plugin will not be recognized.

#### ❌ Invalid - Plugin Will Not Load

```csharp
// Missing [Plugin] attribute entirely
public class MyPlugin : GamePluginAPI<MySettings>
{
    // This plugin will be REJECTED by MotionBridge
}
```

```csharp
// Missing required properties
[Plugin(pluginName = "My Plugin")]  // Missing author, game, version
public class MyPlugin : GamePluginAPI<MySettings>
{
    // This plugin will be REJECTED by MotionBridge
}
```

#### ✅ Valid - Plugin Will Load

```csharp
[Plugin(
    pluginAuthor = "Ryan Suber",
    gameName = "Microsoft Flight Simulator 2020",
    pluginName = "MSFS2020 Plugin",
    pluginVersion = "1.0.0"
)]
public class MSFSPlugin : GamePluginAPI<MSFS_Settings>
{
    // This plugin will be loaded successfully
}
```

---

### `[PluginSetting]` Attribute

Declares configuration properties with default values. Applied to properties in your settings class.

```csharp
public class MySettings
{
    [PluginSetting("C:\\path\\to\\game.exe", SettingKind.ExecutablePath)]
    public string pathToExe { get; set; }
    
    [PluginSetting(12345, SettingKind.Port)]
    public int port { get; set; }
}
```

**Constructor:**
```csharp
PluginSettingAttribute(object value, SettingKind settingKind)
```
- `value` - Default value for this setting
- `settingKind` - Categorization hint for the UI/system

---

### `SettingKind` Enum

Categorizes plugin settings for UI presentation and validation.

```csharp
public enum SettingKind
{
    IPAddress,                    // IP address string
    Port,                         // Network port number
    ConfigPath,                   // Path to configuration file
    ExecutablePath,               // Path to game executable
    FilePath,                     // Generic file path
    Heave,                        // Heave axis configuration
    Surge,                        // Surge axis configuration
    Sway,                         // Sway axis configuration
    Roll,                         // Roll axis configuration
    Pitch,                        // Pitch axis configuration
    Yaw,                          // Yaw axis configuration
    Acceleration,                 // Acceleration configuration
    ExclusionFromProccessStart    // Disable auto-start (see below)
}
```

---

## ⚠️ CRITICAL: Auto-Start Behavior

### `SettingKind.ExclusionFromProccessStart`

**This setting controls whether MotionBridge will attempt to automatically launch the game.**

#### When to USE this setting:

Use `ExclusionFromProccessStart` when your game/simulator **MUST be launched manually** because:

- It uses an external API that requires the game to be running first (e.g., MSFS 2020 with SimConnect)
- The game has a launcher or authentication system
- Auto-starting would break the connection mechanism
- The plugin connects to an already-running process

**Example (MSFS 2020 - requires manual start):**
```csharp
public class MSFS_Settings
{
    [PluginSetting("C:\\path\\to\\MSFS.exe", SettingKind.ExecutablePath)]
    public string pathToExe { get; set; }
    
    // This tells MotionBridge: DO NOT auto-start this game
    [PluginSetting(true, SettingKind.ExclusionFromProccessStart)]
    public bool exclusionFromProccessStart { get; set; }
}
```

#### When to NOT USE this setting:

**DO NOT include this setting** if your game can be auto-started normally, such as:

- Games with UDP telemetry that broadcast automatically on launch
- Games that can be launched via executable without issues
- Games where the plugin doesn't depend on connection order

**Example (Live for Speed - can auto-start):**
```csharp
public class LFS_Settings
{
    [PluginSetting("C:\\path\\to\\LFS.exe", SettingKind.ExecutablePath)]
    public string pathToExe { get; set; }
    
    [PluginSetting(20777, SettingKind.Port)]
    public int port { get; set; }
    
    // NO ExclusionFromProccessStart setting = auto-start is ENABLED
}
```

#### Implementation Details

**Value:** Always use `true` when including this setting (it's a flag, not a toggle)

```csharp
// Correct
[PluginSetting(true, SettingKind.ExclusionFromProccessStart)]
public bool exclusionFromProccessStart { get; set; }

// Don't do this - if you need the setting, set it to true
[PluginSetting(false, SettingKind.ExclusionFromProccessStart)]
public bool exclusionFromProccessStart { get; set; }  // Wrong - just omit it instead
```

**If the setting is present (with value `true`):** MotionBridge will NOT attempt to launch the game executable. The user must start the game manually.

**If the setting is absent:** MotionBridge WILL automatically launch the game using the `ExecutablePath` setting.

---

## Game Plugin API

### `GamePluginAPI<TSettings>`

Abstract base class for game plugins. Inherit from this to create a new game integration.

```csharp
public class MyGamePlugin : GamePluginAPI<MyGameSettings>
{
    // Implement required methods
}
```

---

### Lifecycle Methods

#### `Initialize(IPAddress address)`

Called once during plugin startup. Set up network connections, initialize libraries, start background threads.

**Parameters:**
- `address` - IP address for UDP communication (typically localhost)

**Returns:** `bool`
- `true` - Initialization successful
- `false` - Initialization failed, plugin will not be used

**Example:**
```csharp
public override bool Initialize(IPAddress address)
{
    running = true;
    workerThread = new Thread(WorkerMethod)
    {
        IsBackground = true,
        Name = "Game Data Thread"
    };
    workerThread.Start();
    return true;
}
```

---

#### `Start()`

Called when the system is ready to begin capturing telemetry. Start data requests, enable polling, etc.

**Returns:** `bool`
- `true` - Successfully started capturing
- `false` - Failed to start (game not running, connection failed, etc.)

**Example:**
```csharp
public override bool Start()
{
    if (gameConnection == null) return false;
    
    gameConnection.RequestDataStream();
    isCapturing = true;
    return true;
}
```

---

#### `Shutdown()`

Called when the plugin is being unloaded. Clean up resources, stop threads, close connections.

**Must be graceful** - this is called during normal application shutdown.

**Example:**
```csharp
public override void Shutdown()
{
    running = false;
    
    if (connection != null)
    {
        connection.Dispose();
        connection = null;
    }
    
    if (workerThread != null && workerThread.IsAlive)
    {
        workerThread.Join(1000); // Wait up to 1 second
    }
}
```

---

#### `AttemptTelemetryFrame(ref TelemetryFrame telemetryFrame)`

Called repeatedly (high frequency) to poll for new telemetry data. Populate the provided frame if new data is available.

**Parameters:**
- `telemetryFrame` - Reference to frame struct to populate

**Returns:** `bool`
- `true` - New data available, frame was populated
- `false` - No new data, frame unchanged

**Important:** This is called at high frequency. Return `false` quickly if no data is available.

**Example:**
```csharp
public override bool AttemptTelemetryFrame(ref TelemetryFrame telemetryFrame)
{
    if (!newDataAvailable) return false;
    
    lock (_lockObject)
    {
        telemetryFrame.pitch = currentData.Pitch;
        telemetryFrame.roll = currentData.Roll;
        telemetryFrame.yaw = currentData.Yaw;
        // ... populate other fields ...
        
        newDataAvailable = false;
    }
    
    return true;
}
```

---

#### `SetPathToConfigFile(string path)`

Called to provide the path to the game's configuration file (if needed for patching).

**Parameters:**
- `path` - Full path to configuration file

**Note:** Some games don't need config file modification. Implement as empty method if not needed.

---

#### `PatchConfigFile()`

Called to modify the game's configuration to enable telemetry output.

**Returns:** `bool`
- `true` - Config successfully patched (or patching not needed)
- `false` - Failed to patch config

**Note:** Return `true` if your plugin doesn't need config patching (e.g., using an API like SimConnect).

---

### Properties

#### `Settings`

Strongly-typed settings object of type `TSettings`.

```csharp
public TSettings Settings { get; set; }
```

Access your plugin's configuration:
```csharp
string exePath = Settings.pathToExe;
int port = Settings.port;
```

#### `Metadata`

Plugin metadata from the `[Plugin]` attribute. Set by the system.

```csharp
public PluginAttribute Metadata { get; set; }
```

---

## Motion Interface API

### `MotionInterfaceAPI<TSettings>`

Abstract base class for motion platform interfaces. Inherit from this to support a new motion platform.

```csharp
public class MyMotionPlatform : MotionInterfaceAPI<MyPlatformSettings>
{
    // Implement required methods
}
```

---

### Lifecycle Methods

#### `Initialize(IPAddress address, int port)`

Set up connection to the motion platform.

**Parameters:**
- `address` - Motion platform IP address
- `port` - Motion platform port number

**Returns:** `bool`
- `true` - Successfully connected
- `false` - Connection failed

---

#### `Start()`

Begin accepting telemetry data for transmission.

**Returns:** `bool`
- `true` - Ready to send data
- `false` - Not ready

---

#### `Shutdown()`

Close connection to motion platform, clean up resources.

**Returns:** `bool`
- `true` - Clean shutdown
- `false` - Shutdown issues (non-critical)

---

#### `SendTelemetry(ref TelemetryFrame telemetryFrame)`

Send telemetry data to the motion platform.

**Parameters:**
- `telemetryFrame` - Reference to telemetry data

**Returns:** `bool`
- `true` - Data sent successfully
- `false` - Send failed (connection lost, buffer full, etc.)

**Example:**
```csharp
public override bool SendTelemetry(ref TelemetryFrame telemetryFrame)
{
    try
    {
        byte[] packet = SerializeTelemetry(telemetryFrame);
        udpClient.Send(packet, packet.Length);
        return true;
    }
    catch (SocketException)
    {
        return false;
    }
}
```

---

## Settings System

### How Settings Work

1. Define a settings class with properties
2. Annotate properties with `[PluginSetting]` attributes
3. Pass the settings class as the generic type parameter to `GamePluginAPI<TSettings>`
4. The system automatically applies default values from attributes on construction

**Example:**
```csharp
public class MyGameSettings
{
    [PluginSetting("C:\\Games\\MyGame\\game.exe", SettingKind.ExecutablePath)]
    public string pathToExe { get; set; }
    
    [PluginSetting(20777, SettingKind.Port)]
    public int udpPort { get; set; }
}

[Plugin(
    pluginAuthor = "Me",
    gameName = "My Game",
    pluginName = "My Game Plugin",
    pluginVersion = "1.0.0"
)]
public class MyGamePlugin : GamePluginAPI<MyGameSettings>
{
    public override bool Initialize(IPAddress address)
    {
        // Settings.pathToExe is already set to default value
        string exe = Settings.pathToExe;
        // ...
    }
}
```

### Default Value Application

The system automatically applies default values from `[PluginSetting]` attributes during construction:

- If a property is `null`, the default is applied
- If a property is a value type and equals its default value (e.g., `0` for `int`), the default is applied
- Otherwise, the existing value is preserved

This happens in the constructor before any of your methods are called.

---

## Best Practices

### Threading

- Use background threads for blocking operations (network polling, game API message loops)
- Protect shared data with locks when accessed from multiple threads
- Make `AttemptTelemetryFrame()` fast - it's called at high frequency

### Error Handling

- Return `false` from lifecycle methods on failure rather than throwing exceptions
- Use try-catch blocks around external API calls (game APIs, network operations)
- Log errors for debugging but don't crash

### Resource Cleanup

- Always implement proper cleanup in `Shutdown()`
- Dispose of unmanaged resources (network connections, API handles)
- Join background threads with timeouts to avoid hanging

### Data Freshness

- Use a flag or timestamp to track when new data arrives
- Return `false` from `AttemptTelemetryFrame()` if no new data is available
- Don't block waiting for data in `AttemptTelemetryFrame()`

---

## Common Patterns

### Background Thread for Data Collection

```csharp
private Thread dataThread;
private volatile bool running;
private bool newDataAvailable;
private readonly object lockObject = new object();

public override bool Initialize(IPAddress address)
{
    running = true;
    dataThread = new Thread(DataCollectionLoop)
    {
        IsBackground = true,
        Name = "Data Collection"
    };
    dataThread.Start();
    return true;
}

private void DataCollectionLoop()
{
    while (running)
    {
        // Collect data from game
        var data = GetGameData();
        
        lock (lockObject)
        {
            cachedData = data;
            newDataAvailable = true;
        }
    }
}

public override bool AttemptTelemetryFrame(ref TelemetryFrame telemetryFrame)
{
    if (!newDataAvailable) return false;
    
    lock (lockObject)
    {
        PopulateFrame(ref telemetryFrame, cachedData);
        newDataAvailable = false;
    }
    
    return true;
}
```

---

## Plugin Validation Checklist

Before submitting your plugin, verify:

- ✅ `[Plugin]` attribute is present on your plugin class with all four required properties
- ✅ Your class inherits from `GamePluginAPI<TSettings>` or `MotionInterfaceAPI<TSettings>`
- ✅ All abstract methods are implemented
- ✅ Settings class has `[PluginSetting]` attributes for configuration properties
- ✅ `ExclusionFromProccessStart` is correctly set (or omitted) based on your game's requirements
- ✅ `Shutdown()` properly cleans up all resources
- ✅ `AttemptTelemetryFrame()` returns quickly when no data is available
- ✅ Thread safety is handled correctly for shared data

---

## Support

For questions, bug reports, or feature requests, contact the MotionBridge development team.

**Version:** 1.0.0  
**Last Updated:** December 2024