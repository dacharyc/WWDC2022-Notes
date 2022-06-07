# What's New in SwiftUI

Presenters: 
- Nick Teissler, SwiftUI Engineer
- Franck Ndame Mpouli, SwiftUI Engineer

(Paused at 13:18)

## Intro

- Introduce you to Swift Charts
- Show off SwiftUI's data-driven, strongly-typed model for navigation and new window techniques
- Suite of new controls and deeper customization of existing controls
- Transferable protocol
- New graphics APIs and advanced layout APIs

## Swift Charts

Declarative framework for building beautiful, state-driven charts.

- Different types of charts, as easy as changing from BarMark to LineMark
- Can add modifiers to do things like assign colors, add points
- Can use SwiftUI Views within a chart
- Swift Charts handles localization, Dark Mode, and Dynamic Type automatically
- Works across all platforms

## Navigation and windows

SwiftUI already supports the most common navigation patterns, such as:
- Push-and-pop navigation stacks
- Expansive, detail-rich split views
- Powerful multi-window experiences

Updates for all three patterns this year

### Stacks

New container view this year: `NavigationStack` - supports push-and-pop style navigation

```swift
NavigationStack {
    List(foodItems) { item in
        NavigationLink {
            FoodDetailView(item: item)
        } label: {
            FoodRow(food: item)
        }
    }
    .navigationTitle("Party Food")
}
```

Data-driven NavigationStack:

```swift
NavigationStack {
    List(foodItems) { item in
        NavigationLink(value: item) {
            Label(item.title, image: item.iconName)
        }
    }
    .navigationTitle("Party Food")
    .navigationDestination(for: FoodItem.self) { item in
        FoodDetailView(item: item)
    }
}
```

With data-driven `NavigationStack` you can use state to track navigation:

```swift
// NavigationPath

var foodItems: [FoodItem]
@State private var selectedFoodItems: [FoodItem] = []

var body: some View {
    NavigationStack(path: $selectedFodItems) {
        List(foodItems) { item in
            NavigationLink(value: item) {
                Label(item.title, image: item.iconName)
            }
        }
        .navigationTitle("Party Food")
        .navigationDestination(for: FoodItem.self) { item in
            FoodDetailView(item: item, path: $selectedFoodItems)
        }
    }
}
```

With this, we can easily add a button to return to the first item:

```swift
// FoodDetailView.body

Button("Back to First Item") {
    // Remove all but first element
    selectedFodItems.removeSubrange(1...)
}
```

### Split views for multi-column navigation

NavigationSplitView is for multicolumn navigation. `NavigationSplitView` can declare two- and three-column layouts

```swift
NavigationSplitView {
    List(PartyTask.allCases, selection: $selectedTask) { task in
        NavigationLink(value: $0) {
            TaskLabel(task: $0)
        }
        .listItemTint(task.color)
    }
} detail: {
    switch selectedTask {
    case .food:
        FoodOverview()
    case .music:
        MusicOverview()
    }
}
```

```swift
NavigationSplitView {
    List(PartyTask.allCases, selection: $selectedTask) { task in
        NavigationLink(value: task) {
            TaskLabel(task: task)
        }
        .listItemTint(task.color)
    }
} detail: {
    selectedTask.flatMap { $0.color } ?? .white
}
```

### Scene APIs/Windows

New Window API - can open from menu or add keyboard shortcuts to open

```swift
@main
struct PartyPlanner: App {
    var body: some Scene {
        WindowGroup("Party Planner") {
            PartyPlannerHome()
        }

        Window("Party Budget", id: "budget") {
            Text("Budget View")
        }
        .keyboardShortcut("0")
    }
}
```

Toolbar button with action that shows the window:

```swift
struct DetailView: View {
    @Environment(\.openWindow) var openWindow

    var body: some View {
        Text("Detail View")
            .toolbar {
                Button {
                    openWindow(id: "budget")
                } label: {
                    Image(systemName: "dollarsign")
                }
            }
    }
}
```

Can programmatically open new windows, and there is a whole suite of new window customization options.

#### Resizable Sheets

For a multi-platform app, can use resizable sheets:

```swift
struct PartyPlannerHome: View {
    @State private var selectedTask: PartyTask?
    @State private var presented: Bool = false

    var body: some View {
        NavigationSplitView {
            List(PartyTask.allCases, selection: $selectedTask) { task in
                NavigationLink(value: task) {
                    TaskLabel(task: task)
                }
                .listItemTint(task.color)
            }
        } detail: {
            if case .food = selectedTask {
                FoodsListView()
            } else {
                selectedTask.flatMap { $0.color } ?? .white
            }
        }
        .sheet(isPresented: $presented) {
            Text("Budget View")
                .presentationDetents([.height(250), .medium])
                .presentationDragIndicator(.visible)
        }
    }
}
```

#### Menu-Bar Extras

```swift
@main
struct PartyPlanner: App {
    var body: some Scene {
        Window("Party Budget", id: "budget") {
            Text("Budget View")
        }

        MenuBarExtra("Bulletin Board", systemImage: "quote.bubble") {
            BulletinBoard()
        }
        .menuBarExtraStyle(.window)
    }
}
```

## Related Sessions

- Hello Swift Charts
- Raise the bar
- The SwiftUI cookbook for navigation
- What's new in Xcode
- Use Xcode to develop a multiplatform app
- Bring multiple windows to your SwiftUI app
