# Day 9: Fluent Wait and Custom ExpectedConditions

## Learning Objectives

By the end of this lesson, you will be able to:

- Configure a fluent wait with custom polling intervals
- Ignore specific exceptions during the polling cycle
- Write custom expected conditions for non-standard scenarios
- List all built-in ExpectedConditions and know when to use each
- Decide when to use fluent wait vs standard explicit wait

---

## Core Concept Explanation

A **fluent wait** is an explicit wait with extra configuration options. While a standard `WebDriverWait` polls every 500ms and only ignores `NotFoundException`, a fluent wait lets you:

1. **Set a custom polling interval** -- check every 250ms, every 1 second, or any interval you choose.
2. **Ignore specific exception types** -- suppress `StaleElementReferenceException`, `ElementNotInteractableException`, or others during polling.
3. **Define a custom message** for the timeout exception.

In Selenium 4, `WebDriverWait` itself supports all fluent wait features. In C#, you can also use `DefaultWait<IWebDriver>` for a more explicit fluent API. In Python, `WebDriverWait` accepts `poll_frequency` and `ignored_exceptions` directly in its constructor.

### When to Use Fluent Wait

- Elements that appear and disappear rapidly (reduce polling interval)
- Elements that go stale during the wait (ignore `StaleElementReferenceException`)
- Slow-loading pages where you want to reduce polling overhead (increase interval)
- Custom conditions that are not covered by built-in `ExpectedConditions`

---

## How It Works

The fluent wait loop works as follows:

```
start_time = now()
while (now() - start_time < timeout):
    try:
        result = condition(driver)
        if result is truthy:
            return result
    except ignored_exceptions:
        pass  # swallow and retry
    sleep(polling_interval)
raise TimeoutException("Condition not met within timeout")
```

Each iteration calls your condition function, passing the driver. If the condition returns a truthy value (a non-null element, `True`, etc.), the wait returns immediately. If it throws an ignored exception, the wait swallows it and retries. If the timeout expires, a `TimeoutException` is thrown.

---

## Code Example: C#

```csharp
// File: Day09_FluentWaitDemo.cs
// Prerequisites:
//   dotnet add package Selenium.WebDriver
//   dotnet add package Selenium.Support
//   dotnet add package DotNetSeleniumExtras.WaitHelpers
//   dotnet add package NUnit
//   dotnet add package NUnit3TestAdapter

using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using SeleniumExtras.WaitHelpers;
using System;

namespace Day09
{
    [TestFixture]
    public class FluentWaitDemo
    {
        private IWebDriver driver;

        [SetUp]
        public void SetUp()
        {
            driver = new ChromeDriver();
            driver.Manage().Window.Maximize();
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        // --- FLUENT WAIT USING WebDriverWait (Selenium 4) ---
        [Test]
        public void FluentWaitWithWebDriverWait()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_loading/2");
            driver.FindElement(By.CssSelector("#start button")).Click();

            // WebDriverWait in Selenium 4 supports fluent configuration
            WebDriverWait wait = new WebDriverWait(driver, TimeSpan.FromSeconds(15))
            {
                // Poll every 250ms instead of the default 500ms
                PollingInterval = TimeSpan.FromMilliseconds(250),
                // Custom timeout message
                Message = "The result element did not become visible within 15 seconds"
            };

            // Ignore StaleElementReferenceException during polling
            wait.IgnoreExceptionTypes(typeof(StaleElementReferenceException));

            IWebElement result = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("finish"))
            );

            Assert.That(result.Text, Does.Contain("Hello World"));
        }

        // --- FLUENT WAIT USING DefaultWait<IWebDriver> ---
        [Test]
        public void FluentWaitWithDefaultWait()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_loading/2");
            driver.FindElement(By.CssSelector("#start button")).Click();

            // DefaultWait provides a more explicit fluent API
            DefaultWait<IWebDriver> fluentWait = new DefaultWait<IWebDriver>(driver)
            {
                Timeout = TimeSpan.FromSeconds(15),
                PollingInterval = TimeSpan.FromMilliseconds(300),
                Message = "Element not found within timeout"
            };

            // Ignore multiple exception types
            fluentWait.IgnoreExceptionTypes(
                typeof(NoSuchElementException),
                typeof(StaleElementReferenceException)
            );

            // Use a lambda as the condition function
            IWebElement result = fluentWait.Until(d =>
            {
                IWebElement el = d.FindElement(By.Id("finish"));
                return el.Displayed ? el : null;
            });

            Assert.That(result.Text, Does.Contain("Hello World"));
        }

        // --- CUSTOM EXPECTED CONDITION ---
        [Test]
        public void CustomExpectedCondition()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_controls");

            WebDriverWait wait = new WebDriverWait(driver, TimeSpan.FromSeconds(15));

            // Click "Remove" to remove the checkbox
            driver.FindElement(By.CssSelector("#checkbox-example button")).Click();

            // Custom condition: wait until the checkbox is gone AND the message appears
            bool result = wait.Until(d =>
            {
                // Check that the checkbox no longer exists
                var checkboxes = d.FindElements(By.Id("checkbox"));
                bool checkboxGone = checkboxes.Count == 0;

                // Check that the success message is displayed
                var message = d.FindElements(By.Id("message"));
                bool messageVisible = message.Count > 0 && message[0].Displayed;

                // Both conditions must be true
                return checkboxGone && messageVisible;
            });

            Assert.That(result, Is.True);

            string messageText = driver.FindElement(By.Id("message")).Text;
            Assert.That(messageText, Is.EqualTo("It's gone!"));
        }

        // --- CUSTOM CONDITION: WAIT FOR ELEMENT COUNT ---
        [Test]
        public void WaitForElementCount()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/tables");

            WebDriverWait wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));

            // Custom condition: wait until there are at least 4 rows in the table
            var rows = wait.Until(d =>
            {
                var tableRows = d.FindElements(By.CssSelector("#table1 tbody tr"));
                return tableRows.Count >= 4 ? tableRows : null;
            });

            Assert.That(rows.Count, Is.GreaterThanOrEqualTo(4));
            TestContext.WriteLine($"Found {rows.Count} rows in the table");
        }

        // --- CUSTOM CONDITION: WAIT FOR ATTRIBUTE VALUE ---
        [Test]
        public void WaitForAttributeValue()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_controls");

            WebDriverWait wait = new WebDriverWait(driver, TimeSpan.FromSeconds(15));

            // Click "Enable" to enable the text input
            driver.FindElement(By.CssSelector("#input-example button")).Click();

            // Custom condition: wait until the input's "disabled" attribute is removed
            IWebElement input = wait.Until(d =>
            {
                IWebElement el = d.FindElement(By.CssSelector("#input-example input"));
                // Element is enabled when the disabled attribute is null
                return el.GetAttribute("disabled") == null ? el : null;
            });

            input.SendKeys("Selenium is great!");
            Assert.That(input.GetAttribute("value"), Is.EqualTo("Selenium is great!"));
        }
    }
}
```

