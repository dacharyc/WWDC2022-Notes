# SwiftUI Cookbook for Navigation

Presenter:
- Curt Clifton, SwiftUI Engineer

New data-driven navigation APIs

## New Navigation APIs

The existing APIs are based on links that send views that are shown in other columns or on a stack. With existing navigation, to present a link programmatically, add a binding to the link. Lots of bindings. 

### Navigation Stack

New APIs push that up to the entire container, called a NavigationStack. The path here is a collection that represent all the values pushed on the stack:

```swift
NavigationStack(path: $path) {
    NavigationLink("Details", value: value)
}
```

### Navigation Split View

Good for multi-column apps on mac or ipad. Automatically adapts to a single-column stack.

Two-column experience:

```swift
NavigationSplitView {
    RecipeCategories()
} content: {
    RecipeList()
} detail: {
    RecipeGrid()
}
```

Three-column experience:

```swift
NavigationSplitView {
    RecipeCategories()
} detail: {
    RecipeGrid()
}
```

### Navigation Link Variants

Title and view to present (old style):

```swift
NavigationLink("Show detail") {
    DetailView()
}
```

Title and value (new variant):

```swift
NavigationLink("Apple Pie", value: applePieRecipe)
```

The link's behavior depends on the context in which it appears.

## Recipes for Navigation

### Pushable Stack

Basic navigation (similar to existing navigation paradigms) - single stack

Use:
- NavigationStack
- NavigationLink (the value variant)
- .navigationDestination

```swift
struct PushableStack: View {
    @State private var path: [Recipe] = []
    @StateObject private var dataModel = DataModel()

    var body: some View {
        NavigationStack(path: $path) {
            List(Category.allCases) { category in
                Section(category.localizedName) {
                    ForEach(dataModel.recipes(in: category)) { recipe in
                        NavigationLink(recipe.name, value: recipe)
                    }
                }
            }
            .navigationTitle("Categories")
            .navigationDestination(for: Recipe.self) { recipe in
                RecipeDetail(recipe: recipe)
            }
        }
        .environmentObject(dataModel)
    }
}
```

