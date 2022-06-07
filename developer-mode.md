# Get to know developer mode

Presenter: 
- Pavlo Malynin, Security Engineer

Changes to how you develop, test, and deploy apps

## What is Developer Mode?

- A new mode for iOS 16 watchOS 9 that enables developer workflows
- Disabled by default, requires explicit enrollment
- Developer features are being abused in targeted attacks, while most people don't need to use developer features by default

Distribution and testing flows that do not require Developer Mode:

- Test Flight
- Enterprise (In-House) distribution
- App Store

Only required for local development

## When and How to Turn On?

Turn on developer mode when you need to:
- Run and install Development-signed applications
- Debug and instrument your applications
- Automate testing

How to turn on:
- Connect your device to Xcode
- Developer Mode controls live in Settings -> Privacy & Security
- You must reboot your device, and confirm the decision again

For automation, use `devmodectl`

## Automation Flows

One limitation: only devices without passcodes can be automatically enrolled into Developer Mode

macOS Ventura ships with `devmodectl` that can automate Developer Mode:
- Single device one-off mode
- Streaming mode

## Related Sessions

- What's new in notarization for Mac apps
