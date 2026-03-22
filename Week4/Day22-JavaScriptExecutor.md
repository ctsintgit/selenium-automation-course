# Day 22: JavaScriptExecutor — Scroll, Click, Get Values, Inject JS

## Learning Objectives

By the end of this lesson, you will be able to:

- Understand what JavaScriptExecutor is and when to use it over standard Selenium methods
- Execute JavaScript commands through Selenium in both C# and Python
- Scroll pages in multiple ways: to element, to bottom, by pixel offset
- Click elements that are not interactable via normal Selenium click
- Read element properties such as innerText, value, and custom attributes
- Modify element styles for debugging and visual verification
- Remove overlays and popups that block element interaction
- Extract data from lazy-loaded content after scrolling

---

## 1. Core Concept Explanation

### What Is JavaScriptExecutor?

JavaScriptExecutor is a Selenium interface that lets you run raw JavaScript code directly inside the browser. Every modern browser has a built-in JavaScript engine, and JavaScriptExecutor gives your test code direct access to it.

Think of it this way: normal Selenium commands talk to the browser through the WebDriver protocol. JavaScriptExecutor bypasses that layer and speaks directly to the browser's JavaScript engine, exactly as if you typed code into the browser's Developer Tools console.

### When Should You Use It?

Use JavaScriptExecutor when standard Selenium methods fall short:

| Situation | Why Standard Selenium Fails | JavaScriptExecutor Solution |
|-----------|---------------------------|---------------------------|
| Element is off-screen | Click throws "element not interactable" | Scroll to element first, or JS click |
| Sticky header/footer covers element | Click hits the overlay instead | JS click bypasses overlay |
| Hidden input field | Selenium refuses to type into hidden fields | Set value via JS |
| Need scroll position | No native Selenium scroll API | `window.scrollTo()` or `element.scrollIntoView()` |
| Lazy-loaded content | Content does not exist until you scroll | Scroll via JS, then wait for content |
| Read computed styles | Selenium only reads attributes, not computed CSS | `getComputedStyle()` |
| Remove cookie banners | No Selenium method to remove elements from DOM | `element.remove()` via JS |

### Important Warning

JavaScriptExecutor is powerful but should be your **second choice**, not your first. Always try standard Selenium methods first. JS interactions skip browser-level events (hover, focus) that real users trigger, so overusing JS can hide real bugs in your application.

---

## 2. How It Works (Technical Breakdown)

### The Execution Flow

