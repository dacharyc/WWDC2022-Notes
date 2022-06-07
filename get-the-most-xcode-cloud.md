# Get the most out of Xcode Cloud

Presenters:
- Adam Smythe: Developer Experience Manager
- Sasan Naderi: Xcode Cloud Engineer

## Overview

- Xcode Cloud announced at WWDC21
- CI/CD for Apple devs
- Accelerates development and delivery of high-quality items by:
  - Helping you build apps
  - Run automated tests in parallel
  - Deliver apps to testers
  - View and manage user feedback

## Existing Workflow

- Does it get you the build and test results you want as quickly as possible? Can you save time and resources?

Look at Build Details Overview
- How long does it take to build?
- Time associated with usage?

Usage distribution helps figure out how you can possibly optimize.

Actions:
- Analyze
- Archive
- Build
- Test

Actions are performed in parallel, so the duration of the build is the duration of the longest action. In this case, it's a build that takes 14 minutes.
Usage is the total duration of all the actions that are executing in parallel.

### Optimizing

Workflows define when to perform a build with start conditions. Define your start conditions so you don't perform a build unnecessarily.

- Prefer branch changes vs. scheduled start condition to avoid builds with no new conditions
- Branch changes run when a new commit is pushed to a remote branch; can customize branch names or build from any branch
- Can specify files and folders to skip/start builds

Select reasonable test destinations

- Each destination runs in parallel
- Make sure to select a concise set of simulators
- Xcode Cloud provides an alias for recommended destinations; curated lists of simulators that represent a cross-section of screen sizes

Skip builds and CI depending on the type of change being committed

- Append `[ci skip]` to the end of the commit message, so Xcode Cloud will ignore the event

Optimize custom scripts and tests

- Tidy up unused dependencies
- Resiliently retry API requests that are known to be unreliable
- Ensure that flakey and unreliable tests are corrected quickly