If you need to use multiple types in a navigation stack, check out the new, type-erasing [`NaviationPath`](https://developer.apple.com/documentation/swiftui/navigationpath/) for mixed data. With NavigationStack, you can:

#### Jump to a specific destination in the stack

Add a func to jump to a specific destination

```swift
@State private var path: [Recipe] = []

var body: some View {
    NavigationStack(path: $path) { ... }
}

func showRecipeOfTheDay() {
    path = [dataModel.recipeOfTheDay]
}
```

#### Pop back to the root

Create a func to pop back to the root by removing everything from the path stack

```swift
@State private var path: [Recipe] = []

var body: some View {
    NavigationStack(path: $path) { ... }
}

func popToRoot() {
    path.removeAll()
}
```

### Multcolumn Presentation Without Stacks

For a multi-column navigation with columns showing progressively more info, use:
- NavigationSplitView
- NavigationLink
- List

#### Three-column

```swift
@State private var selectedCategory: Category?
@State private var selectedRecipe: Recipe?

var body: some View {
    NavigationSplitView {
        List(Category.allCases, selection: $selectedCategory) { category in
            NavigationLink(category.localizedName, value: category)
        }
        .navigationTitle("Categories")
    } content: {
        List(
            dataModel.recipes(in: selectedCategory),
            selection: $selectedRecipe)
        { recipe in
            NavigationLink(recipe.name, value: recipe)
        }
        .navigationTitle(selectedCategory?.localizedName ?? "Recipes")
    } detail: {
        RecipeDetail(recipe: selectedRecipe)
    }
}
```

Use a func to navigate to specific destination:

```swift
@State private var selectedCategory: Category?
@State private var selectedRecipe: Recipe?

var body: some View {
    NavigationSplitView { ... }
}

func showRecipeOfTheDay() {
    let recipe = dataModel.recipeOfTheDay
    selectedCategory = recipe.category
    selectedRecipe = recipe
}
```

### Multiple Column Presentation With Stacks

Navigate between related information

Use:
- NavigationSplitView
- NavigationStack
- NavigationLink
- .navigationDestination(for: )
- List

```swift
// Multiple columns with a stack
struct MultipleColumnsWithStack: View {
    @State private var selectedCategory: Category?
    @State private var path: [Recipe] = []
    @StateObject private var dataModel = DataModel()

    var body: some View {
        NavigationSplitView {
            List(Category.allCases, selection: $selectedCategory) { category in
                NavigationLink(category.localizedName, value: category)
            }
            .navigationTitle("Categories")
        } detail: {
            NavigationStack(path: $path) {
                RecipeGrid(category: selectedCategory)
            }
        }
        .environmentObject(dataModel)
    }
}
```

We can put a NavigationStack inside a column in the NavigationSplitView. So then:

```swift
struct RecipeGrid: View {
    @EnvironmentObject private var dataModel: DataModel
    var category: Category?

    var body: some View {
        if let category = category {
            ScrollView {
                LazyVGrid(columns: columns) {
                    ForEach(dataModel.recipes(in: category)) { recipe in
                        NavigationLink(value: recipe) {
                            RecipeTile(recipe: recipe)
                        }
                    }
                }
            }
            .navigationTitle(category.localizedName)
            // Don't attach this directly to the NavigationLink. Lazy loading in the LazyVGrid
            // means the thing might not load, *and* it repeats for every item in grid.
            .navigationDestination(for: Recipe.self) { recipe in
                RecipeDetail(recipe: recipe)
            }
        } else {
            Text("Select a category")
        }
    }

    var columns: [GridItem] { [GridItem(.adaptive(minimum: 240))] }
}
```

And for the detail view:

```swift
struct RecipeDetail: View {
    @EnvironmentObject private var dataModel: DataModel
    var recipe: Recipe

    var body: some View {
        Text("Recipe details go here")
            .navigationTitle(recipe.name)
        ForEach(recipe.related.compactMap { dataModel[$0] }) { related in
            NavigationLink(related.name, value: related)
        }
    }
}
```

Method to show recipe of the day with this navigation paradigm:

```swift
@State private var selectedCategory: Category?
@State private var path: [Recipe] = []

var body: some View {
    NavigationSplitView { ... }
}

func showRecipeOfTheDay() {
    let recipe = dataModel.recipeOfTheDay
    selectedCategory = recipe.category
    path = [recipe]
}
```

## Persistent State

To maintain people's place in the app, we need:
- Codable
- SceneStorage

To do this:

1. Move navigation state into a model type
2. Make the navigation model Codable
3. Use SceneStorage to save and restore

### Move Navigation State into a Model Type

Starting state from last recipe:

```swift
@State private var selectedCategory: Category?
@State private var path: [Recipe] = []

var body: some View {
    NavigationSplitView {
        List(Category.allCases, selection: $selectedCategory) { category in
            NavigationLink(category.localizedName, value: category)
        }
        .navigationTitle("Categories")
    } detail: {
        NavigationStack(path: $path) {
            RecipeGrid(category: selectedCategory)
        }
    }
}
```

Move navigation state into a model type. Introduce a NavigationModel class that is observable:

```swift
class NavigationModel: ObservableObject {
    @Published private var selectedCategory: Category?
    @Published private var path: [Recipe] = []
}
```

Then, introduce a state object in the view to hold an instance of the navigation model:

```swift
@StateObject private var navModel = NavigationModel()

var body: some View {
    NavigationSplitView {
        List(Category.allCases, selection: $navModel.selectedCategory) { category in
            NavigationLink(category.localizedName, value: category)
        }
        .navigationTitle("Categories")
    } detail: {
        NavigationStack(path: $navModel.path) {
            RecipeGrid(category: navModel.selectedCategory)
        }
    }
}
```

### Make the Navigation Model Codable

```swift
class NavigationModel: ObservableObject, Codable {
    @Published private var selectedCategory: Category?
    @Published private var path: [Recipe] = []

    enum CodingKeys: String, CodingKey {
        case selectedCategory
        case recipePathIds
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encodeIfPresent(selectedCategory, forKey: .selectedCategory)
        try container.encode(path.map(\.id), forKey: .recipePathIds)
    }

    required init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.selectedCategory = try container.decodeIfPresent(
            Category.self, forKey: .selectedCategory)

        // Decode recipe ID and convert it back into recipes. Use compactMap
        // to discard any values that can't be decoded to recipes, as if the recipe is deleted
        let recipePathIds = try container.decode([Recipe.ID].self, forKey: .recipePathIds)
        self.path = recipePathIds.compactMap { DataModel.shared[$0] }

        // Add a computed property for reading and writing the model as JSON data
        var jsonData: Data? { ... }
    }
}
```

We implement a custom conformance to Codable because we don't want to store the entire model value:
- This repeats information that already exists elsewhere
- If the recipe database can change independently, as in Syncing new data, the model could persist stale data

### Use SceneStorage to Save and Restore

Current state of main view:

```swift
@StateObject private var navModel = NavigationModel()

var body: some View {
    NavigationSplitView { ... }
}
```

Introduce SceneStorage to persist model:

```swift
@StateObject private var navModel = NavigationModel()
@SceneStorage("navigation") private var data: Data?

var body: some View {
    NavigationSplitView { ... }
}
```

Then add a task modifier to view:

```swift
@StateObject private var navModel = NavigationModel()
@SceneStorage("navigation") private var data: Data?

var body: some View {
    NavigationSplitView { ... }
    .task {
        if let data = data {
            navModel.jsonData = data
        }
        for await _ in navModel.objectWillChangeSequence {
            data = navModel.jsonData
        }
    }
}
```

## Tips

- Adopt the new navigation APIs ASAP
  - Old-style programmatic NavigationLink that takes a binding is deprecated in new builds
- Compose `NavigationSplitView`, `NavigationStack`, and `List`
- Put `navigationDestination` modifiers within easy reach (but not inside lazy containers)
- Start with `NavigationSplitView` when it makes sense - it automatically adapts to iPhone, and makes it easy to support additional real estate in larger platforms

## Related Sessions

- SwiftUI on iPad: Organize your interface
- Build a productivity app for Apple Watch
- Bring multiple windows to your SwiftUI app
