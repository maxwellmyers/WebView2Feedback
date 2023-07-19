# Background

JavaScript's Screen Capture API `GetDisplayMedia` allows developers to get a video stream of a 
user's tabs, windows, or desktop. This API is available in WebView2, but the current default UI 
has some problems that we need to fix to make sure that hybrid apps using WebView2 have a more 
seamless web/native experience. This includes removing the tab column in the UI, replacing 
default strings and icons that do not match in WV2, and potentially having the ability to 
customize the UI itself. 

These apps also expect that the screen capture dialog has an event before the UI is shown to give 
the host app an opportunity to block or allow UI from showing at all.

In this document we describe the updated API. We'd appreciate your feedback.

# Description

We propose introducing the `ScreenCaptureRequested` event. This event will be raised whenever 
the WebView2 and/or iframe corresponding to the CoreWebView2Frame or any of its descendant iframes 
requests permission to use the Screen Capture API before the UI is shown. 

For convenience of the end developer, by default we plan to raise
`ScreenCaptureRequested` on both `CoreWebView2Frame` and `CoreWebView2`. The
`CoreWebView2Frame` event handlers will be invoked first,
before the `CoreWebView2` event handlers. If `Handled` is set true as part of
the `CoreWebView2Frame` event handlers, then the `ScreenCaptureRequested` event
will not be raised on the `CoreWebView2`, and its event handlers will not be
invoked.

In the case of a nested iframe requesting permission, we will raise the event
off of the top level iframe.

# Examples
## C++: Registering Screen Requested Handler on CoreWebView2
``` cpp
wil::com_ptr<ICoreWebView2> m_webviewEventSource;
EventRegistrationToken m_screenCaptureRequestedToken = {};

m_webviewEventSource->add_ScreenCaptureRequested(
    Callback<ICoreWebView2ScreenCaptureRequestedEventHandler>(
        [this](ICoreWebView2* sender, ICoreWebView2ScreenCaptureRequestedEventArgs* args)
          -> HRESULT 
            { 
                // Get Frame Info
                wil::com_ptr<ICoreWebView2FrameInfo> frameInfo;
                CHECK_FAILURE(args->get_FrameInfo(&frameInfo));

                // Frame Source
                wil::unique_cotaskmem_string frameSource;
                CHECK_FAILURE(frameInfo->get_Source(&frameSource));

                // If the host app wants to cancel the request for a specific source
                if (frameSource = "https://developer.microsoft.com/en-us/microsoft-edge/webview2/") 
                {
                    CHECK_FAILURE(args->put_Cancel(TRUE));
                }

                return S_OK;
            })
            .Get(),
  &m_screenCaptureRequestedToken);

```
## C++: Registering Screen Requested Handler on CoreWebView2Frame
``` cpp
wil::com_ptr<ICoreWebView2> m_webview;
auto webview4 = m_webview.try_query<ICoreWebView2_4>();
if (webview4) 
{
    EventRegistrationToken m_frameCreatedToken = {};
    EventRegistrationToken m_screenCaptureRequestedToken = {};

    CHECK_FAILURE(webview4->add_FrameCreated(
        Callback<ICoreWebView2FrameCreatedEventHandler>(
            [this](ICoreWebView2* sender, ICoreWebView2FrameCreatedEventArgs* args) 
              -> HRESULT
            {
                wil::com_ptr<ICoreWebView2Frame> webviewFrame;
                CHECK_FAILURE(args->get_Frame(&webviewFrame));

                auto webviewFrame3 = webviewFrame.try_query<ICoreWebView2Frame3>();
                if (webviewFrame3)
                {
                    CHECK_FAILURE(webviewFrame3->add_ScreenCaptureRequested(
                        Callback<ICoreWebView3FrameScreenCaptureRequestedEventHandler>(
                            [this](ICoreWebView3Frame* sender,
                               ICoreWebView2ScreenCaptureRequestedEventArgs* args) -> HRESULT
                            {

                                    // Get Frame Info
                                    wil::com_ptr<ICoreWebView2FrameInfo> frameInfo;
                                    CHECK_FAILURE(args->get_FrameInfo(&frameInfo));

                                    // Frame Source
                                    wil::unique_cotaskmem_string frameSource;
                                    CHECK_FAILURE(frameInfo->get_Source(&frameSource));

                                    // If the host app wants to cancel the request for a specific source
                                    if (frameSource = "https://developer.microsoft.com/en-us/microsoft-edge/webview2/") 
                                    {
                                        CHECK_FAILURE(args->put_Cancel(TRUE));
                                    }

                                    // Let CoreWebView2 handler know the event is already handled

                                    // In the case of an iframe requesting permission to use Screen Capture, the default 
                                    // behavior is to first raise the ScreenCaptureRequested event off of the 
                                    // CoreWebView2Frame and invoke it's handlers, and then raise the event off the 
                                    // CoreWebView2 and invoke it's handlers. However, If we set Handled to true on the
                                    // CoreWebView2Frame event handler, then we will not raise the
                                    // ScreenCaptureRequested event off the CoreWebView2.

                                    CHECK_FAILURE(args->put_Handled(true));
                                    return S_OK;
                            })
                            .Get(),
                        &m_screenCaptureRequestedToken));
                }
                return S_OK;
        }).Get(),
        &m_FrameCreatedToken));
}
```

