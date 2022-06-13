# Debug Swift debugging with LLDB

Presenters:
- Adrian Prantl, Debugger Compiler Integration

Learn advanced LLDB use cases suitable for things like:

- Integrating a 3rd-party framework
- Debugging CI or build server artifacts
- Working with custom build systems
- Building software for other developers
- Learning more about LLDB

Example is a text-based adventure written in Swift that runs in Terminal.

Find a bug, set a breakpoint, start debugging:

- First, verify the command was read correctly. 
- Attempting to use the debugger failed. Possible issue with framework - can't view source code even though explicitly downloaded debug build.

What does LLDB need in order to show source code? When the compiler compiles a function, it generates machine code & leaves breadcrumbs for the debugger so an address in the executable can be mapped to a source file and line number and vice versa. These breadcrumbs are called debug info.

Debug info is stored in object files. Debug info can be linked into .dSYM bundles. The debug info linker is called dsymutil. LLDB uses Spotlight to locate .dSYM bundles, so it's quite flexible where on disk they are.

For the example, verify LLDB has found the dSYM for the framework. We can do this with the `image list` command. If LLDB finds the dSYM for the framework, can use `image lookup` to get more info about the current address.

In the example, the build path for the source code points to where the path is on the build server, not where it is on the local machine. We can use LLDB redirect the path. Could type in the command directly, or define a per-project LLDB init file.

Now, when using the breakpoint, we can view source code to better understand/diagnose the issue.

## Source Path Issues

### Remap Source Paths

- LLDB can remap source paths using: `settings set target.source-map prefix new`
- Alternately, each dSYM has a bundle that contains an XML .plist file, where you can put a path prefix remapping dictionary; XML `<UUID>.plist` in `.dSYM` bundle - `DBGSourcePathRemapping` dictionary

### Source Path Canonicalization

To avoid having to define one remap prefix per machine, we can instruct the compiler to canonicalize source paths before putting them into the debug info. Use `-debug-prefix-map` option:

Clang:
`-fdebug-prefix-map $PWD=/BUILDROOT`

Swift:
`-debug-prefix-map $PWD=/BUILDROOT`

## New Command: Swift Healthcheck

Xcode 14 introduces a new `swift-healthcheck` command. It gives us access to a log of the Swift expression evaluator configuration. It's a good first step for figuring out if a module import failed.

It provides a path to the log. When you view the log, the end of the log tells you where potential problems occur.

## Takeaways

- LLDB can access source code using debug info linked into .dSYM bundles. You can check for a dSYM for a framework using `image list` and get more details about the current address using `image lookup`
- LLDB has a robust built-in help option (how to access?)
- To define a per-project LLDB init file, go to: Project -> Scheme -> Edit Scheme, or option+click into Play button
- The console equivalent of the Xcode variable view is `frame variable` or `v` command
- New command `swift-healthcheck` gives access to a log where you may get additional details about issues.
