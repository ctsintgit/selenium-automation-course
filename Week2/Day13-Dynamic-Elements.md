# Day 13: Dynamic Elements -- Changing IDs, AJAX, Lazy Loading

## Learning Objectives

By the end of this lesson, you will be able to:

- Identify elements with dynamically generated IDs and attributes
- Use partial attribute matching strategies (CSS `contains`, `starts-with`, `ends-with`; XPath `contains()`)
- Wait for AJAX calls to complete before interacting with results
- Handle lazy-loaded content by scrolling into view and waiting
- Diagnose and fix StaleElementReferenceException
- Implement retry patterns for flaky element interactions

---

## Core Concept Explanation

Modern web applications are highly dynamic. Elements change after the page loads:

| Challenge | Description | Example |
|-----------|-------------|---------|
| **Dynamic IDs** | Element IDs are generated at runtime and change on each page load | `id="field_abc123"` becomes `id="field_xyz789"` |
| **AJAX updates** | Content loads asynchronously after the initial page render | Search results appearing after typing |
| **Lazy loading** | Content loads only when scrolled into the viewport | Product images, infinite scroll feeds |
| **Stale elements** | A previously found element reference becomes invalid because the DOM was updated | Angular/React re-renders |
| **Dynamic attributes** | Classes, data attributes, or text change based on state | `class="btn btn-loading"` becomes `class="btn btn-success"` |

Static locators like `By.Id("field_abc123")` will fail when the ID changes. You need flexible locator strategies and robust wait patterns.

---

## How It Works

### Flexible Locator Strategies

**CSS Selectors for partial matching:**

| Pattern | CSS Selector | Matches |
|---------|-------------|---------|
| Starts with | `[id^='field_']` | `id="field_abc123"` |
| Ends with | `[id$='_name']` | `id="auto_gen_name"` |
| Contains | `[id*='middle']` | `id="form_middle_section"` |
| Attribute presence | `[data-testid]` | Any element with `data-testid` attribute |

**XPath for partial matching:**

| Pattern | XPath | Matches |
|---------|-------|---------|
| Contains | `//*[contains(@id, 'field')]` | `id="field_abc123"` |
| Starts with | `//*[starts-with(@id, 'field')]` | `id="field_abc123"` |
| Contains text | `//*[contains(text(), 'Submit')]` | `<button>Submit Order</button>` |
| Parent-child | `//div[@class='form']//input` | Input inside a div with class "form" |

### StaleElementReferenceException

This exception occurs when:
1. You find an element and store the reference.
2. The DOM updates (React re-render, AJAX content replacement, page navigation).
3. You try to use the stored reference, but the original DOM node has been replaced.

**Solution:** Re-find the element after the DOM update, or use a retry pattern.

---

## Code Example: C#

