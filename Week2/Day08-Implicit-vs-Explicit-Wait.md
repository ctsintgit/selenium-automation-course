# Day 8: Implicit Wait vs Explicit Wait (WebDriverWait)

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain why timing issues occur in web automation
- Implement implicit waits and understand their drawbacks
- Implement explicit waits using WebDriverWait
- Use built-in ExpectedConditions for common scenarios
- Articulate why explicit waits are always preferred over implicit waits

---

## Core Concept Explanation

Web pages do not load instantly. When Selenium sends a command to find an element, that element may not yet exist in the DOM because JavaScript is still executing, an AJAX call is in progress, or the page has not finished rendering. Without any wait strategy, Selenium will throw a `NoSuchElementException` immediately if the element is not present at the exact millisecond the command runs.

**Waits** tell Selenium to pause execution until a condition is met or a timeout expires. There are two primary approaches:

| Wait Type | Scope | Behavior |
|-----------|-------|----------|
| **Implicit Wait** | Global (applies to every `FindElement` call) | Polls the DOM at regular intervals until the element is found or the timeout expires |
| **Explicit Wait** | Local (applies only where you use it) | Waits for a specific condition on a specific element before proceeding |

A third option, `Thread.Sleep` / `time.sleep`, is a hard pause that always waits the full duration regardless of whether the element appeared earlier. **Never use it in test automation.**

---

## How It Works

### Implicit Wait

When you set an implicit wait, every subsequent `FindElement` call will retry for up to the specified duration before throwing an exception. It is set once and applies globally to the driver instance.

**Problem:** Implicit waits apply to *every* element lookup, including ones you expect to fail. They also mask real performance issues because slow elements silently pass. Mixing implicit and explicit waits can cause unpredictable timeout behavior (timeouts may stack).

### Explicit Wait

An explicit wait wraps a specific condition check in a polling loop. You create a `WebDriverWait` object with a timeout, then call its `Until` method with a condition. The wait polls every 500ms by default and returns as soon as the condition is true, or throws a `TimeoutException` when the timeout expires.

**Why explicit waits win:**
1. You wait only where needed, not everywhere
2. You specify exactly what condition to wait for (visible, clickable, text present, etc.)
3. Timeouts are localized and intentional
4. They make tests self-documenting ("wait until the submit button is clickable")

---

## Code Example: C#

```csharp
// File: Day08_WaitDemo.cs
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

namespace Day08
{
    [TestFixture]
    public class WaitDemo
    {
        private IWebDriver driver;

        [SetUp]
        public void SetUp()
        {
            // Selenium 4 manages ChromeDriver automatically
            driver = new ChromeDriver();
            driver.Manage().Window.Maximize();
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        // --- IMPLICIT WAIT (shown for understanding, NOT recommended) ---
        [Test]
        public void ImplicitWaitExample()
        {
            // Set implicit wait: every FindElement will wait up to 10 seconds
            driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(10);

            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_loading/1");

            // Click the Start button to trigger dynamic content loading
            driver.FindElement(By.CssSelector("#start button")).Click();

            // FindElement will poll for up to 10 seconds until the element appears
            // BUT it only checks for presence in the DOM, NOT visibility
            IWebElement result = driver.FindElement(By.Id("finish"));

            // The element may be in the DOM but still hidden -- implicit wait
            // cannot distinguish between present-and-hidden vs present-and-visible
            Assert.That(result.Text, Does.Contain("Hello World"));
        }

        // --- EXPLICIT WAIT (recommended approach) ---
        [Test]
        public void ExplicitWaitExample()
        {
            // No implicit wait set -- this is intentional
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_loading/1");

            // Click Start to trigger loading
            driver.FindElement(By.CssSelector("#start button")).Click();

            // Create a WebDriverWait with a 10-second timeout
            WebDriverWait wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));

            // Wait until the #finish element is visible (not just in the DOM)
            IWebElement result = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("finish"))
            );

            Assert.That(result.Text, Does.Contain("Hello World"));
        }

        // --- COMMON EXPLICIT WAIT CONDITIONS ---
        [Test]
        public void CommonExplicitWaitConditions()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_loading/1");

            WebDriverWait wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));

            // Wait until element exists in the DOM
            wait.Until(ExpectedConditions.ElementExists(By.CssSelector("#start button")));

            // Wait until element is visible on the page
            IWebElement startBtn = wait.Until(
                ExpectedConditions.ElementIsVisible(By.CssSelector("#start button"))
            );

            // Wait until element is clickable (visible + enabled)
            wait.Until(
                ExpectedConditions.ElementToBeClickable(By.CssSelector("#start button"))
            );
            startBtn.Click();

            // Wait until title contains specific text
            wait.Until(ExpectedConditions.TitleContains("The Internet"));

            // Wait until an element becomes invisible (e.g., a loading spinner)
            wait.Until(
                ExpectedConditions.InvisibilityOfElementLocated(By.Id("loading"))
            );

            // Wait until text is present in element
            wait.Until(
                ExpectedConditions.TextToBePresentInElementLocated(
                    By.Id("finish"), "Hello World"
                )
            );

            IWebElement result = driver.FindElement(By.Id("finish"));
            Assert.That(result.Displayed, Is.True);
        }
    }
}
```