---

## Code Example: Python

```python
# File: test_day09_fluent_wait_demo.py
# Prerequisites:
#   pip install selenium pytest

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import (
    StaleElementReferenceException,
    NoSuchElementException,
)
import pytest


@pytest.fixture
def driver():
    """Create a Chrome browser instance for each test."""
    drv = webdriver.Chrome()
    drv.maximize_window()
    yield drv
    drv.quit()


# --- FLUENT WAIT WITH CUSTOM POLLING AND IGNORED EXCEPTIONS ---
def test_fluent_wait_example(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_loading/2")
    driver.find_element(By.CSS_SELECTOR, "#start button").click()

    # WebDriverWait accepts poll_frequency and ignored_exceptions directly
    wait = WebDriverWait(
        driver,
        timeout=15,
        poll_frequency=0.25,  # Poll every 250ms instead of default 500ms
        ignored_exceptions=[StaleElementReferenceException],
    )

    result = wait.until(EC.visibility_of_element_located((By.ID, "finish")))

    assert "Hello World" in result.text


# --- FLUENT WAIT WITH LAMBDA CONDITION ---
def test_fluent_wait_with_lambda(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_loading/2")
    driver.find_element(By.CSS_SELECTOR, "#start button").click()

    wait = WebDriverWait(
        driver,
        timeout=15,
        poll_frequency=0.3,
        ignored_exceptions=[NoSuchElementException, StaleElementReferenceException],
    )

    # Use a lambda as the condition: find element and check visibility
    result = wait.until(
        lambda d: d.find_element(By.ID, "finish")
        if d.find_element(By.ID, "finish").is_displayed()
        else False
    )

    assert "Hello World" in result.text


# --- CUSTOM EXPECTED CONDITION: FUNCTION ---
def test_custom_expected_condition(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_controls")

    wait = WebDriverWait(driver, 15)

    # Click "Remove" to remove the checkbox
    driver.find_element(By.CSS_SELECTOR, "#checkbox-example button").click()

    # Custom condition function
    def checkbox_removed_and_message_shown(d):
        """Return True only when checkbox is gone AND message is displayed."""
        checkboxes = d.find_elements(By.ID, "checkbox")
        checkbox_gone = len(checkboxes) == 0

        messages = d.find_elements(By.ID, "message")
        message_visible = len(messages) > 0 and messages[0].is_displayed()

        return checkbox_gone and message_visible

    wait.until(checkbox_removed_and_message_shown)

    message = driver.find_element(By.ID, "message").text
    assert message == "It's gone!"


# --- CUSTOM EXPECTED CONDITION: CLASS ---
class ElementHasAttributeValue:
    """Custom expected condition that waits for an element's attribute to have
    a specific value. Returns the element when matched, False otherwise."""

    def __init__(self, locator, attribute, value):
        self.locator = locator
        self.attribute = attribute
        self.value = value

    def __call__(self, driver):
        element = driver.find_element(*self.locator)
        if element.get_attribute(self.attribute) == self.value:
            return element
        return False


def test_custom_condition_class(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_controls")

    wait = WebDriverWait(driver, 15)

    # Click "Enable" to enable the input
    driver.find_element(By.CSS_SELECTOR, "#input-example button").click()

    # Wait until the message shows "It's enabled!"
    element = wait.until(
        ElementHasAttributeValue(
            (By.ID, "message"), "textContent", "It's enabled!"
        )
    )

    assert element is not False


# --- WAIT FOR ELEMENT COUNT ---
def test_wait_for_element_count(driver):
    driver.get("https://the-internet.herokuapp.com/tables")

    wait = WebDriverWait(driver, 10)

    # Wait until there are at least 4 rows in the table
    def at_least_four_rows(d):
        rows = d.find_elements(By.CSS_SELECTOR, "#table1 tbody tr")
        return rows if len(rows) >= 4 else False

    rows = wait.until(at_least_four_rows)

    assert len(rows) >= 4
    print(f"Found {len(rows)} rows in the table")


# --- WAIT FOR URL CHANGE ---
def test_wait_for_url_change(driver):
    driver.get("https://the-internet.herokuapp.com/")

    wait = WebDriverWait(driver, 10)

    # Click a link and wait for the URL to change
    driver.find_element(By.LINK_TEXT, "Dynamic Loading").click()

    wait.until(EC.url_contains("dynamic_loading"))

    assert "dynamic_loading" in driver.current_url
```

