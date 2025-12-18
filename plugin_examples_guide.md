# MotionBridge Plugin Examples Guide

This guide demonstrates how to create plugins for MotionBridge with three complete examples:

1. **Live For Speed (LFS)** - Traditional UDP telemetry
2. **Microsoft Flight Simulator 2020 (MSFS)** - External API integration with SimConnect
3. **Generic Motion Interface** - Motion platform output plugin

---

## Example 1: Live For Speed Plugin (UDP Telemetry)

This example shows a typical UDP-based game plugin that receives telemetry data on a network port.

### Overview

- **Game:** Live For Speed
- **Connection Method:** UDP broadcast from game
- **Auto-Start:** Supported (game can be launched automatically)
- **Config Patching:** Required (enables OutSim telemetry)

### Complete Implementation

```csharp
using System.Net;
using System.Net.Sockets;
using System.Diagnostics;

namespace LiveForSpeed_Plugin;

public class LFSPluginSettings()
{
    [PluginSetting(5930, SettingKind.Port)] 
    public int portNumber { get; set; }
         
    [PluginSetting("C:\\gameExePath", SettingKind.ExecutablePath)]
    public string pathToGameExecutable { get; set; }
         
    [PluginSetting("C:\\gameConfigPath", SettingKind.ConfigPath)]
    public string pathToConfigFile { get; set; }
    
    // Note: No ExclusionFromProccessStart - this game CAN auto-start
}

[Plugin(
    pluginAuthor = "Ryan Suber",
    pluginName = "LiveForSpeed",
    pluginVersion = "1.0.0",
    gameName = "LiveForSpeed"
)]
public class LFSPlugin : GamePluginAPI<LFSPluginSettings>
{
    private UdpClient? udpClient;
    private IPEndPoint? udpEndPoint;
    
    public override bool Initialize(IPAddress address)
    {
        if (udpClient != null && udpEndPoint != null) return true;
        
        udpClient = new UdpClient(Settings.portNumber);
        udpEndPoint = new IPEndPoint(IPAddress.Any, 0);
        
        if (udpClient == null || udpEndPoint == null || 
            udpEndPoint.Address.Equals(IPAddress.None)) 
            return false;

        return true;
    }

    public override bool Start()
    {
        return true;
    }

    public override void Shutdown()
    {
        // UDP client cleanup handled automatically by dispose
    }

    public override bool AttemptTelemetryFrame(ref TelemetryFrame telemetryFrame)
    {
        if (udpClient == null) return false;

        // Non-blocking check if data is available
        if (udpClient.Client.Poll(0, SelectMode.SelectRead))
        {
            try
            {
                byte[] buffer = udpClient.Receive(ref udpEndPoint);
                ParseTelemetryFrame(ref telemetryFrame, buffer);
                return true;
            }
            catch (Exception ex)
            {
                // Handle error silently
            }
        }
        return false;
    }

    private void ParseTelemetryFrame(ref TelemetryFrame telemetryFrame, byte[] buffer)
    {
        using var ms = new MemoryStream(buffer);
        using var br = new BinaryReader(ms);
        
        telemetryFrame.timestamp = br.ReadInt32();
        br.BaseStream.Seek(12, SeekOrigin.Current); // Skip angular velocity
        telemetryFrame.yaw = br.ReadSingle();
        telemetryFrame.pitch = br.ReadSingle();
        telemetryFrame.roll = br.ReadSingle();
        br.BaseStream.Seek(4, SeekOrigin.Current); // Skip lateral accel
        telemetryFrame.surge = br.ReadSingle();
        telemetryFrame.heave = br.ReadSingle();
    }

    public override void SetPathToConfigFile(string path)
    {
        Settings.pathToConfigFile = path;
    }

    public override bool PatchConfigFile()
    {
        bool completed = false;

        if (string.IsNullOrWhiteSpace(Settings.pathToConfigFile))
            throw new Exception("pathToConfigFile is NULL or blank!");
            
        if (File.Exists(Settings.pathToConfigFile))
        {
            string directory = Path.GetDirectoryName(Settings.pathToConfigFile);
            string backupPath = Path.Combine(directory, "cfg.bac");
            File.Copy(Settings.pathToConfigFile, backupPath, true);
            
            string[] lineToEdit = new string[]
            {
                "OutSim Mode",
                "OutSim Delay",
                "OutSim IP",
                "OutSim Port"
            };

            string[] replacement = new string[]
            {
                "OutSim Mode 1",
                "OutSim Delay 1",
                "OutSim IP 127.0.0.1",
                $"OutSim Port {Settings.portNumber}"
            };
            
            completed = PatchCfgFile(Settings.pathToConfigFile, lineToEdit, replacement);
        }
        else
        {
            Debug.WriteLine($"Failed to find config file: {Settings.pathToConfigFile}");
        }

        return completed;
    }

    private bool PatchCfgFile(string path, string[] linesToEdit, string[] replacementLines)
    {
        bool hasEdited = false;
        
        if (File.Exists(path))
        {
            string[] lines = File.ReadAllLines(path);
            
            for (int i = 0; i < linesToEdit.Length; i++)
            {
                for (int j = 0; j < lines.Length; j++)
                {
                    if (lines[j].Contains(linesToEdit[i]))
                    {
                        lines[j] = replacementLines[i];
                        hasEdited = true;
                        break;
                    }
                }
            }

            if (hasEdited)
            {
                File.WriteAllLines(path, lines);
            }
        }

        return hasEdited;
    }
}
```

