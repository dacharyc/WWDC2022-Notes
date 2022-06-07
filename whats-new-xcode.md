# What's New in Xcode

Presenters:
- Lisa Xiao, Engineering Manager
- Jonathon Mah, Engineering Manager

We'll take a look at what's new in Xcode 14, including:
- Source editing
- SwiftUI Previews
- Multiplatform applications
- TestFlight feedback
- Performance improvements

Xcode 14 is 30% smaller. Downloads and installs faster. Download additional platforms and simulators on-demand.

## SwiftUI Previews

- Preview canvas is live by default, changes are live as you make them
- New control to create additional variants of preview without writing new code

## Code completion

- Library of SF Symbols
- Initializer and Codable definitions - Xcode pre-populates them now
- Choose from initializer args when using code completion

## Jump to definition

- Redesigned definition list highlights what's different about each of the results so you can choose the one you want

## Callers

- Command-click to open the callers of the method
- Callers list shows different files and functions that contain calls to the method, along with a preview of each call site

## Regex is a first-class feature

- The compiler checks regex like it does with other code
- Xcode can highlight typos

## Xcode 14 shows definitions containing the visible code

Even when the code is scrolled out of view - i.e. the definition "sticks" to the top of the screen

## Build Performance Improvements

- Increased parallelism in build taks
- 2x faster linking
- Builds projects up to 25% faster

Visualization tool to help you identify unexpectedly long tasks and bottlenecks - build timeline

## Parallel Testing

- Same improvements as build performance
- Could improve test execution time up to 30%

## Faster Notarizing

- Up to 4x improvement notarizing macOS apps

## Multiplatform support

- Use a single target to support multiple platforms

## Memory debugging improvements

- Zero in on shortest paths from root objects to unexpectedly live objects
- See all reference paths in and out of an object
- More thorough explanation of leaks to gauge the total weight of your objects

## Swift Package Plugins

- Packages can integrate plugins that process your code in place, like linters and formatters
- You can invoke them directly from the project navigator
- Integrate build tools that generate code or process resources while building

## Localization improvements

## Run destination chooser

- Prioritizes recent choices
- Filter

## Organizer 

- 2 new reports: Feedback and Hangs

## Icons

- New Single Size feature to tell Xcode to automatically create different sized icons from the one provided

## Related Sessions

- Author Fast and Reliable Tests for Xcode Cloud
- Use Xcode to build a multiplatform app
- Meet Swift Package Plugins
- Create Swift Package Plugins
- Building global apps: Localization by example
- Track down hangs with Xcode and on-device detection
