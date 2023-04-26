# Background
Developers would like to ensure a browser extension is installed so that they may take advantage of the functionality the extension is providing.

# Description
Enable extension services in WebView2. Then end developer can `AddExtension` from folder path, `GetExtensions` to get list of extensions installed. Once an extension is installed,
developers can use `ICoreWebView2Extension` to get id, name of this extension. And also remove or enable/disable this extension.

# Examples

The following code snippet demonstrates how the Extensions related API can be used:

## Win32 C++

```cpp
#include <filesystem>
using namespace std;
// namespace fs = std::filesystem;

void AppWindow::InitializeWebView()
{
    auto options = Microsoft::WRL::Make<CoreWebView2EnvironmentOptions>();
    CHECK_FAILURE(options->put_AreExtensionsEnabled(TRUE));

    // ... other option properties

    // ... CreateCoreWebView2EnvironmentWithOptions

    // Add extension from file path
    AddExtension();

    // Remove an installed extension
    RemoveOrDisableExtension(true);

    // Enable/disable a installed extension
    RemoveOrDisableExtension(false);
}

void ScriptComponent::AddExtension()
{

    // Get the profile object.
    auto webView2_13 = m_webView.try_query<ICoreWebView2_13>();
    wil::com_ptr<ICoreWebView2Profile> webView2Profile;
    CHECK_FAILURE(webView2_13->get_Profile(&webView2Profile));
    auto profile6 = webView2Profile.try_query<ICoreWebView2Profile6>();

    OPENFILENAME openFileName = {};
    openFileName.lStructSize = sizeof(openFileName);
    openFileName.hwndOwner = nullptr;
    openFileName.hInstance = nullptr;
    WCHAR fileName[MAX_PATH] = L"";
    openFileName.lpstrFile = fileName;
    openFileName.lpstrFilter = L"Manifest\0manifest.json\0\0";
    openFileName.nMaxFile = ARRAYSIZE(fileName);
    openFileName.Flags = OFN_OVERWRITEPROMPT;

    if (GetOpenFileName(&openFileName))
    {
        // Remove the filename part of the path.
        *wcsrchr(fileName, L'\\') = L'\0';
        profile6->AddExtension(
            fileName,
            Callback<ICoreWebView2AddExtensionCompletedHandler>(
                [](HRESULT error, ICoreWebViewExtension* extension) -> HRESULT
                {
                    if (error != S_OK)
                    {
                        ShowFailure(error, L"AddExtension failed");
                    }
                    wil::unique_cotaskmem_string id;
                    extension->get_Id(&id);

                    wil::unique_cotaskmem_string name;
                    extension->get_Name(&name);

                    std::wstring extensionInfo;
                    extensionInfo.append(L"Added ");
                    extensionInfo.append(id.get());
                    extensionInfo.append(L" ");
                    extensionInfo.append(name.get());
                    MessageBox(nullptr, extensionInfo.c_str(), L"AddExtension Result", MB_OK);
                    return S_OK;
                })
                .Get());
    }
}

void ScriptComponent::RemoveOrDisableExtension(const bool remove)
{
    // Get the profile object.
    auto webView2_13 = m_webView.try_query<ICoreWebView2_13>();
    wil::com_ptr<ICoreWebView2Profile> webView2Profile;
    CHECK_FAILURE(webView2_13->get_Profile(&webView2Profile));
    auto profile6 = webView2Profile.try_query<ICoreWebView2Profile6>();

    profile6->GetExtensions(
        Callback<ICoreWebView2GetExtensionsCompletedHandler>(
            [this, webView2Profile6,
             remove](HRESULT error, ICoreWebView2ExtensionList* extensions) -> HRESULT
            {
                std::wstring extensionIdString;
                UINT extensionsCount = 0;
                extensions->get_Count(&extensionsCount);

                for (UINT index = 0; index < extensionsCount; ++index)
                {
                    wil::com_ptr<ICoreWebView2Extension> extension;
                    extensions->GetValueAtIndex(index, &extension);

                    wil::unique_cotaskmem_string id;
                    wil::unique_cotaskmem_string name;
                    BOOL enabled = false;

                    extension->get_IsEnabled(&enabled);
                    extension->get_Id(&id);
                    extension->get_Name(&name);

                    extensionIdString += id.get();
                    extensionIdString += L" ";
                    extensionIdString += name.get();
                    if (!enabled)
                    {
                        extensionIdString += L" (disabled)";
                    }
                    else
                    {
                        extensionIdString += L" (enabled)";
                    }
                    extensionIdString += L"\n\r\n";
                }

                TextInputDialog dialog(
                    m_appWindow->GetMainWindow(),
                    remove ? L"Remove Extension" : L"Disable/Enable Extension",
                    L"Extension ID:", extensionIdString.c_str());
                if (dialog.confirmed)
                {
                    for (UINT index = 0; index < extensionsCount; ++index)
                    {
                        wil::com_ptr<ICoreWebView2Extension> extension;
                        extensions->GetValueAtIndex(index, &extension);

                        wil::unique_cotaskmem_string id;
                        wil::unique_cotaskmem_string name;

                        extension->get_Id(&id);
                        if (_wcsicmp(id.get(), dialog.input.c_str()) == 0)
                        {
                            if (remove)
                            {
                                extension->Remove(
                                    Callback<ICoreWebView2RemoveCompletedHandler>(
                                        [](HRESULT error) -> HRESULT
                                        {
                                            if (error != S_OK)
                                            {
                                                ShowFailure(error, L"Remove Extension failed");
                                            }
                                            MessageBox(
                                                nullptr, L"Done", L"Remove Extension", MB_OK);
                                            return S_OK;
                                        })
                                        .Get());
                            }
                            else
                            {
                                BOOL enabled = FALSE;
                                extension->get_IsEnabled(&enabled);

                                extension->SetEnabled(
                                    !enabled,
                                    Callback<ICoreWebView2SetEnabledCompletedHandler>(
                                        [](HRESULT error) -> HRESULT
                                        {
                                            if (error != S_OK)
                                            {
                                                ShowFailure(
                                                    error, L"SetEnabled Extension failed");
                                            }
                                            MessageBox(
                                                nullptr, L"Done", L"Toggled Extension", MB_OK);
                                            return S_OK;
                                        })
                                        .Get());
                            }
                        }
                    }
                }
                return S_OK;
            })
            .Get());
}
```

