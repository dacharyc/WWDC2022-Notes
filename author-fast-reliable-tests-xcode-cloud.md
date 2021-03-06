# Author fast and reliable tests for Xcode Cloud

Presenters:
- Suzy Ratcliff, XCTest Engineer

With Xcode Cloud, you can run tests on multiple destinations, including those running on differint operating systems - i.e. iPhone, iPad, Apple Watch, Apple TV, and Mac. You can run different test plan configurations allowing for runtime analysis tols such as address and thread sanitizers. Xcode Cloud provides all of this without impacting the developer's local desktop cycle of code, compile, and test.

With the expanded test suite, there is potential for an increasing number of unreliable tests. There can also be efficiency impacts.

For the example, we'll look at Xcode Clooud tests for Food Truck app.

## Proper simulator setup

Tests running in Xcode Cloud have a fresh simulator. This may not be the developer's expectation.

- Avoid tests that are time-zone specific.
- Locale-based values, such as number formatting and language directionality, can lead to unexpected results. Explicitly set the simulator's locale.
- Avoid reliance on certain device permissions such as internet access. Mock device permissions in a unit test and use an alert handler in a UI test.
- Some tests rely on pre-loaded data. Provide that data in prepared files during setUp().

## Handle tests that fail to meet preconditions

`XCTSkip` instructs the `XCTest Runner` to cease executing the current test and mark it as skipped. You can use this to:

- Bypass a yet-to-be-supported OS version or device type.
- Set an environment variable to skip tests specific to staging or production environments

## Control test flow using an environment variable

Environment variables can provide parameters to both the XCTest test runner app on your device and the test host running `xcodebuild`. 

In Xcode Cloud, environment variables prefixed with `TEST_RUNNER_` get passed into the XCTest test runner. The prefix is stripped prior to the variable being made available in your code, so for example `TEST_RUNNER_BASE_URL` would become `BASE_URL` as the environment variable in your test code.

Test plans require the same format as test code. We do _not_ add the `TEST_RUNNER_` prefix.

You can use environment variables anywhere in test code; for example, you can use them with `XCTSkip` to skip the test for actually ordering donuts when in a production environment.

Worth noting that redefining an environment variable in multiple places, such as both a test plan and the Xcode Cloud UI, can lead to unexpected results. In this case, Xcode Cloud's Environment Variables take precedence over what you specify in your project's test plans. 

### Set an environment value's value within Xcode Cloud UI

When referencing an environment variable within the test code, you can set its value in the Xcode Cloud User Interface. To do this:

1. Navigate to your Cloud Reports, and ctrl+click on Food Truck (or your project name)
2. To edit environment variables within Workflows, select "Manage Workflows" in the context menu
3. In this example, we're editing the integration workflow, so we'll double click that one.
4. In the sidebar, select "Environment." 
5. In the middle of the sheet, under "Environment Variables," you can add the variable's name and value.

### Set an environment value's value in the Test Plan

Alternately, you can set it within the test plan.

In this example, we don't yet have a test plan. To enable test plans:

1. Open the Scheme editor
2. Select "Test" in the sidebar
3. Click "Convert to Use Test Plans" 

This gives us a test plan.

To add environment variables: 

1. Click on the test plan to open its editor.
2. Near the top, you can select between "Tests" and "Configurations." Select "Configurations."
3. In the "Arguments" section, click on "Environment Variables."
4. Enter the variable's name and value in the popup that appears.

Now we can skip the tes in production.

```swift
func testOrderDonut() throws {
    let host = ProcessInfo.processInfo.environment["BASE_URL"]
    try XCTSkipIf(host == "prod.example.com")

    let expectation = XCTestExpectation(description:L "Order donut")
    truck.order(with: .sprinkles, host: host) { error, donut in
        XCTAssertTrue(donut.hasSprinkles)
        expectation.fulfill()
    }
    wait(for: [expectation], timeout: 5)
}
```

## Test reliability

### Expectation timeouts

Increase `XCTestExpectation` timeout or replace with async/await. Replacing with async/await is preferred.

## Declare tests that are expected to fail

