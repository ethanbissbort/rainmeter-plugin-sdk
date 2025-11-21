# Plugin Lifecycle

Understanding the plugin lifecycle is crucial for developing Rainmeter plugins. This document explains when and how each function is called.

## Overview

A Rainmeter plugin follows a specific lifecycle from creation to destruction:

```
Initialize → Reload → Update (repeated) → Finalize
```

Optional functions can be called at any time:
- `GetString` - Called when Rainmeter needs the string value
- `ExecuteBang` - Called when a bang is sent to the plugin
- Custom section variables (C++ only) - Called when referenced

## Core Lifecycle Functions

### Initialize

**Purpose**: Allocate memory and set up the measure instance.

**When Called**:
- When the skin is loaded or refreshed
- Once per measure instance
- Before any other function

**What to Do**:
- Allocate your measure data structure
- Store the `rm` pointer if needed for later API calls
- Call `RmGetSkin()` if you need to execute commands
- Call `RmGetMeasureName()`, `RmGetSkinName()`, etc. and store if needed

**What NOT to Do**:
- Don't read options here (use Reload for that)
- Don't perform expensive operations
- Don't return values (this is a void function)

#### C++ Signature
```cpp
PLUGIN_EXPORT void Initialize(void** data, void* rm)
```

#### C# Signature
```csharp
[DllExport]
public static void Initialize(ref IntPtr data, IntPtr rm)
```

#### Example (C++)
```cpp
PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    *data = measure;

    // Store the skin pointer for later use with RmExecute
    measure->skin = RmGetSkin(rm);

    // Store the measure name
    measure->name = RmGetMeasureName(rm);
}
```

#### Example (C#)
```csharp
[DllExport]
public static void Initialize(ref IntPtr data, IntPtr rm)
{
    Measure measure = new Measure();
    Rainmeter.API api = new Rainmeter.API(rm);

    // Store API reference for later use
    measure.api = api;

    // Store skin pointer for Execute commands
    measure.skin = api.GetSkin();

    data = GCHandle.ToIntPtr(GCHandle.Alloc(measure));
}
```

---

### Reload

**Purpose**: Read and parse options from the skin file.

**When Called**:
- After Initialize
- When the skin is refreshed
- When `DynamicVariables=1` is set, called on every update cycle

**What to Do**:
- Read all measure options using `RmReadString()`, `RmReadInt()`, etc.
- Parse and validate options
- Set the `maxValue` if your measure has a known maximum
- Log errors for invalid options

**What NOT to Do**:
- Don't perform the main plugin logic here
- Don't assume this is only called once
- Be careful with `DynamicVariables=1` - this will be called frequently

#### C++ Signature
```cpp
PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
```

#### C# Signature
```csharp
[DllExport]
public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
```

#### Example (C++)
```cpp
PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;

    // Read string option with default value
    LPCWSTR type = RmReadString(rm, L"Type", L"Default");

    // Read integer option
    int timeout = RmReadInt(rm, L"Timeout", 30);

    // Read boolean (as integer 0 or 1)
    bool enabled = RmReadInt(rm, L"Enabled", 1) == 1;

    // Validate and log errors
    if (_wcsicmp(type, L"Valid") != 0)
    {
        RmLog(rm, LOG_ERROR, L"Invalid Type specified");
    }

    // Set max value if known (for percentage calculations)
    *maxValue = 100.0;
}
```

#### Example (C#)
```csharp
[DllExport]
public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
    Rainmeter.API api = new Rainmeter.API(rm);

    // Read options
    string type = api.ReadString("Type", "Default");
    int timeout = api.ReadInt("Timeout", 30);
    bool enabled = api.ReadInt("Enabled", 1) == 1;

    // Validate
    if (type != "Valid")
    {
        api.Log(API.LogType.Error, "Invalid Type specified");
    }

    // Set max value
    maxValue = 100.0;
}
```

---

### Update

**Purpose**: Perform the main plugin logic and return a numeric value.

