# What's new in Swift-DocC?

Presenters:
- Franklin Schrans, Software Engineer
- Ethan Kusters, Software Engineer

Improvements to Swift-DocC this year:
- Now supports App projects in addition to frameworks
- You can use it to document Objective-C and C APIs
- Publish to managed hosting services
- New navigation sidebar

## Writing documentation

- Expands to App projects

Open the Product menu, and select "Build Documentation" - Swift-DocC provides stubs you can fill in.

A good pattern is to teach how each API works individually, and then build a higher-level catalog.

Comments that you start with three slashes are visible to DocC and become text in the built documentation page.

```swift
/// A view that displays a sloth. {This text appears right under the page title for the struct SlothView}
/// {Subsequent paragraphs appear in "Overview" section in built docs}
/// This is the main view of ``SlothyApp``. {Using the double backticks turns this into a link}
/// Create a sloth view by providing a binding to a sloth.
/// {You can use Markdown code block syntax to provide example code. The following renders as a code block in docs}
/// ```swift
/// @State private var sloth: Sloth
///
/// var body: some View {
///     SlothView(sloth: $sloth)    
/// }
/// ```
struct SlothView: View {

}
```

To document an initializer:

```swift
/// Creates a view that displays the specified sloth.
/// 
/// - Parameter sloth: The sloth the user will edit. {Appears in Params list}
init(sloth: Binding<Sloth>) {

}
```

### Objective-C Documentation

- Same documentation format
- Source editor integration
- Mixed Swift + Objective-C apps and frameworks

Objective-C uses the same markdown syntax as the Swift code. If both languages are available, the built page renders with a language switcher to toggle between Swift & Obj-C

[Multi-language documentation](https://developer.apple.com/documentation/Xcode/documenting-apps-frameworks-and-packages)

### Add a documentation catalog

Add additional content and media by inserting a Documentation Catalog into the project

## Publishing documentation

New, simplified publishing workflow

When you build in Xcode, DocC produces a static bundle. You can export directly and send. In most cases, deploy by copying DocC archive into root of web server. Compatible with GitHub Pages and other stuff. For something with a `basePath` you must specify it in a DocC build setting.

### Configuring hosting base path

Default:
`url/documentation/nameOfFrameworkOrProject`
`url/tutorials/nameOfFrameworkOrProject`

To publish to a different URL at a specific base path:

Example: `https://username.github.io/repository-name/documentation/nameOfFrameworkOrProject`

You need to tell DocC what this base path is.

Build Settings -> Documentation Compiler - Options -> DocC Archive Hosting Base path

After generating, Export it.

There's a [Swift Package Manager DocC Plugin](https://apple.github.io/swift-docc-plugin/documentation/swiftdoccplugin/) you can use for automated documentation deployment. You can also still use the `xcodebuild docbuild` command that was introduced in Xcode 13.

## Web navigation

New navigation sidebar, Filter bar

## Related Sessions

- [Improve the discoverability of your Swift-DocC content](improve-discoverability-docc.md)
