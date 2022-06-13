# Bring multiple windows to your SwiftUI app

Presenters:
- Jeff Robertson, SwiftUI Engineer

Apps in SwiftUI are composed of Scenes and Views. Scenes commonly represent their contents with a window onscreen.

In the example, there is a single window group that shows a reading list in a platform-appropriate way. On platforms that support multiple windows, sucha s iPadOS and macOS, a scene can represent itself with several such windows.

## Scene Basics

Existing scene types:

- WindowGroup: provides a way to build data-driven applications across all of Apple's platforms
- DocumentGroup: Lets you build document-based apps on iOS and macOS
- Settings: Defines an interface for representing in-app settings values on macOS

We're extending the list of scenes with two new additions:

- Window: represents a single, unique window on all platforms
- MenuBarExtra: renders as a persistent control in the system menu bar (macOS only)

As with all scenes, you can use these as a standalone scene or composed with other scenes in your app.

### Window

Unlike WindowGroup, the [Window](https://developer.apple.com/documentation/swiftui/window) scene only ever represents its contents in a single, unique window instance. This is useful when the contents of your scene represent some global app state that would not necessarily fit well with WIndowGroups' multi-window presentation style on macOS and iPadOS, such as a game.

```swift
import SwiftUI

@main
struct BookClub: App {
    @StateObject private var store = ReadingListStore()

    var body: some Scene {
        WindowGroup {
            ReadingListViewer(store: store)
        }
        Window("Activity", id: "activity") {
            ReadingActivity(store: store)
        }
    }
}
```

### MenuBarExtra

[MenuBarExtra](https://developer.apple.com/documentation/swiftui/menubarextra) is a new macOS-only scene type. Rather than rendering its contents in a window, it places its label in the menu bar and shows its contents in either a menu or window which is anchored to the label. It's usable as long as its associated app is running, regardless of whether that app is frontmost.

```swift
import SwiftUI

@main
struct UtilityApp: App {
    var body: some Scene {
        MenuBarExtra("Utility App", systemImage: "hammer") {
            AppMenu()
        }
    }
}
```

Supports two rendering styles:

- The default style shows contents in a menu which pulls down from the menu bar
- Another style presents its contents in a chromeless window anchored to the menu bar

## Auxiliary Scenes

As an example of a Window scene, add an auxiliary scene to represent reading activity over time. The Activity window's data is derived from overall app state, so a window scene is an ideal choice for it. Opening multiple windows with the same state would not fit well with our design.

```swift
import SwiftUI

@main
struct BookClub: App {
    @StateObject private var store = ReadingListStore()

    var body: some Scene {
        WindowGroup {
            ReadingListViewer(store: store)
        }
        // Title is used as a label for a menu item added to the Window menu
        // When selecting this item, the scene's window is opened if it isn't already
        // If it's already open, it is brought to the front.
        Window("Activity", id: "activity") {
            ReadingActivity(store: store)
        }
    }
}
```

## Scene Navigation

The BookClub app has a context menu that can be invoked for any book in the Content List pane. This context menu includes a button for triggering the window presentation:

```swift
// Presenting a window

struct OpenNoteButton: View {
    var book: Book

    var body: some View {
        Button("Open in New Window") {
            openWindow(value: book.id)
        }
    }
}
```

The value must conform to both `Hashable` and `Codable`. 

Alongside our app's primary WindowGroup and auxiliary window, we'll add an additional WindowGroup for handling our book details:

```swift
struct BookClub: App {
    @StateObject private var store = ReadingListStore()

    var body: some Scene {
        WindowGroup {
            ReadingListViewer(store: store)
        }
        Window("Activity", id: "activity") {
            ReadingActivity(store: store)
        }
        WindowGroup("Book Details", for: Book.ID.self) { $bookId in
            BookDetail(id: $bookId, store: store)
        }
    }
}
```

The book details WindowGroup uses a new initializer. In addition to the title, we're noting that this group presents data for the `Book.ID` type - in our case, UUIDs. This type should match the value that we are passing to the `openWindow` action we added earlier.

When a given value is provided to the WindowGroup for presentation, SwiftUI creates a new child scene for that value, and the root content of that scene's window is defined by that value, using the group's view builder. Each unique presented value creates a new scene. The value's equality determines whether to create a new window or reuse an existing window. If it presents a value for which a window already exists, the group uses that window rather than creating a new one.

### New Environment types

SwiftUI provides several new callable types via the environment for presenting windows tied to the scenes that your app defines:

- openWindow action: present windows for either a WindowGroup or Window scene
- newDocument: supports opening new document windows for both FileDocuments and ReferenceFileDocuments
- openDocument: present windows where the contents are provided by an existing file on disk

#### OpenWindow Action

Things to note about openWindow action:

- The identifier passed to the action must match an identifier for a scene defined in your app
- openWindow action can take a presentation value, which the presented scene will use to display its contents
- The form of the action is only supported by WindowGroup, using a new initializer
- The type of the value must match against the type provided to the scene's initializer

```swift
@Environment(\.openWindow) private var openWindow
...
openWIndow(value: book.id)
...
WIndowGroup("Book Details", for: Book.ID.self) { $bookId in

}
```

#### NewDocument Action

Supports opening new document windows for both FileDocuments and ReferenceFileDocuments. This action requires that the corresponding DocumentGroup in your app is defined with an editor role.

```swift
@Environment(\.newDocument) private var newDocument
...
newDocument(TextFile())
...
DocumentGroup(newDocument: TextFile()) { file in

}
```

#### OpenDocument Action

Opens a file that exists on disk. Takes a URL to a file to open. Your app must define a DocumentGroup for presentign the window, and the document type for that group must allow for reading the type of the file at the provided URL.

```swift
@Environment(\.openDocument) private var openDocument
...
openDocument(at: URL(fileURLWithPath: pathToDocument))
...
struct TextFile: FileDocument {
    static var readableContentTypes: [UTType] { [.plainText] }
}
```

## Scene Customization

Because we've defined our app with two WindowGroup scenes - one for hte main viewer window and one for the detail windows - SwiftUI by default adds a menu item for each group in the File menu. However, the menu item for the detail window doesn't make sense in this context. We'd prefer that those windows can only be opened via the context menu added earlier.

A new scene modifier, `commandsRemoved`, allows you to modify a scene such that it no longer provides its default commands, like the one in the File menu.

```swift
struct BookClub: App {
    @StateObject private var store = ReadingListStore()

    var body: some Scene {
        WindowGroup {
            ReadingListViewer(store: store)
        }
        Window("Activity", id: "activity") {
            ReadingActivity(store: store)
        }
        WindowGroup("Book Details", for: Book.ID.self) { $bookId in
            BookDetail(id: $bookId, store: store)
        }
        .commandsRemoved()
    }
}
```

Another new scene modifier, `defaultPosition` modifier, specifies a position to use when no previous state is available. This position is relative to the screen size and places the window in the appropriate location taking into account the current locale. The different position helps differentiate the Activity window from the other viewing windows on the screen.

```swift
struct ReadingActivityScene: Scene {
   @ObservedObject var store: ReadingListStore

   var body: some Scene {
       Window("Activity", id: "activity") {
           ReadingActivity(store: store)
       }
       .defaultPosition(.topTrailing)
   }
}
```

Alongside `defaultPosition`, you can add a `defaultSize` modifier. This value gives the layout system an initial size for the window.

```swift
struct ReadingActivityScene: Scene {
   @ObservedObject var store: ReadingListStore

   var body: some Scene {
       Window("Activity", id: "activity") {
           ReadingActivity(store: store)
       }
       .defaultPosition(.topTrailing)
       .defaultSize(width: 400, height: 800)
   }
}
```

The `keyboardShortcut` modifier has been expanded to work on scene types. In our example, we'll modify the Activity window to open it with the shortcut Option-Command-0.

```swift
struct ReadingActivityScene: Scene {
   @ObservedObject var store: ReadingListStore

   var body: some Scene {
       Window("Activity", id: "activity") {
           ReadingActivity(store: store)
       }
       .defaultPosition(.topTrailing)
       .defaultSize(width: 400, height: 800)
       .keyboardShortcut("0", modifiers: [.option, .command])
   }
}
```

## Related Sessions 

- SwiftUI on iPad: Organize your interface
- SwiftUI on iPad: Add toolbars, titles, and more
