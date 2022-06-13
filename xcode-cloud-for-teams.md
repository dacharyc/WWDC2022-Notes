# Deep dive into Xcode Cloud for teams

Presenters: 
- John-Robert Cross, Software Development Engineer
- Jo Lynn, Designer

## Integrate Xcode Cloud with existing tools

- Webhooks allow easy integration with tools and services that support them.
- You can connect a webhook in App Store Connect by telling Xcode Cloud whar URL to send the webhook to, and you should start seeing those webhooks come in right away.

You can also use APIs to:

- Create build dashboards
- Extract build artifacts
- Integrate build information into your existing software 

### Integrate build information into existing software

Example uses a very simple Swift On Server-based service to integrate an issue tracker with Xcode Cloud. To speed up development, it uses the Vapor framework.

High-level process:
- Webhook comes from Xcode Cloud to our server
- We read the webhook and check if the commit message written by the committer has a certain string in it which maps to an issue in our tracker
- If it does, we hit the Xcode Cloud API to gather more information about the build
- We can use that information to construct a comment we can post onto our issue tracker that contains the information we're interested in
- We'll call an API on the issue tracker, which saves the message against our issue

The [Xcode Cloud API Documentation](https://developer.apple.com/documentation/appstoreconnectapi/xcode_cloud_workflows_and_builds) lives under the App Store Connect API. You need to set up authentication tokens for the App Store Connect API.

(The session then goes through a _very_ fast list of docs and steps you need to set this up)

## Manage your dependencies

Xcode Cloud supports Swift Package Manager without requiring any additional configuration, if the package's repository is publicly accessible. You can also make Xcode Cloud work with third-party dependency managesr like Cocoapods and Carthage, but you'll have to do extra work by using custom build scripts.

## Follow best practices


### Use Xcode Cloud with SwiftLint

Integrate SwiftLint with Xcode Cloud using a custom build script. We want Xcode Cloud to run the SwiftLint tool after it clones our source code from the team's primary repository.

In the Project navigator, add a `post_clone` script in a `ci_scripts` folder. The Xcode Cloud build environment includes Homebrew, and that's what we're using to install SwiftLint.

```sh
#!/bin/zsh

# ci_post_clone.sh
# Food Truck

# Use Homebrew to install the SwiftLint tool
brew install swiftlint

# Run SwiftLint 
swiftlint $CI_WORKSPACE
```

The script executes within the `ci_scripts` directory, so we have to tell SwiftLint to run within the `ci_workspace` environment variable, which points to our repository.

## Other takeways

- You can Disable-Re-enable workflows
- You can restrict editing access to workflows
- You can reuse the same workflow for multiple start conditions. You might want to do this for external deployment; changes to main branch, release branch, or a scheduled build, you might want to archive, test, and use TestFlight for internal testing.
- Everything you can do in Xcode to set up Xcode Cloud stuff is also available via App Store Connect

## Related Sessions

- Customize your advanced Xcode Cloud workflows: WWDC21
