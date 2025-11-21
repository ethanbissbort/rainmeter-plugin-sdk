# Rainmeter Plugin SDK

## Project Overview

This is the official Rainmeter plugin software development kit (SDK) that contains the necessary tools and examples to build plugins in C/C++ and C#. Plugins built with this SDK require **Rainmeter 4.5 or higher**.

## Project Structure

```
rainmeter-plugin-sdk/
├── API/                    # Core API files for both C++ and C#
│   ├── RainmeterAPI.h     # C++ API header file
│   ├── RainmeterAPI.cs    # C# API wrapper
│   ├── DllExporter.exe    # DLL export utility for C# plugins
│   ├── x32/               # 32-bit libraries
│   │   └── Rainmeter.lib
│   └── x64/               # 64-bit libraries
│       └── Rainmeter.lib
├── C++/                   # C++ plugin examples and solution
│   ├── SDK-CPP.sln        # Visual Studio solution for C++ examples
│   ├── PluginEmpty/       # Blank plugin template
│   ├── PluginSystemVersion/      # Example: System version info
│   ├── PluginRmExecute/          # Example: Execute Rainmeter commands
│   ├── PluginDataHandling/       # Example: Data handling
│   ├── PluginParentChild/        # Example: Parent-child measures
│   └── PluginSectionVariables/   # Example: Section variables
├── C#/                    # C# plugin examples and solution
│   ├── SDK-CS.sln         # Visual Studio solution for C# examples
│   ├── PluginEmpty/       # Blank plugin template
│   ├── PluginSystemVersion/      # Example: System version info
│   ├── PluginRmExecute/          # Example: Execute Rainmeter commands
│   ├── PluginDataHandling/       # Example: Data handling
│   └── PluginParentChild/        # Example: Parent-child measures
└── .github/
    └── workflows/
        └── build.yml      # GitHub Actions CI workflow
```

## Technology Stack

- **Languages**: C/C++, C#
- **Build System**: Visual Studio 2022 (any edition, including Community)
- **Platform**: Windows (x32 and x64)
- **Target Framework**: Requires Rainmeter 4.5+
- **License**: GNU General Public License v2.0 or later

## Key Components

### API Files

- `API/RainmeterAPI.h`: Main C++ API header with exported functions for plugin development
- `API/RainmeterAPI.cs`: C# API wrapper providing access to Rainmeter functionality
- `API/DllExporter.exe`: Utility for exporting functions from C# DLLs

### Plugin Architecture

Both C++ and C# plugins follow a similar lifecycle with these core functions:

1. **Initialize**: Called when a measure is created
2. **Reload**: Called when a measure is reloaded or skin is refreshed
3. **Update**: Called on each update cycle, returns a numeric value
4. **Finalize**: Called when a measure is destroyed (cleanup)

Optional functions:
- **GetString**: Returns a string value for the measure
- **ExecuteBang**: Executes commands sent from the skin
- **Section Variables**: Custom functions for use in section variables

### C++ Plugin Structure

```cpp
struct Measure { /* measure-specific data */ };

PLUGIN_EXPORT void Initialize(void** data, void* rm);
PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue);
PLUGIN_EXPORT double Update(void* data);
PLUGIN_EXPORT LPCWSTR GetString(void* data);  // Optional
PLUGIN_EXPORT void ExecuteBang(void* data, LPCWSTR args);  // Optional
PLUGIN_EXPORT void Finalize(void* data);
```

### C# Plugin Structure

```csharp
namespace PluginName
{
    class Measure { /* measure-specific data */ }

    public class Plugin
    {
        [DllExport]
        public static void Initialize(ref IntPtr data, IntPtr rm);

        [DllExport]
        public static void Reload(IntPtr data, IntPtr rm, ref double maxValue);

        [DllExport]
        public static double Update(IntPtr data);

        [DllExport]  // Optional
        public static IntPtr GetString(IntPtr data);

        [DllExport]  // Optional
        public static void ExecuteBang(IntPtr data, String args);

        [DllExport]
        public static void Finalize(IntPtr data);
    }
}
```

## Building the SDK

### Prerequisites