You can use `XCTExpectFailure` instead of disabling or skipping a test. With `XCTExpectFailure` your test executes normally, and the results are transformed as follows:

- A failure in a test will now be reported as an expected failure. That failed test within its suite will be reported as a pass.

This approach eliminates the noise generated by expected failures.

## Leverage test repetition to diagnose unreliable code

Test repetitions is a tool that runs the same test multiple times waiting for one of the following:

- The first failure
- The first success
- A statistical result

We can run our code and test cases multiple times with repetition to confirm initial app and test code reliability before checking in the code. If you see that a failure exists, use the `repeat-until-failure` mode locally to diagnose the bug. 

For tests that rely on an unreliable external service, leverage the `retry-on-failure` repetition policy to confirm a test can succeed. However, prefer instead to mock external services when possible.

### Enable test repetitions

To enable test repetitions in your test plan:

1. Go to the test plan editor and select "Configurations"
2. Go to the "Test Execution" section.
3. Use the popup to select your test repetition mode.

Consider setting "Maximum Test Repetitions" to do fewer repetitions. 

## Configuring for faster results

### Split tests into multiple test plans

Split tests into multiple test plans. For example:
- Run unit tests along with a key subset of UI tests for a single platform
- Run the full set of tests on all supported tests in the background, not blocking pull requests

With this approach, you can add tests and new platforms while keeping CI timely.

To set up a workflow to run a selected set of tests:

1. Create a new test plan - in our example, called "Pull Requests"
2. Open it in the test plan editor
3. Near the top, you can select between "Tests" and "Configurations." Select "Tests."
4. Choose a subset of tests to verify for a pull request.

Now, to set up a Workflow to run our "Pull Requests" test plan:

1. Go back to Xcode Cloud Manage Workflows
2. Press the "Add" button at the bottom left of the "Manage Workflow" sheet
3. For simplicity for this example, name the new workflow "Pull Requests."
4. Choose a start condition. In the sidebar, to the right of "Start Conditions," press the "Add" button.
5. A menu appears showing start condition options. For our example, we'll select "Pull Request Changes."
6. Add a build action. In the sidebar, below "Start Conditions," select "Build" from the context menu.
7. Now we need to run the tests. Click the "Add" action again, but this time select "Test." Now we have a test action.
8. In the middle of the sheet, there is a drop-down for test. Select the "Pull Requests" test plan.

Now the Workflow is configured to run our test plan on pull requests. To create a second workflow that runs a complete test suite on a schedule, follow this procedure again, but set the "Start Condition" to "On a Schedule for a Branch" - and then set the workflow to run your full suite test plan.

### Run tests concurrently

By default, Xcode Cloud tests your platforms in parallel. In addition, you can enable Xcode to run tests in parallel on a target and test object class level.

To enable parallel test execution in Xcode:

1. Go to our test plan editor, and select "Tests"
2. To the right of the "Food Truck Tests" test bundle, click the "Options" button

One of the options allows us to "Execute in parallel" when possible. If the server has enough cores available, multiple targets and test object classes can be executed concurrently. Enable this option to improve the test suite turnaround time.

Tests must be designed to run independently to take advantage of parallel execution. Proper setup and teardown are essential to reliable test case behavior.

## Limit runaway tests

Runaway tests are tests that don't end in a timely manner. Some examples include an infinite loop or waiting indefinitely for a failed server.

Halt these tests by setting an execution time allowance in the test plan. This specifies the number of seconds for a test to run before it fails with a timeout error. This prevents a test suite from getting stuck on an individual test.

To set an execution time allowance:

1. Go to the test plan editor
2. Select "Configurations"
3. Under the "Test Execution" category, enable "Test Timeouts"
4. Specify the number of seconds to wait. The default is 600 seconds.

## Related Sessions

- Embrace expected failures in XCTest: WWDC 2021
- Testing Tips & Tricks: WWDC 2018
- Diagnose unreliable code with test repetition: WWDC 2021
- Write tests to fail: WWDC 2020
- Testing in Xcode: WWDC 2019
- Get your test results faster: WWDC 2020