## C#: Registering Screen Capture Requested Handler
```c#
private WebView2 webView;
webView.CoreWebView2.ScreenCaptureRequested += (sender, screenCaptureArgs) =>
{   
    // Get Frame Info
    wil::com_ptr<ICoreWebView2FrameInfo> frameInfo;
    frameInfo = screenCaptureArgs.frameInfo

    // Frame Source
    string frameSource;
    frameSource = frameInfo.Source;

    // If the host app wants to cancel the request from a specific frame
    if (frameSource == "https://developer.microsoft.com/en-us/microsoft-edge/webview2/")
    {
        screenCaptureArgs.Cancel = true;
    }
}
```


## C#: Registering IFrame Screen Capture Requested Handler
```c#
private WebView2 webView;

m_webview.CoreWebView2.FrameCreated += (sender, frameCreatedArgs) =>
{
    // Checking for runtime support of CoreWebView2Frame.ScreenCaptureRequested
    try
    {
        frameCreatedArgs.Frame.ScreenCaptureRequested += (frameSender, screenCaptureArgs) =>
        {
            // Get Frame Info
            wil::com_ptr<ICoreWebView2FrameInfo> frameInfo;
            frameInfo = screenCaptureArgs.frameInfo

            // Frame Source
            string frameSource;
            frameSource = frameInfo.Source;

            // If the host app wants to cancel the request from a specific source
            if (frameSource == "https://developer.microsoft.com/en-us/microsoft-edge/webview2/")
            {
                screenCaptureArgs.Cancel = true;
            }

            // Let CoreWebView2 handler know the event is already handled

            // In the case of an iframe requesting permission to use Screen Capture, the default 
            // behavior is to first raise the ScreenCaptureRequested event off of the 
            // CoreWebView2Frame and invoke it's handlers, and then raise the event off the 
            // CoreWebView2 and invoke it's handlers. However, If we set Handled to true on the
            // CoreWebView2Frame event handler, then we will not raise the
            // ScreenCaptureRequested event off the CoreWebView2.
            // 
            // NotImplementedException could be thrown if underlying runtime did not
            // implement Handled. However, we only run this code after checking if
            // CoreWebView2Frame.ScreenCaptureRequested exists, and both exist together,
            // so it would not be a problem.
            args.Handled = true;
        };
    }
    catch (NotImplementedException exception)
    {
        // If the runtime support is not there we probably want this
        // to be a no-op.
    }
};
```

