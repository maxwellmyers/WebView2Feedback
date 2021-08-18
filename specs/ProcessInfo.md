Process Info
===

# Background
<!-- TEMPLATE
    Use this section to provide background context for the new API(s)
    in this spec. Try to briefly provide enough information to be able to read
    the rest of the document.

    This section and the appendix are the only sections that likely
    do not get copied into any official documentation, they're just an aid
    to reading this spec. If you find useful information in the background
    or appendix consider moving it to documentation.
    
    If you're modifying an existing API, included a link here to the
    existing page(s) or spec documentation.

    For example, this section is a place to explain why you're adding this
    API rather than modifying an existing API.

    For example, this is a place to provide a brief explanation of some dependent
    area, just explanation enough to understand this new API, rather than telling
    the reader "go read 100 pages of background information posted at ...". 
-->

# Examples
## ProcessRequested

Feature explanation text goes here, including why an app would use it, how it
replaces or supplements existing functionality.

```c#
void ProcessInfoCmdExecuted(object target, ExecutedRoutedEventArgs e)
{
    CoreWebView2ProcessCollection processList = webView.CoreWebView2.ProcessInfo;
    string result;
    uint processListSize = processList.Count;
    if (processListSize == 0)
    {
        result = "No process found.";
    }
    else
    {
        result = $"{processListSize} child process(s) found\n\n";
        for (uint i = 0; i < processListSize; ++i)
        {
            uint processId = processList.GetProcessIdAtIndex(i);
            CoreWebView2ProcessKind kind = processList.GetProcessTypeAtIndex(i);
            result = result + $"Process ID: {processId}\nProcess Kind: {kind}\n";
        }
    }
    MessageBox.Show(this, result, "Process List");
}
    ```
    
```cpp
ProcessComponent::ProcessComponent(AppWindow* appWindow)
    : m_appWindow(appWindow), m_webView(appWindow->GetWebView())
{
    // Register a handler for the ProcessRequested event.
    //! [ProcessRequested]
    CHECK_FAILURE(m_webView->add_ProcessRequested(
        Callback<ICoreWebView2StagingProcessRequestedEventHandler>(
            [this](
                ICoreWebView2* sender, IUnknown* args) -> HRESULT {
                CHECK_FAILURE(m_webView->get_ProcessInfo(&m_processCollection));

                return S_OK;
            })
            .Get(),
        &m_processRequestedToken));
    //! [ProcessRequested]
}

std::wstring ProcessComponent::ProcessKindToString(const COREWEBVIEW2_PROCESS_KIND kind)
{
    switch (kind)
    {
#define KIND_ENTRY(kindValue)                                                                  \
    case kindValue:                                                                            \
        return L#kindValue;

        KIND_ENTRY(COREWEBVIEW2_PROCESS_KIND_BROWSER_PROCESS);
        KIND_ENTRY(COREWEBVIEW2_PROCESS_KIND_RENDERER_PROCESS);
        KIND_ENTRY(COREWEBVIEW2_PROCESS_KIND_UTILITY_PROCESS);
        KIND_ENTRY(COREWEBVIEW2_PROCESS_KIND_SANDBOX_HELPER_PROCESS);
        KIND_ENTRY(COREWEBVIEW2_PROCESS_KIND_GPU_PROCESS);
        KIND_ENTRY(COREWEBVIEW2_PROCESS_KIND_PPAPI_PLUGIN_PROCESS);
        KIND_ENTRY(COREWEBVIEW2_PROCESS_KIND_PPAPI_BROKER_PROCESS);
        KIND_ENTRY(COREWEBVIEW2_PROCESS_KIND_UNKNOWN_PROCESS);

#undef KIND_ENTRY
    }

    return L"PROCESS KIND: " + std::to_wstring(static_cast<uint32_t>(kind));
}

// Get the process info
//! [ProcessRequested]
void ProcessComponent::ProcessInfo()
{
    std::wstring result;
    UINT process_list_size;
    CHECK_FAILURE(m_processCollection->get_Count(&process_list_size));

    if (process_list_size == 0)
    {
        result += L"No process found.";
    }
    else
    {
        result += std::to_wstring(process_list_size) + L" process(s) found";
        result += L"\n\n";
        for (UINT i = 0; i < process_list_size; ++i)
        {
            UINT32 processId = 0;
            CHECK_FAILURE(
                m_processCollection->GetProcessIdAtIndex(i, &processId));

            WCHAR buffer[4096] = L"";
            StringCchPrintf(buffer, ARRAYSIZE(buffer), L"Process ID: %u\n", processId);

            result += buffer;

            COREWEBVIEW2_PROCESS_KIND kind;
            CHECK_FAILURE(m_processCollection->GetProcessTypeAtIndex(i, &kind));

            result += L"Process Kind: " + ProcessKindToString(kind);
            result += L"\n";
        }
    }
    MessageBox(nullptr, result.c_str(), L"GetProcessesInfo Result", MB_OK);
}
//! [ProcessRequested]
```