---

## Code Example: Python

```python
# File: test_day08_wait_demo.py
# Prerequisites:
#   pip install selenium pytest

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import pytest


@pytest.fixture
def driver():
    """Create a Chrome browser instance for each test."""
    # Selenium 4 manages ChromeDriver automatically
    drv = webdriver.Chrome()
    drv.maximize_window()
    yield drv
    drv.quit()


# --- IMPLICIT WAIT (shown for understanding, NOT recommended) ---
def test_implicit_wait_example(driver):
    # Set implicit wait: every find_element will wait up to 10 seconds
    driver.implicitly_wait(10)

    driver.get("https://the-internet.herokuapp.com/dynamic_loading/1")

    # Click Start to trigger dynamic content
    driver.find_element(By.CSS_SELECTOR, "#start button").click()

    # find_element will poll for up to 10 seconds until the element is in the DOM
    # BUT it only checks DOM presence, NOT visibility
    result = driver.find_element(By.ID, "finish")

    # May fail because element can be in DOM but hidden
    assert "Hello World" in result.text


# --- EXPLICIT WAIT (recommended approach) ---
def test_explicit_wait_example(driver):
    # No implicit wait set -- intentional
    driver.get("https://the-internet.herokuapp.com/dynamic_loading/1")

    # Click Start to trigger loading
    driver.find_element(By.CSS_SELECTOR, "#start button").click()

    # Create a WebDriverWait with a 10-second timeout
    wait = WebDriverWait(driver, 10)

    # Wait until the #finish element is visible (not just in the DOM)
    result = wait.until(
        EC.visibility_of_element_located((By.ID, "finish"))
    )

    assert "Hello World" in result.text


# --- COMMON EXPLICIT WAIT CONDITIONS ---
def test_common_explicit_wait_conditions(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_loading/1")

    wait = WebDriverWait(driver, 10)

    # Wait until element exists in the DOM
    wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, "#start button")))

    # Wait until element is visible
    start_btn = wait.until(
        EC.visibility_of_element_located((By.CSS_SELECTOR, "#start button"))
    )

    # Wait until element is clickable (visible + enabled)
    wait.until(
        EC.element_to_be_clickable((By.CSS_SELECTOR, "#start button"))
    )
    start_btn.click()

    # Wait until title contains specific text
    wait.until(EC.title_contains("The Internet"))

    # Wait until element becomes invisible (e.g., loading spinner)
    wait.until(
        EC.invisibility_of_element_located((By.ID, "loading"))
    )

    # Wait until text is present in element
    wait.until(
        EC.text_to_be_present_in_element((By.ID, "finish"), "Hello World")
    )

    result = driver.find_element(By.ID, "finish")
    assert result.is_displayed()
```

---

## Step-by-Step Walkthrough

1. **Navigate to a dynamic page** -- The demo page hides content behind a "Start" button that triggers a loading bar.
2. **Click Start** -- JavaScript begins loading content asynchronously.
3. **Wait for the result** -- The `#finish` element is either hidden initially (example 1) or not yet in the DOM (example 2).
4. **Implicit wait approach** -- Sets a blanket timeout on all `FindElement` calls. Works sometimes, but cannot check visibility.
5. **Explicit wait approach** -- Creates a `WebDriverWait` and specifies `ElementIsVisible`. The wait polls every 500ms. As soon as the element becomes visible, execution continues. If 10 seconds pass without visibility, a `TimeoutException` is thrown.
6. **Assert** -- Verify the loaded text matches expectations.