---

## Step-by-Step Walkthrough

1. **Standard explicit wait** -- `WebDriverWait(driver, 10)` polls every 500ms, ignoring only `NotFoundException`. Good for most cases.
2. **Fluent configuration** -- Add `PollingInterval` (C#) or `poll_frequency` (Python) to change the polling rate. Useful when elements flicker rapidly.
3. **Ignoring exceptions** -- Call `IgnoreExceptionTypes` (C#) or pass `ignored_exceptions` (Python) to suppress exceptions like `StaleElementReferenceException` that occur when the DOM is rebuilding.
4. **Custom conditions with lambdas** -- Pass a lambda/function that receives the driver and returns a truthy value when the condition is met, or a falsy value/exception to keep polling.
5. **Custom condition classes (Python)** -- Create a callable class with `__call__` for reusable conditions. The class stores locator info and returns the element or `False`.
6. **Combining conditions** -- Check multiple things in a single custom condition (e.g., element removed AND message shown).

---

## Built-in ExpectedConditions Reference

### C# (SeleniumExtras.WaitHelpers.ExpectedConditions)

| Condition | Returns | Use When |
|-----------|---------|----------|
| `ElementExists(locator)` | `IWebElement` | Need element in DOM (may be hidden) |
| `ElementIsVisible(locator)` | `IWebElement` | Need to read text or verify display |
| `ElementToBeClickable(locator)` | `IWebElement` | About to click an element |
| `InvisibilityOfElementLocated(locator)` | `bool` | Waiting for spinner/overlay to disappear |
| `TextToBePresentInElement(element, text)` | `bool` | Waiting for specific text content |
| `TextToBePresentInElementLocated(locator, text)` | `bool` | Same, but finds element by locator |
| `TitleContains(text)` | `bool` | Page title should include text |
| `TitleIs(text)` | `bool` | Page title should match exactly |
| `UrlContains(text)` | `bool` | URL should include substring |
| `UrlToBe(url)` | `bool` | URL should match exactly |
| `FrameToBeAvailableAndSwitchToIt(locator)` | `IWebDriver` | Switching into a frame |
| `AlertIsPresent()` | `IAlert` | Waiting for a JavaScript alert |
| `ElementToBeSelected(element)` | `bool` | Checkbox/radio should be selected |
| `StalenessOf(element)` | `bool` | Old element should be removed from DOM |

### Python (selenium.webdriver.support.expected_conditions)

| Condition | Returns | Use When |
|-----------|---------|----------|
| `presence_of_element_located(locator)` | `WebElement` | Need element in DOM |
| `visibility_of_element_located(locator)` | `WebElement` | Need visible element |
| `element_to_be_clickable(locator)` | `WebElement` | About to click |
| `invisibility_of_element_located(locator)` | `bool/WebElement` | Spinner disappearing |
| `text_to_be_present_in_element(locator, text)` | `bool` | Waiting for text |
| `title_contains(text)` | `bool` | Title check |
| `title_is(text)` | `bool` | Exact title match |
| `url_contains(text)` | `bool` | URL substring check |
| `url_to_be(url)` | `bool` | Exact URL match |
| `frame_to_be_available_and_switch_to_it(locator)` | `bool` | Frame switching |
| `alert_is_present()` | `Alert` | Alert waiting |
| `element_to_be_selected(element)` | `bool` | Selection check |
| `staleness_of(element)` | `bool` | Element removed |
| `number_of_windows_to_be(count)` | `bool` | Multiple windows |
| `new_window_is_opened(handles)` | `bool` | New window opened |

---

## Common Mistakes & How to Avoid Them

### 1. Setting polling interval too low
Polling every 10ms generates excessive WebDriver traffic and can slow down the browser. Keep polling at 250ms or higher.

### 2. Ignoring all exceptions
```python
# BAD: hides real errors
ignored_exceptions=[Exception]
```
Only ignore specific, expected exceptions like `NoSuchElementException` or `StaleElementReferenceException`.

### 3. Returning None from a custom condition
In Python, `None` is falsy, so returning `None` means "keep waiting." If your condition function accidentally returns `None`, the wait will always time out.

### 4. Forgetting to return the element
```python
# BAD: returns True instead of the element
def my_condition(d):
    el = d.find_element(By.ID, "result")
    if el.is_displayed():
        return True  # You lose the element reference

# GOOD: return the element itself
def my_condition(d):
    el = d.find_element(By.ID, "result")
    return el if el.is_displayed() else False
```

---

## Best Practices

1. **Start with standard WebDriverWait.** Only add fluent configuration when you have a specific reason.
2. **Use the shortest timeout that works.** Long timeouts slow down failure detection.
3. **Name custom condition functions descriptively.** `checkbox_removed_and_message_shown` is much clearer than `check_condition`.
4. **Make custom conditions reusable.** Use classes (Python) or helper methods (C#) that accept parameters.
5. **Ignore only expected exceptions.** Every ignored exception is a potential bug you will not see.
6. **Log what you are waiting for.** Use the `Message` property to provide context in timeout errors.

---

## Hands-On Exercise

**Goal:** Build a custom wait that handles a common real-world scenario -- waiting for a table to finish loading and contain data.

1. Navigate to `https://the-internet.herokuapp.com/tables`.
2. Create a custom expected condition that:
   - Finds all rows in `#table1 tbody`
   - Returns the list of rows only if there are at least 4 rows
   - Returns `False` (or `null`) otherwise
3. Use a fluent wait with 500ms polling interval.
4. Once the rows are returned, print each person's last name from the first column.
5. Assert that "Smith" appears in the list of last names.

**Expected output:**
```
Found 4 table rows
Last names: Smith, Bach, Doe, Conway
Assertion passed: Smith is in the list
```

---

## Real-World Scenario

Consider a dashboard that loads multiple widgets asynchronously. Each widget fetches data from a different API endpoint. You need to verify that the "Revenue" widget shows a dollar amount.

```python
# The revenue widget might show a loading skeleton, then the actual value
# The element exists immediately but contains "Loading..." text first

class WidgetShowsDollarAmount:
    """Wait until a widget displays a dollar amount (starts with $)."""
    def __init__(self, locator):
        self.locator = locator

    def __call__(self, driver):
        element = driver.find_element(*self.locator)
        text = element.text.strip()
        if text.startswith("$") and len(text) > 1:
            return element
        return False

wait = WebDriverWait(driver, 20, poll_frequency=1)
revenue = wait.until(WidgetShowsDollarAmount((By.ID, "revenue-widget")))
print(f"Revenue: {revenue.text}")
```

Standard `ExpectedConditions` cannot express "text starts with $" -- this is exactly where custom conditions shine.

---

## Resources

- [Selenium Waits Documentation](https://www.selenium.dev/documentation/webdriver/waits/)
- [DefaultWait API (C#)](https://www.selenium.dev/selenium/docs/api/dotnet/html/T_OpenQA_Selenium_Support_UI_DefaultWait_1.htm)
- [WebDriverWait API (Python)](https://www.selenium.dev/selenium/docs/api/py/webdriver_support/selenium.webdriver.support.wait.html)
- [The Internet Herokuapp - Dynamic Controls](https://the-internet.herokuapp.com/dynamic_controls)