### Key Features

**UDP Polling Pattern:**
- Uses non-blocking `Poll()` to check for data availability
- Returns `false` immediately if no data is ready
- Efficient for high-frequency polling

**Config Patching:**
- Creates backup before modifying game config
- Enables OutSim telemetry output
- Configures game to send data to correct port

**Binary Data Parsing:**
- Uses `BinaryReader` for structured data
- Seeks past unwanted data fields
- Directly populates telemetry frame

---

## Example 2: MSFS 2020 Plugin (SimConnect API)

This example demonstrates integration with an external game API that requires special handling.

### Overview

- **Game:** Microsoft Flight Simulator 2020
- **Connection Method:** SimConnect COM API
- **Auto-Start:** NOT supported (must be launched manually)
- **Config Patching:** Not required (API handles configuration)

### Required Dependencies

**CRITICAL:** Use `System.Windows.Forms` from the .NET Framework for SimConnect compatibility:

```xml
<ItemGroup>
    <Reference Include="System.Windows.Forms" />
    <PackageReference Include="Microsoft.FlightSimulator.SimConnect" Version="..." />
</ItemGroup>
```

**Do NOT use** `Microsoft.Windows.Forms` or other modern WPF/WinUI packages - they are incompatible with SimConnect's message pump requirements.

### Settings Class

```csharp
using System.Net;
using System.Runtime.InteropServices;
using Microsoft.FlightSimulator.SimConnect;
using System.Windows.Forms;

namespace MSFS2020_Plugin;

public class MSFS_Settings
{
    [PluginSetting("C:\\pathToExe", SettingKind.ExecutablePath)]
    public string pathToExe { get; set; }
    
    [PluginSetting("C:\\pathToConfigNotNeeded", SettingKind.ConfigPath)]
    public string pathToConfigNotNeeded { get; set; }
    
    // CRITICAL: This flag prevents auto-start
    [PluginSetting(true, SettingKind.ExclusionFromProccessStart)]
    public bool exclusionFromProccessStart { get; set; }
}
```

### Plugin Implementation