## .NET and WinRT

```c#
/// Create WebView Environment with option

void CreateEnvironmentWithOption()
{
    CoreWebView2EnvironmentOptions options = new CoreWebView2EnvironmentOptions();
    options.AreExtensionsEnabled = true;
    CoreWebView2Environment environment = await CoreWebView2Environment.CreateAsync(BrowserExecutableFolder, UserDataFolder, options);
}

void ExtensionsCommandExecuted(object target, ExecutedRoutedEventArgs e)
{
    if (_isControlInVisualTree)
    {
        RemoveControlFromVisualTree(webView);
    }
    webView.Dispose();
    webView = GetReplacementControl(false, true);
    AttachControlToVisualTree(webView);
    webView.CoreWebView2InitializationCompleted += (sender, err) =>
    {
        if (err.IsSuccess)
        {
            new Extensions(webView.CoreWebView2).Show();
        }
    };
}

void ExtensionsCommandCanExecute(object sender, CanExecuteRoutedEventArgs e)
{
    e.CanExecute = true;
}

// Extensions class
/// <summary>
/// Interaction logic for Extensions.xaml
/// </summary>
public partial class Extensions : Window
{
    private CoreWebView2 m_coreWebView2;
    public Extensions(CoreWebView2 coreWebView2)
    {
        m_coreWebView2 = coreWebView2;
        InitializeComponent();
        _ = FillViewAsync();
    }

    public class ListEntry
    {
        public string Name;
        public string Id;
        public bool Enabled;

        public override string ToString()
        {
            return (Enabled ? "" : "Disabled ") + Name + " (" + Id + ")";
        }
    }

    List<ListEntry> m_listData = new List<ListEntry>();

    private async System.Threading.Tasks.Task FillViewAsync()
    {
        var extensions = await m_coreWebView2.Profile.GetExtensionsAsync();

        m_listData.Clear();
        for (uint idx = 0; idx < extensions.Count; ++idx)
        {
            ListEntry entry = new ListEntry();
            var extension = extensions.GetValueAtIndex(idx);
            entry.Name = extension.Name;
            entry.Id = extension.Id;
            entry.Enabled = extension.IsEnabled;
            m_listData.Add(entry);
        }
        ExtensionsList.ItemsSource = m_listData;
        ExtensionsList.Items.Refresh();
    }

    private void ExtensionsToggleEnabled(object sender, RoutedEventArgs e)
    {
        _ = ExtensionsToggleEnabledAsync(sender, e);
    }

    private async System.Threading.Tasks.Task ExtensionsToggleEnabledAsync(object sender, RoutedEventArgs e)
    {
        ListEntry entry = (ListEntry)ExtensionsList.SelectedItem;
        var extensions = await m_coreWebView2.Profile.GetExtensionsAsync();
        bool found = false;
        for (uint idx = 0; idx < extensions.Count; ++idx)
        {
            CoreWebView2Extension extension = extensions.GetValueAtIndex(idx);
            if (extension.Id == entry.Id)
            {
                try
                {
                    await extension.SetEnabledAsync(extension.IsEnabled ? false : true);
                }
                catch (Exception exception)
                {
                    MessageBox.Show("Failed to toggle extension enabled: " + exception);
                }
                found = true;
                break;
            }
        }
        if (!found)
        {
            MessageBox.Show("Failed to find extension");
        }
        await FillViewAsync();
    }

    private void ExtensionsAdd(object sender, RoutedEventArgs e)
    {
        _ = ExtensionsAddAsync(sender, e);
    }

    private async System.Threading.Tasks.Task ExtensionsAddAsync(object sender, RoutedEventArgs e)
    {
        var dialog = new TextInputDialog(
            title: "Add extension",
            description: "Enter the absolute Windows file path to the unpackaged browser extension",
            defaultInput: "");
        if (dialog.ShowDialog() == true)
        {
            try
            {
                CoreWebView2Extension extension = await m_coreWebView2.Profile.AddExtensionAsync(dialog.Input.Text);
                MessageBox.Show("Added extension " + extension.Name + " (" + extension.Id + ")");
                await FillViewAsync();
            }
            catch (Exception exception)
            {
                MessageBox.Show("Failed to add extension: " + exception);
            }
        }
    }

    private void ExtensionsRemove(object sender, RoutedEventArgs e)
    {
        _ = ExtensionsRemoveAsync(sender, e);
    }

    private async System.Threading.Tasks.Task ExtensionsRemoveAsync(object sender, RoutedEventArgs e)
    {
        ListEntry entry = (ListEntry)ExtensionsList.SelectedItem;
        if (MessageBox.Show("Remove extension " + entry + "?", "Confirm removal", MessageBoxButton.OKCancel) == MessageBoxResult.OK)
        {
            var extensions = await m_coreWebView2.Profile.GetExtensionsAsync();
            bool found = false;
            for (uint idx = 0; idx < extensions.Count; ++idx)
            {
                CoreWebView2Extension extension = extensions.GetValueAtIndex(idx);
                if (extension.Id == entry.Id)
                {
                    try
                    {
                        await extension.RemoveAsync();
                    }
                    catch (Exception exception)
                    {
                        MessageBox.Show("Failed to remove extension: " + exception);
                    }
                    found = true;
                    break;
                }
            }
            if (!found)
            {
                MessageBox.Show("Failed to find extension");
            }
        }
        await FillViewAsync();
    }
}

```

