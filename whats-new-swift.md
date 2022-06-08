# What's new in Swift

Presenters:
- Angela Laar, Swift Team
- Becca Royal-Gordon, Swift Team

New features in Swift 5.7. Goal to make your life as a developer easier.

- Community update
- Swift packages
- Performance improvements
- Concurrency updates
- Expressive Swift

## Community Update

Community involvement is at Swift's core. DocC & Swift.org open-sourced. Workgroup support:

- Swift on Server
- Diversity in Swift
- Swift Website (New)
- C++ Interoperability (New)

We need mentors for Diversity in Swift mentorship program. Check out Swift blog post about it. Visit swift.org/diversity to learn more about Diversity in Swift.

## Cross-Platform Support

RPMs
- Amazon Linux 2
- CentOS 7

## Swift Packages

Dropped dependency on external Unicode library, added a smaller native solution. Statically-linked standard library to make Swift suitable for restricted environments.

### TOFU

Trust on First Use. New security protocol. Compares subsequent downloads with initial fingerprint.

### Command Plugins

Command Plugins
- Generated documentation
- Reformat source code
- Generate test reports

Open-source formatters and linters.

DocC got Obj-c & C support this year. Command plugin code example for DocC:

```swift
@main struct MyPlugin: CommandPlugins {

    func performCommand(context: PluginContext, arguments: [String]) throws {
        let process = try Process.run(doccExec, arguments: doccArgs)
        process.waitUntilExit()
    }
}
```

```sh
> swift package generate-documentation
```

### Build Tool Plugins

Provide a scalable way for packages to define plugins that can implement additional build steps.

Examples:
- Source code generation
- Resource processing

Use a build tool plugin:

```swift
import PackagePlugin

@main struct MyCoolPlugin: BuildToolPlugin {
    func createBuildCommands(context: TargetBuildContext) throws -> [Command] {
        // Run some command
    }
}
```

Implement a build tool plugin:

```swift
import PackagePlugin

@main struct MyCoolPlugin: BuildToolPlugin {
    func createBuildCommands(context: TargetBuildContext) throws -> [Command] {

        let generatedSources = context.pluginWorkDirectory.appending("GeneratedSources")

        return [
            .buildCommand(
                displayName: "Running MyTool",
                executable: try context.tool(named: "mycooltool").path,
                arguments: ["create"],
                outputFilesDirectory: generatedSources)
        ]
    }
}
```

Module disambiguation allows you to distinguish between modules - i.e. rename them.

## Performance Improvements

### New Swift driver settings

Swift driver settings
- Integrated compiler
- Eager compiliation
- Eager linking

Improvements in build time range from 5% to 25%

### Faster type-checking of generics

Improve compile time for things like protocols and where clauses. In the old system, it could take a long time to type check. 

### Runtime improvements

Optimized protocol conformance checking. Protocols used to be computed every launch - now, they're cached.

## Concurrency updates

Data-race safety at the forefront of model improvements.

Back deploy to iOS 13/macOS Catalina.

Distributed actors.

### Data race safety

Actors were the first step in data race safety. But what happens when different threads want to query the information via different actors?

Distributed Actors put actors on different machines with a network between them.

```swift
distributed actor Player {
   
    var ai: PlayerBotAI?
    var gameState: GameState
    
    distributed func makeMove() -> GameMove {
        return ai.decideNextMove(given: &gameState)
    }
}
```

Then, to call the distributed actors:

```swift
func endOfRound(players: [Player]) async throws {
    // Have each of the players make their move
    for player in players {
        let move = try await player.makeMove()
    }
}
```

Distributed Actors Package
- Integrated networking layer with SwiftNIO
- Implements SWIM

### Async Algorithms Package

- Integration w/async/await
- Async sequences

### Concurrency optimizations

