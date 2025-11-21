# Advanced Topics

Advanced plugin development topics, patterns, and best practices.

## Table of Contents

1. [Parent-Child Pattern](#parent-child-pattern)
2. [Data Persistence](#data-persistence)
3. [Section Variables](#section-variables)
4. [Threading and Async Operations](#threading-and-async-operations)
5. [Performance Optimization](#performance-optimization)
6. [Error Handling](#error-handling)
7. [Memory Management](#memory-management)
8. [Interoperability](#interoperability)
9. [Security Considerations](#security-considerations)
10. [Best Practices](#best-practices)

---

## Parent-Child Pattern

### When to Use

Use the parent-child pattern when:
- One operation is expensive (network query, file parsing, etc.)
- Multiple measures need different parts of the same data
- You want to avoid redundant operations
- Data should be shared across measures in the same skin

### Implementation Strategy

**Core Concept**: One parent measure queries/calculates data, multiple child measures access different aspects.

### C++ Implementation

```cpp
// Parent stores data
struct ParentMeasure
{
    void* skin;
    LPCWSTR name;
    ChildMeasure* ownerChild;

    // Your shared data
    std::vector<std::wstring> items;
    std::map<std::wstring, int> values;
};

// Child accesses parent data
struct ChildMeasure
{
    ParentMeasure* parent;
    int index;  // Which item to return
};

// Global registry (one per DLL)
std::vector<ParentMeasure*> g_Parents;

PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    ChildMeasure* child = new ChildMeasure;
    *data = child;

    void* skin = RmGetSkin(rm);
    LPCWSTR parentName = RmReadString(rm, L"ParentName", L"");

    if (!*parentName)
    {
        // Create parent
        child->parent = new ParentMeasure;
        child->parent->name = RmGetMeasureName(rm);
        child->parent->skin = skin;
        child->parent->ownerChild = child;
        g_Parents.push_back(child->parent);
    }
    else
    {
        // Find existing parent (match name AND skin!)
        for (auto p : g_Parents)
        {
            if (_wcsicmp(p->name, parentName) == 0 && p->skin == skin)
            {
                child->parent = p;
                return;
            }
        }
        RmLog(rm, LOG_ERROR, L"Parent not found");
    }
}

PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    ChildMeasure* child = (ChildMeasure*)data;

    // Read child options
    child->index = RmReadInt(rm, L"Index", 0);

    // Read parent options (only for owner)
    if (child->parent && child->parent->ownerChild == child)
    {
        LPCWSTR file = RmReadString(rm, L"File", L"");
        // Load data into parent...
    }
}

PLUGIN_EXPORT double Update(void* data)
{
    ChildMeasure* child = (ChildMeasure*)data;
    ParentMeasure* parent = child->parent;

    if (!parent) return 0.0;

    // Parent updates data (only for owner)
    if (parent->ownerChild == child)
    {
        // Do expensive operation here
        // parent->items = QueryData();
    }

    // Child accesses data
    if (child->index < parent->items.size())
        return (double)parent->items[child->index].length();

    return 0.0;
}

PLUGIN_EXPORT void Finalize(void* data)
{
    ChildMeasure* child = (ChildMeasure*)data;
    ParentMeasure* parent = child->parent;

    // Only owner deletes parent
    if (parent && parent->ownerChild == child)
    {
        g_Parents.erase(
            std::remove(g_Parents.begin(), g_Parents.end(), parent),
            g_Parents.end());
        delete parent;
    }

    delete child;
}
```

### C# Implementation

```csharp
internal class ParentMeasure : Measure
{
    internal static List<ParentMeasure> Parents = new List<ParentMeasure>();

    internal string Name;
    internal IntPtr Skin;

    // Your shared data
    internal List<string> Items = new List<string>();

    internal ParentMeasure()
    {
        Parents.Add(this);
    }

    internal override void Dispose()
    {
        Parents.Remove(this);
    }

    internal void UpdateData()
    {
        // Expensive operation here
        // Items = QueryData();
    }
}

internal class ChildMeasure : Measure
{
    private ParentMeasure parent;
    private int index;

    internal override void Reload(API api, ref double maxValue)
    {
        string parentName = api.ReadString("ParentName", "");
        IntPtr skin = api.GetSkin();

        // Find parent
        parent = ParentMeasure.Parents.Find(p =>
            p.Skin.Equals(skin) && p.Name.Equals(parentName));

        if (parent == null)
            api.Log(API.LogType.Error, "Parent not found");

        index = api.ReadInt("Index", 0);
    }

    internal override double Update()
    {
        if (parent == null) return 0.0;

        if (index < parent.Items.Count)
            return parent.Items[index].Length;

        return 0.0;
    }
}
```

### Important Considerations

1. **Match skin AND name**: Multiple skins may use same measure name
2. **Owner child**: One child "owns" the parent and handles cleanup
3. **Update once**: Parent updates data once, children just access it
4. **Thread safety**: If using threads, add locks around shared data

---

## Data Persistence

### Rainmeter.data File

The `Rainmeter.data` file is an INI file for persistent storage.

**Location**: `%APPDATA%\Rainmeter\Rainmeter.data`

### Reading Data (C++)

```cpp
PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;

    // Get file path
    LPCWSTR dataFile = RmGetSettingsFile();

    // Read integer
    int value = GetPrivateProfileInt(
        L"MyPlugin",      // Section name
        L"Counter",       // Key name
        0,                // Default value
        dataFile          // File path
    );

    // Read string
    WCHAR buffer[256];
    GetPrivateProfileString(
        L"MyPlugin",
        L"LastUser",
        L"",
        buffer,
        256,
        dataFile
    );
    std::wstring lastUser = buffer;
}
```

### Writing Data (C++)

```cpp
PLUGIN_EXPORT void Finalize(void* data)
{
    Measure* measure = (Measure*)data;

    LPCWSTR dataFile = RmGetSettingsFile();

    // Write integer
    WCHAR buffer[32];
    _itow_s(measure->counter, buffer, 10);
    WritePrivateProfileString(
        L"MyPlugin",
        L"Counter",
        buffer,
        dataFile
    );

    // Write string
    WritePrivateProfileString(
        L"MyPlugin",
        L"LastUser",
        L"JohnDoe",
        dataFile
    );

    delete measure;
}
```

### Reading Data (C#)

```csharp
[DllImport("kernel32.dll", CharSet = CharSet.Unicode)]
static extern int GetPrivateProfileString(
    string section, string key, string defaultValue,
    [In, Out] char[] value, int size, string filePath);

[DllExport]
public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
    string dataFile = API.GetSettingsFile();

    // Read value
    char[] buffer = new char[256];
    GetPrivateProfileString("MyPlugin", "Counter", "0",
        buffer, 256, dataFile);

    try
    {
        string value = new string(buffer).TrimEnd('\0');
        measure.counter = Convert.ToInt32(value);
    }
    catch (Exception ex)
    {
        // Handle error
    }
}
```

### Writing Data (C#)

```csharp
[DllImport("kernel32", CharSet = CharSet.Unicode)]
[return: MarshalAs(UnmanagedType.Bool)]
private static extern bool WritePrivateProfileString(
    string section, string key, string value, string filePath);

[DllExport]
public static void Finalize(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    string dataFile = API.GetSettingsFile();
    WritePrivateProfileString("MyPlugin", "Counter",
        measure.counter.ToString(), dataFile);

    GCHandle.FromIntPtr(data).Free();
}
```

### Best Practices for Persistence

1. **Use unique section names**: Prefix with your plugin name
2. **Handle missing values**: Always provide defaults
3. **Validate loaded data**: Check for corrupted values
4. **Save in Finalize**: Not in Update (too frequent)
5. **Optional persistence**: Give users control (SaveData option)
6. **Document saved data**: Tell users what's stored

---

## Section Variables

### C++ Only Feature

Custom section variables allow inline function calls:
```ini
[!SetVariable Text "[MyMeasure:ToUpper(hello)]"]
```

### Function Signature

```cpp
PLUGIN_EXPORT LPCWSTR FunctionName(void* data,
                                   const int argc,
                                   const WCHAR* argv[])
```

### Implementation

```cpp
struct Measure
{
    std::wstring input;
    std::wstring buffer;  // For return values
};

PLUGIN_EXPORT LPCWSTR ToUpper(void* data, const int argc, const WCHAR* argv[])
{
    Measure* measure = (Measure*)data;

    // Check arguments
    if (argc > 0)
    {
        measure->buffer = argv[0];  // Use first argument
    }
    else
    {
        measure->buffer = measure->input;  // Use stored value
    }

    // Transform
    std::transform(measure->buffer.begin(), measure->buffer.end(),
                   measure->buffer.begin(), std::towupper);

    return measure->buffer.c_str();
}

// Multiple arguments example
PLUGIN_EXPORT LPCWSTR Substring(void* data, const int argc, const WCHAR* argv[])
{
    Measure* measure = (Measure*)data;

    if (argc < 2)
    {
        measure->buffer = L"";
        return measure->buffer.c_str();
    }

    std::wstring text = argv[0];
    int start = _wtoi(argv[1]);
    int length = (argc > 2) ? _wtoi(argv[2]) : text.length() - start;

    measure->buffer = text.substr(start, length);
    return measure->buffer.c_str();
}
```

### Usage in Skins

```ini
[mString]
Measure=Plugin
Plugin=MyPlugin
Input=Default Text

[Meter1]
Meter=String
Text=[mString:ToUpper()]
DynamicVariables=1

[Meter2]
Meter=String
Text=[mString:ToUpper(custom text)]
DynamicVariables=1

[Meter3]
Meter=String
Text=[mString:Substring(Hello World, 0, 5)]
DynamicVariables=1
```

### C# Alternative with ExecuteBang

```csharp
[DllExport]
public static void ExecuteBang(IntPtr data,
    [MarshalAs(UnmanagedType.LPWStr)] String args)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    // Parse command and arguments
    var parts = args.Split(new[] { ' ' }, 2);
    string command = parts[0];
    string argument = parts.Length > 1 ? parts[1] : "";

    switch (command.ToLowerInvariant())
    {
        case "toupper":
            measure.result = argument.ToUpper();
            break;

        case "tolower":
            measure.result = argument.ToLower();
            break;

        case "substring":
            var subParts = argument.Split(',');
            if (subParts.Length >= 2)
            {
                int start = int.Parse(subParts[0].Trim());
                int length = int.Parse(subParts[1].Trim());
                measure.result = measure.input.Substring(start, length);
            }
            break;
    }
}
```

Skin usage:
```ini
[!CommandMeasure MyMeasure "ToUpper hello world"]
```

---

## Threading and Async Operations

### When to Use Threads

- Network requests
- File I/O operations
- Heavy computations
- Any operation that blocks for > 10ms

### C++ Threading

```cpp
#include <thread>
#include <atomic>
#include <mutex>

struct Measure
{
    std::thread worker;
    std::atomic<bool> running;
    std::atomic<bool> dataReady;
    std::mutex dataMutex;

    std::wstring result;
    bool threadStarted;

    Measure() : running(false), dataReady(false), threadStarted(false) {}
};

void WorkerThread(Measure* measure)
{
    while (measure->running)
    {
        // Do work
        std::wstring newResult = FetchData();

        // Update with lock
        {
            std::lock_guard<std::mutex> lock(measure->dataMutex);
            measure->result = newResult;
            measure->dataReady = true;
        }

        std::this_thread::sleep_for(std::chrono::seconds(5));
    }
}

PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;
    *data = measure;
}

PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;

    if (!measure->threadStarted)
    {
        measure->running = true;
        measure->worker = std::thread(WorkerThread, measure);
        measure->threadStarted = true;
    }
}

PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;

    // Check if new data is ready
    if (measure->dataReady.load())
    {
        return 1.0;
    }

    return 0.0;
}

PLUGIN_EXPORT LPCWSTR GetString(void* data)
{
    Measure* measure = (Measure*)data;

    // Read with lock
    std::lock_guard<std::mutex> lock(measure->dataMutex);
    return measure->result.c_str();
}

PLUGIN_EXPORT void Finalize(void* data)
{
    Measure* measure = (Measure*)data;

    // Stop thread
    if (measure->threadStarted)
    {
        measure->running = false;
        if (measure->worker.joinable())
        {
            measure->worker.join();
        }
    }

    delete measure;
}
```

### C# Threading

```csharp
using System.Threading;
using System.Threading.Tasks;

internal class Measure
{
    private Task workerTask;
    private CancellationTokenSource cancellation;
    private readonly object lockObj = new object();
    private string result = "";
    private bool dataReady = false;

    public void StartWorker()
    {
        cancellation = new CancellationTokenSource();

        workerTask = Task.Run(async () =>
        {
            while (!cancellation.Token.IsCancellationRequested)
            {
                // Do work
                string newResult = await FetchDataAsync();

                // Update with lock
                lock (lockObj)
                {
                    result = newResult;
                    dataReady = true;
                }

                await Task.Delay(5000, cancellation.Token);
            }
        }, cancellation.Token);
    }

    public double Update()
    {
        lock (lockObj)
        {
            return dataReady ? 1.0 : 0.0;
        }
    }

    public string GetResult()
    {
        lock (lockObj)
        {
            return result;
        }
    }

    public void Dispose()
    {
        cancellation?.Cancel();
        workerTask?.Wait(1000);  // Wait up to 1 second
    }
}

[DllExport]
public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
    measure.StartWorker();
}

[DllExport]
public static void Finalize(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
    measure.Dispose();
    GCHandle.FromIntPtr(data).Free();
}
```

### Threading Best Practices

1. **Always use locks** for shared data
2. **Stop threads in Finalize** with timeout
3. **Handle exceptions** in thread code
4. **Use atomic types** for simple flags
5. **Don't block Update()** waiting for thread
6. **Consider thread pools** for many tasks

---

## Performance Optimization

### Profiling

Use Visual Studio's profiler to identify bottlenecks:
1. Debug → Performance Profiler
2. Select CPU Usage
3. Start profiling
4. Use skin normally
5. Stop and analyze results

### Common Optimizations

**1. Cache Expensive Results**
```cpp
struct Measure
{
    std::wstring cachedPath;
    FILETIME lastModified;
    Data cachedData;
};

PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;

    // Check if file changed
    FILETIME current;
    GetFileTime(measure->cachedPath, &current);

    if (CompareFileTime(&current, &measure->lastModified) != 0)
    {
        // File changed, reload
        measure->cachedData = LoadData(measure->cachedPath);
        measure->lastModified = current;
    }

    // Use cached data
    return ProcessData(measure->cachedData);
}
```

**2. Lazy Initialization**
```cpp
PLUGIN_EXPORT double Update(void* data)
{
    Measure* measure = (Measure*)data;

    // Only initialize when needed
    if (!measure->initialized)
    {
        measure->expensiveResource = CreateResource();
        measure->initialized = true;
    }

    return UseResource(measure->expensiveResource);
}
```

**3. Avoid String Operations in Update**
```cpp
// BAD
PLUGIN_EXPORT double Update(void* data)
{
    std::wstring result = L"Value: " + std::to_wstring(value);
    // String construction every update!
}

// GOOD
PLUGIN_EXPORT double Update(void* data)
{
    measure->cachedValue = value;
    return value;
}

PLUGIN_EXPORT LPCWSTR GetString(void* data)
{
    _snwprintf_s(measure->buffer, _TRUNCATE, L"Value: %d",
                 measure->cachedValue);
    return measure->buffer;
}
```

**4. Pre-allocate Buffers**
```cpp
struct Measure
{
    std::vector<int> buffer;

    Measure()
    {
        buffer.reserve(1000);  // Pre-allocate
    }
};
```

**5. Use Appropriate Data Structures**
```cpp
// For lookups
std::unordered_map<std::wstring, int> fastLookup;  // O(1)

// For ordered iteration
std::vector<Item> items;  // Fast iteration

// For sorted data
std::set<int> sortedValues;  // Auto-sorted
```

---

## Error Handling

### C++ Error Handling

```cpp
PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    Measure* measure = (Measure*)data;

    try
    {
        LPCWSTR file = RmReadPath(rm, L"File", L"");

        // Validate
        if (wcslen(file) == 0)
        {
            RmLog(rm, LOG_ERROR, L"File option is required");
            return;
        }

        // Check file exists
        if (GetFileAttributes(file) == INVALID_FILE_ATTRIBUTES)
        {
            RmLogF(rm, LOG_ERROR, L"File not found: %s", file);
            return;
        }

        // Try to open
        HANDLE hFile = CreateFile(file, GENERIC_READ, FILE_SHARE_READ,
                                  NULL, OPEN_EXISTING, 0, NULL);
        if (hFile == INVALID_HANDLE_VALUE)
        {
            DWORD error = GetLastError();
            RmLogF(rm, LOG_ERROR, L"Failed to open file: Error %d", error);
            return;
        }

        // Success
        CloseHandle(hFile);
    }
    catch (const std::exception& e)
    {
        RmLog(rm, LOG_ERROR, L"Exception in Reload");
    }
}
```

### C# Error Handling

```csharp
[DllExport]
public static void Reload(IntPtr data, IntPtr rm, ref double maxValue)
{
    try
    {
        Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
        Rainmeter.API api = new Rainmeter.API(rm);

        string file = api.ReadPath("File", "");

        // Validate
        if (string.IsNullOrEmpty(file))
        {
            api.Log(API.LogType.Error, "File option is required");
            return;
        }

        // Check file exists
        if (!File.Exists(file))
        {
            api.Log(API.LogType.Error, $"File not found: {file}");
            return;
        }

        // Try to read
        string content = File.ReadAllText(file);
        measure.data = ParseData(content);
    }
    catch (Exception ex)
    {
        Rainmeter.API api = new Rainmeter.API(rm);
        api.Log(API.LogType.Error, $"Exception: {ex.Message}");
    }
}
```

### Error Handling Best Practices

1. **Validate all inputs** in Reload
2. **Log errors** with descriptive messages
3. **Provide defaults** when values are missing
4. **Handle file/network errors** gracefully
5. **Don't crash** - catch exceptions
6. **Use appropriate log levels**

---

## Memory Management

### C++ Memory Guidelines

**1. Always Match New/Delete**
```cpp
PLUGIN_EXPORT void Initialize(void** data, void* rm)
{
    Measure* measure = new Measure;  // Allocate
    *data = measure;
}

PLUGIN_EXPORT void Finalize(void* data)
{
    Measure* measure = (Measure*)data;
    delete measure;  // Free
}
```

**2. Use Smart Pointers**
```cpp
struct Measure
{
    std::unique_ptr<Resource> resource;
    std::shared_ptr<SharedData> shared;
};

// Automatically cleaned up!
```

**3. Avoid Leaks**
```cpp
// BAD
PLUGIN_EXPORT double Update(void* data)
{
    Data* temp = new Data;  // LEAK!
    return temp->value;
}

// GOOD
PLUGIN_EXPORT double Update(void* data)
{
    Data temp;  // Stack allocation
    return temp.value;
}
```

### C# Memory Guidelines

**1. Always Free GCHandles**
```csharp
[DllExport]
public static void Initialize(ref IntPtr data, IntPtr rm)
{
    data = GCHandle.ToIntPtr(GCHandle.Alloc(new Measure()));
}

[DllExport]
public static void Finalize(IntPtr data)
{
    GCHandle.FromIntPtr(data).Free();  // MUST DO THIS!
}
```

**2. Dispose Pattern**
```csharp
internal class Measure : IDisposable
{
    private FileStream stream;

    public void Dispose()
    {
        stream?.Dispose();
    }
}

[DllExport]
public static void Finalize(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;
    measure.Dispose();
    GCHandle.FromIntPtr(data).Free();
}
```

**3. Using Statements**
```csharp
[DllExport]
public static double Update(IntPtr data)
{
    Measure measure = (Measure)GCHandle.FromIntPtr(data).Target;

    using (var file = File.OpenRead(measure.path))
    using (var reader = new StreamReader(file))
    {
        return ParseData(reader.ReadToEnd());
    }
    // Automatically disposed
}
```

---

## Interoperability

### Calling Native APIs from C#

```csharp
// Windows API
[DllImport("user32.dll")]
static extern IntPtr FindWindow(string lpClassName, string lpWindowName);

[DllImport("kernel32.dll", CharSet = CharSet.Unicode)]
static extern uint GetPrivateProfileString(/* ... */);

// Use
IntPtr hwnd = FindWindow(null, "Notepad");
```

### Using C++ DLLs from C#

```csharp
[DllImport("MyNativeLib.dll", CallingConvention = CallingConvention.Cdecl)]
static extern int NativeFunction(int param);

[DllImport("MyNativeLib.dll")]
static extern IntPtr GetString();

// Marshal strings
string result = Marshal.PtrToStringAnsi(GetString());
```

---

## Security Considerations

### Input Validation

```cpp
PLUGIN_EXPORT void Reload(void* data, void* rm, double* maxValue)
{
    // Validate numeric ranges
    int value = RmReadInt(rm, L"Value", 50);
    if (value < 0 || value > 100)
    {
        RmLog(rm, LOG_ERROR, L"Value must be 0-100");
        value = 50;
    }

    // Validate paths
    LPCWSTR path = RmReadPath(rm, L"Path", L"");
    if (!IsPathSafe(path))
    {
        RmLog(rm, LOG_ERROR, L"Invalid path");
        return;
    }

    // Sanitize strings
    std::wstring input = RmReadString(rm, L"Input", L"");
    input = SanitizeInput(input);
}
```

### File Operations

```cpp
bool IsPathSafe(const std::wstring& path)
{
    // Don't allow directory traversal
    if (path.find(L"..") != std::wstring::npos)
        return false;

    // Check it's in allowed directory
    std::wstring basePath = GetBasePath();
    if (path.substr(0, basePath.length()) != basePath)
        return false;

    return true;
}
```

### Command Execution

```cpp
// Never execute arbitrary user input!
// BAD
RmExecute(skin, userInput.c_str());

// GOOD - validate/sanitize
if (command == L"Reset")
    RmExecute(skin, L"!SetVariable Counter 0");
```

---

## Best Practices Summary

### Do

✅ Validate all inputs
✅ Log errors with descriptive messages
✅ Free all allocated memory
✅ Use threads for blocking operations
✅ Cache expensive results
✅ Handle exceptions gracefully
✅ Test with both x32 and x64
✅ Document your options
✅ Provide sensible defaults
✅ Use const-correctness (C++)

### Don't

❌ Read options in Update (use Reload)
❌ Calculate in GetString (use Update)
❌ Block Update() for long periods
❌ Leak memory
❌ Execute arbitrary user input
❌ Assume DLL paths
❌ Ignore errors silently
❌ Use deprecated APIs
❌ Hard-code paths
❌ Forget thread safety

---

## Next Steps

- Review [Examples](examples.md) to see these concepts in action
- Check [API Reference](api-reference-cpp.md) for function details
- Study [Plugin Lifecycle](plugin-lifecycle.md) for the big picture
- Read [Building and Setup](building-and-setup.md) for project configuration

## Additional Resources

- Rainmeter Forums: https://forum.rainmeter.net/
- Official Docs: https://docs.rainmeter.net/developers/
- GitHub Issues: https://github.com/rainmeter/rainmeter-plugin-sdk/issues