---

## Common Mistakes & How to Avoid Them

### 1. Mixing implicit and explicit waits
```
BAD:  driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(10);
      new WebDriverWait(driver, TimeSpan.FromSeconds(10)).Until(...);
```
Mixing them causes unpredictable behavior. The implicit wait may interfere with the explicit wait's polling, resulting in waits of up to 20 seconds instead of 10. **Pick one strategy and stick with it -- always explicit.**

### 2. Using Thread.Sleep / time.sleep
```
BAD:  Thread.Sleep(5000);  // C#
BAD:  time.sleep(5)        # Python
```
Hard sleeps waste time when the element appears early and fail when it appears late. Explicit waits return immediately once the condition is met.

### 3. Waiting for presence instead of visibility
```csharp
// BAD: element might be in the DOM but display:none
wait.Until(ExpectedConditions.ElementExists(By.Id("finish")));

// GOOD: waits until the element is actually visible
wait.Until(ExpectedConditions.ElementIsVisible(By.Id("finish")));
```

### 4. Setting implicit wait too high
A 30-second implicit wait means every *failed* element lookup takes 30 seconds. If you have 50 negative checks in your test suite, that adds 25 minutes of wasted time.

### 5. Not resetting implicit wait
Once set, implicit wait persists for the driver's lifetime. If you must use it temporarily, reset it to zero afterward.

---

## Best Practices

1. **Use explicit waits exclusively.** Remove all implicit waits from your code.
2. **Create a reusable wait helper.** Centralize your `WebDriverWait` creation so timeout values are consistent.
3. **Choose the right condition.** Use `ElementToBeClickable` before clicking, `ElementIsVisible` before reading text, `InvisibilityOfElementLocated` before verifying something disappeared.
4. **Keep timeouts reasonable.** 10 seconds is a good default. Use longer timeouts only for known slow operations.
5. **Catch `TimeoutException` when appropriate.** Sometimes you want to check if something exists without failing the test.
6. **Name your waits in comments.** Explain *what* you are waiting for to make tests readable.

---

## Hands-On Exercise

**Goal:** Automate the dynamic loading pages on The Internet Herokuapp.

1. Navigate to `https://the-internet.herokuapp.com/dynamic_loading/2` (element is not in the DOM until loading finishes).
2. Click the "Start" button.
3. Use an explicit wait to wait until the loading spinner disappears.
4. Use another explicit wait to wait until the result text is visible.
5. Assert the result text is "Hello World!".
6. Print the time it took from clicking Start to seeing the result.

**Expected output:**
```
Loading completed in approximately 5 seconds
Result text: Hello World!
Test passed.
```

---

## Real-World Scenario

Consider an e-commerce checkout flow:

1. User clicks "Place Order" -- a loading spinner appears.
2. The server processes payment (2-8 seconds depending on the payment gateway).
3. A confirmation page appears with an order number.

Without waits, Selenium would try to read the order number while the spinner is still showing. With an implicit wait, you could detect the element in the DOM but it might still be hidden behind the spinner overlay. With an explicit wait, you can:

```csharp
// Wait for the spinner to disappear
wait.Until(ExpectedConditions.InvisibilityOfElementLocated(By.CssSelector(".loading-spinner")));

// Wait for the order confirmation to be visible
IWebElement orderNumber = wait.Until(
    ExpectedConditions.ElementIsVisible(By.Id("order-confirmation-number"))
);
```

This approach is reliable, self-documenting, and returns as fast as possible.

---

## Resources

- [Selenium Waits Official Docs](https://www.selenium.dev/documentation/webdriver/waits/)
- [The Internet Herokuapp - Dynamic Loading](https://the-internet.herokuapp.com/dynamic_loading)
- [SeleniumExtras.WaitHelpers NuGet](https://www.nuget.org/packages/DotNetSeleniumExtras.WaitHelpers)
- [Python expected_conditions API](https://www.selenium.dev/selenium/docs/api/py/webdriver_support/selenium.webdriver.support.expected_conditions.html)