# API Details
## C++
```
interface ICoreWebView2_20;
interface ICoreWebView2ScreenCaptureRequestedEventArgs;
interface ICoreWebView2ScreenCaptureRequestedEventHandler;

interface ICoreWebView2Frame3;
interface ICoreWebView2FrameScreenCaptureRequestedEventHandler;
interface ICoreWebView2FrameScreenCaptureRequestedEventArgs;

/// This interface is an extension of `ICoreWebView2` that supports the ScreenCaptureRequested event. 
// MSOWNERS: stevenwei@microsoft.com 
[uuid(accc0e97-fa2d-4a8d-ad61-cc9ae57a1825), object, pointer_default(unique)] 
interface ICoreWebView2_20 : IUnknown { 
  /// Add an event handler for the `ScreenCaptureRequested` event. 
  /// `ScreenCaptureRequested` event is raised when the Screen Capture API is requested by the user using getDisplayMedia(). 
  HRESULT add_ScreenCaptureRequested( 
      [in] ICoreWebView2ScreenCaptureRequestedEventHandler* eventHandler, 
      [out] EventRegistrationToken* token); 
  /// Remove an event handler previously added with `add_ScreenCaptureRequested`. 
  // MSOWNERS: stevenwei@microsoft.com 
  HRESULT remove_ScreenCaptureRequested( 
      [in] EventRegistrationToken token); 
}   
/// Receives `ScreenCaptureRequested` events. 
// MSOWNERS: stevenwei@microsoft.com 
[uuid(9b5bbea1-4a58-4567-8b42-8781d3986cb4), object, pointer_default(unique)] 
interface ICoreWebView2ScreenCaptureRequestedEventHandler : IUnknown { 
  /// Called to provide the event args when a screen capture is requested on 
  /// a WebView element. 
  HRESULT Invoke( 
      [in] ICoreWebView2* sender, 
      [in] ICoreWebView2ScreenCaptureRequestedEventArgs* args); 
} 
/// Event args for the `ScreenCaptureRequested` event. 
// MSOWNERS: stevenwei@microsoft.com 
[uuid(a1d309ee-c03f-11eb-8529-0242ac130003), object, pointer_default(unique)] 
interface ICoreWebView2ScreenCaptureRequestedEventArgs : IUnknown { 
  /// The associated frame information that requests the screen capture  
  /// permission. This can be used to grab the frame source, name, frameId,  
  /// and parent frame information. 
  [propget] HRESULT FrameInfo([out, retval] ICoreWebView2FrameInfo**  
     frameInfo); 
  /// By default, both the `ScreenCaptureRequested` event handlers on the 
  /// `CoreWebView2Frame` and the `CoreWebView2` will be invoked, with the 
  /// `CoreWebView2Frame` event handlers invoked first. The host may 
  /// set this flag to `TRUE` within the `CoreWebView2Frame` event handlers 
  /// to prevent the remaining `CoreWebView2` event handlers from being  
  /// invoked. 
  /// 
  /// If a deferral is taken on the event args, then you must synchronously 
  /// set `Handled` to TRUE prior to taking your deferral to prevent the 
  /// `CoreWebView2`s event handlers from being invoked. 
  [propget] HRESULT Handled([out, retval] BOOL* handled); 
  /// Sets the `Handled` property. 
  [propput] HRESULT Handled([in] BOOL handled); 
  /// The host may set this flag to cancel the screen capture. If canceled, 
  /// the screen capture UI is not displayed regardless of the 
  /// `Handled` property. 
  /// On the script side, it will return with a NotAllowedError as Permission denied.
  [propget] HRESULT Cancel([out, retval] BOOL* cancel); 
  /// Sets the `Cancel` property. 
  [propput] HRESULT Cancel([in] BOOL cancel); 
  /// Returns an `ICoreWebView2Deferral` object. Use this deferral to 
  /// defer the decision to show the Screen Capture UI. 
  /// 
  /// Returns an `ICoreWebView2Deferral` object.    
  HRESULT GetDeferral([out, retval] ICoreWebView2Deferral** deferral); 
} 
/// This is an extension of the ICoreWebView2Frame interface that supports ScreenCaptureRequested 
// MSOWNERS: stevenwei@microsoft.com 
[uuid(12885cda-9caa-4793-9c38-f15827dbab1f), object, pointer_default(unique)] 
interface ICoreWebView2Frame3 : IUnknown { 
  /// Add an event handler for the `ScreenCaptureRequested` event. 
  /// `ScreenCaptureRequested is raised when content in an iframe or any of its 
  /// descendant iframes requests permission to use the Screen Capture  
  /// API from getDisplayMedia() 
  /// 
  /// This relates to the `ScreenCaptureRequested` event on the  
  /// CoreWebView2`. 
  /// Both these events will be raised in the case of an iframe requesting 
  /// permission. The `CoreWebView2Frame`'s event handlers will be invoked 
  /// before the event handlers on the `CoreWebView2`. If the `Handled`  
  /// property of the `ScreenCaptureRequestedEventArgs` is set to TRUE  
  /// within the`CoreWebView2Frame` event handler, then the event will not  
  /// be raised on the `CoreWebView2`, and its event handlers will not be   
  /// invoked. 
  /// 
  HRESULT add_ScreenCaptureRequested( 
      [in] ICoreWebView2FrameScreenCaptureRequestedEventHandler* handler, 
      [out] EventRegistrationToken* token); 
  
  /// Remove an event handler previously added with  
  /// `add_ScreenCaptureRequested` 
  HRESULT remove_ScreenCaptureRequested( 
      [in] EventRegistrationToken token); 
} 
 
