# Selenium Glossary

A comprehensive glossary of terms you will encounter throughout this course and in Selenium automation work. Entries are organized alphabetically.

---

### Actions (C#) / ActionChains (Python)

A class that lets you chain low-level mouse and keyboard interactions: hover, double-click, right-click, drag-and-drop, key presses. In C# it is `OpenQA.Selenium.Interactions.Actions`; in Python it is `selenium.webdriver.common.action_chains.ActionChains`. You build a sequence of actions and call `.Perform()` (C#) or `.perform()` (Python) to execute them.

### Alert

A browser-native dialog box (alert, confirm, or prompt). Selenium cannot interact with alerts using normal element methods. You must switch to the alert first with `driver.SwitchTo().Alert()` (C#) or `driver.switch_to.alert` (Python), then accept, dismiss, or read its text.

### Assertion

A check that verifies an expected condition is true. If the assertion fails, the test fails. NUnit uses `Assert.That(...)` and PyTest uses Python's built-in `assert` statement.

### Browser Options

Configuration objects passed to the driver at startup to customize browser behavior. Examples: `ChromeOptions`, `FirefoxOptions`, `EdgeOptions`. Common uses include running headless, setting window size, disabling notifications, and adding proxy settings.

### ChromeDriver

The WebDriver implementation that controls Google Chrome. It is a standalone executable that acts as a bridge between your Selenium code and the Chrome browser. Managed automatically by Selenium Manager or WebDriverManager.

### ChromeOptions

A configuration class used to set Chrome-specific launch options. Common arguments: `--headless=new`, `--start-maximized`, `--disable-notifications`, `--incognito`.

### conftest.py

A special file recognized by PyTest. Fixtures and hooks defined in `conftest.py` are automatically available to all test files in the same directory and its subdirectories, without requiring an import.

### CSS Selector

A pattern used to select HTML elements based on their tag, class, id, attributes, and structural position. Generally faster than XPath and more readable for simple selections. Example: `div.container > p.intro`.

### DesiredCapabilities (Legacy)

An older mechanism for specifying browser/platform requirements when creating a WebDriver session. Replaced by browser-specific `Options` classes in Selenium 4+. You may still encounter it in legacy codebases.

### DOM (Document Object Model)

The tree-structured representation of an HTML page in memory. Selenium interacts with web pages by finding and manipulating elements within the DOM. When the page changes dynamically (via JavaScript), the DOM updates accordingly.

### Driver

Short for WebDriver. The object your test code uses to control a browser. Each browser has its own driver implementation (ChromeDriver, GeckoDriver, EdgeDriver). The driver translates your Selenium commands into browser-specific instructions.

### EdgeDriver

The WebDriver implementation for Microsoft Edge (Chromium-based). Since modern Edge is built on Chromium, EdgeDriver is similar to ChromeDriver in behavior and configuration.

### ElementNotInteractableException

Thrown when an element exists in the DOM but cannot be interacted with -- for example, because it is obscured by another element, has zero size, or is not within the viewport. Common fix: scroll the element into view or wait for an overlay to disappear.

### ExplicitWait

See **WebDriverWait**. An explicit wait pauses execution until a specific condition is met or a timeout is reached. More targeted and reliable than implicit waits. Always preferred for dynamic content.

### ExpectedConditions

A collection of predefined conditions used with WebDriverWait. Examples: `ElementIsVisible`, `ElementToBeClickable`, `AlertIsPresent`, `TitleContains`. In C# these are in `SeleniumExtras.WaitHelpers.ExpectedConditions`; in Python they are in `selenium.webdriver.support.expected_conditions`.

### FirefoxOptions

A configuration class for Firefox-specific launch options, analogous to `ChromeOptions`. Use it to set headless mode, set Firefox binary path, or add Firefox-specific preferences.

### Fluent Wait

A variant of explicit wait that gives you control over the polling interval and which exceptions to ignore while waiting. Useful when the default polling frequency of WebDriverWait is too fast or you need to swallow specific exceptions during polling.

### Frame / iFrame

An HTML element (`<frame>` or `<iframe>`) that embeds another HTML document within the current page. Selenium cannot see elements inside a frame until you switch into it with `driver.SwitchTo().Frame(...)`. You must switch back to the parent frame or default content when done.

### GeckoDriver

The WebDriver implementation for Mozilla Firefox. GeckoDriver translates WebDriver commands into the Marionette protocol that Firefox understands. Required for Selenium to control Firefox.

### Headless Mode

Running a browser without a visible window. The browser renders pages in memory but nothing appears on screen. Useful for CI/CD pipelines and faster test execution. Enable with `--headless=new` in Chrome or `-headless` in Firefox.

### Hub (Selenium Grid)

The central server in a Selenium Grid setup. The Hub receives test requests from clients and distributes them to available Nodes. Tests connect to the Hub URL and the Hub routes them to the appropriate Node based on requested capabilities.

### Implicit Wait

A global timeout set on the driver that tells it to poll the DOM for a specified duration when trying to find an element. Applies to every `FindElement` call. Set once and it persists for the life of the driver. Avoid mixing with explicit waits, as the behaviors can conflict unpredictably.

### InvalidSelectorException

Thrown when a locator expression is syntactically invalid -- for example, a malformed XPath or CSS selector. Fix the selector syntax to resolve this.

### JavaScriptExecutor

An interface that lets you run arbitrary JavaScript in the browser context. In C# cast the driver to `IJavaScriptExecutor`; in Python call `driver.execute_script()`. Common uses: scrolling, clicking hidden elements, reading computed styles, modifying the DOM.

### Locator

A strategy for finding elements on a web page. Selenium provides eight locator types: ID, Name, ClassName, TagName, LinkText, PartialLinkText, CssSelector, and XPath. Choosing the right locator is one of the most important skills in Selenium automation.

### Node (Selenium Grid)

A machine (physical or virtual) in a Selenium Grid that has one or more browsers available. Nodes register with the Hub and execute tests on behalf of the Hub. A Grid can have many Nodes with different browsers and operating systems.

### NoSuchElementException

Thrown when `FindElement` cannot locate an element matching the given locator. Common causes: wrong locator, element has not loaded yet (use a wait), element is inside a frame (switch to it first), or the page URL is wrong.

### NoSuchFrameException

Thrown when trying to switch to a frame that does not exist. Verify the frame name, id, or index. Use a wait to ensure the frame has loaded before switching.

### NoSuchWindowException

Thrown when trying to switch to a window handle that no longer exists, usually because the window was closed.

### NUnit

A popular open-source unit testing framework for .NET / C#. Provides attributes like `[Test]`, `[SetUp]`, `[TearDown]`, and `[TestCase]` to structure tests, plus `Assert.That(...)` for assertions.

### Page Factory

A pattern (built into Selenium for Java; available via `SeleniumExtras` for C#) where elements on a Page Object are initialized using annotations/attributes. Less common in Python where simple locator tuples are preferred.

### Page Object Model (POM)

A design pattern where each web page (or component) is represented by a class. The class encapsulates locators and page-specific actions as methods. Tests interact with page methods instead of raw Selenium calls. Benefits: reduces duplication, improves readability, and makes tests easier to maintain when the UI changes.

### Parallel Execution

Running multiple tests at the same time to reduce total execution time. In NUnit, use `[Parallelizable]` attributes. In PyTest, use the `pytest-xdist` plugin with `pytest -n auto`. Requires that tests are independent and do not share state.

### PyTest

A popular Python testing framework. Features include simple `assert` statements, powerful fixtures with dependency injection, parametrization via `@pytest.mark.parametrize`, and a rich plugin ecosystem. The primary testing framework used in the Python track of this course.

### RemoteWebDriver

A WebDriver implementation that connects to a browser running on a different machine (or in a container). Used with Selenium Grid, cloud providers (BrowserStack, Sauce Labs), or any standalone WebDriver server. You provide a remote URL and desired capabilities/options.

### Selector

A general term for a locator expression -- either a CSS selector string or an XPath expression -- used to identify elements in the DOM.

### Selenium

An open-source suite of tools for automating web browsers. The core component is Selenium WebDriver, which provides a programming interface for controlling browsers. Other components include Selenium Grid (for distributed testing) and Selenium IDE (a record-and-playback browser extension).

### Selenium Grid

A system for running Selenium tests on multiple machines in parallel. Consists of a Hub (central coordinator) and one or more Nodes (machines with browsers). Enables cross-browser and cross-platform testing at scale.

### Selenium IDE

A browser extension (Chrome/Firefox) that records user interactions and generates Selenium test scripts. Useful for quick prototyping but not recommended for production test suites due to fragile locators and lack of programming logic.

### Selenium Manager

A tool bundled with Selenium 4.6+ that automatically downloads and configures the correct browser driver for your installed browser version. Eliminates the need for manual driver management or third-party tools like WebDriverManager in many cases.

### Shadow DOM

A browser feature that encapsulates a subtree of DOM elements, hiding them from the main document's DOM. Regular `FindElement` calls cannot reach inside a shadow DOM. You must first get the shadow root element and then search within it.

### StaleElementReferenceException

Thrown when you try to interact with an element reference that is no longer attached to the DOM. This happens when the page or a portion of it has been refreshed or re-rendered since you found the element. Fix: re-locate the element before interacting with it.

### TestCase (NUnit) / parametrize (PyTest)

Mechanisms for data-driven testing. `[TestCase]` in NUnit and `@pytest.mark.parametrize` in PyTest let you run the same test method with different input values, generating a separate test result for each set of inputs.

### TimeoutException

Thrown when a wait (explicit or fluent) exceeds its timeout without the expected condition becoming true. Common fix: increase the timeout, verify the condition is correct, or check that the element/page is actually loading.

### W3C WebDriver Protocol

The standardized protocol (a W3C Recommendation) that defines how client libraries communicate with browser drivers over HTTP. Selenium 4 uses the W3C protocol exclusively, replacing the older JSON Wire Protocol. This standardization means better cross-browser consistency.

### WebDriver

The core interface in Selenium that represents a browser session. Provides methods for navigation, element finding, window management, and more. Each browser has a concrete implementation (ChromeDriver, GeckoDriver, EdgeDriver). In C# the interface is `IWebDriver`; in Python you use `webdriver.Chrome()`, `webdriver.Firefox()`, etc.

### WebDriverException

The base exception class for all Selenium WebDriver errors. More specific exceptions (NoSuchElementException, TimeoutException, StaleElementReferenceException) inherit from it. Catching `WebDriverException` will catch any Selenium error, which can be useful as a broad safety net but makes debugging harder.

### WebDriverManager

A third-party library (available for C#, Java, and Python) that automatically downloads and manages browser driver executables. In C# it is the `WebDriverManager` NuGet package. In Python it is the `webdriver-manager` pip package. Being partially superseded by Selenium Manager in Selenium 4.6+.

### WebDriverWait

A class that implements explicit waiting. You provide a driver, a timeout, and a condition. The wait polls the condition at regular intervals until it returns a truthy value or the timeout expires. The single most important tool for handling dynamic web pages reliably.

### WebElement

The object returned when you find an element on the page. It represents a single DOM element and provides methods for clicking, typing, reading text, getting attributes, and more. In C# the interface is `IWebElement`; in Python it is a `WebElement` instance.

### Window Handle

A unique string identifier assigned to each browser window or tab. Use `driver.CurrentWindowHandle` (C#) or `driver.current_window_handle` (Python) to get the current one, and `driver.WindowHandles` / `driver.window_handles` to get all of them. Required when switching between multiple windows or tabs.

### XPath

XML Path Language -- a query language for selecting nodes in an XML/HTML document. More powerful than CSS selectors (supports text matching, parent traversal, and complex conditions) but generally slower and more verbose. Two types: absolute (`/html/body/div`) which is fragile, and relative (`//div[@id='main']`) which is preferred.

---