```csharp
[Plugin(
    pluginAuthor = "Ryan Suber",
    gameName = "Microsoft Flight Simulator 2020",
    pluginName = "MSFS2020 Plugin",
    pluginVersion = "1.0.0"
)]
public class MSFSPlugin : GamePluginAPI<MSFS_Settings>
{
    private Thread simConnectThread;
    private SimConnect simConnect;
    private volatile bool running;
    private bool newDataAvailable = false;
    private SimConnectForm form;
    private readonly object _lockObject = new object();

    [StructLayout(LayoutKind.Sequential)]
    private struct AircraftData
    {
        public double Pitch;
        public double Bank;
        public double Heading;
        public double AccelZ;
    }
    
    private AircraftData aircraftData = new AircraftData();
    
    private enum DATA_DEF { Aircraft }
    private enum DATA_REQ { Aircraft }
    private enum SYSTEM_EVENT { SimStart }
    
    public override bool Initialize(IPAddress address)
    {
        running = true;
        simConnectThread = new Thread(SimConnectThread)
        {
            IsBackground = true,
            Name = "MSFS SimConnect Thread"
        };
        simConnectThread.Start();
        return true;
    }

    public override bool Start()
    {
        if (simConnect == null) return false;

        lock (_lockObject)
        {
            newDataAvailable = false;
        }

        return true;
    }

    public override void Shutdown()
    {
        running = false;
        
        if (simConnect != null)
        {
            simConnect.Dispose();
            simConnect = null;
        }

        if (form != null && !form.IsDisposed)
        {
            form.Invoke(new Action(() => { form.Close(); }));
        }

        if (simConnectThread != null && simConnectThread.IsAlive)
        {
            simConnectThread.Join(500);
        }
    }

    public override bool AttemptTelemetryFrame(ref TelemetryFrame telemetryFrame)
    {
        if (!newDataAvailable) return false;

        lock (_lockObject)
        {
            telemetryFrame.pitch = (float)aircraftData.Pitch;
            telemetryFrame.roll = (float)aircraftData.Bank;
            telemetryFrame.yaw = (float)aircraftData.Heading;
            
            double rawAccelZ = aircraftData.AccelZ;
            if (rawAccelZ > 0.5f || rawAccelZ < -0.5f)
            {
                float accelZ = (float)rawAccelZ - 32.174f;
                // TODO: Apply filtering and scaling
            }
            else
            {
                telemetryFrame.heave = 0.0f;
            }
            
            newDataAvailable = false;
        }

        return true;
    }

    public override void SetPathToConfigFile(string path)
    {
        // Not needed for SimConnect
    }

    public override bool PatchConfigFile()
    {
        // SimConnect handles configuration
        return true;
    }

    private void SimConnectThread()
    {
        try
        {
            form = new SimConnectForm();
            form.Load += (s, e) =>
            {
                simConnect = new SimConnect(
                    "MotionBridge MSFS",
                    form.Handle,
                    0x0402,
                    null,
                    0
                );
                
                form.SetMessageHandler(() =>
                {
                    try
                    {
                        simConnect?.ReceiveMessage();
                    }
                    catch (Exception ex)
                    {
                        // Handle SimConnect errors
                    }
                });
                
                RegisterSimConnect();
            };
            
            System.Windows.Forms.Application.Run(form);
        }
        catch (Exception ex)
        {
            throw new Exception($"Failed to launch SimConnect: {ex}");
        }
    }

    private void RegisterSimConnect()
    {
        simConnect.AddToDataDefinition(
            DATA_DEF.Aircraft, 
            "PLANE PITCH DEGREES",
            "radians",
            SIMCONNECT_DATATYPE.FLOAT64,
            0,
            SimConnect.SIMCONNECT_UNUSED
        );
        
        simConnect.AddToDataDefinition(
            DATA_DEF.Aircraft, 
            "PLANE BANK DEGREES",
            "radians",
            SIMCONNECT_DATATYPE.FLOAT64,
            0,
            SimConnect.SIMCONNECT_UNUSED
        );
        
        simConnect.AddToDataDefinition(
            DATA_DEF.Aircraft, 
            "PLANE HEADING DEGREES",
            "radians",
            SIMCONNECT_DATATYPE.FLOAT64,
            0,
            SimConnect.SIMCONNECT_UNUSED
        );
        
        simConnect.AddToDataDefinition(
            DATA_DEF.Aircraft, 
            "ACCELERATION BODY Z",
            "feet per second squared",
            SIMCONNECT_DATATYPE.FLOAT64,
            0,
            SimConnect.SIMCONNECT_UNUSED
        );
        
        simConnect.RegisterDataDefineStruct<AircraftData>(DATA_DEF.Aircraft);
        simConnect.SubscribeToSystemEvent(
            SYSTEM_EVENT.SimStart,
            "SimStart"
        );
        
        simConnect.OnRecvSimobjectData += OnSimRecvData;
        
        // Request data updates on every visual frame
        simConnect.RequestDataOnSimObject(
            DATA_REQ.Aircraft, 
            DATA_DEF.Aircraft,
            SimConnect.SIMCONNECT_OBJECT_ID_USER,
            SIMCONNECT_PERIOD.VISUAL_FRAME,
            SIMCONNECT_DATA_REQUEST_FLAG.DEFAULT,
            0, 0, 0
        );
    }

    private void OnSimRecvData(SimConnect sender, SIMCONNECT_RECV_SIMOBJECT_DATA data)
    {
        if (data.dwRequestID != (uint)DATA_REQ.Aircraft) return;

        lock (_lockObject)
        {
            aircraftData = (AircraftData)data.dwData[0];
            newDataAvailable = true;
        }
    }
}
```

