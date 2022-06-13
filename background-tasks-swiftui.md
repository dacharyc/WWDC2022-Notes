# Efficiency awaits: Background tasks in SwiftUI

Presenters:
- John Gallagher, watchOS Software Engineer

New SwiftUI API for handling background tasks using Swift Concurrency

This API is shared across: watchOS, iOS, tvOS, Mac Catalyst, and widgets, including iOS apps running on the Mac.

Example app (Stormy):
- Sends a notification on stormy days reminding users to take a photo
- Uploads the photo in the background

## How the notification only gets sent on stormy days using background tasks

When the app first launches in the foreground by the user, we schedule a background app refresh task for noon. When the user leaves the app and the app is suspended, the system knows to wake the application again in the background at the scheduled time.

In the background, we need to figure out if it is stormy outside, and if it is, send a notification.

First, we need to check with a weather service for the current weather. With the `URLSession` scheduled for background, the application can suspend and wait for the network request to complete. Once that completes, our app is given background runtime again with a new `URLSession` background task. The app can use the results of the weather query to decide whether to send a notification to the user.

When the system sends the app refresh background task, we make our network API call in the background. If it comes back in time, we can send a notification to the user. If it's taking too long, the system signals to the app that it is running low on background runtime for the current task, giving us a chance to handle the situation gracefully. 

**IMPORTANT:** If apps do not signal they have completed their background work before runtime expires, the app may be quit by the system, and throttled for future background task requests.

For this example, we can make our network request as a background network request, which allows us to complete our app refresh task immediately and get woken again for additional background runtime when the network request completes.

### BackgroundTask API

Create a basic application, and a function to schedule background app refresh for noon tomorrow. We create a date representing noon tomorrow, and then create a background app request refresh for noon tomorrow to submit to the system scheduler. This is what tells the system to wake our app tomorrow at noon:

```swift
struct StormyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

func scheduleAppRefresh() {
    let today = Calendar.current.startOfDay(for: .now)
    let tomorrow = Calendar.current.date(byAdding: .day, value: 1, to: today)!
    let noonComponent = DateComponents(hour: 12)
    let noon = Calendar.current.date(byAdding: noonComponent, to: tomorrow)

    let request = BGAppRefreshTaskRequest(identifier: "StormyNoon")
    request.earliestBeginDate = noon
    try? BGTaskScheduler.shared.submit(request)
}
```

We want to call this function when the user first opens the application and requests daily storm notifications at noon. We can register a handler corresponding to the background task we scheduled by using the new background task scene modifier:

```swift
struct StormyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .backgroundTask(.appRefresh("StormyNoon")) {
            scheduleAppRefresh()
            if await isStormy() {
                await notifyForPhoto()
            }
        }
    }
}
```

Many APIS across Apple Platforms support Swift Concurrency for async operations, such as sending notifications:

```swift
struct StormyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .backgroundTask(.appRefresh("StormyNoon")) {
            scheduleAppRefresh()
            if await isStormy() {
                await notifyForPhoto()
            }
        }
    }
}

func notifyForPhoto() async {
    let notificationRequest = photoUploadNotification()
    do {
        try await UNUserNotificationCenter.current().add(notificationRequest)
    } catch {
        print("Notification failed with error: \(String(describing: error))")
    }
}
```

URLSession has adopted Swift Concurrency and has a method for downloading data from the network that can be awaited from async contexts:

```swift
struct StormyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .backgroundTask(.appRefresh("StormyNoon")) {
            scheduleAppRefresh()
            if await isStormy() {
                await notifyForPhoto()
            }
        }
    }
}

func isStormy() async -> Bool {
    let session = URLSession.shared

    let request = URLRequest(url: WEATHER_URL)

    let response = try? await session.data(for: request)

    if let data = response {
        return parseWeather(data)
    }
    return false
}
```

What if it can't complete the request before the runtime expires? We want to make sure we set up the URLSession as a background session and to ensure it will send launch events to the app using a URLSession background task. Instead of a shared URLSession, we should create from a background configuration with the `sessionSendsLaunchEvents` property set to true:

```swift
func isStormy() async -> Bool {
    let config = URLSessionConfiguration.background(withIdentifier: "isStormy")
    config.sessionLaunchEvents = true
    let session = URLSession(configuration: config)

    let request = URLRequest(url: WEATHER_URL)

    let response = try? await session.data(for: request)

    if let data = response {
        return parseWeather(data)
    }
    return false
}
```

When our background task runtime is expiring, the system will cancel the async task that is running the closure provided to the background task modifier. The network request will also be canceled when our background runtime is expiring. To respond and handle that cancellation, we can use the `withTaskCancellationHandler` function built in to Swift Concurrency. Instead of awaiting the result directly, we place our download int oa `withTaskCancellationHandler` call and await this as well:

```swift
func isStormy() async -> Bool {
    let config = URLSessionConfiguration.background(withIdentifier: "isStormy")
    config.sessionLaunchEvents = true
    let session = URLSession(configuration: config)

    let request = URLRequest(url: WEATHER_URL)

    let response = await withTaskCancellationHandler {
        try? await session.data(for: request)
    } onCancel: {
        let task = session.downloadTask(with: request)
        task.resume()
    }

    if let data = response {
        return parseWeather(data)
    }
    return false
}
```

Finally, we need to make sure the app is set up to handle a launch from a background URLSession. Use the background task modifier again, but this time with URLSession task type:

```swift
struct StormyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .backgroundTask(.appRefresh("StormyNoon")) {
            scheduleAppRefresh()
            if await isStormy() {
                await notifyForPhoto()
            }
        }
        .backgroundTask(.urlSession("isStormy")) {
            // ...
        }
    }
}
```
