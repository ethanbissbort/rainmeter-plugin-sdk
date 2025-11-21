# C# API Reference

Complete reference for the Rainmeter C# Plugin API (`RainmeterAPI.cs`).

## Table of Contents

1. [API Class](#api-class)
2. [Reading Options](#reading-options)
3. [String Operations](#string-operations)
4. [Executing Commands](#executing-commands)
5. [Getting Information](#getting-information)
6. [Logging](#logging)
7. [StringBuffer Class](#stringbuffer-class)
8. [DllExport Attribute](#dllexport-attribute)
9. [Complete Example](#complete-example)

---

## API Class

The `Rainmeter.API` class wraps the Rainmeter C API for use in C#.

### Creating an API Instance

```csharp
Rainmeter.API api = new Rainmeter.API(rm);
// or using implicit conversion:
Rainmeter.API api = (Rainmeter.API)rm;
```

---

## Reading Options

### ReadString

Retrieves a string option from the plugin measure.

```csharp
public string ReadString(string option, string defValue, bool replaceMeasures = true)
```

**Parameters:**
- `option` - Name of the option to read
- `defValue` - Default value if option not found
- `replaceMeasures` - If true, replaces section variables in the string

**Returns:** The option value as a string

**Example:**
```csharp
[DllExport]
public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
    Rainmeter.API api = new Rainmeter.API(rm);

    // Read with section variable replacement (default)
    string path = api.ReadString("FilePath", "C:\\Default\\");

    // Read without replacement (for actions/bangs)
    string command = api.ReadString("OnUpdate", "", false);
}
```

---

### ReadStringFromSection

Retrieves a string option from another measure or meter.

```csharp
public string ReadStringFromSection(string section, string option,
                                    string defValue, bool replaceMeasures = true)
```

**Parameters:**
- `section` - Name of the measure/meter section
- `option` - Name of the option to read
- `defValue` - Default value if not found
- `replaceMeasures` - If true, replaces section variables

**Returns:** The option value as a string

**Note:** Returns default value in older Rainmeter versions without support.

**Example:**
```csharp
string value = api.ReadStringFromSection("MyOtherMeasure", "Text", "");
```

---

### ReadInt

Retrieves an option as an integer.

```csharp
public int ReadInt(string option, int defValue)
```

**Parameters:**
- `option` - Name of the option
- `defValue` - Default value if not found or invalid

**Returns:** The option value as an integer

**Example:**
```csharp
int maxRetries = api.ReadInt("MaxRetries", 3);
bool enabled = api.ReadInt("Enabled", 1) == 1;  // Boolean as 0/1
```

---

### ReadIntFromSection

Retrieves an integer option from another section.

```csharp
public int ReadIntFromSection(string section, string option, int defValue)
```

**Note:** Returns default value in older Rainmeter versions.

---

### ReadDouble

Retrieves an option as a double.

```csharp
public double ReadDouble(string option, double defValue)
```

**Parameters:**
- `option` - Name of the option
- `defValue` - Default value if not found or invalid

**Returns:** The option value as a double

**Example:**
```csharp
double threshold = api.ReadDouble("Threshold", 0.75);
double timeout = api.ReadDouble("Timeout", 30.0);
```

---

### ReadDoubleFromSection

Retrieves a double option from another section.

```csharp
public double ReadDoubleFromSection(string section, string option, double defValue)
```

**Note:** Returns default value in older Rainmeter versions.

---

### ReadPath

Retrieves an option and converts relative path to absolute.

```csharp
public string ReadPath(string option, string defValue)
```

**Parameters:**
- `option` - Name of the option
- `defValue` - Default value if not found

**Returns:** Absolute path as a string

**Example:**
```csharp
// Converts "..\\Images\\bg.png" to absolute path
string imagePath = api.ReadPath("ImagePath", "");
```

---

## String Operations

### ReplaceVariables

Replaces variables and section variables in a string.

```csharp
public string ReplaceVariables(string str)
```

**Parameters:**
- `str` - String with unresolved variables (e.g., `#Variable#` or `[MeasureName]`)

**Returns:** String with variables replaced

**Example:**
```csharp
string resolved = api.ReplaceVariables("#MyVar# - [MeasureName]");

// Check if a variable equals something
string myVar = api.ReplaceVariables("#MyVar#").ToUpperInvariant();
if (myVar == "SOMETHING")
{
    // Do something
}
```

---

## Executing Commands

### Execute (Instance Method)

Executes a Rainmeter bang/command using the current measure's skin.

```csharp
public void Execute(string command)
```

**Parameters:**
- `command` - Bang to execute

**Example:**
```csharp
[DllExport]
public static double Update(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    // Execute commands
    measure.api.Execute("!SetVariable Count 10");
    measure.api.Execute("!UpdateMeter MyMeter");
    measure.api.Execute("!Redraw");

    return 0.0;
}
```

---

### Execute (Static Method)

Executes a Rainmeter bang/command with explicit skin pointer.

```csharp
public static void Execute(IntPtr skin, string command)
```

**Parameters:**
- `skin` - Pointer to the skin (from `api.GetSkin()`)
- `command` - Bang to execute

**Example:**
```csharp
IntPtr skin = api.GetSkin();
Rainmeter.API.Execute(skin, "!SetVariable Count 10");
```

---

## Getting Information

### GetMeasureName

Gets the name of the current measure.

```csharp
public string GetMeasureName()
```

**Returns:** The measure name as a string

**Example:**
```csharp
[DllExport]
public static void Initialize(ref IntPtr data, IntPtr rm)
{
    Measure measure = new Measure();
    Rainmeter.API api = new Rainmeter.API(rm);

    measure.name = api.GetMeasureName();  // Store for later use
    measure.api = api;

    data = GCHandle.ToIntPtr(GCHandle.Alloc(measure));
}
```

---

### GetSkin

Gets a pointer to the current skin (needed for Execute).

```csharp
public IntPtr GetSkin()
```

**Returns:** IntPtr to the current skin

**Example:**
```csharp
[DllExport]
public static void Initialize(ref IntPtr data, IntPtr rm)
{
    Measure measure = new Measure();
    Rainmeter.API api = new Rainmeter.API(rm);

    measure.skin = api.GetSkin();  // Store for API.Execute
    measure.api = api;

    data = GCHandle.ToIntPtr(GCHandle.Alloc(measure));
}
```

---

### GetSettingsFile (Static)

Gets the path to the Rainmeter.data file.

```csharp
public static string GetSettingsFile()
```

**Returns:** Path to Rainmeter.data as a string

**Example:**
```csharp
string dataFile = API.GetSettingsFile();

// Use with P/Invoke functions for reading/writing
GetPrivateProfileString("MyPlugin", "Key", "0", buffer, size, dataFile);
WritePrivateProfileString("MyPlugin", "Key", "Value", dataFile);
```

---

### GetSkinName

Gets the full path and name of the skin.

```csharp
public string GetSkinName()
```

**Returns:** Skin name as a string (e.g., "MySkin\\SubFolder")

**Example:**
```csharp
string skinName = api.GetSkinName();
```

---

### GetSkinWindow

Gets the handle to the skin window.

```csharp
public IntPtr GetSkinWindow()
```

**Returns:** IntPtr to the skin window handle (HWND)

**Example:**
```csharp
IntPtr hwnd = api.GetSkinWindow();
```

---

## Logging

### Log (Instance Method)

Sends a message to the Rainmeter log with source.

```csharp
public void Log(LogType type, string message)
```

**Log Types:**
- `API.LogType.Error` - Error messages
- `API.LogType.Warning` - Warning messages
- `API.LogType.Notice` - Notice messages
- `API.LogType.Debug` - Debug messages (only shown when Rainmeter is in debug mode)

**Example:**
```csharp
api.Log(API.LogType.Error, "Failed to open file");
api.Log(API.LogType.Warning, "Using default value");
api.Log(API.LogType.Notice, "Plugin initialized successfully");
api.Log(API.LogType.Debug, "Variable x = 42");
```

---

### Log (Static Method)

Sends a message to the Rainmeter log with explicit rm pointer.

```csharp
public static void Log(IntPtr rm, LogType type, string message)
```

**Example:**
```csharp
Rainmeter.API.Log(rm, API.LogType.Notice, "Message");
```

---

### LogF (Instance Method)

Sends a formatted message to the Rainmeter log.

```csharp
public void LogF(LogType type, string format, params Object[] args)
```

**Example:**
```csharp
int count = 42;
double value = 3.14;
api.LogF(API.LogType.Notice, "Count: {0}, Value: {1:F2}", count, value);

string name = "MyPlugin";
api.LogF(API.LogType.Debug, "Plugin '{0}' loaded", name);
```

---

### LogF (Static Method)

Sends a formatted message with explicit rm pointer.

```csharp
public static void LogF(IntPtr rm, LogType type, string format, params Object[] args)
```

**Example:**
```csharp
Rainmeter.API.LogF(rm, API.LogType.Notice, "Count: {0}", count);
```

---

## StringBuffer Class

Helper class for returning strings from `GetString()` to Rainmeter.

### Update

Updates the string buffer and returns a pointer to it.

```csharp
public static IntPtr Update(string value)
```

**Parameters:**
- `value` - The string to return (or null for number-only measures)

**Returns:** IntPtr to the string buffer

**Example:**
```csharp
[DllExport]
public static IntPtr GetString(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    // Return null for number-only measures
    if (measure.type == MeasureType.Number)
        return Rainmeter.StringBuffer.Update(null);

    // Return the string value
    return Rainmeter.StringBuffer.Update(measure.stringValue);
}
```

---

### Get

Gets the current string buffer pointer without updating.

```csharp
public static IntPtr Get()
```

**Returns:** IntPtr to the current buffer

---

## DllExport Attribute

Marks methods for export by DllExporter.exe.

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class DllExport : Attribute
```

**Usage:**
```csharp
[DllExport]
public static void Initialize(ref IntPtr data, IntPtr rm)
{
    // ...
}

[DllExport]
public static void Finalize(IntPtr data)
{
    // ...
}
```

**Important:** The post-build event runs DllExporter.exe to export these functions:
```
"$(SolutionDir)..\API\DllExporter.exe" "$(ConfigurationName)" "$(PlatformName)" "$(TargetDir)\" "$(TargetFileName)"
```

---

## Complete Example

```csharp
using System;
using System.Runtime.InteropServices;
using Rainmeter;

namespace MyPlugin
{
    internal class Measure
    {
        public Rainmeter.API api;
        public IntPtr skin;
        public string name;
        public int counter;
        public string output;

        public Measure()
        {
            counter = 0;
            output = "";
        }
    }

    public class Plugin
    {
        [DllExport]
        public static void Initialize(ref IntPtr data, IntPtr rm)
        {
            Measure measure = new Measure();
            Rainmeter.API api = new Rainmeter.API(rm);

            // Store API and info for later use
            measure.api = api;
            measure.skin = api.GetSkin();
            measure.name = api.GetMeasureName();

            data = GCHandle.ToIntPtr(GCHandle.Alloc(measure));

            api.Log(API.LogType.Notice, "Plugin initialized");
        }

        [DllExport]
        public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
        {
            Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
            Rainmeter.API api = new Rainmeter.API(rm);

            // Read options
            int startValue = api.ReadInt("StartValue", 0);
            measure.counter = startValue;

            // Log with formatting
            api.LogF(API.LogType.Debug, "StartValue set to {0}", startValue);

            maxValue = 100.0;
        }

        [DllExport]
        public static double Update(IntPtr data)
        {
            Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

            // Increment counter
            measure.counter++;

            // Prepare string for GetString
            measure.output = $"Count: {measure.counter}";

            // Execute a bang at certain values
            if (measure.counter == 50)
            {
                measure.api.Execute("!SetVariable HalfwayReached 1");
            }

            return (double)measure.counter;
        }

        [DllExport]
        public static IntPtr GetString(IntPtr data)
        {
            Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
            return Rainmeter.StringBuffer.Update(measure.output);
        }

        [DllExport]
        public static void ExecuteBang(IntPtr data,
            [MarshalAs(UnmanagedType.LPWStr)] String args)
        {
            Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

            if (args.ToLowerInvariant() == "reset")
            {
                measure.counter = 0;
            }
        }

        [DllExport]
        public static void Finalize(IntPtr data)
        {
            Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

            // Clean up resources if needed
            // measure.Dispose();

            GCHandle.FromIntPtr(data).Free();
        }
    }
}
```

---

## GCHandle Usage

C# plugins use `GCHandle` to pass managed objects through unmanaged pointers.

### Allocating

```csharp
Measure measure = new Measure();
data = GCHandle.ToIntPtr(GCHandle.Alloc(measure));
```

### Retrieving

```csharp
Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
```

### Freeing

```csharp
GCHandle.FromIntPtr(data).Free();
```

**Important:** Always free in Finalize to prevent memory leaks!

---

## P/Invoke for Rainmeter.data

To read/write to Rainmeter.data file:

```csharp
[DllImport("kernel32.dll", CharSet = CharSet.Unicode)]
static extern int GetPrivateProfileString(string section, string key,
    string defaultValue, [In, Out] char[] value, int size, string filePath);

[DllImport("kernel32", CharSet = CharSet.Unicode, SetLastError = true)]
[return: MarshalAs(UnmanagedType.Bool)]
private static extern bool WritePrivateProfileString(string section,
    string key, string value, string filePath);
```

**Usage:**
```csharp
string dataFile = API.GetSettingsFile();
char[] buffer = new char[256];

// Read
GetPrivateProfileString("MyPlugin", "Count", "0", buffer, 256, dataFile);
string value = new string(buffer);
int count = Convert.ToInt32(value);

// Write
WritePrivateProfileString("MyPlugin", "Count", count.ToString(), dataFile);
```

---

## Best Practices

1. **Store API reference**: Store the API object in Initialize for reuse
2. **Use try-catch for parsing**: Wrap conversions in try-catch blocks
3. **Validate inputs**: Check options for valid values
4. **Use appropriate log levels**: Debug for detailed info, Error for failures
5. **Always free GCHandles**: Free in Finalize to prevent leaks
6. **Handle null gracefully**: Check for null from API functions
7. **Use string interpolation**: Modern C# `$"Count: {value}"` syntax
8. **Read actions without replacement**: Use `false` for replaceMeasures when reading bangs

---

## Section Variables (C# Alternative)

C# doesn't support custom section variables like C++. However, you can:

1. Use `ExecuteBang` with `!CommandMeasure`
2. Use parent-child pattern to share data
3. Return values via `GetString` that other measures can read

**Example with ExecuteBang:**
```csharp
[DllExport]
public static void ExecuteBang(IntPtr data,
    [MarshalAs(UnmanagedType.LPWStr)] String args)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    // Parse command and arguments
    string[] parts = args.Split(new[] { ' ' }, 2);
    string command = parts[0].ToLowerInvariant();
    string argument = parts.Length > 1 ? parts[1] : "";

    switch (command)
    {
        case "toupper":
            measure.result = argument.ToUpper();
            break;

        case "tolower":
            measure.result = argument.ToLower();
            break;
    }
}
```

**Skin usage:**
```ini
[!CommandMeasure MyMeasure "ToUpper Hello World"]
```

---

## DllExporter.exe

The C# SDK uses `DllExporter.exe` to export functions from managed DLLs.

**How it works:**
1. Build your C# project
2. Post-build event runs DllExporter.exe
3. DllExporter modifies the DLL to export [DllExport] methods
4. Rainmeter can now load the DLL like a C++ plugin

**Post-build event:**
```
"$(SolutionDir)..\API\DllExporter.exe" "$(ConfigurationName)" "$(PlatformName)" "$(TargetDir)\" "$(TargetFileName)"
```

**Don't modify the post-build event unless you know what you're doing!**