### SimConnect Form Helper

**CRITICAL:** This custom Form class is required for SimConnect to function. Without overriding `WndProc` and calling `ReceiveMessage()`, SimConnect will silently fail to deliver data.

```csharp
using System.Windows.Forms;

namespace MSFS2020_Plugin;

public class SimConnectForm : Form
{
    private const int WM_USER_SIMCONNECT = 0x0402;
    private Action _onMessageReceived;

    public void SetMessageHandler(Action handler)
    {
        _onMessageReceived = handler;
    }

    protected override void WndProc(ref Message m)
    {
        if (m.Msg == WM_USER_SIMCONNECT)
        {
            _onMessageReceived?.Invoke();
        }
        base.WndProc(ref m);
    }
}
```

### Key Features

**Background Thread with Message Loop:**
- SimConnect requires a Windows Forms message pump
- Thread runs `Application.Run(form)` to process Windows messages
- Form is invisible but necessary for message handling

**Custom WndProc Override:**
- **CRITICAL:** Without this, SimConnect data callbacks will never fire
- `ReceiveMessage()` must be called when `WM_USER_SIMCONNECT` message arrives
- This is the most common pitfall when using SimConnect

**Thread-Safe Data Access:**
- Uses `lock` to protect shared `aircraftData` struct
- `volatile bool` flag for cross-thread communication
- `newDataAvailable` flag prevents stale data transmission

**Graceful Shutdown:**
- Disposes SimConnect connection
- Closes form on UI thread using `Invoke()`
- Joins background thread with timeout

### Common SimConnect Pitfalls

❌ **Don't do this:**
- Forget to override `WndProc` (data will never arrive)
- Use wrong Windows Forms package (incompatible with SimConnect)
- Call SimConnect methods from wrong thread
- Forget to call `ReceiveMessage()` in message handler

✅ **Do this:**
- Always use custom Form with `WndProc` override
- Use `System.Windows.Forms` from .NET Framework
- Handle all SimConnect operations on the dedicated thread
- Set `ExclusionFromProccessStart` to prevent auto-launch

---

## Example 3: Generic Motion Interface Plugin

This example shows how to create a motion platform output plugin.

### Overview

- **Type:** Motion Interface (not a game plugin)
- **Connection Method:** UDP transmission to motion platform
- **Features:** Configurable axis mapping

### Complete Implementation