```csharp
// File: Day13_DynamicElementsDemo.cs
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
using System.Collections.Generic;

namespace Day13
{
    [TestFixture]
    public class DynamicElementsDemo
    {
        private IWebDriver driver;
        private WebDriverWait wait;

        [SetUp]
        public void SetUp()
        {
            driver = new ChromeDriver();
            driver.Manage().Window.Maximize();
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(15));
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        // --- PARTIAL ATTRIBUTE MATCHING ---
        [Test]
        public void PartialAttributeMatch_CssSelector()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_controls");

            // CSS: attribute starts with
            IWebElement checkbox = wait.Until(
                ExpectedConditions.ElementIsVisible(By.CssSelector("[type^='check']"))
            );
            Assert.That(checkbox.Displayed, Is.True);

            // CSS: attribute contains
            IWebElement container = driver.FindElement(
                By.CssSelector("[id*='checkbox']")
            );
            Assert.That(container, Is.Not.Null);

            // CSS: attribute ends with
            IWebElement button = driver.FindElement(
                By.CssSelector("button[onclick$='swapCheckbox()']")
            );
            // Note: the above works only if onclick attribute exists and ends with that value

            TestContext.WriteLine("Partial CSS selectors working correctly");
        }

        [Test]
        public void PartialAttributeMatch_XPath()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_controls");

            // XPath: contains
            IWebElement example = driver.FindElement(
                By.XPath("//*[contains(@id, 'checkbox-example')]")
            );
            Assert.That(example.Displayed, Is.True);

            // XPath: starts-with
            IWebElement startsWith = driver.FindElement(
                By.XPath("//*[starts-with(@id, 'checkbox')]")
            );
            Assert.That(startsWith, Is.Not.Null);

            // XPath: contains text
            IWebElement removeBtn = driver.FindElement(
                By.XPath("//button[contains(text(), 'Remove')]")
            );
            Assert.That(removeBtn.Text, Does.Contain("Remove"));

            // XPath: parent-child navigation
            IWebElement inputInExample = driver.FindElement(
                By.XPath("//div[@id='input-example']//input")
            );
            Assert.That(inputInExample, Is.Not.Null);

            TestContext.WriteLine("Partial XPath selectors working correctly");
        }

        // --- WAITING FOR AJAX/DYNAMIC CONTENT ---
        [Test]
        public void WaitForDynamicContentAfterAction()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_controls");

            // Click "Remove" to trigger an AJAX call that removes the checkbox
            IWebElement removeBtn = wait.Until(
                ExpectedConditions.ElementToBeClickable(
                    By.CssSelector("#checkbox-example button"))
            );
            removeBtn.Click();

            // Wait for the loading indicator to appear then disappear
            wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("loading"))
            );
            wait.Until(
                ExpectedConditions.InvisibilityOfElementLocated(By.Id("loading"))
            );

            // Wait for the success message
            IWebElement message = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("message"))
            );
            Assert.That(message.Text, Is.EqualTo("It's gone!"));

            // Verify the checkbox is actually gone
            var checkboxes = driver.FindElements(By.CssSelector("#checkbox"));
            Assert.That(checkboxes.Count, Is.EqualTo(0));

            TestContext.WriteLine("Dynamic AJAX content handled successfully");
        }

        // --- HANDLING STALE ELEMENT REFERENCE ---
        [Test]
        public void HandleStaleElementReference()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_controls");

            // Find the checkbox
            IWebElement checkbox = driver.FindElement(
                By.CssSelector("#checkbox-example input[type='checkbox']")
            );
            checkbox.Click(); // Check it

            // Now remove it (DOM will change)
            driver.FindElement(By.CssSelector("#checkbox-example button")).Click();

            // Wait for removal
            wait.Until(
                ExpectedConditions.InvisibilityOfElementLocated(By.Id("loading"))
            );

            // The original 'checkbox' reference is now stale
            Assert.Throws<StaleElementReferenceException>(() =>
            {
                bool _ = checkbox.Displayed; // This will throw!
            });

            TestContext.WriteLine("StaleElementReferenceException caught as expected");

            // Now click "Add" to add the checkbox back
            driver.FindElement(By.CssSelector("#checkbox-example button")).Click();
            wait.Until(
                ExpectedConditions.InvisibilityOfElementLocated(By.Id("loading"))
            );

            // Re-find the checkbox -- the old reference is invalid
            IWebElement newCheckbox = wait.Until(
                ExpectedConditions.ElementIsVisible(
                    By.CssSelector("#checkbox-example input[type='checkbox']"))
            );

            Assert.That(newCheckbox.Displayed, Is.True);
            TestContext.WriteLine("Re-found the checkbox with a fresh reference");
        }

        // --- RETRY PATTERN FOR FLAKY ELEMENTS ---
        [Test]
        public void RetryPatternForFlakiness()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_content");

            // This page has content that changes on each refresh
            // Use a retry pattern to handle potential staleness
            string text = RetryOnStale(() =>
            {
                IWebElement content = driver.FindElement(
                    By.CssSelector(".large-10.columns:first-of-type")
                );
                return content.Text;
            }, maxRetries: 3);

            Assert.That(text.Length, Is.GreaterThan(0));
            TestContext.WriteLine($"Content (with retry): {text.Substring(0, 50)}...");
        }

        /// <summary>
        /// Retry a function up to maxRetries times if StaleElementReferenceException occurs.
        /// </summary>
        private T RetryOnStale<T>(Func<T> action, int maxRetries = 3)
        {
            for (int attempt = 1; attempt <= maxRetries; attempt++)
            {
                try
                {
                    return action();
                }
                catch (StaleElementReferenceException) when (attempt < maxRetries)
                {
                    TestContext.WriteLine(
                        $"StaleElementReferenceException on attempt {attempt}, retrying..."
                    );
                }
            }
            throw new Exception("Max retries exceeded");
        }

        // --- SCROLLING TO LAZY-LOADED CONTENT ---
        [Test]
        public void ScrollToLazyLoadedContent()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/infinite_scroll");

            IJavaScriptExecutor js = (IJavaScriptExecutor)driver;

            // Scroll down to trigger lazy loading
            for (int i = 0; i < 5; i++)
            {
                // Scroll to bottom of page
                js.ExecuteScript("window.scrollTo(0, document.body.scrollHeight);");

                // Wait for new content to load
                wait.Until(d =>
                {
                    var paragraphs = d.FindElements(By.CssSelector(".jscroll-added"));
                    return paragraphs.Count > i;
                });
            }

            // Count loaded paragraphs
            var allParagraphs = driver.FindElements(By.CssSelector(".jscroll-added"));
            TestContext.WriteLine($"Lazy-loaded paragraphs: {allParagraphs.Count}");
            Assert.That(allParagraphs.Count, Is.GreaterThanOrEqualTo(5));
        }

        // --- SCROLL ELEMENT INTO VIEW ---
        [Test]
        public void ScrollElementIntoView()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/large");

            IJavaScriptExecutor js = (IJavaScriptExecutor)driver;

            // The page has a large table. Find an element at the bottom.
            IWebElement lastCell = driver.FindElement(By.Id("sibling-50.3"));

            // Scroll the element into the viewport
            js.ExecuteScript("arguments[0].scrollIntoView({behavior: 'smooth', block: 'center'});", lastCell);

            // Wait for the element to be visible in viewport
            wait.Until(d =>
            {
                bool isInViewport = (bool)js.ExecuteScript(
                    "var rect = arguments[0].getBoundingClientRect();" +
                    "return rect.top >= 0 && rect.bottom <= window.innerHeight;",
                    lastCell
                );
                return isInViewport;
            });

            TestContext.WriteLine($"Scrolled to element: {lastCell.Text}");
            Assert.That(lastCell.Displayed, Is.True);
        }

        // --- WAITING FOR AJAX WITH JQUERY ---
        [Test]
        public void WaitForAjaxToComplete()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_loading/2");
            driver.FindElement(By.CssSelector("#start button")).Click();

            IJavaScriptExecutor js = (IJavaScriptExecutor)driver;

            // Wait until all AJAX calls are complete (jQuery-based sites)
            // This checks if jQuery exists and if active AJAX calls are zero
            wait.Until(d =>
            {
                bool jQueryDefined = (bool)js.ExecuteScript(
                    "return typeof jQuery !== 'undefined';"
                );

                if (!jQueryDefined)
                    return true; // No jQuery = no jQuery AJAX to wait for

                bool ajaxComplete = (bool)js.ExecuteScript(
                    "return jQuery.active === 0;"
                );
                return ajaxComplete;
            });

            // Also wait for the specific element we need
            IWebElement result = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("finish"))
            );

            Assert.That(result.Text, Does.Contain("Hello World"));
        }

        // --- USING data-testid ATTRIBUTES ---
        [Test]
        public void UseDataTestIdAttributes()
        {
            // Many modern apps add data-testid attributes specifically for testing
            // These are stable and do not change with styling updates
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/challenging_dom");

            // When data-testid is not available, use other stable attributes
            // CSS selector with multiple attribute checks
            IWebElement table = driver.FindElement(By.CssSelector("table"));

            // Find cells by content when structure is dynamic
            var cells = driver.FindElements(By.CssSelector("table td"));
            TestContext.WriteLine($"Total table cells: {cells.Count}");

            // Find a specific row using XPath with text content
            IWebElement row = driver.FindElement(
                By.XPath("//td[contains(text(), 'Iuvaret0')]/parent::tr")
            );
            var rowCells = row.FindElements(By.TagName("td"));
            TestContext.WriteLine($"Row cells: {rowCells.Count}");
        }
    }
}
```