- Visual Studio 2022 (any edition)
  - Download Community edition: https://www.visualstudio.com/downloads/
- Windows SDK

### Build Instructions

1. Open the appropriate solution file:
   - For C++: `C++/SDK-CPP.sln`
   - For C#: `C#/SDK-CS.sln`

2. Select the desired configuration:
   - Debug or Release
   - x86 (32-bit) or x64 (64-bit)

3. Build the solution (Ctrl+Shift+B)

### Platform Toolset

The examples use Visual Studio 2022 platform toolset (v143). The project files (.vcxproj, .csproj) specify the appropriate toolset versions.

## Development Guidelines

### Creating a New Plugin

1. **Choose a template**: Start with `PluginEmpty` for either C++ or C#
2. **Copy the template**: Create a new directory with your plugin name
3. **Rename files**: Update class names and namespaces to match your plugin
4. **Implement functionality**: Add your plugin logic in the lifecycle functions
5. **Build**: Compile for both x32 and x64 if needed
6. **Test**: Test with Rainmeter 4.5 or higher

### Common API Functions (C++)

- `RmReadString()`: Read string options from the measure
- `RmReadInt()`: Read integer options
- `RmReadDouble()`: Read double options
- `RmExecute()`: Execute Rainmeter commands
- `RmLog()`: Log messages to Rainmeter log
- `RmGet()`: Get measure/meter values

### Common API Functions (C#)

The C# API is accessed through the `Rainmeter.API` class:
- `api.ReadString()`: Read string options
- `api.ReadInt()`: Read integer options
- `api.ReadDouble()`: Read double options
- `api.Execute()`: Execute Rainmeter commands
- `api.Log()`: Log messages
- `Rainmeter.StringBuffer.Update()`: Return strings from GetString()

## Example Plugins

### PluginEmpty
Blank canvas template for starting a new plugin.

### PluginSystemVersion
Demonstrates retrieving system information and returning both numeric and string values.

### PluginRmExecute
Shows how to execute Rainmeter commands (bangs) from within a plugin.

### PluginDataHandling
Demonstrates data management and measure option handling.

### PluginParentChild
Shows how to create parent-child measure relationships.

### PluginSectionVariables (C++ only)
Demonstrates implementing custom section variable functions.

## Documentation

Official documentation is available at: https://docs.rainmeter.net/developers/

### Key Documentation Pages
- Plugin API reference
- C++ plugin guide
- C# plugin guide
- Best practices
- Distribution guidelines

## CI/CD

The project includes GitHub Actions workflow (`.github/workflows/build.yml`) that automatically builds the SDK on commits.

## Important Notes

1. **Memory Management**:
   - C++: Manually manage memory allocation/deallocation
   - C#: Use GCHandle for data pointer management

2. **String Handling**:
   - C++: Uses wide character strings (LPCWSTR)
   - C#: C# strings are marshaled to unmanaged wide strings

3. **Platform Considerations**:
   - Build separate DLLs for x32 and x64 architectures
   - Rainmeter automatically loads the appropriate version

4. **Version Compatibility**:
   - SDK requires Rainmeter 4.5+
   - Some API functions may not be available in older versions

## Common Tasks

### Adding a new example plugin
1. Copy `PluginEmpty` directory
2. Rename all files and update project settings
3. Add to the appropriate solution file (SDK-CPP.sln or SDK-CS.sln)

### Testing a plugin
1. Build the plugin DLL
2. Copy to Rainmeter's Plugins folder
3. Create or modify a skin to use the plugin
4. Load/refresh the skin in Rainmeter

### Debugging
1. Set Rainmeter.exe as the debug executable in project properties
2. Set breakpoints in your plugin code
3. Start debugging (F5) - this launches Rainmeter
4. Load a skin that uses your plugin

## Troubleshooting

- **Build errors**: Ensure Visual Studio 2022 and Windows SDK are installed
- **Plugin not loading**: Check Rainmeter version (must be 4.5+)
- **Export errors (C#)**: Verify DllExporter.exe is present in API folder
- **Platform mismatch**: Ensure plugin architecture matches Rainmeter version