# API Details    
```
interface ICoreWebView2StagingProcessCollection;
interface ICoreWebView2StagingProcessRequestedEventHandler;

[v1_enum]
typedef enum COREWEBVIEW2_PROCESS_KIND {
  /// Indicates the browser process kind.
  COREWEBVIEW2_PROCESS_KIND_BROWSER_PROCESS,

  /// Indicates the render process kind.
  COREWEBVIEW2_PROCESS_KIND_RENDERER_PROCESS,

  /// Indicates the utility process kind.
  COREWEBVIEW2_PROCESS_KIND_UTILITY_PROCESS,

  /// Indicates the sandbox helper process kind.
  COREWEBVIEW2_PROCESS_KIND_SANDBOX_HELPER_PROCESS,

  /// Indicates the GPU process kind.
  COREWEBVIEW2_PROCESS_KIND_GPU_PROCESS,

  /// Indicates the PPAPI plugin process kind.
  COREWEBVIEW2_PROCESS_KIND_PPAPI_PLUGIN_PROCESS,

  /// Indicates the PPAPI plugin broker process kind.
  COREWEBVIEW2_PROCESS_KIND_PPAPI_BROKER_PROCESS,

  /// Indicates the process of unspecified kind.
  COREWEBVIEW2_PROCESS_KIND_UNKNOWN_PROCESS,
} COREWEBVIEW2_PROCESS_KIND;

[uuid(B625A89E-368F-43F5-BCBA-39AA6234CCF8), object, pointer_default(unique)]
interface ICoreWebView2Staging2 : ICoreWebView2Staging {
  /// Adds an event handler for the `ProcessRequested` event.
  /// 
  /// \snippet ProcessComponent.cpp ProcessRequested
  HRESULT add_ProcessRequested(
      [in] ICoreWebView2StagingProcessRequestedEventHandler* eventHandler,
      [out] EventRegistrationToken* token);

  /// Remove an event handler previously added with `add_ProcessRequested`.
  HRESULT remove_ProcessRequested(
      [in] EventRegistrationToken token);

  /// Returns the `ICoreWebView2StagingProcessCollection`
  [propget] HRESULT ProcessInfo([out, retval]ICoreWebView2StagingProcessCollection** value);
}

/// A list containing process id and corresponding process type.
/// \snippet ProcessComponent.cpp get_ProcessInfo
[uuid(5356F3B3-4859-4763-9C95-837CDEEE8912), object, pointer_default(unique)]
interface ICoreWebView2StagingProcessCollection : IUnknown {
  /// The number of process contained in the ICoreWebView2StagingProcessCollection.
  [propget] HRESULT Count([out, retval] UINT* count);

  /// Gets the process id at the given index.
  HRESULT GetProcessIdAtIndex([in] UINT index, [out, retval] UINT32* value);

  /// Gets the process type at the given index.
  HRESULT GetProcessTypeAtIndex([in] UINT index, [out, retval] COREWEBVIEW2_PROCESS_KIND* processKind);
}

/// An event handler for the `ProcessRequested` event.
[uuid(CFF13C72-2E3B-4812-96FB-DFDDE67FBE90), object, pointer_default(unique)]
interface ICoreWebView2StagingProcessRequestedEventHandler : IUnknown {
  /// Provides the event args for the corresponding event.  No event args exist
  /// and the `args` parameter is set to `null`.
  HRESULT Invoke([in] ICoreWebView2* sender, [in] IUnknown* args);
}
```

```c# (but really MIDL3)
namespace Microsoft.Web.WebView2.Core
{
    // ...
    runtimeclass CoreWebView2ProcessCollection;

    /// Kind of process type used in the CoreWebView2ProcessCollection.
    enum CoreWebView2ProcessKind
    {
        BrowserProcess = 0,
        RendererProcess = 1,
        UtilityProcess = 2,
        SandboxHelperProcess = 3,
        GpuProcess = 4,
        PpapiPluginProcess = 5,
        PpapiBrokerProcess = 6,
        UnknownProcess = 7,
    };

    runtimeclass CoreWebView2
    {
        /// Gets a list of process.
        CoreWebView2ProcessCollection ProcessInfo { get; };
        event Windows.Foundation.TypedEventHandler<CoreWebView2, Object> ProcessRequested;

        // ...
    }

    runtimeclass CoreWebView2ProcessCollection
    {
        // ICoreWebView2ProcessCollection members
        /// Process count.
        UInt32 Count { get; };

        /// Process id.
        UInt32 GetProcessIdAtIndex(UInt32 index);

        /// Process type.
        CoreWebView2ProcessKind GetProcessTypeAtIndex(UInt32 index);
    }

    // ...
}
```