---

## Code Example: Python

```python
# File: test_day13_dynamic_elements_demo.py
# Prerequisites:
#   pip install selenium pytest

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import StaleElementReferenceException
import pytest


@pytest.fixture
def driver():
    """Create a Chrome browser instance for each test."""
    drv = webdriver.Chrome()
    drv.maximize_window()
    yield drv
    drv.quit()


# --- PARTIAL ATTRIBUTE MATCHING ---
def test_partial_attribute_match_css(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_controls")

    wait = WebDriverWait(driver, 10)

    # CSS: attribute starts with
    checkbox = wait.until(
        EC.visibility_of_element_located((By.CSS_SELECTOR, "[type^='check']"))
    )
    assert checkbox.is_displayed()

    # CSS: attribute contains
    container = driver.find_element(By.CSS_SELECTOR, "[id*='checkbox']")
    assert container is not None

    print("Partial CSS selectors working correctly")


def test_partial_attribute_match_xpath(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_controls")

    # XPath: contains
    example = driver.find_element(
        By.XPATH, "//*[contains(@id, 'checkbox-example')]"
    )
    assert example.is_displayed()

    # XPath: starts-with
    starts_with = driver.find_element(
        By.XPATH, "//*[starts-with(@id, 'checkbox')]"
    )
    assert starts_with is not None

    # XPath: contains text
    remove_btn = driver.find_element(
        By.XPATH, "//button[contains(text(), 'Remove')]"
    )
    assert "Remove" in remove_btn.text

    # XPath: parent-child navigation
    input_el = driver.find_element(
        By.XPATH, "//div[@id='input-example']//input"
    )
    assert input_el is not None

    print("Partial XPath selectors working correctly")


# --- WAITING FOR AJAX CONTENT ---
def test_wait_for_dynamic_content_after_action(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_controls")

    wait = WebDriverWait(driver, 15)

    # Click "Remove" to trigger AJAX removal of checkbox
    remove_btn = wait.until(
        EC.element_to_be_clickable((By.CSS_SELECTOR, "#checkbox-example button"))
    )
    remove_btn.click()

    # Wait for loading indicator to appear then disappear
    wait.until(EC.visibility_of_element_located((By.ID, "loading")))
    wait.until(EC.invisibility_of_element_located((By.ID, "loading")))

    # Wait for success message
    message = wait.until(EC.visibility_of_element_located((By.ID, "message")))
    assert message.text == "It's gone!"

    # Verify checkbox is gone
    checkboxes = driver.find_elements(By.CSS_SELECTOR, "#checkbox")
    assert len(checkboxes) == 0

    print("Dynamic AJAX content handled successfully")


# --- HANDLING STALE ELEMENT REFERENCE ---
def test_handle_stale_element_reference(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_controls")

    wait = WebDriverWait(driver, 15)

    # Find and click the checkbox
    checkbox = driver.find_element(
        By.CSS_SELECTOR, "#checkbox-example input[type='checkbox']"
    )
    checkbox.click()

    # Remove the checkbox (DOM will change)
    driver.find_element(By.CSS_SELECTOR, "#checkbox-example button").click()

    # Wait for removal
    wait.until(EC.invisibility_of_element_located((By.ID, "loading")))

    # The original checkbox reference is now stale
    with pytest.raises(StaleElementReferenceException):
        _ = checkbox.is_displayed()

    print("StaleElementReferenceException caught as expected")

    # Click "Add" to bring the checkbox back
    driver.find_element(By.CSS_SELECTOR, "#checkbox-example button").click()
    wait.until(EC.invisibility_of_element_located((By.ID, "loading")))

    # Re-find the checkbox with a fresh reference
    new_checkbox = wait.until(
        EC.visibility_of_element_located(
            (By.CSS_SELECTOR, "#checkbox-example input[type='checkbox']")
        )
    )

    assert new_checkbox.is_displayed()
    print("Re-found the checkbox with a fresh reference")


# --- RETRY PATTERN ---
def retry_on_stale(action, max_retries=3):
    """Retry a function up to max_retries times on StaleElementReferenceException."""
    for attempt in range(1, max_retries + 1):
        try:
            return action()
        except StaleElementReferenceException:
            if attempt == max_retries:
                raise
            print(f"Stale element on attempt {attempt}, retrying...")


def test_retry_pattern_for_flakiness(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_content")

    # Use retry pattern to handle potential staleness
    text = retry_on_stale(lambda: driver.find_element(
        By.CSS_SELECTOR, ".large-10.columns"
    ).text)

    assert len(text) > 0
    print(f"Content (with retry): {text[:50]}...")


# --- SCROLLING TO LAZY-LOADED CONTENT ---
def test_scroll_to_lazy_loaded_content(driver):
    driver.get("https://the-internet.herokuapp.com/infinite_scroll")

    wait = WebDriverWait(driver, 15)

    # Scroll down multiple times to trigger lazy loading
    for i in range(5):
        # Scroll to the bottom of the page
        driver.execute_script(
            "window.scrollTo(0, document.body.scrollHeight);"
        )

        # Wait for new content to load
        wait.until(
            lambda d, idx=i: len(
                d.find_elements(By.CSS_SELECTOR, ".jscroll-added")
            ) > idx
        )

    # Count loaded paragraphs
    paragraphs = driver.find_elements(By.CSS_SELECTOR, ".jscroll-added")
    print(f"Lazy-loaded paragraphs: {len(paragraphs)}")
    assert len(paragraphs) >= 5


# --- SCROLL ELEMENT INTO VIEW ---
def test_scroll_element_into_view(driver):
    driver.get("https://the-internet.herokuapp.com/large")

    # Find an element at the bottom of the large page
    last_cell = driver.find_element(By.ID, "sibling-50.3")

    # Scroll the element into the viewport
    driver.execute_script(
        "arguments[0].scrollIntoView({behavior: 'smooth', block: 'center'});",
        last_cell,
    )

    # Wait for the element to be in viewport
    wait = WebDriverWait(driver, 10)
    wait.until(lambda d: d.execute_script(
        "var rect = arguments[0].getBoundingClientRect();"
        "return rect.top >= 0 && rect.bottom <= window.innerHeight;",
        last_cell,
    ))

    print(f"Scrolled to element: {last_cell.text}")
    assert last_cell.is_displayed()


# --- WAITING FOR AJAX WITH JAVASCRIPT ---
def test_wait_for_ajax_to_complete(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_loading/2")
    driver.find_element(By.CSS_SELECTOR, "#start button").click()

    wait = WebDriverWait(driver, 15)

    # Wait for jQuery AJAX to complete (if jQuery is on the page)
    def ajax_complete(d):
        jquery_defined = d.execute_script(
            "return typeof jQuery !== 'undefined';"
        )
        if not jquery_defined:
            return True  # No jQuery means no jQuery AJAX
        return d.execute_script("return jQuery.active === 0;")

    wait.until(ajax_complete)

    # Also wait for the specific element
    result = wait.until(
        EC.visibility_of_element_located((By.ID, "finish"))
    )

    assert "Hello World" in result.text


# --- CUSTOM WAIT: ELEMENT ATTRIBUTE CHANGES ---
def test_wait_for_class_change(driver):
    driver.get("https://the-internet.herokuapp.com/dynamic_controls")

    wait = WebDriverWait(driver, 15)

    # Click Enable
    driver.find_element(By.CSS_SELECTOR, "#input-example button").click()

    # Wait until the input element no longer has the disabled attribute
    def input_is_enabled(d):
        el = d.find_element(By.CSS_SELECTOR, "#input-example input")
        return el if el.is_enabled() else False

    enabled_input = wait.until(input_is_enabled)
    enabled_input.send_keys("Dynamic elements mastered!")

    assert enabled_input.get_attribute("value") == "Dynamic elements mastered!"
    print("Successfully typed into dynamically enabled input")
```