```csharp
using System.Net;
using System.Net.Sockets;

namespace GenericInterface;

public class GenericMotionSettings
{
    [PluginSetting("127.0.0.1", SettingKind.IPAddress)]
    public string ipAddress { get; set; }
    
    [PluginSetting(5931, SettingKind.Port)]
    public int port { get; set; }
    
    // Configurable axis positions in output array
    [PluginSetting(0, SettingKind.Heave)]
    public int heavePos { get; set; }
    
    [PluginSetting(1, SettingKind.Surge)]
    public int surgePos { get; set; }
    
    [PluginSetting(2, SettingKind.Sway)]
    public int swayPos { get; set; }
    
    [PluginSetting(3, SettingKind.Roll)]
    public int rollPos { get; set; }
    
    [PluginSetting(4, SettingKind.Pitch)]
    public int pitchPos { get; set; }
    
    [PluginSetting(5, SettingKind.Yaw)]
    public int yawPos { get; set; }
    
    [PluginSetting(6, SettingKind.Acceleration)]
    public int accelerationPos { get; set; }
}

[Plugin(
    pluginAuthor = "Ryan Suber",
    pluginName = "Generic Motion Interface",
    pluginVersion = "1.0.0",
    gameName = "Generic Motion Interface"
)]
public class GenericMotionInterface : MotionInterfaceAPI<GenericMotionSettings> 
{
    private UdpClient? client;
    private IPEndPoint? endpoint;

    public override bool Initialize(IPAddress address, int port)
    {
        Settings.port = port;
        Settings.ipAddress = address.ToString();
        
        client = new UdpClient();
        endpoint = new IPEndPoint(address, port);
        
        if (client != null && endpoint != null) return true;
        return false;
    }

    public override bool Start()
    {
        return true;
    }

    public override bool Shutdown()
    {
        client?.Dispose();
        return true;
    }

    public override bool SendTelemetry(ref TelemetryFrame telemetryFrame)
    {
        if (client == null || endpoint == null) return false;
        
        // Create output array with configurable axis positions
        float[] data = new float[6];
        data[Settings.heavePos] = telemetryFrame.heave;
        data[Settings.surgePos] = telemetryFrame.surge;
        data[Settings.swayPos] = telemetryFrame.sway;
        data[Settings.rollPos] = telemetryFrame.roll;
        data[Settings.pitchPos] = telemetryFrame.pitch;
        data[Settings.yawPos] = telemetryFrame.yaw;
        
        // Convert float array to byte array
        byte[] buffer = new byte[data.Length * sizeof(float)];
        Buffer.BlockCopy(data, 0, buffer, 0, buffer.Length);
        
        client?.Send(buffer, buffer.Length, endpoint);
        return true;
    }
}
```

### Key Features

**Configurable Axis Mapping:**
- Motion platforms may expect axes in different orders
- Settings allow reordering without code changes
- Default positions work for most platforms

**Efficient Data Serialization:**
- Uses `Buffer.BlockCopy` for fast memory copy
- Converts float array directly to bytes
- No serialization overhead

**Simple Lifecycle:**
- Minimal initialization required
- UDP client automatically managed
- Clean resource disposal

---

## Comparison Summary

| Feature | LFS (UDP) | MSFS (SimConnect) | Motion Interface |
|---------|-----------|-------------------|------------------|
| **Base Class** | `GamePluginAPI` | `GamePluginAPI` | `MotionInterfaceAPI` |
| **Auto-Start** | ✅ Yes | ❌ No (manual) | N/A |
| **Config Patching** | ✅ Required | ❌ Not needed | N/A |
| **Threading** | Main thread only | Background thread | Main thread only |
| **Special Requirements** | Binary parsing | Windows Forms API | Axis mapping |
| **Complexity** | Low | High | Low |

---

## Best Practices by Pattern

### UDP Polling Pattern (LFS)

✅ **Do:**
- Use non-blocking `Poll()` for availability check
- Handle exceptions silently in `AttemptTelemetryFrame()`
- Create backups before patching config files

❌ **Don't:**
- Block waiting for UDP data
- Throw exceptions on missing data
- Modify configs without backups

### External API Pattern (MSFS)

✅ **Do:**
- Use dedicated background thread for API message loop
- Override `WndProc` in custom Form class
- Protect shared data with locks
- Set `ExclusionFromProccessStart` flag
- Use `System.Windows.Forms` from .NET Framework

❌ **Don't:**
- Call API methods from wrong thread
- Forget `ReceiveMessage()` call
- Use modern WPF/WinUI Forms packages
- Attempt auto-launch for API-based games

### Motion Interface Pattern

✅ **Do:**
- Make axis mapping configurable
- Use efficient serialization (BlockCopy)
- Handle UDP send failures gracefully

❌ **Don't:**
- Hardcode axis positions
- Use slow serialization methods
- Crash on network errors

---

## Quick Start Checklist

When creating a new plugin, verify:

- ✅ `[Plugin]` attribute with all required fields
- ✅ Inherits from correct base class (`GamePluginAPI` or `MotionInterfaceAPI`)
- ✅ Settings class with `[PluginSetting]` attributes
- ✅ `ExclusionFromProccessStart` correctly set (or omitted)
- ✅ All abstract methods implemented
- ✅ Proper resource cleanup in `Shutdown()`
- ✅ Thread-safe if using background threads
- ✅ Returns `false` quickly from `AttemptTelemetryFrame()` when no data

---

## Support

For additional help or questions about plugin development, contact the MotionBridge development team.

**Version:** 1.0.0  
**Last Updated:** December 2024