/// Receives `ScreenCaptureRequested` events for iframes. 
// MSOWNERS: stevenwei@microsoft.com 
[uuid(c07ac75c-2105-4bb8-9c57-21b6ed8fb381), object, pointer_default(unique)] 
interface ICoreWebView2FrameScreenCaptureRequestedEventHandler : IUnknown { 
  /// Provides the event args for the corresponding event. 
  HRESULT Invoke( 
      [in] ICoreWebView2Frame* sender, 
      [in] ICoreWebView2ScreenCaptureRequestedEventArgs * args); 
} 

```

## C#
```c#
namespace Microsoft.Web.WebView2.Core
{
    runtimeclass CoreWebView2ScreenCaptureRequestedEventArgs
    {

        [interface_name("Microsoft.Web.WebView2.Core.ICoreWebView2ScreenCaptureRequestedEventArgs")]
        {
            CoreWebView2FrameInfo FrameInfo { get; };
            Boolean Cancel { get; set; };
            Boolean Handled { get; set; };
        }

    }

    runtimeclass CoreWebView2
    {
        // ...
        event Windows.Foundation.TypedEventHandler<CoreWebView2, 
            CoreWebView2ScreenCaptureRequestedEventArgs> ScreenCaptureRequested;
    };

    runtimeclass CoreWebView2Frame
    {
        // ...
        // ICoreWebView2Frame3 members
        event Windows.Foundation.TypedEventHandler<CoreWebView2Frame, 
            CoreWebView2ScreenCaptureRequestedEventArgs> ScreenCaptureRequested;
    }

}
```

# UI Changes

Having the column for specific tabs/WV2s doesn’t make sense in the vast majority of cases, so we 
will remove that column entirely. Apps that want to offer the ability to select a specific Tab/WV2 
will need to use the full API when we have it available to construct their own UI. 


Next, the URL of the WV2 is used in a handful of locations that we should replace by default: 

“Choose what to share with <url>” in the main dialog. 

“<url> is sharing a window” in the sharing bar when sharing a window. 

All of these should be replaced with “this app”. 


When the sharing bar is open, the icon for it is a WV2 icon, not the host app’s icon. We should 
use the host app’s icon (or no icon?) 