- Actor prioritization
- Priority-inversion avoidance (less-important work can't block higher-priority work)

### Swift concurrency instruments

Help you visualize and optimize concurrency code. Provides stats including simultaneous tasks and all tasks that have been created. Task Forest provides a graphical representation of parent-child relationships between tasks.

## Expressive Swift

### Optional unwrapping

```swift
if let mailmapURL = mailmapURL {

    mailmapLines = try String(contentsOf: mailmapURL).split(separator: "\n")
    
}
```

Optional unwrapping with long variable names

```swift
if let workingDirectoryMailmapURL = workingDirectoryMailmapURL {

    mailmapLines = try String(contentsOf: workingDirectoryMailmapURL).split(separator: "\n")
    
}
```

Unwrapping in Swift 5.7:

```swift
if let workingDirectoryMailmapURL {
  
    mailmapLines = try String(contentsOf: workingDirectoryMailmapURL).split(separator: "\n")

}

guard let workingDirectoryMailmapURL else { return }

mailmapLines = try String(contentsOf: workingDirectoryMailmapURL).split(separator: "\n")
```

### Closure improvements

Closures can now more easily/accurately infer type, even for more complex closures that have a do/catch

### Permitted pointer conversions

Swift/C-family interoperability. Swift now has a separate set of rules for calls that allow pointer conversions that would be legal in C even if they're not in Swift.

### String parsing

Regex literals

```swift
func parseLine(_ line: Substring) throws -> MailmapEntry {

    let regex = /\h*([^<#]+?)??\h*<([^>#]+)>\h*(?:#|\Z)/

    guard let match = line.prefixMatch(of: regex) else {
        throw MailmapError.badLine
    }

    return MailmapEntry(name: match.1, email: match.2)
}
```

Use words instead of symbols to create Regex w/RegexBuilder:

```swift
import RegexBuilder

let regex = Regex {
    ZeroOrMore(.horizontalWhitespace)
    Optionally {
        Capture(OneOrMore(.noneOf("<#")))
    }
        .repetitionBehavior(.reluctant)
    ZeroOrMore(.horizontalWhitespace)
    "<"
    Capture(OneOrMore(.noneOf(">#")))
    ">"
    ZeroOrMore(.horizontalWhitespace)
    ChoiceOf {
       "#"
       Anchor.endOfSubjectBeforeNewline
    }
}
```

Turn a Regex into a reusable Regex component:

```swift
struct MailmapLine: RegexComponent {
    @RegexComponentBuilder
    var regex: Regex<(Substring, Substring?, Substring)> {
        ZeroOrMore(.horizontalWhitespace)
        Optionally {
            Capture(OneOrMore(.noneOf("<#")))
        }
            .repetitionBehavior(.reluctant)
        ZeroOrMore(.horizontalWhitespace)
        "<"
        Capture(OneOrMore(.noneOf(">#")))
        ">"
        ZeroOrMore(.horizontalWhitespace)
        ChoiceOf {
           "#"
            Anchor.endOfSubjectBeforeNewline
        }
    }
}
```

Swift Regex
- Uses a custom engine written in swift
- Literal dialect based on UTS #18 w/Extensions
- Must run on an OS w/Regex engine built in: macOS 13, iOS 16, tvOS 16, watchOS 9

### Generics & Protocols

Something that conforms to Mailmap vs. a box whose contents conform to Mailmap. They look alike so it's hard to figure out which you're using. For a box whose contents conform to a type, use `any Mailmap` - i.e. `any` keyword.

Primary associated types.

`AnyCollection` is a type-erasing wrapper, where `any Collection` is a built-in language feature that does basically the same thing without the type-erasing wrapper. See if you can reimplement them with `any` types.

Use generics when possible. Swift making generics as easy to use as any types.

Write with `some` keyword as a shorthand.

## Related Sessions

- Meet Swift Package plugins
- Create Swift Package plugins
- Demystify parallelization in Xcode Builds
- Improve app size and runtime performance
- Eliminate data races using Swift concurrency
- Meet distributed actors in Swift
- Meet Swift Async Algorithms
- Visualize and optimize Swift concurrency
- Meet Swift Regex
- Swift Regex: Beyond the basics
- Embrace Swift generics
- Design protocol interfaces in Swift