---

## Step-by-Step Walkthrough

1. **Identify the dynamic element** -- Inspect the element in DevTools. Reload the page and see if the ID/class changes.
2. **Choose a stable locator** -- Use partial matching (`contains`, `starts-with`) on the stable part of the attribute. Or use `data-testid` if available.
3. **Wait for the element** -- Use explicit waits. For AJAX content, wait for the loading indicator to disappear, then wait for the content to appear.
4. **Handle staleness** -- If the DOM updates after you find an element, re-find it. Use the retry pattern for repetitive actions.
5. **Scroll for lazy content** -- Use `scrollIntoView()` or `window.scrollTo()` via JavaScript execution, then wait for the content to appear.

---

## Common Mistakes & How to Avoid Them

### 1. Using generated IDs as locators
```python
# BAD: this ID changes every page load
driver.find_element(By.ID, "ember1234")

# GOOD: use a stable partial match or parent-child relationship
driver.find_element(By.CSS_SELECTOR, "[data-testid='username-input']")
driver.find_element(By.XPATH, "//label[text()='Username']/following-sibling::input")
```

### 2. Not waiting after triggering AJAX
```csharp
// BAD: content has not loaded yet
driver.FindElement(By.Id("search-btn")).Click();
var results = driver.FindElements(By.CssSelector(".result-item")); // Empty!

// GOOD: wait for results to appear
driver.FindElement(By.Id("search-btn")).Click();
wait.Until(d => d.FindElements(By.CssSelector(".result-item")).Count > 0);
```

