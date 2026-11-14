# UI/UX optimizations

# **EyeDropper API**

To fill this gap between web apps and native apps, the Microsoft Edge team proposed and implemented the new EyeDropper API. We enabled it for all our users when the feature shipped on Chromium-based browsers in the week of 21 Oct 2021.

### **Speeding up UI tests with CodeceptJS and Playwright**

The new framework needed to:

- Support Typescript so that it was consistent with our frontend codebase.
- Have active development accepting external contributions so that it's less likely to become outdated.
- Support our existing test setup (Cucumber) while we migrated to the new framework.
- Support using drivers and libraries such as Playwright for web tests and Appium for web and mobile tests.
- Support testing on multiple browsers and devices, including using cloud-based browser testing services such as Saucelabs, Browerstack, or LambdaTest.
- Support headless browsers such as those provided by Playwright.

**CodeceptJS** fulfilled all our requirements and has good documentation and a great community.

 We first decided to create a wrapper around the CodeceptJS API that resembled the Chimp API to support both test frameworks. This would allow us to inject the correct framework during runtime, depending on which framework we wanted to run.

WebdriverIO allowed the use of multiple selector strategies which involved chaining find element commands. We introduced a pseudo-element object when using the find element command that internally kept track of the selectors. When actions were triggered, we would join the underlying selectors in the pseudo-element into one selector.

We used **ts-node-dev** to watch for any code changes and programmatically invoke our tests if any changes are made. Playwright allows Webkit testing without the need for cloud-based cross-browser testing services. Using Playwright in our CI pipelines halved our CI Webkit test run times and increased our test coverage by 30%. 

To add `await`s, we customized the eslint [**no-floating-promise**](https://github.com/typescript-eslint/typescript-eslint/blob/main/packages/eslint-plugin/docs/rules/no-floating-promises.md) rule to find asynchronous functions that were not awaited and wrote a fix function specific to our scenarios that would safely add awaits.

To add `async`s, we used [**ts-morph**](https://github.com/dsherret/ts-morph), a Typescript AST, to transform function expressions and arrow functions to asynchronous functions if they contained the keyword `await`.

To reduce the start-up wait times, we programmatically generated the Cucumber configuration file to contain only the feature files and steps definition files required for the test. First, we used the [**Gherkin AST**](https://github.com/cucumber/gherkin) to read the feature file steps that we wanted to run. Second, we used [**ts-morph**](https://github.com/dsherret/ts-morph) to extract all step regexes from step definitions files and matched the regexes with the feature file steps mentioned previously. Using this method reduced the start-up time to between 6 and 10 seconds. This was a significant reduction in wait times for local execution, but we also saw faster executions in CI builds.

Additionally, we introduced a cache mechanism that would search the step definitions used from the previous run before searching all the step definitions. Using this cache meant searching only about 20 step definition files instead of 600. In most cases, a developer would want to run the same test multiple times, which worked great and reduced the start-up time to between 2 and 4 seconds.