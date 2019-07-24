---
description: More advanced scenarios with concurrency and asynchrony in C++/WinRT.
title: More advanced concurrency and asynchrony with C++/WinRT
ms.date: 07/23/2019
ms.topic: article
keywords: windows 10, uwp, standard, c++, cpp, winrt, projection, concurrency, async, asynchronous, asynchrony
ms.localizationpriority: medium
---

# More advanced concurrency and asynchrony with C++/WinRT

This topic describes more advanced scenarios with concurrency and asynchrony in C++/WinRT.

For an introduction to this subject, first read [Concurrency and asynchronous operations](concurrency.md).

## Offloading work onto the Windows thread pool

A coroutine is a function like any other in that a caller is blocked until a function returns execution to it. And, the first opportunity for a coroutine to return is the first `co_await`, `co_return`, or `co_yield`.

So, before you do compute-bound work in a coroutine, you need to return execution to the caller (in other words, introduce a suspension point) so that the caller isn't blocked. If you're not already doing that by `co_await`-ing some other operation, then you can `co_await` the [**winrt::resume_background**](/uwp/cpp-ref-for-winrt/resume-background) function. That returns control to the caller, and then immediately resumes execution on a thread pool thread.

The thread pool being used in the implementation is the low-level [Windows thread pool](https://docs.microsoft.com/windows/desktop/ProcThread/thread-pool-api), so it's optimially efficient.

```cppwinrt
IAsyncOperation<uint32_t> DoWorkOnThreadPoolAsync()
{
    co_await winrt::resume_background(); // Return control; resume on thread pool.

    uint32_t result;
    for (uint32_t y = 0; y < height; ++y)
    for (uint32_t x = 0; x < width; ++x)
    {
        // Do compute-bound work here.
    }
    co_return result;
}
```

## Programming with thread affinity in mind

This scenario expands on the previous one. You offload some work onto the thread pool, but then you want to display progress in the user interface (UI).

```cppwinrt
IAsyncAction DoWorkAsync(TextBlock textblock)
{
    co_await winrt::resume_background();
    // Do compute-bound work here.

    textblock.Text(L"Done!"); // Error: TextBlock has thread affinity.
}
```

The code above throws a [**winrt::hresult_wrong_thread**](/uwp/cpp-ref-for-winrt/error-handling/hresult-wrong-thread) exception, because a **TextBlock** must be updated from the thread that created it, which is the UI thread. One solution is to capture the thread context within which our coroutine was originally called. To do that, instantiate a [**winrt::apartment_context**](/uwp/cpp-ref-for-winrt/apartment-context) object, do background work, and then `co_await` the **apartment_context** to switch back to the calling context.

```cppwinrt
IAsyncAction DoWorkAsync(TextBlock textblock)
{
    winrt::apartment_context ui_thread; // Capture calling context.

    co_await winrt::resume_background();
    // Do compute-bound work here.

    co_await ui_thread; // Switch back to calling context.

    textblock.Text(L"Done!"); // Ok if we really were called from the UI thread.
}
```

As long as the coroutine above is called from the UI thread that created the **TextBlock**, then this technique works. There will be many cases in your app where you're certain of that.

For a more general solution to updating UI, which covers cases where you're uncertain about the calling thread, you can `co_await` the [**winrt::resume_foreground**](/uwp/cpp-ref-for-winrt/resume-foreground) function to switch to a specific foreground thread. In the code example below, we specify the foreground thread by passing the dispatcher object associated with the **TextBlock** (by accessing its [**Dispatcher**](/uwp/api/windows.ui.xaml.dependencyobject.dispatcher#Windows_UI_Xaml_DependencyObject_Dispatcher) property). The implementation of **winrt::resume_foreground** calls [**CoreDispatcher.RunAsync**](/uwp/api/windows.ui.core.coredispatcher.runasync) on that dispatcher object to execute the work that comes after it in the coroutine.

```cppwinrt
IAsyncAction DoWorkAsync(TextBlock textblock)
{
    co_await winrt::resume_background();
    // Do compute-bound work here.

    co_await winrt::resume_foreground(textblock.Dispatcher()); // Switch to the foreground thread associated with textblock.

    textblock.Text(L"Done!"); // Guaranteed to work.
}
```

## Execution contexts, resuming, and switching in a coroutine

Broadly speaking, after a suspension point in a coroutine, the original thread of execution may go away and resumption may occur on any thread (in other words, any thread may call the **Completed** method for the async operation).

But if you `co_await` any of the four Windows Runtime asynchronous operation types (**IAsyncXxx**), then C++/WinRT captures the calling context at the point you `co_await`. And it ensures that you're still on that context when the continuation resumes. C++/WinRT does this by checking whether you're already on the calling context and, if not, switching to it. If you were on a single-threaded apartment (STA) thread before `co_await`, then you'll be on the same one afterward; if you were on a multi-threaded apartment (MTA) thread before `co_await`, then you'll be on one afterward.

```cppwinrt
IAsyncAction ProcessFeedAsync()
{
    Uri rssFeedUri{ L"https://blogs.windows.com/feed" };
    SyndicationClient syndicationClient;

    // The thread context at this point is captured...
    SyndicationFeed syndicationFeed{ co_await syndicationClient.RetrieveFeedAsync(rssFeedUri) };
    // ...and is restored at this point.
}
```

The reason you can rely on this behavior is because C++/WinRT provides code to adapt those Windows Runtime asynchronous operation types to the C++ coroutine language support (these pieces of code are called wait adapters). The remaining awaitable types in C++/WinRT are simply thread pool wrappers and/or helpers; so they complete on the thread pool.

```cppwinrt
using namespace std::chrono;
IAsyncOperation<int> return_123_after_5s()
{
    // No matter what the thread context is at this point...
    co_await 5s;
    // ...we're on the thread pool at this point.
    co_return 123;
}
```

If you `co_await` some other type&mdash;even within a C++/WinRT coroutine implementation&mdash;then another library provides the adapters, and you'll need to understand what those adapters do in terms of resumption and contexts.

To keep context switches down to a minimum, you can use some of the techniques that we've already seen in this topic. Let's see some illustrations of doing that. In this next pseudo-code example, we show the outline of an event handler that calls a Windows Runtime API to load an image, drops onto a background thread to process that image, and then returns to the UI thread to display the image in the UI.

```cppwinrt
IAsyncAction MainPage::ClickHandler(IInspectable /* sender */, RoutedEventArgs /* args */)
{
    // We begin in the UI context.

    // Call StorageFile::OpenAsync to load an image file.

    // The call to OpenAsync occurred on a background thread, but C++/WinRT has restored us to the UI thread by this point.

    co_await winrt::resume_background();

    // We're now on a background thread.

    // Process the image.

    co_await winrt::resume_foreground(this->Dispatcher());

    // We're back on MainPage's UI thread.

    // Display the image in the UI.
}
```

For this scenario, there's a little bit of ineffiency around the call to **StorageFile::OpenAsync**. There's a necessary context switch to a background thread (so that the handler can return execution to the caller), on resumption after which C++/WinRT restores the UI thread context. But, in this case, it's not necessary to be on the UI thread until we're about to update the UI. The more Windows Runtime APIs we call *before* our call to **winrt::resume_background**, the more unnecessary back-and-forth context switches we incur. The solution is not to call *any* Windows Runtime APIs before then. Move them all after the **winrt::resume_background**.

```cppwinrt
IAsyncAction MainPage::ClickHandler(IInspectable /* sender */, RoutedEventArgs /* args */)
{
    // We begin in the UI context.

    co_await winrt::resume_background();

    // We're now on a background thread.

    // Call StorageFile::OpenAsync to load an image file.

    // Process the image.

    co_await winrt::resume_foreground(this->Dispatcher());

    // We're back on MainPage's UI thread.

    // Display the image in the UI.
}
```

If you want to do something more advanced, then you could write your own await adapters. For example, if you want a `co_await` to resume on the same thread that the async action completes on (so, there's no context switch), then you could begin by writing await adapters similar to the ones shown below.

> [!NOTE]
> The code example below is provided for educational purposes only; it's to get you started understanding how await adapters work. If you want to use this technique in your own codebase, then we recommend that you develop and test your own await adapter struct(s). For example, you could write **complete_on_any**, **complete_on_current**, and **complete_on(dispatcher)**. Also consider making them templates that take the **IAsyncXxx** type as a template parameter.

```cppwinrt
struct no_switch
{
    no_switch(Windows::Foundation::IAsyncAction const& async) : m_async(async)
    {
    }

    bool await_ready() const
    {
        return m_async.Status() == Windows::Foundation::AsyncStatus::Completed;
    }

    void await_suspend(std::experimental::coroutine_handle<> handle) const
    {
        m_async.Completed([handle](Windows::Foundation::IAsyncAction const& /* asyncInfo */, Windows::Foundation::AsyncStatus const& /* asyncStatus */)
        {
            handle();
        });
    }

    auto await_resume() const
    {
        return m_async.GetResults();
    }

private:
    Windows::Foundation::IAsyncAction const& m_async;
};
```

To understand how to use the **no_switch** await adapters, you'll first need to know that when the C++ compiler encounters a `co_await` expression it looks for functions called **await_ready**, **await_suspend**, and **await_resume**. The C++/WinRT library provides those functions so that you get reasonable behavior by default, like this.

```cppwinrt
IAsyncAction async{ ProcessFeedAsync() };
co_await async;
```

To use the **no_switch** await adapters, just change the type of that `co_await` expression from **IAsyncXxx** to **no_switch**, like this.

```cppwinrt
IAsyncAction async{ ProcessFeedAsync() };
co_await static_cast<no_switch>(async);
```

Then, instead of looking for the three **await_xxx** functions that match **IAsyncXxx**, the C++ compiler looks for functions that match **no_switch**.

## Canceling an asychronous operation, and cancellation callbacks

The Windows Runtime's features for asynchronous programming allow you to cancel an in-flight asynchronous action or operation. Here's an example that calls [**StorageFolder::GetFilesAsync**](/uwp/api/windows.storage.storagefolder.getfilesasync) to retrieve a potentially large collection of files, and it stores the resulting asynchronous operation object in a data member. The user has the option to cancel the operation.

```cppwinrt
// MainPage.xaml
...
<Button x:Name="workButton" Click="OnWork">Work</Button>
<Button x:Name="cancelButton" Click="OnCancel">Cancel</Button>
...

// MainPage.h
...
#include <winrt/Windows.Foundation.Collections.h>
#include <winrt/Windows.Storage.Search.h>

using namespace winrt;
using namespace Windows::Foundation;
using namespace Windows::Foundation::Collections;
using namespace Windows::Storage;
using namespace Windows::Storage::Search;
using namespace Windows::UI::Xaml;
...
struct MainPage : MainPageT<MainPage>
{
    MainPage()
    {
        InitializeComponent();
    }

    IAsyncAction OnWork(IInspectable /* sender */, RoutedEventArgs /* args */)
    {
        workButton().Content(winrt::box_value(L"Working..."));

        // Enable the Pictures Library capability in the app manifest file.
        StorageFolder picturesLibrary{ KnownFolders::PicturesLibrary() };

        m_async = picturesLibrary.GetFilesAsync(CommonFileQuery::OrderByDate, 0, 1000);

        IVectorView<StorageFile> filesInFolder{ co_await m_async };

        workButton().Content(box_value(L"Done!"));

        // Process the files in some way.
    }

    void OnCancel(IInspectable const& /* sender */, RoutedEventArgs const& /* args */)
    {
        if (m_async.Status() != AsyncStatus::Completed)
        {
            m_async.Cancel();
            workButton().Content(winrt::box_value(L"Canceled"));
        }
    }

private:
    IAsyncOperation<::IVectorView<StorageFile>> m_async;
};
...
```

For the implementation side of cancellation, let's begin with a simple example.

```cppwinrt
// main.cpp
#include <iostream>
#include <winrt/Windows.Foundation.h>

using namespace winrt;
using namespace Windows::Foundation;
using namespace std::chrono_literals;

IAsyncAction ImplicitCancellationAsync()
{
    while (true)
    {
        std::cout << "ImplicitCancellationAsync: do some work for 1 second" << std::endl;
        co_await 1s;
    }
}

IAsyncAction MainCoroutineAsync()
{
    auto implicit_cancellation{ ImplicitCancellationAsync() };
    co_await 3s;
    implicit_cancellation.Cancel();
}

int main()
{
    winrt::init_apartment();
    MainCoroutineAsync().get();
}
```

If you run the example above, then you'll see **ImplicitCancellationAsync** print one message per second for three seconds, after which time it automatically terminates as a result of being canceled. This works because, on encountering a `co_await` expression, a coroutine checks whether it has been cancelled. If it has, then it short-circuits out; and if it hasn't, then it suspends as normal.

Cancellation can, of course, happen while the coroutine is suspended. Only when the coroutine resumes, or hits another `co_await`, will it check for cancellation. The issue is one of potentially too-coarse-grained latency in responding to cancellation.

So, another option is to explicitly poll for cancellation from within your coroutine. Update the example above with the code in the listing below. In this new example, **ExplicitCancellationAsync** retrieves the object returned by the [**winrt::get_cancellation_token**](/uwp/cpp-ref-for-winrt/get-cancellation-token) function, and uses it to periodically check whether the coroutine has been canceled. As long as it's not canceled, the coroutine loops indefinitely; once it is canceled, the loop and the function exit normally. The outcome is the same as the previous example, but here exiting happens explicitly, and under control.

```cppwinrt
IAsyncAction ExplicitCancellationAsync()
{
    auto cancellation_token{ co_await winrt::get_cancellation_token() };

    while (!cancellation_token())
    {
        std::cout << "ExplicitCancellationAsync: do some work for 1 second" << std::endl;
        co_await 1s;
    }
}

IAsyncAction MainCoroutineAsync()
{
    auto explicit_cancellation{ ExplicitCancellationAsync() };
    co_await 3s;
    explicit_cancellation.Cancel();
}
...
```

Waiting on **winrt::get_cancellation_token** retrieves a cancellation token with knowledge of the **IAsyncAction** that the coroutine is producing on your behalf. You can use the function call operator on that token to query the cancellation state&mdash;essentially polling for cancellation. If you're performing some compute-bound operation, or iterating through a large collection, then this is a reasonable technique.

### Register a cancellation callback

The Windows Runtime's cancellation doesn't automatically flow to other asynchronous objects. But&mdash;introduced in version 10.0.17763.0 (Windows 10, version 1809) of the Windows SDK&mdash;you can register a cancellation callback. This is a pre-emptive hook by which cancellation can be propagated, and makes it possible to integrate with existing concurrency libraries.

In this next code example, **NestedCoroutineAsync** does the work, but it has no special cancellation logic in it. **CancellationPropagatorAsync** is essentially a wrapper on the nested coroutine; the wrapper forwards cancellation pre-emptively.

```cppwinrt
// main.cpp
#include <iostream>
#include <winrt/Windows.Foundation.h>

using namespace winrt;
using namespace Windows::Foundation;
using namespace std::chrono_literals;

IAsyncAction NestedCoroutineAsync()
{
    while (true)
    {
        std::cout << "NestedCoroutineAsync: do some work for 1 second" << std::endl;
        co_await 1s;
    }
}

IAsyncAction CancellationPropagatorAsync()
{
    auto cancellation_token{ co_await winrt::get_cancellation_token() };
    auto nested_coroutine{ NestedCoroutineAsync() };

    cancellation_token.callback([=]
    {
        nested_coroutine.Cancel();
    });

    co_await nested_coroutine;
}

IAsyncAction MainCoroutineAsync()
{
    auto cancellation_propagator{ CancellationPropagatorAsync() };
    co_await 3s;
    cancellation_propagator.Cancel();
}

int main()
{
    winrt::init_apartment();
    MainCoroutineAsync().get();
}
```

**CancellationPropagatorAsync** registers a lambda function for its own cancellation callback, and then it awaits (it suspends) until the nested work completes. When or if **CancellationPropagatorAsync** is canceled, it propagates the cancellation to the nested coroutine. There's no need to poll for cancellation; nor is cancellation blocked indefinitely. This mechanism is flexible enough for you to use it to interop with a coroutine or concurrency library that knows nothing of C++/WinRT.

## Reporting progress

If your coroutine returns either [**IAsyncActionWithProgress**](/uwp/api/windows.foundation.iasyncactionwithprogress_tprogress_), or [**IAsyncOperationWithProgress**](/uwp/api/windows.foundation.iasyncoperationwithprogress_tresult_tprogress_), then you can retrieve the object returned by the [**winrt::get_progress_token**](/uwp/cpp-ref-for-winrt/get-progress-token) function, and use it to report progress back to a progress handler. Here's a code example.

```cppwinrt
// main.cpp
#include <iostream>
#include <winrt/Windows.Foundation.h>

using namespace winrt;
using namespace Windows::Foundation;
using namespace std::chrono_literals;

IAsyncOperationWithProgress<double, double> CalcPiTo5DPs()
{
    auto progress{ co_await winrt::get_progress_token() };

    co_await 1s;
    double pi_so_far{ 3.1 };
    progress(0.2);

    co_await 1s;
    pi_so_far += 4.e-2;
    progress(0.4);

    co_await 1s;
    pi_so_far += 1.e-3;
    progress(0.6);

    co_await 1s;
    pi_so_far += 5.e-4;
    progress(0.8);

    co_await 1s;
    pi_so_far += 9.e-5;
    progress(1.0);
    co_return pi_so_far;
}

IAsyncAction DoMath()
{
    auto async_op_with_progress{ CalcPiTo5DPs() };
    async_op_with_progress.Progress([](auto const& /* sender */, double progress)
    {
        std::wcout << L"CalcPiTo5DPs() reports progress: " << progress << std::endl;
    });
    double pi{ co_await async_op_with_progress };
    std::wcout << L"CalcPiTo5DPs() is complete !" << std::endl;
    std::wcout << L"Pi is approx.: " << pi << std::endl;
}

int main()
{
    winrt::init_apartment();
    DoMath().get();
}
```

> [!NOTE]
> It's not correct to implement more than one *completion handler* for an asynchronous action or operation. You can have either a single delegate for its completed event, or you can `co_await` it. If you have both, then the second will fail. Either one of the following two kinds of completion handlers is appropriate; not both for the same async object.

```cppwinrt
auto async_op_with_progress{ CalcPiTo5DPs() };
async_op_with_progress.Completed([](auto const& sender, AsyncStatus /* status */)
{
    double pi{ sender.GetResults() };
});
```

```cppwinrt
auto async_op_with_progress{ CalcPiTo5DPs() };
double pi{ co_await async_op_with_progress };
```

For more info about completion handlers, see [Delegate types for asynchronous actions and operations](handle-events.md#delegate-types-for-asynchronous-actions-and-operations).

## Fire and forget

Sometimes, you have a task that can be done concurrently with other work, and you don't need to wait for that task to complete (no other work depends on it), nor do you need it to return a value. In that case, you can fire off the task and forget it. You can do that by writing a coroutine whose return type is [**winrt::fire_and_forget**](/uwp/cpp-ref-for-winrt/fire-and-forget) (instead of one of the Windows Runtime asynchronous operation types, or **concurrency::task**).

```cppwinrt
// main.cpp
#include <winrt/Windows.Foundation.h>

using namespace winrt;
using namespace std::chrono_literals;

winrt::fire_and_forget CompleteInFiveSeconds()
{
    co_await 5s;
}

int main()
{
    winrt::init_apartment();
    CompleteInFiveSeconds();
    // Do other work here.
}
```

**winrt::fire_and_forget** is also useful as the return type of your event handler when you need to perform asynchronous operations in it. Here's an example (also see [Strong and weak references in C++/WinRT](/windows/uwp/cpp-and-winrt-apis/weak-references#safely-accessing-the-this-pointer-in-a-class-member-coroutine)).

```cppwinrt
winrt::fire_and_forget MyClass::MyMediaBinder_OnBinding(MediaBinder const&, MediaBindingEventArgs args)
{
    auto lifetime{ get_strong() }; // Prevent *this* from prematurely being destructed.
    auto ensure_completion{ unique_deferral(args.GetDeferral()) }; // Take a deferral, and ensure that we complete it.

    auto file{ co_await StorageFile::GetFileFromApplicationUriAsync(Uri(L"ms-appx:///video_file.mp4")) };
    args.SetStorageFile(file);

    // The destructor of unique_deferral completes the deferral here.
}
```

The first argument (the *sender*) is left unnamed, because we never use it. For that reason we're safe to leave it as a reference. But observe that *args* is passed by value. See the [Parameter-passing](concurrency.md#parameter-passing) section above.

## Awaiting a kernel handle

C++/WinRT provides a **resume_on_signal** class, which you can use to suspend until a kernel event is signaled. You're responsible for ensuring that the handle remains valid until your `co_await resume_on_signal(h)` returns. **resume_on_signal** itself can't do that for you, because you may have lost the handle even before the **resume_on_signal** starts, as in this first example.

```cppwinrt
IAsyncAction Async(HANDLE event)
{
    co_await DoWorkAsync();
    co_await resume_on_signal(event); // The incoming handle is not valid here.
}
```

The incoming **HANDLE** is valid only until the function returns, and this function (which is a coroutine) returns at the first suspension point (the first `co_await` in this case). While awaiting **DoWorkAsync**, control has returned to the caller, the calling frame has gone out of scope, and you no longer know whether the handle will be valid when your coroutine resumes.

Technically, our coroutine is receiving its parameters by value, as it should (see [Parameter-passing](concurrency.md#parameter-passing) above). But in this case we need to go a step further so that we're following the *spirit* of that guidance (rather than just the letter). We need to pass a strong reference (in other words, ownership) along with the handle. Here's how.

```cppwinrt
IAsyncAction Async(winrt::handle event)
{
    co_await DoWorkAsync();
    co_await resume_on_signal(event); // The incoming handle *is* not valid here.
}
```

Passing a [**winrt::handle**](/uwp/cpp-ref-for-winrt/handle) by value provides ownership semantics, which ensures that the kernel handle remains valid for the lifetime of the coroutine.

Here's how you might call that coroutine.

```cppwinrt
namespace
{
    winrt::handle duplicate(winrt::handle const& other, DWORD access)
    {
        winrt::handle result;
        if (other)
        {
            winrt::check_bool(::DuplicateHandle(::GetCurrentProcess(),
		        other.get(), ::GetCurrentProcess(), result.put(), access, FALSE, 0));
        }
        return result;
    }

    winrt::handle make_manual_reset_event(bool initialState = false)
    {
        winrt::handle event{ ::CreateEvent(nullptr, true, initialState, nullptr) };
        winrt::check_bool(static_cast<bool>(event));
        return event;
    }
}

IAsyncAction SampleCaller()
{
    handle event{ make_manual_reset_event() };
    auto async{ Async(duplicate(event)) };

    ::SetEvent(event.get());
    event.close(); // Our handle is closed, but Async still has a valid handle.

    co_await async; // Will wake up when *event* is signaled.
}
```

## Important APIs
* [IAsyncAction interface](/uwp/api/windows.foundation.iasyncaction)
* [IAsyncActionWithProgress&lt;TProgress&gt; interface](/uwp/api/windows.foundation.iasyncactionwithprogress_tprogress_)
* [IAsyncOperation&lt;TResult&gt; interface](/uwp/api/windows.foundation.iasyncoperation_tresult_)
* [IAsyncOperationWithProgress&lt;TResult, TProgress&gt; interface](/uwp/api/windows.foundation.iasyncoperationwithprogress_tresult_tprogress_)
* [SyndicationClient::RetrieveFeedAsync method](/uwp/api/windows.web.syndication.syndicationclient.retrievefeedasync)
* [winrt::get_cancellation_token](/uwp/cpp-ref-for-winrt/get-cancellation-token)
* [winrt::get_progress_token](/uwp/cpp-ref-for-winrt/get-progress-token)
* [winrt::fire_and_forget](/uwp/cpp-ref-for-winrt/fire-and-forget)

## Related topics
* [Concurrency and asynchronous operations](concurrency.md)
* [Handle events by using delegates in C++/WinRT](handle-events.md)