### 3. Catching StaleElementReferenceException too broadly
```python
# BAD: hiding real bugs
try:
    element.click()
except:  # Catches everything
    pass

# GOOD: catch only staleness and retry specifically
try:
    element.click()
except StaleElementReferenceException:
    element = driver.find_element(By.ID, "my-element")
    element.click()
```

### 4. Using Thread.Sleep for AJAX
```python
# BAD: arbitrary sleep
driver.find_element(By.ID, "load-btn").click()
time.sleep(5)  # Might be too short or too long

# GOOD: explicit wait for the expected result
wait.until(EC.visibility_of_element_located((By.ID, "results")))
```

---

## Best Practices

1. **Request `data-testid` attributes** from developers. These are stable, unique, and exist solely for testing.
2. **Use CSS selectors with partial matching** as your first choice for dynamic attributes.
3. **Wait for loading indicators to disappear** before asserting on dynamic content.
4. **Re-find elements after DOM mutations.** Never cache element references across page state changes.
5. **Implement a generic retry helper** for actions that might hit staleness.
6. **Use `FindElements` (plural) for existence checks.** It returns an empty list instead of throwing, making it safe for conditional logic.
7. **Scroll into view before interacting** with elements below the fold.

---

## Hands-On Exercise

**Goal:** Automate the Dynamic Controls page end-to-end.