1. Your test code calls `ExecuteScript()` (C#) or `execute_script()` (Python)
2. Selenium serializes the JavaScript string and sends it to the browser via WebDriver protocol
3. The browser's JS engine executes the code in the context of the current page
4. Any return value is serialized back and returned to your test code

### Passing Arguments

You can pass arguments from your test code into the JavaScript. Inside the JS string, use `arguments[0]`, `arguments[1]`, etc. to reference them.

```
ExecuteScript("arguments[0].click();", someElement)
```

Here, `someElement` (a Selenium WebElement) is passed as `arguments[0]` into the JavaScript. The browser receives the actual DOM element reference.

### Return Values

JavaScript can return values back to your test code:

| JS Return Type | C# Type | Python Type |
|---------------|---------|-------------|
| string | string | str |
| number | long | int or float |
| boolean | bool | bool |
| DOM element | IWebElement | WebElement |
| array | ReadOnlyCollection<object> | list |
| null/undefined | null | None |

---

## 3. Code Example: C# (Complete, Runnable)

```csharp
// File: Day22_JavaScriptExecutor.cs
// Prerequisites:
//   dotnet new nunit -n Day22
//   cd Day22
//   dotnet add package Selenium.WebDriver
//   dotnet add package Selenium.Support

using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;
using System.IO;

namespace Day22
{
    [TestFixture]
    public class JavaScriptExecutorTests
    {
        private IWebDriver driver;
        private IJavaScriptExecutor js;
        private WebDriverWait wait;

        [SetUp]
        public void SetUp()
        {
            // Selenium 4 auto-manages the driver binary
            var options = new ChromeOptions();
            options.AddArgument("--start-maximized");
            driver = new ChromeDriver(options);

            // Cast driver to IJavaScriptExecutor once and reuse
            js = (IJavaScriptExecutor)driver;

            // Explicit wait — never Thread.Sleep
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        // ----- SCROLLING -----

        [Test]
        public void ScrollToBottomOfPage()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/large");

            // Scroll to absolute bottom of the page
            js.ExecuteScript("window.scrollTo(0, document.body.scrollHeight);");

            // Wait until the bottom element is visible
            var bottomElement = wait.Until(d =>
            {
                var el = d.FindElement(By.Id("page-footer"));
                return el.Displayed ? el : null;
            });

            Assert.That(bottomElement, Is.Not.Null, "Footer should be visible after scrolling");
            Console.WriteLine("Successfully scrolled to page bottom.");
        }

        [Test]
        public void ScrollToSpecificElement()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/large");

            // Find an element that is off-screen
            IWebElement table = driver.FindElement(By.Id("large-table"));

            // scrollIntoView with smooth behavior
            js.ExecuteScript(
                "arguments[0].scrollIntoView({behavior: 'smooth', block: 'center'});",
                table
            );

            // Verify element is now in the viewport
            bool isInViewport = (bool)js.ExecuteScript(
                @"var rect = arguments[0].getBoundingClientRect();
                  return (
                      rect.top >= 0 &&
                      rect.left >= 0 &&
                      rect.bottom <= window.innerHeight &&
                      rect.right <= window.innerWidth
                  );",
                table
            );

            Console.WriteLine($"Element in viewport after scroll: {isInViewport}");
        }

        [Test]
        public void ScrollByPixelAmount()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/large");

            // Scroll down by 500 pixels
            js.ExecuteScript("window.scrollBy(0, 500);");

            // Read current scroll position
            long scrollY = (long)js.ExecuteScript("return window.pageYOffset;");
            Console.WriteLine($"Scrolled to Y position: {scrollY}");

            Assert.That(scrollY, Is.GreaterThanOrEqualTo(500),
                "Page should have scrolled down at least 500 pixels");
        }

        // ----- CLICKING -----

        [Test]
        public void ClickElementViaJavaScript()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/challenging_dom");

            // Sometimes normal click fails because an overlay covers the element.
            // JS click bypasses the overlay entirely.
            IWebElement button = driver.FindElement(By.CssSelector(".button"));

            // JS click — fires click event directly on the DOM element
            js.ExecuteScript("arguments[0].click();", button);

            Console.WriteLine("JS click executed successfully on button.");
        }

        // ----- GETTING VALUES -----

        [Test]
        public void GetElementProperties()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

            IWebElement usernameInput = driver.FindElement(By.Id("username"));

            // Type something first so we can read it back
            usernameInput.SendKeys("testuser");

            // Get the value property (what the user typed)
            string inputValue = (string)js.ExecuteScript(
                "return arguments[0].value;", usernameInput
            );
            Console.WriteLine($"Input value: {inputValue}");
            Assert.That(inputValue, Is.EqualTo("testuser"));

            // Get innerText of a label
            IWebElement heading = driver.FindElement(By.TagName("h2"));
            string headingText = (string)js.ExecuteScript(
                "return arguments[0].innerText;", heading
            );
            Console.WriteLine($"Heading text: {headingText}");

            // Get a custom attribute
            string typeAttr = (string)js.ExecuteScript(
                "return arguments[0].getAttribute('type');", usernameInput
            );
            Console.WriteLine($"Input type attribute: {typeAttr}");

            // Get computed style
            string bgColor = (string)js.ExecuteScript(
                "return window.getComputedStyle(arguments[0]).backgroundColor;",
                usernameInput
            );
            Console.WriteLine($"Background color: {bgColor}");
        }

        // ----- CHANGING STYLES -----

        [Test]
        public void HighlightElementForDebugging()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

            IWebElement loginButton = driver.FindElement(
                By.CssSelector("button[type='submit']")
            );

            // Add a bright red border to highlight the element
            js.ExecuteScript(
                "arguments[0].style.border = '3px solid red';",
                loginButton
            );

            // Add background color highlight
            js.ExecuteScript(
                "arguments[0].style.backgroundColor = 'yellow';",
                loginButton
            );

            Console.WriteLine("Element highlighted with red border and yellow background.");

            // You could take a screenshot here for debugging evidence
        }

        // ----- REMOVING OVERLAYS -----

        [Test]
        public void RemoveBlockingOverlay()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/entry_ad");

            // Wait for the modal overlay to appear
            wait.Until(d =>
            {
                try
                {
                    var modal = d.FindElement(By.Id("modal"));
                    return modal.Displayed;
                }
                catch (NoSuchElementException)
                {
                    return false;
                }
            });

            // Remove the modal overlay from the DOM entirely
            js.ExecuteScript(
                "var modal = document.getElementById('modal');" +
                "if (modal) modal.remove();"
            );

            // Also remove any backdrop/overlay divs
            js.ExecuteScript(
                "var overlays = document.querySelectorAll('.modal-backdrop, .overlay');" +
                "overlays.forEach(function(el) { el.remove(); });"
            );

            Console.WriteLine("Overlay removed from DOM.");
        }

        // ----- SETTING VALUES -----

        [Test]
        public void SetHiddenInputValue()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

            IWebElement usernameInput = driver.FindElement(By.Id("username"));

            // Set value directly via JS (useful for hidden or readonly fields)
            js.ExecuteScript("arguments[0].value = 'tomsmith';", usernameInput);

            // Verify the value was set
            string setValue = (string)js.ExecuteScript(
                "return arguments[0].value;", usernameInput
            );
            Assert.That(setValue, Is.EqualTo("tomsmith"));
            Console.WriteLine($"Value set via JS: {setValue}");
        }

        // ----- RETURN COMPLEX DATA -----

        [Test]
        public void GetPageMetaData()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

            // Get page title via JS (alternative to driver.Title)
            string title = (string)js.ExecuteScript("return document.title;");
            Console.WriteLine($"Page title: {title}");

            // Get all links on the page and return their text
            var linkTexts = js.ExecuteScript(
                @"var links = document.querySelectorAll('a');
                  var texts = [];
                  links.forEach(function(link) {
                      if (link.innerText.trim() !== '') {
                          texts.push(link.innerText.trim());
                      }
                  });
                  return texts;"
            );

            // The return type is ReadOnlyCollection<object>
            var collection = (System.Collections.ObjectModel.ReadOnlyCollection<object>)linkTexts;
            Console.WriteLine($"Found {collection.Count} links with text.");
            foreach (var text in collection)
            {
                Console.WriteLine($"  - {text}");
            }
        }

        // ----- ASYNC SCRIPT EXECUTION -----

        [Test]
        public void ExecuteAsyncScript()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

            // Set script timeout
            driver.Manage().Timeouts().AsynchronousJavaScript =
                TimeSpan.FromSeconds(10);

            // ExecuteAsyncScript waits for a callback (the last argument)
            // The callback signals that the async operation is done
            string result = (string)js.ExecuteAsyncScript(
                @"var callback = arguments[arguments.length - 1];
                  setTimeout(function() {
                      callback('Async script completed after delay');
                  }, 1000);"
            );

            Console.WriteLine(result);
            Assert.That(result, Does.Contain("completed"));
        }
    }
}
```

---

## 4. Code Example: Python (Complete, Runnable)

```python
# File: test_day22_javascript_executor.py
# Prerequisites:
#   pip install selenium pytest

import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import time


@pytest.fixture
def driver():
    """Set up Chrome browser. Selenium 4 auto-manages chromedriver."""
    options = Options()
    options.add_argument("--start-maximized")
    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


@pytest.fixture
def wait(driver):
    """Explicit wait instance — never use time.sleep."""
    return WebDriverWait(driver, 10)


# ----- SCROLLING -----

class TestScrolling:

    def test_scroll_to_bottom(self, driver, wait):
        """Scroll to the absolute bottom of a long page."""
        driver.get("https://the-internet.herokuapp.com/large")

        # Scroll to bottom using document.body.scrollHeight
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")

        # Wait until footer is visible
        footer = wait.until(
            EC.visibility_of_element_located((By.ID, "page-footer"))
        )

        assert footer.is_displayed(), "Footer should be visible after scrolling"
        print("Successfully scrolled to page bottom.")

    def test_scroll_to_element(self, driver, wait):
        """Scroll until a specific element is in view."""
        driver.get("https://the-internet.herokuapp.com/large")

        # Find an element that is off-screen
        table = driver.find_element(By.ID, "large-table")

        # scrollIntoView centers the element in the viewport
        driver.execute_script(
            "arguments[0].scrollIntoView({behavior: 'smooth', block: 'center'});",
            table
        )

        # Check if element is in viewport
        is_in_viewport = driver.execute_script("""
            var rect = arguments[0].getBoundingClientRect();
            return (
                rect.top >= 0 &&
                rect.left >= 0 &&
                rect.bottom <= window.innerHeight &&
                rect.right <= window.innerWidth
            );
        """, table)

        print(f"Element in viewport after scroll: {is_in_viewport}")

    def test_scroll_by_pixels(self, driver):
        """Scroll down by a specific number of pixels."""
        driver.get("https://the-internet.herokuapp.com/large")

        # Scroll down by 500 pixels
        driver.execute_script("window.scrollBy(0, 500);")

        # Read current scroll position
        scroll_y = driver.execute_script("return window.pageYOffset;")
        print(f"Scrolled to Y position: {scroll_y}")

        assert scroll_y >= 500, "Page should have scrolled at least 500px"


# ----- CLICKING -----

class TestJSClick:

    def test_click_via_javascript(self, driver):
        """Click an element using JS when normal click is blocked."""
        driver.get("https://the-internet.herokuapp.com/challenging_dom")

        # Find the button
        button = driver.find_element(By.CSS_SELECTOR, ".button")

        # JS click — bypasses overlays and interactability checks
        driver.execute_script("arguments[0].click();", button)

        print("JS click executed successfully on button.")


# ----- GETTING VALUES -----

class TestGetValues:

    def test_get_element_properties(self, driver):
        """Read various properties from elements using JS."""
        driver.get("https://the-internet.herokuapp.com/login")

        username_input = driver.find_element(By.ID, "username")

        # Type something so we can read it back
        username_input.send_keys("testuser")

        # Get the value property
        input_value = driver.execute_script(
            "return arguments[0].value;", username_input
        )
        print(f"Input value: {input_value}")
        assert input_value == "testuser"

        # Get innerText of a heading
        heading = driver.find_element(By.TAG_NAME, "h2")
        heading_text = driver.execute_script(
            "return arguments[0].innerText;", heading
        )
        print(f"Heading text: {heading_text}")

        # Get a custom attribute
        type_attr = driver.execute_script(
            "return arguments[0].getAttribute('type');", username_input
        )
        print(f"Input type attribute: {type_attr}")

        # Get computed style
        bg_color = driver.execute_script(
            "return window.getComputedStyle(arguments[0]).backgroundColor;",
            username_input
        )
        print(f"Background color: {bg_color}")


# ----- CHANGING STYLES -----

class TestStyles:

    def test_highlight_element(self, driver):
        """Add visual highlights to elements for debugging."""
        driver.get("https://the-internet.herokuapp.com/login")

        login_button = driver.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        )

        # Add red border
        driver.execute_script(
            "arguments[0].style.border = '3px solid red';", login_button
        )

        # Add yellow background
        driver.execute_script(
            "arguments[0].style.backgroundColor = 'yellow';", login_button
        )

        print("Element highlighted with red border and yellow background.")


# ----- REMOVING OVERLAYS -----

class TestOverlayRemoval:

    def test_remove_blocking_overlay(self, driver, wait):
        """Remove a modal overlay from the DOM."""
        driver.get("https://the-internet.herokuapp.com/entry_ad")

        # Wait for modal to appear
        wait.until(
            EC.visibility_of_element_located((By.ID, "modal"))
        )

        # Remove the modal from the DOM
        driver.execute_script("""
            var modal = document.getElementById('modal');
            if (modal) modal.remove();
        """)

        # Remove any backdrop overlays too
        driver.execute_script("""
            var overlays = document.querySelectorAll('.modal-backdrop, .overlay');
            overlays.forEach(function(el) { el.remove(); });
        """)

        print("Overlay removed from DOM.")


# ----- SETTING VALUES -----

class TestSetValues:

    def test_set_hidden_input_value(self, driver):
        """Set a value directly on an input via JavaScript."""
        driver.get("https://the-internet.herokuapp.com/login")

        username_input = driver.find_element(By.ID, "username")

        # Set value directly — works even on hidden or readonly fields
        driver.execute_script(
            "arguments[0].value = 'tomsmith';", username_input
        )

        # Verify value was set
        set_value = driver.execute_script(
            "return arguments[0].value;", username_input
        )
        assert set_value == "tomsmith"
        print(f"Value set via JS: {set_value}")


# ----- RETURNING COMPLEX DATA -----

class TestComplexReturns:

    def test_get_page_metadata(self, driver):
        """Get multiple values from the page in one JS call."""
        driver.get("https://the-internet.herokuapp.com/")

        # Get page title via JS
        title = driver.execute_script("return document.title;")
        print(f"Page title: {title}")

        # Get all link texts on the page
        link_texts = driver.execute_script("""
            var links = document.querySelectorAll('a');
            var texts = [];
            links.forEach(function(link) {
                if (link.innerText.trim() !== '') {
                    texts.push(link.innerText.trim());
                }
            });
            return texts;
        """)

        # Returns a Python list
        print(f"Found {len(link_texts)} links with text.")
        for text in link_texts:
            print(f"  - {text}")


# ----- ASYNC SCRIPT -----

class TestAsyncScript:

    def test_execute_async_script(self, driver):
        """Execute an async JS script with a callback."""
        driver.get("https://the-internet.herokuapp.com/")

        # Set script timeout
        driver.set_script_timeout(10)

        # execute_async_script provides a callback as the last argument
        result = driver.execute_async_script("""
            var callback = arguments[arguments.length - 1];
            setTimeout(function() {
                callback('Async script completed after delay');
            }, 1000);
        """)

        print(result)
        assert "completed" in result


# ----- LAZY-LOADED CONTENT EXERCISE -----

class TestLazyLoading:

    def test_scroll_and_extract_lazy_content(self, driver, wait):
        """
        Exercise: Scroll to trigger lazy loading, then extract data.
        Uses infinite scroll page as example.
        """
        driver.get("https://the-internet.herokuapp.com/infinite_scroll")

        paragraphs_before = len(
            driver.find_elements(By.CSS_SELECTOR, ".jscroll-added")
        )
        print(f"Paragraphs before scrolling: {paragraphs_before}")

        # Scroll down multiple times to trigger lazy loading
        for i in range(5):
            driver.execute_script(
                "window.scrollTo(0, document.body.scrollHeight);"
            )
            # Wait for new content to load (explicit wait on element count)
            try:
                wait.until(lambda d: len(
                    d.find_elements(By.CSS_SELECTOR, ".jscroll-added")
                ) > paragraphs_before + i)
            except Exception:
                pass  # Some scrolls may not trigger new content

        paragraphs_after = len(
            driver.find_elements(By.CSS_SELECTOR, ".jscroll-added")
        )
        print(f"Paragraphs after scrolling: {paragraphs_after}")

        # Extract text from all loaded paragraphs
        all_text = driver.execute_script("""
            var paras = document.querySelectorAll('.jscroll-added p');
            var texts = [];
            paras.forEach(function(p) {
                texts.push(p.innerText.substring(0, 50) + '...');
            });
            return texts;
        """)

        print(f"Extracted {len(all_text)} paragraph previews.")
        for preview in all_text[:3]:
            print(f"  {preview}")

        assert paragraphs_after > paragraphs_before, \
            "Lazy loading should have added more content"
```

---

## 5. Step-by-Step Walkthrough

### Scrolling to an Element

1. **Find the element** using any standard locator (`FindElement` / `find_element`)
2. **Call `scrollIntoView()`** via JavaScriptExecutor, passing the element as `arguments[0]`
3. The browser scrolls until the element is within the visible viewport
4. **Optionally verify** the element is visible by checking its bounding rectangle

### JS Click on a Blocked Element

1. **Attempt a normal Selenium click** first
2. If you get `ElementClickInterceptedException`, switch to JS click
3. Pass the element to `ExecuteScript` / `execute_script` and call `arguments[0].click()`
4. The JS click fires the `click` event directly on the DOM element, bypassing any overlay

### Reading Element Properties

1. Use `return arguments[0].propertyName;` to get the property value
2. Common properties: `.value` (input fields), `.innerText` (visible text), `.innerHTML` (HTML markup)
3. For attributes, use `.getAttribute('name')`
4. For CSS, use `getComputedStyle(arguments[0]).propertyName`

---

## 6. Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `return` in JS | Method returns `null` instead of the value | Always write `return someValue;` |
| Using JS click for everything | Hides real UI bugs (overlays, disabled buttons) | Try normal click first, JS click as fallback |
| Setting `.value` without triggering events | Frameworks like React/Angular do not detect the change | Fire `input` and `change` events after setting value |
| Not waiting after scroll | Element exists in DOM but is not rendered yet | Use explicit wait after scrolling |
| Passing wrong argument index | `arguments[1]` when you only passed one element | Count your arguments starting at 0 |
| Using `innerText` on hidden elements | `innerText` returns empty string for hidden elements | Use `textContent` instead |

### Triggering Events After Setting Value via JS

React, Angular, and Vue listen for `input` events, not direct `.value` changes. After setting a value with JS, dispatch the event:

```javascript
arguments[0].value = 'newvalue';
arguments[0].dispatchEvent(new Event('input', { bubbles: true }));
arguments[0].dispatchEvent(new Event('change', { bubbles: true }));
```

---

## 7. Best Practices

1. **Use JS as a fallback, not a default.** Standard Selenium methods simulate real user behavior more accurately.

2. **Create reusable helper methods** for common JS operations:
   ```csharp
   // C# helper
   public void ScrollToElement(IWebElement element)
   {
       ((IJavaScriptExecutor)driver).ExecuteScript(
           "arguments[0].scrollIntoView({block: 'center'});", element);
   }
   ```

3. **Keep JS strings short and focused.** If you need complex logic, consider loading a `.js` file instead of writing inline strings.

4. **Always validate after JS operations.** After a JS click, verify the expected result (page change, element appearance) with an explicit wait.

5. **Log JS operations** for debugging. When a test fails, knowing whether the JS executed is critical.

6. **Handle null returns gracefully.** JS can return `null` or `undefined`, which become `null`/`None` in your test code.

---

## 8. Hands-On Exercise

### Task: Scroll to Lazy-Loaded Content and Extract Data

**Objective:** Navigate to a page with lazy-loaded content, scroll down to trigger loading, then extract all the loaded content.

**Steps:**
1. Navigate to `https://the-internet.herokuapp.com/infinite_scroll`
2. Count the initial number of content blocks
3. Scroll to the bottom of the page 5 times, waiting for new content each time
4. Count the final number of content blocks
5. Extract the first 50 characters of each paragraph
6. Assert that more content loaded after scrolling

**Expected Output:**
```
Paragraphs before scrolling: 0
Paragraphs after scrolling: 5 (or more)
Extracted 5 paragraph previews.
  Lorem ipsum dolor sit amet, consectetur adipisci...
  ...
```

---

## 9. Real-World Scenario

### Scenario: E-Commerce Product Gallery with Lazy Loading

An e-commerce site loads product images only when you scroll near them. The "Load More" button at the bottom is covered by a sticky cookie consent banner.

```python
def load_all_products(driver, wait):
    """Scroll through product gallery, dismiss cookie banner, load all items."""

    # Step 1: Remove cookie consent banner
    driver.execute_script("""
        var banner = document.querySelector('.cookie-consent');
        if (banner) banner.remove();
    """)

    # Step 2: Scroll and click "Load More" until no more products
    while True:
        try:
            load_more = wait.until(
                EC.presence_of_element_located((By.ID, "load-more"))
            )
            # Scroll to the button
            driver.execute_script(
                "arguments[0].scrollIntoView({block: 'center'});",
                load_more
            )
            # JS click in case anything overlaps
            driver.execute_script("arguments[0].click();", load_more)
            # Wait for new products to appear
            wait.until(lambda d: len(
                d.find_elements(By.CSS_SELECTOR, ".product-card")
            ) > current_count)
        except TimeoutException:
            break  # No more "Load More" button — all products loaded

    # Step 3: Extract all product data
    products = driver.execute_script("""
        var cards = document.querySelectorAll('.product-card');
        var data = [];
        cards.forEach(function(card) {
            data.push({
                name: card.querySelector('.title').innerText,
                price: card.querySelector('.price').innerText,
                image: card.querySelector('img').src
            });
        });
        return data;
    """)

    return products
```

---

## 10. Resources

- [Selenium JavaScriptExecutor Documentation](https://www.selenium.dev/documentation/webdriver/interactions/javascript/)
- [MDN: Element.scrollIntoView()](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView)
- [MDN: Window.scrollTo()](https://developer.mozilla.org/en-US/docs/Web/API/Window/scrollTo)
- [MDN: getComputedStyle()](https://developer.mozilla.org/en-US/docs/Web/API/Window/getComputedStyle)
- [The Internet - Heroku Test App](https://the-internet.herokuapp.com/)
