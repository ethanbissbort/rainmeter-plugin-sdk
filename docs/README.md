# Rainmeter Plugin SDK Documentation

Welcome to the Rainmeter Plugin SDK documentation. This SDK provides everything you need to create plugins for Rainmeter 4.5 and higher in C/C++ or C#.

## Table of Contents

1. [Getting Started](#getting-started)
2. [Documentation Structure](#documentation-structure)
3. [Quick Start](#quick-start)
4. [Resources](#resources)

## Getting Started

### Prerequisites

- **Visual Studio 2022** (any edition including Community)
  - Download: https://www.visualstudio.com/downloads/
- **Windows SDK** (included with Visual Studio)
- **Rainmeter 4.5 or higher** for testing

### What's Included

The SDK contains:

- **API Files** (`API/`)
  - `RainmeterAPI.h` - C++ API header
  - `RainmeterAPI.cs` - C# API wrapper
  - `DllExporter.exe` - C# DLL export utility
  - Pre-built libraries for x32 and x64

- **C++ Examples** (`C++/`)
  - Empty template
  - SystemVersion - Reading system information
  - RmExecute - Executing Rainmeter commands
  - DataHandling - Data persistence
  - ParentChild - Parent-child measure relationships
  - SectionVariables - Custom section variables

- **C# Examples** (`C#/`)
  - Same examples as C++, implemented in C#

## Documentation Structure

- **[API Reference - C++](api-reference-cpp.md)** - Complete C++ API documentation
- **[API Reference - C#](api-reference-csharp.md)** - Complete C# API documentation
- **[Plugin Lifecycle](plugin-lifecycle.md)** - Understanding plugin initialization, updates, and finalization
- **[C++ Examples](examples-cpp.md)** - Detailed C++ example walkthroughs
- **[C# Examples](examples-csharp.md)** - Detailed C# example walkthroughs
- **[Building and Setup](building-and-setup.md)** - Build instructions and project setup
- **[Advanced Topics](advanced-topics.md)** - Parent-child patterns, section variables, data persistence

## Quick Start

### Creating Your First C++ Plugin

1. Copy the `C++/PluginEmpty` directory
2. Rename the directory and files to match your plugin name
3. Open `C++/SDK-CPP.sln` in Visual Studio 2022
4. Add your renamed project to the solution
5. Implement your plugin logic in the lifecycle functions
6. Build for x64 and/or x32
7. Copy the DLL to Rainmeter's Plugins folder

### Creating Your First C# Plugin

1. Copy the `C#/PluginEmpty` directory
2. Rename the directory and files to match your plugin name
3. Open `C#/SDK-CS.sln` in Visual Studio 2022
4. Add your renamed project to the solution
5. Implement your plugin logic using the `Rainmeter.API` class
6. Build for x64 and/or x86
7. The post-build event will run `DllExporter.exe` to export functions
8. Copy the DLL to Rainmeter's Plugins folder

## Quick Example: System Version Plugin

### C++ Version

```cpp
struct Measure
{
    MeasureType type;
};

PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    *data = measure;
}

PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;
    LPCWSTR value = RmReadString(rm, L"Type", L"");
    // Parse options...
}

PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;
    // Return numeric value
    return 10.0;
}

PLUGIN_EXPORT void Finalize(void* data)
{
    Measure* measure = (Measure*)data;
    delete measure;
}
```

### C# Version

```csharp
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
        Rainmeter.API api = new Rainmeter.API(rm);
        string value = api.ReadString("Type", "");
        // Parse options...
    }

    [DllExport]
    public static double Update(IntPtr data)
    {
        Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
        // Return numeric value
        return 10.0;
    }

    [DllExport]
    public static void Finalize(IntPtr data)
    {
        GCHandle.FromIntPtr(data).Free();
    }
}
```

## Resources

### Official Documentation
- Rainmeter Developer Docs: https://docs.rainmeter.net/developers/
- Plugin API Guide: https://docs.rainmeter.net/developers/plugin/

### Repository
- GitHub: https://github.com/rainmeter/rainmeter-plugin-sdk

### Support
- Rainmeter Forums: https://forum.rainmeter.net/
- GitHub Issues: https://github.com/rainmeter/rainmeter-plugin-sdk/issues

## License

This SDK is licensed under the GNU General Public License v2.0 or later.
See the license headers in individual files for details.

## Next Steps

- Review the [Plugin Lifecycle](plugin-lifecycle.md) to understand how plugins work
- Check out the [API Reference](api-reference-cpp.md) for available functions
- Explore the [Examples](examples-cpp.md) to see common patterns
- Read [Building and Setup](building-and-setup.md) for detailed build instructions