1. Navigate to `https://the-internet.herokuapp.com/dynamic_controls`.
2. Check the checkbox, then click "Remove" and wait for it to be removed.
3. Assert the message says "It's gone!".
4. Click "Add" and wait for the checkbox to reappear.
5. Assert the message says "It's back!".
6. Click "Enable" on the input and wait for it to become enabled.
7. Type "Automation Complete" into the enabled input.
8. Assert the input value is "Automation Complete".

**Expected output:**
```
Checkbox removed -- message: It's gone!
Checkbox restored -- message: It's back!
Input enabled -- typed: Automation Complete
All dynamic element tests passed!
```

---

## Real-World Scenario

Consider a search-as-you-type feature (like Google Suggest):

```python
search_box = driver.find_element(By.ID, "search-input")
search_box.send_keys("selenium")

wait = WebDriverWait(driver, 10)

# Wait for the suggestions dropdown to appear
suggestions = wait.until(
    EC.visibility_of_element_located((By.CSS_SELECTOR, ".suggestions-dropdown"))
)

# Wait until at least 3 suggestions are loaded
wait.until(
    lambda d: len(d.find_elements(By.CSS_SELECTOR, ".suggestion-item")) >= 3
)

# Click the first suggestion
# Use a fresh find to avoid staleness (suggestions may re-render)
first_suggestion = driver.find_element(By.CSS_SELECTOR, ".suggestion-item:first-child")
first_suggestion.click()

# Wait for search results page to load
wait.until(EC.url_contains("search?q="))
```

This combines several techniques: waiting for AJAX results, counting dynamic elements, and avoiding stale references.

---

## Resources

- [Selenium Waits Documentation](https://www.selenium.dev/documentation/webdriver/waits/)
- [The Internet Herokuapp - Dynamic Controls](https://the-internet.herokuapp.com/dynamic_controls)
- [The Internet Herokuapp - Dynamic Content](https://the-internet.herokuapp.com/dynamic_content)
- [The Internet Herokuapp - Infinite Scroll](https://the-internet.herokuapp.com/infinite_scroll)
- [CSS Attribute Selectors (MDN)](https://developer.mozilla.org/en-US/docs/Web/CSS/Attribute_selectors)
- [XPath Tutorial (W3Schools)](https://www.w3schools.com/xml/xpath_syntax.asp)
