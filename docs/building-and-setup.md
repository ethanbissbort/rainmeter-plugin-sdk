# Building and Setup Guide

Complete guide for setting up your development environment and building Rainmeter plugins.

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Visual Studio Setup](#visual-studio-setup)
3. [Building C++ Plugins](#building-cpp-plugins)
4. [Building C# Plugins](#building-csharp-plugins)
5. [Project Configuration](#project-configuration)
6. [Debugging Plugins](#debugging-plugins)
7. [Deploying Plugins](#deploying-plugins)
8. [Troubleshooting](#troubleshooting)

---

## System Requirements

### Required

- **Windows 10 or higher**
- **Visual Studio 2022** (any edition)
  - Download Community (free): https://www.visualstudio.com/downloads/
- **Windows SDK** (included with Visual Studio)
- **Rainmeter 4.5 or higher** for testing
  - Download: https://www.rainmeter.net/

### Optional

- Git for version control
- Text editor for skin files (Notepad++, VS Code, etc.)

---

## Visual Studio Setup

### Installation

1. Download Visual Studio 2022
2. Run the installer
3. Select workloads:
   - **Desktop development with C++** (for C++ plugins)
   - **.NET desktop development** (for C# plugins)
4. Individual components (verify these are included):
   - Windows 10/11 SDK
   - C++ core features
   - MSVC v143 (VS 2022) C++ build tools
   - .NET Framework 4.7.1 targeting pack (for C#)

### First Time Setup

1. Clone or download the SDK
   ```bash
   git clone https://github.com/rainmeter/rainmeter-plugin-sdk.git
   ```

2. Open the appropriate solution:
   - C++: `C++/SDK-CPP.sln`
   - C#: `C#/SDK-CS.sln`

3. Set the configuration:
   - **Configuration**: Debug or Release
   - **Platform**: x86 (32-bit) or x64 (64-bit)

4. Build the solution: `Ctrl+Shift+B`

---

## Building C++ Plugins

### Project Structure

```
C++/
├── SDK-CPP.sln          # Visual Studio solution
├── PluginEmpty/         # Template plugin
│   ├── PluginEmpty.cpp
│   ├── PluginEmpty.vcxproj
│   └── PluginEmpty.rc
└── [Other examples...]
```

### Build Configurations

| Configuration | Platform | Output Directory |
|--------------|----------|------------------|
| Debug | Win32 | `x32/Debug/` |
| Debug | x64 | `x64/Debug/` |
| Release | Win32 | `x32/Release/` |
| Release | x64 | `x64/Release/` |

### Building from Visual Studio

1. Open `C++/SDK-CPP.sln`
2. Select configuration (Debug/Release) and platform (x86/x64)
3. Right-click project → Build
4. Find DLL in output directory

### Building from Command Line

```cmd
# Set up Visual Studio environment
"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat"

# Build for x64 Release
msbuild C++\SDK-CPP.sln -t:rebuild -p:Configuration=Release -p:Platform=x64

# Build for x86 Release
msbuild C++\SDK-CPP.sln -t:rebuild -p:Configuration=Release -p:Platform=x86
```

### Project Settings

**Platform Toolset**: v143 (Visual Studio 2022)

**Character Set**: Unicode

**Configuration Type**: Dynamic Library (.dll)

**C++ Language Standard**: C++14 or higher

**Include Directories**: `../../API` (for RainmeterAPI.h)

**Linker Settings**:
- Additional Library Directories: `../../API/x32` or `../../API/x64`
- Additional Dependencies: `Rainmeter.lib` (for advanced scenarios)

---

## Building C# Plugins

### Project Structure

```
C#/
├── SDK-CS.sln           # Visual Studio solution
├── PluginEmpty/         # Template plugin
│   ├── PluginEmpty.cs
│   ├── AssemblyInfo.cs
│   └── PluginEmpty.csproj
└── [Other examples...]
```

### Build Configurations

| Configuration | Platform | Output Directory |
|--------------|----------|------------------|
| Debug | x86 | `x32/Debug/` |
| Debug | x64 | `x64/Debug/` |
| Release | x86 | `x32/Release/` |
| Release | x64 | `x64/Release/` |

### Building from Visual Studio

1. Open `C#/SDK-CS.sln`
2. Select configuration (Debug/Release) and platform (x86/x64)
3. Right-click project → Build
4. **Post-build event** runs automatically:
   - Executes `DllExporter.exe`
   - Exports [DllExport] functions
5. Find DLL in output directory

### Building from Command Line

```cmd
# Build for x64 Release
msbuild C#\SDK-CS.sln -t:rebuild -p:Configuration=Release -p:Platform=x64

# Build for x86 Release
msbuild C#\SDK-CS.sln -t:rebuild -p:Configuration=Release -p:Platform=x86
```

### Project Settings

**Target Framework**: .NET Framework 4.7.1

**Platform Target**: x86 or x64 (must match!)

**Output Type**: Class Library

**Compile Files**:
- Your plugin .cs file
- AssemblyInfo.cs
- `$(SolutionDir)..\API\RainmeterAPI.cs` (linked)

**Post-Build Event** (IMPORTANT):
```
"$(SolutionDir)..\API\DllExporter.exe" "$(ConfigurationName)" "$(PlatformName)" "$(TargetDir)\" "$(TargetFileName)"
```

**Don't remove this!** It's required to export functions from the managed DLL.

---

## Project Configuration

### Creating a New C++ Plugin

1. **Copy the template**:
   ```
   Copy C++/PluginEmpty to C++/MyPlugin
   ```

2. **Rename files**:
   - `PluginEmpty.cpp` → `MyPlugin.cpp`
   - `PluginEmpty.rc` → `MyPlugin.rc`
   - `PluginEmpty.vcxproj` → `MyPlugin.vcxproj`

3. **Update .vcxproj**:
   - Open in text editor
   - Replace all instances of "PluginEmpty" with "MyPlugin"
   - Update `<RootNamespace>` and `<ProjectGuid>`

4. **Update .rc file**:
   - Change `VALUE "OriginalFilename"` to "MyPlugin.dll"
   - Update version info

5. **Add to solution**:
   - Open SDK-CPP.sln
   - Right-click solution → Add → Existing Project
   - Select MyPlugin.vcxproj

6. **Build and test**

### Creating a New C# Plugin

1. **Copy the template**:
   ```
   Copy C#/PluginEmpty to C#/MyPlugin
   ```

2. **Rename files**:
   - `PluginEmpty.cs` → `MyPlugin.cs`
   - `PluginEmpty.csproj` → `MyPlugin.csproj`

3. **Update .csproj**:
   - Open in text editor
   - Update `<RootNamespace>` to "MyPlugin"
   - Update `<AssemblyName>` to "MyPlugin"
   - Update `<ProjectGuid>` (generate new GUID)

4. **Update .cs file**:
   - Change namespace from "PluginEmpty" to "MyPlugin"

5. **Update AssemblyInfo.cs**:
   - Update title, description, etc.

6. **Add to solution**:
   - Open SDK-CS.sln
   - Right-click solution → Add → Existing Project
   - Select MyPlugin.csproj

7. **Verify post-build event exists**

8. **Build and test**

---

## Debugging Plugins

### Setting Up Debugging

#### C++

1. Right-click project → Properties
2. Debugging section:
   - **Command**: Path to `Rainmeter.exe`
     ```
     C:\Program Files\Rainmeter\Rainmeter.exe
     ```
   - **Command Arguments**: (optional)
     ```
     !ActivateConfig "MySkin" "skin.ini"
     ```

3. Copy DLL to Rainmeter plugins folder:
   ```
   C:\Users\[YourName]\Documents\Rainmeter\Plugins\
   ```

4. Create a test skin that uses your plugin

5. Press F5 to start debugging

#### C#

Same as C++, but note:
- Managed debugging can be slower
- Set "Enable Just My Code" for better experience
- Use `api.Log()` liberally for debugging

### Debugging Tips

1. **Use logging extensively**:
   ```cpp
   RmLogF(rm, LOG_DEBUG, L"Value: %d", myValue);
   ```

2. **Set breakpoints** in your plugin code

3. **Watch variables** in Visual Studio

4. **Check Rainmeter log**:
   - Rainmeter → Manage → Log tab
   - Or About → Log

5. **Use debug build** for better symbols

6. **Attach to process**:
   - Debug → Attach to Process
   - Select Rainmeter.exe

### Common Debug Scenarios

**Plugin not loading:**
- Check Rainmeter log for errors
- Verify platform (x86 vs x64) matches
- Ensure all dependencies are present

**Crash on load:**
- Check Initialize function
- Verify memory allocation
- Look for null pointer access

**Wrong values:**
- Log option values in Reload
- Check data types and conversions
- Verify formulas are parsed correctly

---

## Deploying Plugins

### File Structure

```
@Resources/
    Plugins/
        MyPlugin_x32.dll  (32-bit version)
        MyPlugin_x64.dll  (64-bit version)
MySkin.ini
```

### Skin Configuration

```ini
[Rainmeter]
Update=1000

[MeasurePlugin]
Measure=Plugin
Plugin=MyPlugin
; Options here
```

Rainmeter automatically loads the correct architecture (x32/x64).

### Distribution

1. **Create .rmskin package**:
   - Include both x32 and x64 DLLs
   - Place in `@Resources\Plugins`
   - Use Rainmeter Skin Packager

2. **Manual installation**:
   - Users copy DLLs to:
     ```
     Documents\Rainmeter\Plugins\
     ```

3. **Documentation**:
   - Include readme with options
   - Provide example skins
   - List requirements (Rainmeter version, etc.)

### Version Information

**C++ (.rc file)**:
```rc
VS_VERSION_INFO VERSIONINFO
 FILEVERSION 1,0,0,0
 PRODUCTVERSION 1,0,0,0
 FILEFLAGSMASK 0x17L
 FILEFLAGS 0x0L
 FILEOS 0x4L
 FILETYPE 0x2L
 FILESUBTYPE 0x0L
BEGIN
    // ... version strings
END
```

**C# (AssemblyInfo.cs)**:
```csharp
[assembly: AssemblyVersion("1.0.0.0")]
[assembly: AssemblyFileVersion("1.0.0.0")]
```

---

## Troubleshooting

### Build Errors

**Error: Cannot open include file 'RainmeterAPI.h'**
- Solution: Check include directories point to `../../API`

**Error: Platform toolset 'v143' not installed**
- Solution: Install Visual Studio 2022 C++ tools

**Error: DllExporter.exe not found (C#)**
- Solution: Ensure `API/DllExporter.exe` exists
- Check post-build event path

**Error: Target framework not supported**
- Solution: Install .NET Framework 4.7.1 targeting pack

### Runtime Errors

**Plugin not appearing in Rainmeter**
- Check plugin name in skin matches DLL name
- Verify correct architecture (x32/x64)
- Look in Rainmeter log for load errors

**Crash on skin load**
- Check Initialize function
- Verify no null pointer access
- Use try-catch in C# Initialize

**Wrong behavior**
- Add debug logging
- Verify options are read correctly
- Check Update cycle timing

### Performance Issues

**Slow Update function**
- Move expensive operations to background thread
- Cache results when possible
- Consider using parent-child pattern

**Memory leaks**
- Always free memory in Finalize
- Check for unreleased resources
- Use memory profiler

---

## CI/CD Integration

The SDK includes GitHub Actions workflow (`.github/workflows/build.yml`):

```yaml
name: build
on: push
jobs:
  build:
    runs-on: windows-2025
    steps:
      - uses: actions/checkout@v4
      - uses: microsoft/setup-msbuild@v2
      - name: Build C# SDK
        run: msbuild.exe C#\SDK-CS.sln -t:rebuild -p:Configuration=Release -p:Platform=x64
      - name: Build C++ SDK
        run: msbuild.exe C++\SDK-CPP.sln -t:rebuild -p:Configuration=Release -p:Platform=x64
```

Automatically builds on every push to verify no build errors.

---

## Best Practices

1. **Always build both x32 and x64** for compatibility
2. **Test with both Debug and Release** builds
3. **Use Release builds** for distribution
4. **Version your plugins** properly
5. **Include debug symbols** (.pdb) during development
6. **Clean and rebuild** when changing project settings
7. **Test on clean Rainmeter installation** before release
8. **Keep SDK updated** to latest version

---

## Quick Reference

### Build Commands

```cmd
# C++ Debug x64
msbuild C++\SDK-CPP.sln /p:Configuration=Debug /p:Platform=x64

# C++ Release x86
msbuild C++\SDK-CPP.sln /p:Configuration=Release /p:Platform=Win32

# C# Release x64
msbuild C#\SDK-CS.sln /p:Configuration=Release /p:Platform=x64

# Build all configurations
msbuild C++\SDK-CPP.sln /t:rebuild /p:Configuration=Release /p:Platform="x64;x86"
```

### File Locations

- **API Headers**: `API/RainmeterAPI.h`, `API/RainmeterAPI.cs`
- **Libraries**: `API/x32/Rainmeter.lib`, `API/x64/Rainmeter.lib`
- **DllExporter**: `API/DllExporter.exe`
- **Output**: `[Project]/x32/[Config]/`, `[Project]/x64/[Config]/`
- **Rainmeter Plugins**: `Documents\Rainmeter\Plugins\`

---

## Next Steps

- Review [API Reference](api-reference-cpp.md) for available functions
- Check [Examples](examples-cpp.md) for common patterns
- Read [Plugin Lifecycle](plugin-lifecycle.md) for understanding flow
- Explore [Advanced Topics](advanced-topics.md) for complex scenarios