# API Notes

See [API Details](#api-details) section below for API reference.

# API Details

## Win32 C++

```IDL
interface ICoreWebView2Profile6;
interface ICoreWebView2EnvironmentOptions6;
interface ICoreWebView2RemoveCompletedHandler;
interface ICoreWebView2SetEnabledCompletedHandler;
interface ICoreWebView2Extension;
interface ICoreWebView2ExtensionList;
interface ICoreWebView2AddExtensionCompletedHandler;
interface ICoreWebView2GetExtensionsCompletedHandler;

/// Additional options used to create WebView2 Environment.
[uuid(4B1F63E9-F7A5-4EA5-8D84-2B30F2404E82), object, pointer_default(unique)]
interface ICoreWebView2EnvironmentOptions6 : IUnknown {
  [propget] HRESULT AreExtensionsEnabled([out, retval] BOOL* value);
  [propput] HRESULT IsExtensionEnabled([in] BOOL value);
}

/// This is the ICoreWebView2 profile.
[uuid(8B16D238-9508-4C36-B4D4-749EB9AC4AD0), object, pointer_default(unique)]
interface ICoreWebView2Profile6 : IUnknown {
    /// Adds the Extension using the extension path for unpacked extensions
    /// from the local device. The extension path is the top directory of a
    /// specific extension where its manifest file lives.
    HRESULT AddExtension([in] LPCWSTR extensionPath, [in] ICoreWebView2AddExtensionCompletedHandler* handler);
    /// Gets the Extensions for the Profile.
    HRESULT GetExtensions([in] ICoreWebView2GetExtensionsCompletedHandler* handler);
}

/// Provides a set of properties for managing an Extension, which includes
/// an ID, name, and whether it is enabled or not, and the ability to Remove
/// the Extension, and enable or disable it.
[uuid(BCFC3E36-1BAD-4009-BFFC-A372C469F6BA), object, pointer_default(unique)]
interface ICoreWebView2Extension : IUnknown {
    /// This is the Extension's ID.
    /// The caller must free the returned string with `CoTaskMemFree`.  See
    /// [API Conventions](/microsoft-edge/webview2/concepts/win32-api-conventions#strings).
    [propget] HRESULT Id([out, retval] LPWSTR* value);
    /// This is the Extension's name.
    /// The caller must free the returned string with `CoTaskMemFree`.  See
    /// [API Conventions](/microsoft-edge/webview2/concepts/win32-api-conventions#strings).
    [propget] HRESULT Name([out, retval] LPWSTR* value);
    /// Removes the Extension from the WebView2 Profile while the app is running.
    HRESULT Remove([in] ICoreWebView2RemoveCompletedHandler* handler);
    /// If isEnabled is true then the Extension is enabled and running in WebView instances.
    /// If it is false then the Extension is disabled and not running in WebView instances.
    [propget] HRESULT IsEnabled([out, retval] BOOL* value);
    /// Sets whether the Extension is enabled or disabled based on isEnabled.
    HRESULT SetIsEnabled([in] BOOL isEnabled, [in] ICoreWebView2SetEnabledCompletedHandler* handler);
}

/// Provides a set of properties for managing Extension Lists. This
/// includes the number of Extensions in the list, and the ability
/// to get an Extension from the list at a particular index.
[uuid(59251055-F2F1-448F-A096-F996FB9ACBE2), object, pointer_default(unique)]
interface ICoreWebView2ExtensionList : IUnknown {
    /// The number of Extensions in the list.
    [propget] HRESULT Count([out, retval] UINT* count);
    /// Gets the Extension located in the Extension List at the given index.
    HRESULT GetValueAtIndex([in] UINT index, [out, retval] ICoreWebView2Extension** extension);
}

// Maybe should be named EnvironmentGetExtensionsCompletedHandler.
/// The caller implements this interface to receive the result of
/// getting the Extensions.
[uuid(2A05565F-DE9D-46E3-AAF7-B866966828F4), object, pointer_default(unique)]
interface ICoreWebView2GetExtensionsCompletedHandler : IUnknown {
    HRESULT Invoke([in] HRESULT errorCode, [in] ICoreWebView2ExtensionList* extensionList);
}

// Maybe should be named EnvironmentAddExtensionCompletedHandler.
/// The caller implements this interface to receive the result
/// of loading an Extension.
[uuid(E81499FE-8BC6-4BA6-BAD7-D21FFA4C3266), object, pointer_default(unique)]
interface ICoreWebView2AddExtensionCompletedHandler : IUnknown {
    HRESULT Invoke([in] HRESULT errorCode, [in] ICoreWebView2Extension* extension);
}

/// The caller implements this interface to receive the result of removing
/// the Extension from the Profile.
[uuid(BD73ED6B-08A3-4A57-9DD0-2903067632B4), object, pointer_default(unique)]
interface ICoreWebView2RemoveCompletedHandler : IUnknown {
    HRESULT Invoke([in] HRESULT errorCode);
}

/// The caller implements this interface to receive the result of setting the
/// Extension as enabled or disabled. If enabled, the Extension is running in WebView
/// instances. If disabled, the Extension is not running in WebView instances.
[uuid(0A5F7098-2265-49C7-BA60-5A511E80805A), object, pointer_default(unique)]
interface ICoreWebView2SetEnabledCompletedHandler : IUnknown {
    HRESULT Invoke([in] HRESULT errorCode);
}
```

## .NET and WinRT

```c#
namespace Microsoft.Web.WebView2.Core
{
    
    // ...
    runtimeclass CoreWebView2ExtensionList;
    runtimeclass CoreWebView2Extension;
    
    // ...
    runtimeclass CoreWebView2EnvironmentOptions
    {
        // ...
        Boolean AreExtensionsEnabled { get; set; };
    }

    // ...
    runtimeclass CoreWebView2Profile
    {
        // ...
        Windows.Foundation.IAsyncOperation<CoreWebView2Extension> AddExtensionAsync(String extensionPath);
        Windows.Foundation.IAsyncOperation<CoreWebView2ExtensionList> GetExtensionsAsync();
    }

    runtimeclass CoreWebView2ExtensionList
    {
        UInt32 Count { get; };
        CoreWebView2Extension GetValueAtIndex(UInt32 index);
    }

    runtimeclass CoreWebView2Extension
    {
        String Id { get; };
        String Name { get; };
        Boolean IsEnabled { get; };
        Windows.Foundation.IAsyncAction RemoveAsync();
        Windows.Foundation.IAsyncAction SetEnabledAsync(Boolean IsEnabled);
    }

    // ...
}
```