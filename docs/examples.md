# Plugin Examples

Comprehensive guide to all example plugins included in the SDK, with explanations for both C++ and C# implementations.

## Table of Contents

1. [PluginEmpty](#pluginempty) - Blank template
2. [PluginSystemVersion](#pluginsystemversion) - Reading system information
3. [PluginRmExecute](#pluginrmexecute) - Executing Rainmeter commands
4. [PluginDataHandling](#plugindatahandling) - Data persistence
5. [PluginParentChild](#pluginparentchild) - Parent-child measure relationships
6. [PluginSectionVariables](#pluginsectionvariables) - Custom section variables (C++ only)

---

## PluginEmpty

**Location**: `C++/PluginEmpty`, `C#/PluginEmpty`

**Purpose**: Blank canvas template for creating new plugins.

### Description

This is the simplest possible plugin with all core lifecycle functions implemented. It does nothing except demonstrate the basic structure. Use this as a starting point for your own plugins.

### Features

- Basic measure data structure
- All required lifecycle functions
- Optional functions commented out
- Ready to build and extend

### C++ Implementation

```cpp
struct Measure
{
    Measure() {}
};

PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    *data = measure;
}

PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;
}

PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;
    return 0.0;
}

PLUGIN_EXPORT void Finalize(void* data)
{
    Measure* measure = (Measure*)data;
    delete measure;
}
```

### C# Implementation

```csharp
class Measure
{
    // Include your measure data/functions here.
}

public class Plugin
{
    [DllExport]
    public static void Initialize(ref IntPtr data, IntPtr rm)
    {
        data = GCHandle.ToIntPtr(GCHandle.Alloc(new Measure()));
    }

    [DllExport]
    public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
    {
        Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
    }

    [DllExport]
    public static double Update(IntPtr data)
    {
        Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
        return 0.0;
    }

    [DllExport]
    public static void Finalize(IntPtr data)
    {
        GCHandle.FromIntPtr(data).Free();
    }
}
```

### Usage

Perfect starting point for any new plugin. Copy, rename, and extend with your functionality.

---

## PluginSystemVersion

**Location**: `C++/PluginSystemVersion`, `C#/PluginSystemVersion`

**Purpose**: Demonstrates reading system information and returning both numeric and string values.

### Description

This plugin retrieves the operating system version information. It shows how to:
- Handle different measure types via options
- Return numeric values from Update()
- Return string values from GetString()
- Calculate derived values
- Prepare strings in Update() for efficiency

### Features

- Multiple output types: Major, Minor, Number, String
- Demonstrates type-based behavior
- Shows proper GetString() usage
- Error handling with RmLog

### Key Concepts

1. **Type Selection**: Uses "Type" option to determine what to return
2. **Numeric vs String**: Returns appropriate value based on type
3. **String Preparation**: Calculates string in Update(), returns in GetString()
4. **Null Return**: GetString() returns nullptr/null for numeric types

### C++ Implementation Highlights

```cpp
enum MeasureType
{
    MEASURE_MAJOR,
    MEASURE_MINOR,
    MEASURE_NUMBER,
    MEASURE_STRING
};

struct Measure
{
    MeasureType type;
    std::wstring strValue;
};

PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;
    LPCWSTR value = RmReadString(rm, L"Type", L"");

    if (_wcsicmp(value, L"Major") == 0)
        measure->type = MEASURE_MAJOR;
    else if (_wcsicmp(value, L"String") == 0)
        measure->type = MEASURE_STRING;
    else
        RmLog(rm, LOG_ERROR, L"Invalid \"Type\"");
}

PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;
    OSVERSIONINFOEX osvi = {sizeof(OSVERSIONINFOEX)};
    GetVersionEx((OSVERSIONINFO*)&osvi);

    switch (measure->type)
    {
        case MEASURE_MAJOR:
            return (double)osvi.dwMajorVersion;

        case MEASURE_STRING:
            // Prepare string HERE, not in GetString
            WCHAR buffer[128];
            _snwprintf_s(buffer, _TRUNCATE, L"%i.%i (Build %i)",
                osvi.dwMajorVersion, osvi.dwMinorVersion, osvi.dwBuildNumber);
            measure->strValue = buffer;
            break;
    }
    return 0.0;
}

PLUGIN_EXPORT LPCWSTR GetString(void* data)
{
    Measure* measure = (Measure*)data;

    // Only return string for string types
    if (measure->type == MEASURE_STRING)
        return measure->strValue.c_str();

    // Return nullptr for numeric types
    return nullptr;
}
```

### C# Implementation Highlights

```csharp
internal class Measure
{
    enum MeasureType { Major, Minor, Number, String }
    private MeasureType Type = MeasureType.Major;

    internal void Reload(Rainmeter.API api, ref double maxValue)
    {
        string type = api.ReadString("Type", "");
        switch (type.ToLowerInvariant())
        {
            case "major": Type = MeasureType.Major; break;
            case "string": Type = MeasureType.String; break;
            default:
                api.Log(API.LogType.Error, $"Type={type} not valid");
                break;
        }
    }

    internal double Update()
    {
        switch (Type)
        {
            case MeasureType.Major:
                return (double)Environment.OSVersion.Version.Major;
            // ... other cases
        }
        return 0.0;
    }

    internal string GetString()
    {
        switch (Type)
        {
            case MeasureType.String:
                return $"{Environment.OSVersion.Version.Major}." +
                       $"{Environment.OSVersion.Version.Minor} " +
                       $"(Build {Environment.OSVersion.Version.Build})";
        }
        return null;  // Numeric types
    }
}
```

### Sample Skin

```ini
[Rainmeter]
Update=1000

[mString]
Measure=Plugin
Plugin=SystemVersion
Type=String

[mMajor]
Measure=Plugin
Plugin=SystemVersion
Type=Major

[Text]
Meter=String
MeasureName=mString
MeasureName2=mMajor
Text="String: %1#CRLF#Major: %2"
```

### Key Takeaways

- Use enums for type selection
- Validate options and log errors
- Calculate strings in Update(), not GetString()
- Return nullptr/null from GetString() for numeric-only measures

---

## PluginRmExecute

**Location**: `C++/PluginRmExecute`, `C#/PluginRmExecute`

**Purpose**: Demonstrates executing Rainmeter commands (bangs) from a plugin.

### Description

This plugin implements a timer that executes a command after a specified time. It shows how to:
- Store the skin pointer for executing commands
- Read actions without replacing section variables
- Use RmExecute to run bangs
- Implement timer logic
- Work with time calculations

### Features

- Configurable timer
- Execute commands at intervals
- Store skin pointer in Initialize
- Read actions with replaceMeasures=FALSE

### Key Concepts

1. **Skin Pointer**: Store `RmGetSkin()` in Initialize for later use
2. **Action Reading**: Use `replaceMeasures=FALSE` when reading bangs
3. **Timer Logic**: Track time and execute when threshold reached
4. **Dynamic Variables Alternative**: Shows how to avoid DV=1 performance hit

### C++ Implementation Highlights

```cpp
struct Measure
{
    std::wstring command;
    double updateRate;
    time_t timer;
    void* skin;  // Important!

    Measure() : updateRate(0), timer(std::time(nullptr)), skin(nullptr) {}
};

PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    *data = measure;

    // Store skin pointer for RmExecute
    measure->skin = RmGetSkin(rm);
}

PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;
    measure->updateRate = RmReadDouble(rm, L"Timer", 1.0);

    // Read action WITHOUT replacing variables (FALSE parameter)
    // This allows variables to be evaluated when executed
    measure->command = RmReadString(rm, L"OnTimer", L"", FALSE);
}

PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;

    double difference = std::difftime(time(nullptr), measure->timer);
    if (difference >= measure->updateRate)
    {
        // Execute the command
        RmExecute(measure->skin, measure->command.c_str());
        measure->timer = std::time(nullptr);
    }

    return measure->updateRate - difference;
}
```

### C# Implementation Highlights

```csharp
class Measure
{
    public string myCommand = "";
    public int updateRate;
    public DateTime timer;
    public Rainmeter.API api;
}

[DllExport]
public static void Initialize(ref IntPtr data, IntPtr rm)
{
    Measure measure = new Measure();
    Rainmeter.API api = new Rainmeter.API(rm);

    measure.api = api;
    measure.timer = DateTime.UtcNow;

    data = GCHandle.ToIntPtr(GCHandle.Alloc(measure));
}

[DllExport]
public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
    Rainmeter.API api = new Rainmeter.API(rm);

    measure.updateRate = api.ReadInt("Timer", 1);

    // Read without replacement (false parameter)
    measure.myCommand = api.ReadString("OnTimer", "", false);
}

[DllExport]
public static double Update(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    int timePassed = (int)(DateTime.UtcNow - measure.timer).TotalSeconds;

    if (timePassed >= measure.updateRate)
    {
        // Execute the command
        measure.api.Execute(measure.myCommand);
        measure.timer = DateTime.UtcNow;
    }

    return measure.updateRate - timePassed;
}
```

### Sample Skin

```ini
[Rainmeter]
Update=1000

[Variables]
Count=0
Timer=5

[mTimer]
Measure=Plugin
Plugin=RmExecute
Timer=#Timer#
OnTimer=[!SetVariable Count "(#Count# + 1)"]
DynamicVariables=1

[Text]
Meter=String
Text=Updated #Count# times
DynamicVariables=1
```

### Key Takeaways

- Always store skin pointer in Initialize
- Read actions with `replaceMeasures=FALSE`
- This allows variables to update dynamically
- Use time tracking for periodic actions
- Return meaningful values (time remaining)

---

## PluginDataHandling

**Location**: `C++/PluginDataHandling`, `C#/PluginDataHandling`

**Purpose**: Demonstrates data persistence across skin reloads using the Rainmeter.data file.

### Description

This plugin implements a counter that persists across skin reloads and even Rainmeter restarts. It shows how to:
- Store data in Rainmeter.data file
- Read from Rainmeter.data on load
- Use GetPrivateProfileInt/WritePrivateProfileString
- Handle optional vs persistent data
- Save data in Finalize

### Features

- Counter that persists across reloads
- Optional starting value
- Conditional save (SaveData/StoreData option)
- Uses Windows INI file API

### Key Concepts

1. **Rainmeter.data**: Central storage file for persistent plugin data
2. **GetSettingsFile()**: Returns path to Rainmeter.data
3. **Private Profile API**: Windows functions for INI file access
4. **Save in Finalize**: Write data when plugin unloads

### C++ Implementation Highlights

```cpp
struct Measure
{
    int counter;
    bool saveData;
    LPCWSTR dataFile;

    Measure() : counter(0), saveData(false), dataFile(nullptr) {}
};

PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    *data = measure;

    // Get path to Rainmeter.data
    measure->dataFile = RmGetSettingsFile();
}

PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;

    // Check for explicit starting value
    measure->counter = RmReadInt(rm, L"StartingValue", -1);

    // If not set, read from data file
    if (measure->counter < 0)
    {
        measure->counter = (int)GetPrivateProfileInt(
            L"Plugin_DataHandling",  // Section
            L"Count",                 // Key
            0,                        // Default
            measure->dataFile         // File path
        );
    }

    // Check if we should save on exit
    measure->saveData = RmReadInt(rm, L"SaveData", 0) == 1;
}

PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;

    if (++measure->counter < 0)
        measure->counter = 0;

    return measure->counter;
}

PLUGIN_EXPORT void Finalize(void* data)
{
    Measure* measure = (Measure*)data;

    if (measure->saveData)
    {
        // Save to data file
        WCHAR buffer[SHRT_MAX];
        _itow_s(measure->counter, buffer, 10);
        WritePrivateProfileString(
            L"Plugin_DataHandling",
            L"Count",
            buffer,
            measure->dataFile
        );
    }

    delete measure;
}
```

### C# Implementation Highlights

```csharp
// P/Invoke declarations
[DllImport("kernel32.dll", CharSet = CharSet.Unicode)]
static extern int GetPrivateProfileString(string section, string key,
    string defaultValue, [In, Out] char[] value, int size, string filePath);

[DllImport("kernel32", CharSet = CharSet.Unicode)]
[return: MarshalAs(UnmanagedType.Bool)]
private static extern bool WritePrivateProfileString(string section,
    string key, string value, string filePath);

class Measure
{
    public int myCounter = 0;
    public bool storeData = false;
}

[DllExport]
public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
    Rainmeter.API api = new Rainmeter.API(rm);

    int startValue = api.ReadInt("StartingValue", -1);

    if (startValue == -1)
    {
        char[] buffer = new char[256];
        GetPrivateProfileString("DataHandling", "StoredCount", "0",
            buffer, 256, API.GetSettingsFile());

        try
        {
            measure.myCounter = Convert.ToInt32(new string(buffer));
        }
        catch
        {
            api.Log(API.LogType.Error, "Error converting stored value");
        }
    }
    else
    {
        measure.myCounter = startValue;
    }

    measure.storeData = api.ReadInt("StoreData", 0) == 1;
}

[DllExport]
public static void Finalize(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    if (measure.storeData)
    {
        WritePrivateProfileString("DataHandling", "StoredCount",
            measure.myCounter.ToString(), API.GetSettingsFile());
    }

    GCHandle.FromIntPtr(data).Free();
}
```

### Sample Skin

```ini
[Rainmeter]
Update=1000

[mCount]
Measure=Plugin
Plugin=DataHandling
SaveData=1

[mCountAt0]
Measure=Plugin
Plugin=DataHandling
StartingValue=0

[MeterCount]
Meter=String
MeasureName=mCount
Text=Persistent: %1

[MeterCountAt0]
Meter=String
MeasureName=mCountAt0
Y=5R
Text=Reset: %1
```

### Key Takeaways

- Use Rainmeter.data for persistent storage
- Get file path once in Initialize
- Use Private Profile API for reading/writing
- Save in Finalize, not Update
- Provide options for both persistent and non-persistent modes
- Handle conversion errors (especially in C#)

---

## PluginParentChild

**Location**: `C++/PluginParentChild`, `C#/PluginParentChild`

**Purpose**: Demonstrates parent-child measure relationships for sharing data between measures.

### Description

This plugin shows how to implement a parent measure that holds data, with child measures that access that data. This pattern is useful when:
- One measure queries expensive data
- Multiple measures need different aspects of that data
- You want to avoid redundant queries

### Features

- Parent measure stores shared data
- Child measures reference parent by name
- Global registry of parent measures
- Type-based value selection

### Key Concepts

1. **Parent Measure**: Holds shared data, identified by measure name
2. **Child Measures**: Reference parent via "ParentName" option
3. **Global Registry**: Static list tracks all parent measures
4. **Skin Verification**: Match both name AND skin pointer

### C++ Implementation Highlights

```cpp
struct ParentMeasure
{
    void* skin;
    LPCWSTR name;
    ChildMeasure* ownerChild;
    int valueA, valueB, valueC;
};

struct ChildMeasure
{
    MeasureType type;
    ParentMeasure* parent;
};

// Global registry
std::vector<ParentMeasure*> g_ParentMeasures;

PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    ChildMeasure* child = new ChildMeasure;
    *data = child;

    void* skin = RmGetSkin(rm);
    LPCWSTR parentName = RmReadString(rm, L"ParentName", L"");

    if (!*parentName)
    {
        // This is a parent measure
        child->parent = new ParentMeasure;
        child->parent->name = RmGetMeasureName(rm);
        child->parent->skin = skin;
        child->parent->ownerChild = child;
        g_ParentMeasures.push_back(child->parent);
    }
    else
    {
        // This is a child - find parent
        for (auto iter = g_ParentMeasures.begin();
             iter != g_ParentMeasures.end(); ++iter)
        {
            // Match BOTH name AND skin!
            if (_wcsicmp((*iter)->name, parentName) == 0 &&
                (*iter)->skin == skin)
            {
                child->parent = (*iter);
                return;
            }
        }
        RmLog(rm, LOG_ERROR, L"Invalid ParentName");
    }
}

PLUGIN_EXPORT void Finalize(void* data)
{
    ChildMeasure* child = (ChildMeasure*)data;
    ParentMeasure* parent = child->parent;

    // Only delete parent if this is the owner
    if (parent && parent->ownerChild == child)
    {
        g_ParentMeasures.erase(
            std::remove(g_ParentMeasures.begin(),
                        g_ParentMeasures.end(), parent),
            g_ParentMeasures.end());
        delete parent;
    }

    delete child;
}
```

### C# Implementation Highlights

```csharp
internal class ParentMeasure : Measure
{
    internal static List<ParentMeasure> ParentMeasures =
        new List<ParentMeasure>();

    internal string Name;
    internal IntPtr Skin;
    internal int ValueA, ValueB, ValueC;

    internal ParentMeasure()
    {
        ParentMeasures.Add(this);
    }

    internal override void Dispose()
    {
        ParentMeasures.Remove(this);
    }
}

internal class ChildMeasure : Measure
{
    private ParentMeasure ParentMeasure = null;

    internal override void Reload(Rainmeter.API api, ref double maxValue)
    {
        base.Reload(api, ref maxValue);

        string parentName = api.ReadString("ParentName", "");
        IntPtr skin = api.GetSkin();

        // Find parent
        ParentMeasure = null;
        foreach (ParentMeasure pm in ParentMeasure.ParentMeasures)
        {
            if (pm.Skin.Equals(skin) && pm.Name.Equals(parentName))
            {
                ParentMeasure = pm;
            }
        }

        if (ParentMeasure == null)
        {
            api.Log(API.LogType.Error, $"ParentName={parentName} not valid");
        }
    }
}

[DllExport]
public static void Initialize(ref IntPtr data, IntPtr rm)
{
    Rainmeter.API api = new Rainmeter.API(rm);
    string parent = api.ReadString("ParentName", "");

    Measure measure;
    if (String.IsNullOrEmpty(parent))
        measure = new ParentMeasure();
    else
        measure = new ChildMeasure();

    data = GCHandle.ToIntPtr(GCHandle.Alloc(measure));
}
```

### Sample Skin

```ini
[Rainmeter]
Update=1000

[mParent]
Measure=Plugin
Plugin=ParentChild
ValueA=111
ValueB=222
ValueC=333
Type=A

[mChild1]
Measure=Plugin
Plugin=ParentChild
ParentName=mParent
Type=B

[mChild2]
Measure=Plugin
Plugin=ParentChild
ParentName=mParent
Type=C

[Text]
Meter=String
MeasureName=mParent
MeasureName2=mChild1
MeasureName3=mChild2
Text="Parent: %1#CRLF#Child1: %2#CRLF#Child2: %3"
```

### Key Takeaways

- Use static/global list to track parents
- Match both name AND skin pointer (important for multiple skins)
- Parent owns the data, children just access it
- Owner child is responsible for parent cleanup
- Useful for expensive operations done once, accessed many times

---

## PluginSectionVariables

**Location**: `C++/PluginSectionVariables`, `C#/PluginSectionVariables`

**Purpose**: Demonstrates custom section variables (inline function calls).

### Description

This plugin shows how to create custom functions callable from skins using section variable syntax: `[MeasureName:FunctionName(args)]`

**Note**: C++ only! C# can use ExecuteBang as an alternative.

### Features (C++)

- Custom ToUpper/ToLower functions
- Accept optional arguments
- Return transformed strings
- Use measure's stored value if no args

### Key Concepts

1. **Section Variable Syntax**: `[Measure:Function(arg1, arg2)]`
2. **Function Signature**: `LPCWSTR FuncName(void* data, const int argc, const WCHAR* argv[])`
3. **Arguments**: Passed as array, argc is count
4. **DynamicVariables**: Required in meters using section variables

### C++ Implementation

```cpp
struct Measure
{
    std::wstring inputStr;
    std::wstring buffer;  // For return values
};

PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;
    measure->inputStr = RmReadString(rm, L"Input", L"");
}

PLUGIN_EXPORT LPCWSTR GetString(void* data)
{
    Measure* measure = (Measure*)data;
    return measure->inputStr.c_str();
}

// Custom section variable function
PLUGIN_EXPORT LPCWSTR ToUpper(void* data, const int argc, const WCHAR* argv[])
{
    Measure* measure = (Measure*)data;

    // Use argument if provided, otherwise use Input option
    if (argc > 0)
        measure->buffer = argv[0];  // First argument only
    else
        measure->buffer = measure->inputStr;

    // Transform to uppercase
    std::transform(measure->buffer.begin(), measure->buffer.end(),
                   measure->buffer.begin(), std::towupper);

    return measure->buffer.c_str();
}

PLUGIN_EXPORT LPCWSTR ToLower(void* data, const int argc, const WCHAR* argv[])
{
    Measure* measure = (Measure*)data;

    if (argc > 0)
        measure->buffer = argv[0];
    else
        measure->buffer = measure->inputStr;

    std::transform(measure->buffer.begin(), measure->buffer.end(),
                   measure->buffer.begin(), std::towlower);

    return measure->buffer.c_str();
}
```

### C# Alternative (ExecuteBang)

```csharp
[DllExport]
public static void ExecuteBang(IntPtr data,
    [MarshalAs(UnmanagedType.LPWStr)] String args)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
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

### Sample Skin (C++)

```ini
[Rainmeter]
Update=1000

[Variables]
TextString=click me to make me uppercase!

[mString]
Measure=Plugin
Plugin=SectionVariables
Input=I AM A SECTION VARIABLE!

[TextLower]
Meter=String
Text=[mString:ToLower()]
DynamicVariables=1

[TextUpper]
Meter=String
Y=5R
Text=#TextString#
LeftMouseUpAction=[!SetOption TextUpper Text "[mString:ToUpper(#TextString#)]"]
```

### Sample Skin (C# with ExecuteBang)

```ini
[TextUpper]
Meter=String
Text=#TextString#
LeftMouseUpAction=[!CommandMeasure mString "ToUpper #TextString#"]
```

### Key Takeaways

- Section variables are C++ only
- Must use DynamicVariables=1 in meters
- Store return values in measure buffer
- argc tells you how many arguments
- C# can achieve similar with ExecuteBang
- Parse arguments carefully

---

## Summary Table

| Example | Key Learning | Difficulty |
|---------|-------------|------------|
| PluginEmpty | Basic structure | Beginner |
| PluginSystemVersion | Type handling, GetString | Beginner |
| PluginRmExecute | Executing commands, timers | Intermediate |
| PluginDataHandling | Data persistence | Intermediate |
| PluginParentChild | Measure relationships | Advanced |
| PluginSectionVariables | Custom functions (C++) | Advanced |

---

## Next Steps

- Try modifying each example to understand the code
- Combine concepts from multiple examples
- Build your own plugin using learned patterns
- Review [API Reference](api-reference-cpp.md) for more functions
- Check [Advanced Topics](advanced-topics.md) for complex scenarios