**When Called**:
- On every update cycle (based on the skin's Update interval)
- Not called if the measure is disabled or paused

**What to Do**:
- Perform your plugin's main work
- Calculate and return a numeric value
- Update any internal state
- Store string values in your measure structure (don't calculate them in GetString)

**What NOT to Do**:
- Don't read options here (use Reload)
- Don't return strings (use GetString for that)
- Keep the function as fast as possible

#### C++ Signature
```cpp
PLUGIN_EXPORT double Update(void* data)
```

#### C# Signature
```csharp
[DllExport]
public static double Update(IntPtr data)
```

#### Example (C++)
```cpp
PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;

    // Perform calculations
    int value = measure->counter++;

    // If returning a string value, prepare it here
    if (measure->returnString)
    {
        WCHAR buffer[128];
        _snwprintf_s(buffer, _TRUNCATE, L"Count: %d", value);
        measure->stringValue = buffer;
    }

    return (double)value;
}
```

#### Example (C#)
```csharp
[DllExport]
public static double Update(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    // Perform main logic
    int value = measure.counter++;

    // Prepare string if needed
    if (measure.returnString)
    {
        measure.stringValue = $"Count: {value}";
    }

    return (double)value;
}
```

---

### Finalize

**Purpose**: Clean up resources and free memory.

**When Called**:
- When the skin is unloaded or refreshed
- Once per measure instance
- Last function called for the measure

**What to Do**:
- Free any allocated memory
- Close file handles, network connections, etc.
- Save persistent data if needed
- Delete your measure structure

**What NOT to Do**:
- Don't access Rainmeter API functions (rm pointer may be invalid)
- Don't try to execute commands

#### C++ Signature
```cpp
PLUGIN_EXPORT void Finalize(void* data)
```

#### C# Signature
```csharp
[DllExport]
public static void Finalize(IntPtr data)
```

#### Example (C++)
```cpp
PLUGIN_EXPORT void Finalize(void* data)
{
    Measure* measure = (Measure*)data;

    // Clean up resources
    if (measure->fileHandle)
    {
        CloseHandle(measure->fileHandle);
    }

    // Free memory
    delete measure;
}
```

#### Example (C#)
```csharp
[DllExport]
public static void Finalize(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    // Clean up resources
    measure.Dispose();

    // Free GC handle
    GCHandle.FromIntPtr(data).Free();
}
```

---

## Optional Functions

### GetString

**Purpose**: Return a string representation of the measure.

**When Called**:
- When Rainmeter needs the string value
- Can be called multiple times per update cycle
- Not called if the measure returns `nullptr` (C++) or `null` (C#)

**Important**: Do NOT perform calculations here! Calculate in `Update()` and store the result, then return it here.

#### C++ Signature
```cpp
PLUGIN_EXPORT LPCWSTR GetString(void* data)
```

#### C# Signature
```csharp
[DllExport]
public static IntPtr GetString(IntPtr data)
```

#### Example (C++)
```cpp
PLUGIN_EXPORT LPCWSTR GetString(void* data)
{
    Measure* measure = (Measure*)data;

    // Return nullptr to indicate this is a number-only measure
    if (measure->type == TYPE_NUMBER)
        return nullptr;

    // Return the string calculated in Update()
    return measure->stringValue.c_str();
}
```

#### Example (C#)
```csharp
[DllExport]
public static IntPtr GetString(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    // Return null to indicate this is a number-only measure
    if (measure.type == MeasureType.Number)
        return Rainmeter.StringBuffer.Update(null);

    // Return the string calculated in Update()
    return Rainmeter.StringBuffer.Update(measure.stringValue);
}
```

---

### ExecuteBang

**Purpose**: Handle custom commands sent from the skin.

**When Called**:
- When a `!CommandMeasure` bang is executed
- Format: `[!CommandMeasure MeasureName "Arguments"]`

#### C++ Signature
```cpp
PLUGIN_EXPORT void ExecuteBang(void* data, LPCWSTR args)
```

#### C# Signature
```csharp
[DllExport]
public static void ExecuteBang(IntPtr data,
    [MarshalAs(UnmanagedType.LPWStr)] String args)
```

#### Example (C++)
```cpp
PLUGIN_EXPORT void ExecuteBang(void* data, LPCWSTR args)
{
    Measure* measure = (Measure*)data;

    if (_wcsicmp(args, L"Reset") == 0)
    {
        measure->counter = 0;
    }
    else if (_wcsnicmp(args, L"SetValue ", 9) == 0)
    {
        measure->counter = _wtoi(args + 9);
    }
}
```

---

### Custom Section Variables (C++ Only)

**Purpose**: Provide custom functions callable from skins using section variables.

**Format**: `[MeasureName:FunctionName(arg1, arg2)]`

#### C++ Signature
```cpp
PLUGIN_EXPORT LPCWSTR FunctionName(void* data, const int argc, const WCHAR* argv[])
```

#### Example
```cpp
PLUGIN_EXPORT LPCWSTR ToUpper(void* data, const int argc, const WCHAR* argv[])
{
    Measure* measure = (Measure*)data;

    // Use first argument if provided, otherwise use stored value
    std::wstring text = (argc > 0) ? argv[0] : measure->inputStr;

    // Transform to uppercase
    std::transform(text.begin(), text.end(), text.begin(), std::towupper);

    measure->buffer = text;
    return measure->buffer.c_str();
}
```

---

## Lifecycle Diagram

```
┌─────────────┐
│   Skin      │
│   Loads     │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Initialize  │ ◄─── Allocate memory, store rm/skin pointers
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Reload    │ ◄─── Read options from skin
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Update    │ ◄─── Main plugin logic (returns number)
└──────┬──────┘
       │
       ├──────► GetString (if string value requested)
       │
       ├──────► ExecuteBang (if command sent)
       │
       │  ┌────────────────────┐
       └──┤ Update continues   │
          │ on each cycle      │
          └────────┬───────────┘
                   │
         ┌─────────▼────────┐
         │  Skin Unloads    │
         │  or Refreshes    │
         └─────────┬────────┘
                   │
                   ▼
           ┌───────────────┐
           │   Finalize    │ ◄─── Clean up, free memory
           └───────────────┘
```

## Best Practices

1. **Initialize**: Keep it light - just allocate memory and store pointers
2. **Reload**: Validate all options and provide good error messages
3. **Update**: Do the main work here, keep it fast
4. **GetString**: Just return pre-calculated strings, don't compute here
5. **Finalize**: Always clean up to prevent memory leaks

## Common Pitfalls

1. **Reading options in Update**: Options should be read in Reload
2. **Calculating in GetString**: Calculate in Update, return in GetString
3. **Not handling DynamicVariables=1**: Reload will be called every update
4. **Forgetting to free memory**: Always clean up in Finalize
5. **Storing rm pointer without checking version**: Some functions require newer Rainmeter versions
