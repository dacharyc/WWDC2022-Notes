# What's New in SwiftUI

Presenters: 
- Nick Teissler, SwiftUI Engineer
- Franck Ndame Mpouli, SwiftUI Engineer

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

## SwiftUI Control Enhancements

### Forms

Settings interfaces are control heavy. New settings designed to present controls in a consistent and well-organized fashion. Different types of controls grouped into sections. `.grouped` formStyle.

```swift
Form {
    Section {
        LabeledContent("Location", value: address)
        DatePicker("Date", selection: $date)
    }
    Section("Vibe") {
        Picker("Accent color", selection: $accent) { ... }
        Picker("Color scheme", selection: $scheme) { ... }
        Toggle(isOn: $extraGuests) {
            Text("Allow extra guests")
            Text("The more the merrier!")
        }
    }
}
.formStyle(.grouped)
```

Controls consistently align labels, content grouped under headings. 

#### Labeled content

Can display content, and allow selecting the text. Can also wrap any kind of view.

```swift
Section {
    LabeledContent("Location") {
        AddressView(location)
    }
    DatePicker("Date", selection: $date)
}
```

### Controls

Other new control features:
- New `.axis` parameter for TextFields to expand them vertically
- `.lineLimit` modifier supports more advanced behaviors
- `MultiDatePicker` control - supports non-contiguous date selection
- Mixed-state controls
  - Disclosure group with toggles; each inner toggle has an individual binding, while the aggregate toggle takes a collection
  - Displays a mixed state if the values don't all match

```swift
DisclosureGroup {
    Toggle("Balloons", isOn: $includeBalloons)
    Toggle("Confetti", isOn: $includeConfetti)
    Toggle("Inflatables", isOn: $includeInflatables)
    Toggle("Party Horns", isOn: $includePartyHorns)
} label: {
    Toggle("All Decorations", isOn: [
        $includeBalloons,
        $includeConfetti,
        $includeInflatables,
        $includePartyHorns
    ])
}
```

- Button-style toggles
- Steppers
- Accessibility quick actions

### Tables

Tables now supported on iPad OS. Uses same Tables API introduced last year.

```swift
@State private var attendees: [Attendee]

var body: some View {
    Table(attendees) {
        TableColumn("Name") { attendee in
            AttendeeRow(attendee)
        }
        TableColumn("City", value: \.city)
        TableColumn("Status") { attendee in
            StatusRow(attendee)
        }
    }
}
```

There is a new selection-based `contentMenu` modifier:

```swift
@State private var attendees: [Attendee]
@State private var selection: Set<Attendee.ID>

var body: some View {
    Table(attendees, selection: $selection) {
        ...
    }
    .contextMenu(forSelectionType: Attendee.ID.self) { selection in
        if selection.isEmpty {
            Button("New Invitation") { addInvitation() }
        } else if selection.count = 1 {
            Button("Mark as VIP") { markVIPs(selection) }
        } else {
            Button("Mark as VIPs") { markVIPs(selection) }
        }
    }
}
```

Users can customize and reorder toolbars:

```swift
Table(attendees, selection: $selection) {
    ...
}
.toolbar(id: "invitations") {
    ToolbarItem(id: "new", placement: .secondaryAction) {
        Button(action: sendNewInvitation) {
            Label("New Invitation", systemImage: "envelope")
        }
    }
}
```

## Search improvements

New this year, `.searchable` can support tokenized inputs and suggestions:

```swift
@State private var queryText: String
@State private var queryTokens: [InvitationToken]

var body: some View {
    InvitationsContentView()
    .searchable(text: $queryText, tokens: $queryTokens) { token in
        Label(token.displayName, systemImage: token.systemImage)
    }
}
```

SwiftUI now supports search scopes, which appear in a scope bar:

```swift
@State private var queryText: String
@State private var queryTokens: [InvitationToken]
@State private var scope: AttendanceScope

var body: some View {
    InvitationsContentView()
    .searchable(text: $queryText, tokens: $queryTokens, scope: $scope) { token in
        Label(token.displayName, systemImage: token.systemImage)
    } scopes: {
        Text("In Person").tag(AttendanceScope.inPerson)
        Text("Online").tag(AttendanceScope.online)
    }
}
```

## Sharing

### Photos

New `PhotosPicker` view:

```swift
@State private var selection: PhotosPickerItem?

var body: some View {
    Gallery(...)
    .toolbar {
        PhotosPicker(
            selection: $selection, matching: .images
        ) {
            Label("Pick a photo", systemImage: "plus.app")
        }
    }
}
```

### Sharing

New ShareLink API:

```swift
Gallery(...)
.toolbar {
    ShareLink(
        item: image,
        preview: SharePreview("Birthday Effects")
    )
}
```

### Transferable

New protocol. A Swift-first declarative way to describe how types are transferred across applications. This powers SwiftUI features, like drag-and-drop.

New DropDestination API, which accepts a payload type - in this case, just an image:

```swift
@State private var image: Image?

var body: some View {
    Gallery(..)
    .dropDestination(
        payloadType: Image.self
    ) { receivedImages, location in
        image = receivedImages.first
        return !receivedImages.isEmpty
    }
}
```

Many standard types, such as string and image, already conform to Transferrable:

- String
- Data
- Attributed String
- Image
- ...

You can implement it in your own custom types.

```swift
struct BirthdayFilter: Codable {
    ...
}

extension BirthdayFilter: Transferable {
    static var representation: some TransferRepresentation {
        CodableRepresentation(contentType: .birthdayFilter)
    }
}
```

## Graphics and Layout

### ShapeStyles has new APIs

- Color has a new style that adds a subtle gradient: `.backgroundStyle(.blue.gradient)`
- New Shadow modifier: 

```swift
.foregroundStyle(
    .white.shadow(.drop(radius: 1, y: 1.5))
)
```

### Text animation improvements

### Layouts

- Grid is a new container view that arranges views in a 2D grid
- Enable automatic alignments across rows and columns

```swift
struct VipDetailView: View {
    var body: some View {
        Grid {
            GridRow {
                NameHeadline()
                .gridCellColumns(2)
            }
            GridRow {
                CalendarIcon()
                SymbolGrid()
            }
        }
    }
}
```

New Layout protocol lets you customize Layout abstractions. Using the new `AnyLayout` type to switch between Grid and custom layout:

```swift
var body: some View {
    let grid = Grid(
        alignment: .center,
        horizontalSpacing: 0,
        veritcalSpacing: 0
    )

    let layout = model.usesGridLayout ? AnyLayout(grid) : AnyLayout(scattered)

    layout {
        ForEach(rows) { row in
            GridRow {
                ForEach(row.columns) {
                    CellView()
                }
            }
        }
    }
}
```

## SwiftUI Previews

- Preview variants for multiple appearances, type sizes, and orientations without writing any configuration code
- Previews runs in Live mode by default


## Related Sessions

- Hello Swift Charts
- Raise the bar
- The SwiftUI cookbook for navigation
- What's new in Xcode
- Use Xcode to develop a multiplatform app
- Bring multiple windows to your SwiftUI app
- Meet Transferable
- Compose custom layouts with SwiftUI
