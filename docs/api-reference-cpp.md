# C++ API Reference

Complete reference for the Rainmeter C++ Plugin API (`RainmeterAPI.h`).

## Table of Contents

1. [Reading Options](#reading-options)
2. [String Operations](#string-operations)
3. [Executing Commands](#executing-commands)
4. [Getting Information](#getting-information)
5. [Logging](#logging)
6. [Macros and Types](#macros-and-types)

---

## Reading Options

### RmReadString

Retrieves a string option from the plugin measure.

```cpp
LPCWSTR RmReadString(void* rm, LPCWSTR option, LPCWSTR defValue,
                     BOOL replaceMeasures = TRUE);
```

**Parameters:**
- `rm` - Pointer to the plugin measure
- `option` - Name of the option to read
- `defValue` - Default value if option not found
- `replaceMeasures` - If TRUE, replaces section variables in the string

**Returns:** The option value as a wide character string (LPCWSTR)

**Example:**
```cpp
PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;

    // Read with section variable replacement (default)
    LPCWSTR path = RmReadString(rm, L"FilePath", L"C:\\Default\\");

    // Read without replacement (for actions/bangs)
    LPCWSTR command = RmReadString(rm, L"OnUpdate", L"", FALSE);
}
```

---

### RmReadStringFromSection

Retrieves a string option from another measure or meter.

```cpp
LPCWSTR RmReadStringFromSection(void* rm, LPCWSTR section, LPCWSTR option,
                                LPCWSTR defValue, BOOL replaceMeasures = TRUE);
```

**Parameters:**
- `rm` - Pointer to the plugin measure
- `section` - Name of the measure/meter section
- `option` - Name of the option to read
- `defValue` - Default value if not found
- `replaceMeasures` - If TRUE, replaces section variables

**Returns:** The option value as LPCWSTR

**Note:** Returns default value in older Rainmeter versions without support.

**Example:**
```cpp
LPCWSTR value = RmReadStringFromSection(rm, L"MyOtherMeasure", L"Text", L"");
```

---

### RmReadFormula

Retrieves an option and parses it as a formula/number.

```cpp
double RmReadFormula(void* rm, LPCWSTR option, double defValue);
```

**Parameters:**
- `rm` - Pointer to the plugin measure
- `option` - Name of the option
- `defValue` - Default value if not found or parse fails

**Returns:** The parsed numeric value

**Example:**
```cpp
// Can read "42", "1.5", or formulas like "(2 * 10)"
double timeout = RmReadFormula(rm, L"Timeout", 30.0);
```

---

### RmReadFormulaFromSection

Retrieves and parses a formula from another section.

```cpp
double RmReadFormulaFromSection(void* rm, LPCWSTR section, LPCWSTR option,
                                double defValue);
```

**Note:** Returns default value in older Rainmeter versions.

---

### RmReadInt

Retrieves an option as an integer (wrapper around RmReadFormula).

```cpp
int RmReadInt(void* rm, LPCWSTR option, int defValue);
```

**Example:**
```cpp
int maxRetries = RmReadInt(rm, L"MaxRetries", 3);
```

---

### RmReadIntFromSection

Retrieves an integer option from another section.

```cpp
int RmReadIntFromSection(void* rm, LPCWSTR section, LPCWSTR option, int defValue);
```

---

### RmReadDouble

Retrieves an option as a double (wrapper around RmReadFormula).

```cpp
double RmReadDouble(void* rm, LPCWSTR option, double defValue);
```

**Example:**
```cpp
double threshold = RmReadDouble(rm, L"Threshold", 0.75);
```

---

### RmReadDoubleFromSection

Retrieves a double option from another section.

```cpp
double RmReadDoubleFromSection(void* rm, LPCWSTR section, LPCWSTR option,
                               double defValue);
```

---

### RmReadPath

Retrieves an option and converts relative path to absolute.

```cpp
LPCWSTR RmReadPath(void* rm, LPCWSTR option, LPCWSTR defValue);
```

**Example:**
```cpp
// Converts "..\Images\bg.png" to absolute path
LPCWSTR imagePath = RmReadPath(rm, L"ImagePath", L"");
```

---

## String Operations

### RmReplaceVariables

Replaces variables and section variables in a string.

```cpp
LPCWSTR RmReplaceVariables(void* rm, LPCWSTR str);
```

**Parameters:**
- `rm` - Pointer to the plugin measure
- `str` - String with unresolved variables (e.g., `#Variable#` or `[MeasureName]`)

**Returns:** String with variables replaced

**Example:**
```cpp
LPCWSTR resolved = RmReplaceVariables(rm, L"#MyVar# - [MeasureName]");
```

---

### RmPathToAbsolute

Converts a relative path to an absolute path.

```cpp
LPCWSTR RmPathToAbsolute(void* rm, LPCWSTR relativePath);
```

**Example:**
```cpp
LPCWSTR absPath = RmPathToAbsolute(rm, L"..\\Config\\settings.txt");
```

---

## Executing Commands

### RmExecute

Executes a Rainmeter bang/command.

```cpp
void RmExecute(void* skin, LPCWSTR command);
```

**Parameters:**
- `skin` - Pointer to the skin (obtained from `RmGetSkin()`)
- `command` - Bang to execute

**Example:**
```cpp
// Store skin pointer in Initialize:
measure->skin = RmGetSkin(rm);

// Use in Update or elsewhere:
RmExecute(measure->skin, L"!SetVariable Count 10");
RmExecute(measure->skin, L"!UpdateMeter MyMeter");
RmExecute(measure->skin, L"!Redraw");
```

**Common Bangs:**
- `!SetVariable VarName Value` - Set a variable
- `!SetOption MeterName OptionName Value` - Set an option
- `!UpdateMeasure MeasureName` - Update a measure
- `!UpdateMeter MeterName` - Update a meter
- `!Redraw` - Redraw the skin

---

## Getting Information

### RmGet

Low-level function to retrieve information (use helper functions instead).

```cpp
void* RmGet(void* rm, int type);
```

**Types:**
- `RMG_MEASURENAME` (0) - Measure name
- `RMG_SKIN` (1) - Skin pointer
- `RMG_SETTINGSFILE` (2) - Path to Rainmeter.data
- `RMG_SKINNAME` (3) - Skin name
- `RMG_SKINWINDOWHANDLE` (4) - Skin window handle

---

### RmGetMeasureName

Gets the name of the current measure.

```cpp
LPCWSTR RmGetMeasureName(void* rm);
```

**Example:**
```cpp
PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    measure->name = RmGetMeasureName(rm);  // Store for later use
    *data = measure;
}
```

---

### RmGetSkin

Gets a pointer to the current skin (needed for RmExecute).

```cpp
void* RmGetSkin(void* rm);
```

**Example:**
```cpp
PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    measure->skin = RmGetSkin(rm);  // Store for RmExecute
    *data = measure;
}
```

---

### RmGetSettingsFile

Gets the path to the Rainmeter.data file.

```cpp
LPCWSTR RmGetSettingsFile();
```

**Example:**
```cpp
LPCWSTR dataFile = RmGetSettingsFile();
// Use with GetPrivateProfileInt, WritePrivateProfileString, etc.
```

---

### RmGetSkinName

Gets the full path and name of the skin.

```cpp
LPCWSTR RmGetSkinName(void* rm);
```

**Example:**
```cpp
LPCWSTR skinName = RmGetSkinName(rm);
// Returns something like: "MySkin\\SubFolder"
```

---

### RmGetSkinWindow

Gets the handle to the skin window.

```cpp
HWND RmGetSkinWindow(void* rm);
```

**Example:**
```cpp
PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    measure->hwnd = RmGetSkinWindow(rm);
    *data = measure;
}
```

---

## Logging

### RmLog

Sends a message to the Rainmeter log with source.

```cpp
void RmLog(void* rm, int level, LPCWSTR message);
```

**Log Levels:**
- `LOG_ERROR` (1) - Error messages
- `LOG_WARNING` (2) - Warning messages
- `LOG_NOTICE` (3) - Notice messages
- `LOG_DEBUG` (4) - Debug messages (only shown when Rainmeter is in debug mode)

**Example:**
```cpp
RmLog(rm, LOG_ERROR, L"Failed to open file");
RmLog(rm, LOG_WARNING, L"Using default value");
RmLog(rm, LOG_NOTICE, L"Plugin initialized successfully");
RmLog(rm, LOG_DEBUG, L"Variable x = 42");
```

---

### RmLogF

Sends a formatted message to the Rainmeter log.

```cpp
void RmLogF(void* rm, int level, LPCWSTR format, ...);
```

**Example:**
```cpp
int count = 42;
double value = 3.14;
RmLogF(rm, LOG_NOTICE, L"Count: %d, Value: %.2f", count, value);

std::wstring name = L"MyPlugin";
RmLogF(rm, LOG_DEBUG, L"Plugin '%s' loaded", name.c_str());
```

---

### LSLog (Deprecated)

Old logging function, use `RmLog` instead.

```cpp
BOOL LSLog(int level, LPCWSTR unused, LPCWSTR message);
```

---

## Macros and Types

### PLUGIN_EXPORT

Macro for exporting plugin functions.

```cpp
#define PLUGIN_EXPORT EXTERN_C __declspec(dllexport)
```

**Usage:**
```cpp
PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    // ...
}
```

---

### LIBRARY_EXPORT

Used internally by the API (for functions imported from Rainmeter.dll).

```cpp
#define LIBRARY_EXPORT EXTERN_C __declspec(dllimport)
```

---

### LOGLEVEL Enum

```cpp
enum LOGLEVEL
{
    LOG_ERROR   = 1,
    LOG_WARNING = 2,
    LOG_NOTICE  = 3,
    LOG_DEBUG   = 4
};
```

---

### RmGetType Enum

```cpp
enum RmGetType
{
    RMG_MEASURENAME      = 0,
    RMG_SKIN             = 1,
    RMG_SETTINGSFILE     = 2,
    RMG_SKINNAME         = 3,
    RMG_SKINWINDOWHANDLE = 4
};
```

---

## Complete Example

```cpp
#include <Windows.h>
#include <string>
#include "../../API/RainmeterAPI.h"

struct Measure
{
    void* skin;
    std::wstring name;
    int counter;
    std::wstring output;

    Measure() : skin(nullptr), counter(0) {}
};

PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    *data = measure;

    // Get and store info for later use
    measure->skin = RmGetSkin(rm);
    measure->name = RmGetMeasureName(rm);

    RmLog(rm, LOG_NOTICE, L"Plugin initialized");
}

PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;

    // Read options
    int startValue = RmReadInt(rm, L"StartValue", 0);
    measure->counter = startValue;

    // Log with formatting
    RmLogF(rm, LOG_DEBUG, L"StartValue set to %d", startValue);

    *maxValue = 100.0;
}

PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;

    // Increment counter
    measure->counter++;

    // Prepare string for GetString
    WCHAR buffer[128];
    _snwprintf_s(buffer, _TRUNCATE, L"Count: %d", measure->counter);
    measure->output = buffer;

    // Execute a bang at certain values
    if (measure->counter == 50)
    {
        RmExecute(measure->skin, L"!SetVariable HalfwayReached 1");
    }

    return (double)measure->counter;
}

PLUGIN_EXPORT LPCWSTR GetString(void* data)
{
    Measure* measure = (Measure*)data;
    return measure->output.c_str();
}

PLUGIN_EXPORT void ExecuteBang(void* data, LPCWSTR args)
{
    Measure* measure = (Measure*)data;

    if (_wcsicmp(args, L"Reset") == 0)
    {
        measure->counter = 0;
    }
}

PLUGIN_EXPORT void Finalize(void* data)
{
    Measure* measure = (Measure*)data;
    delete measure;
}
```

---

## Best Practices

1. **Always validate inputs**: Check options for valid values
2. **Use appropriate log levels**: DEBUG for detailed info, ERROR for failures
3. **Store pointers in Initialize**: Get skin/measure name once and reuse
4. **Handle nullptr**: Check for null pointers from API functions
5. **Use wide strings**: All strings are wide character (LPCWSTR)
6. **Format numbers carefully**: Use appropriate format specifiers in RmLogF
7. **Read actions without replacement**: Use FALSE for replaceMeasures when reading bangs

## Version Compatibility

Some functions are not available in older Rainmeter versions:

- `RmReadStringFromSection` - Returns default in old versions
- `RmReadFormulaFromSection` - Returns default in old versions
- `RmReadIntFromSection` - Returns default in old versions
- `RmReadDoubleFromSection` - Returns default in old versions

The API handles this gracefully by checking for function availability at runtime.
