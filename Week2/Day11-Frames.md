# Day 11: Frames and iFrames -- Switching In/Out

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain what frames and iframes are and why websites use them
- Switch into a frame by index, name, or WebElement
- Switch back to the main page using DefaultContent and ParentFrame
- Navigate nested frames (frames within frames)
- Find and interact with elements inside frames
- Handle common frame-related exceptions

---

## Core Concept Explanation

An **iframe** (inline frame) embeds a separate HTML document inside the current page. The embedded document has its own DOM, completely isolated from the parent page. Common uses include:

- Embedded videos (YouTube, Vimeo)
- Payment forms (Stripe, PayPal) -- for PCI compliance isolation
- Third-party widgets (chat, ads, social media)
- Rich text editors (TinyMCE, CKEditor)
- Legacy content integration

**Why this matters for Selenium:** When an element lives inside an iframe, Selenium cannot see it from the parent page's context. You must first switch the driver's focus into the frame, interact with elements there, and then switch back.

Think of it like floors in a building. The driver starts on the ground floor (main page). To access a room on another floor (iframe), you must take the elevator (switch to frame) to that floor first.

---

## How It Works

### Switching Into a Frame

Three ways to switch into a frame:

| Method | When to Use |
|--------|------------|
| **By index** (`0`, `1`, `2`...) | When frames have no name/id. Index is order of appearance in DOM. |
| **By name or ID** (`"frameName"`) | When the iframe has a `name` or `id` attribute. |
| **By WebElement** (find the iframe element first) | Most reliable. Works even when name/id is dynamic. |

### Switching Out of a Frame

| Method | Effect |
|--------|--------|
| `SwitchTo().DefaultContent()` | Returns to the top-level page (main document). Always safe. |
| `SwitchTo().ParentFrame()` | Moves up one level. Useful for nested frames. |

### Frame Hierarchy Example

```
Main Page (default content)
  +-- Frame A
  |     +-- Frame A1 (nested)
  +-- Frame B
```

To reach Frame A1: switch to Frame A, then switch to Frame A1.
To go from Frame A1 back to Frame B: either `DefaultContent()` then switch to Frame B, or `ParentFrame()` twice then switch to Frame B.

---

## Code Example: C#

```csharp
// File: Day11_FramesDemo.cs
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

namespace Day11
{
    [TestFixture]
    public class FramesDemo
    {
        private IWebDriver driver;
        private WebDriverWait wait;

        [SetUp]
        public void SetUp()
        {
            driver = new ChromeDriver();
            driver.Manage().Window.Maximize();
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        // --- SWITCH TO FRAME BY ID/NAME ---
        [Test]
        public void SwitchToFrameByName()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/iframe");

            // The TinyMCE editor is inside an iframe with id="mce_0_ifr"
            // Wait for the frame to be available and switch to it
            wait.Until(ExpectedConditions.FrameToBeAvailableAndSwitchToIt("mce_0_ifr"));

            // Now we are inside the iframe -- find the editor body
            IWebElement editorBody = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("tinymce"))
            );

            // Clear existing text and type new content
            editorBody.Clear();
            editorBody.SendKeys("Typed from Selenium inside an iframe!");

            string editorText = editorBody.Text;
            TestContext.WriteLine($"Editor text: {editorText}");
            Assert.That(editorText, Is.EqualTo("Typed from Selenium inside an iframe!"));

            // Switch back to the main page
            driver.SwitchTo().DefaultContent();

            // Now we can interact with elements on the main page
            IWebElement heading = driver.FindElement(By.TagName("h3"));
            Assert.That(heading.Text, Does.Contain("Editor"));
        }

        // --- SWITCH TO FRAME BY INDEX ---
        [Test]
        public void SwitchToFrameByIndex()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/iframe");

            // Switch to the first (and only) iframe on the page by index 0
            driver.SwitchTo().Frame(0);

            IWebElement editorBody = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("tinymce"))
            );

            string text = editorBody.Text;
            TestContext.WriteLine($"Frame content: {text}");
            Assert.That(text.Length, Is.GreaterThan(0));

            // Switch back to main page
            driver.SwitchTo().DefaultContent();
        }

        // --- SWITCH TO FRAME BY WEBELEMENT ---
        [Test]
        public void SwitchToFrameByElement()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/iframe");

            // Find the iframe element first
            IWebElement iframeElement = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("mce_0_ifr"))
            );

            // Switch using the WebElement -- most reliable approach
            driver.SwitchTo().Frame(iframeElement);

            IWebElement editorBody = driver.FindElement(By.Id("tinymce"));
            editorBody.Clear();
            editorBody.SendKeys("Switched by element reference");

            Assert.That(editorBody.Text, Is.EqualTo("Switched by element reference"));

            driver.SwitchTo().DefaultContent();
        }

        // --- NESTED FRAMES ---
        [Test]
        public void HandleNestedFrames()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/nested_frames");

            // Page structure:
            // Main page
            //   +-- frame[name="frame-top"]
            //   |     +-- frame[name="frame-left"]
            //   |     +-- frame[name="frame-middle"]
            //   |     +-- frame[name="frame-right"]
            //   +-- frame[name="frame-bottom"]

            // Step 1: Switch to the top frame
            driver.SwitchTo().Frame("frame-top");

            // Step 2: Switch to the middle frame (nested inside top)
            driver.SwitchTo().Frame("frame-middle");

            // Now we can read content inside frame-middle
            IWebElement content = driver.FindElement(By.Id("content"));
            TestContext.WriteLine($"Middle frame text: {content.Text}");
            Assert.That(content.Text, Is.EqualTo("MIDDLE"));

            // Step 3: Go back to the parent frame (frame-top)
            driver.SwitchTo().ParentFrame();

            // Step 4: Now switch to frame-left (sibling of frame-middle)
            driver.SwitchTo().Frame("frame-left");

            IWebElement leftContent = driver.FindElement(By.TagName("body"));
            TestContext.WriteLine($"Left frame text: {leftContent.Text}");
            Assert.That(leftContent.Text, Is.EqualTo("LEFT"));

            // Step 5: Go all the way back to the main page
            driver.SwitchTo().DefaultContent();

            // Step 6: Switch to the bottom frame
            driver.SwitchTo().Frame("frame-bottom");

            IWebElement bottomContent = driver.FindElement(By.TagName("body"));
            TestContext.WriteLine($"Bottom frame text: {bottomContent.Text}");
            Assert.That(bottomContent.Text, Is.EqualTo("BOTTOM"));

            // Return to main page
            driver.SwitchTo().DefaultContent();
        }

        // --- WAITING FOR FRAME TO BE AVAILABLE ---
        [Test]
        public void WaitForFrameAndSwitch()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/iframe");

            // FrameToBeAvailableAndSwitchToIt waits AND switches in one step
            // By name/id:
            wait.Until(
                ExpectedConditions.FrameToBeAvailableAndSwitchToIt("mce_0_ifr")
            );

            // We are now inside the frame
            IWebElement body = driver.FindElement(By.Id("tinymce"));
            Assert.That(body, Is.Not.Null);

            driver.SwitchTo().DefaultContent();

            // By locator (By object):
            wait.Until(
                ExpectedConditions.FrameToBeAvailableAndSwitchToIt(
                    By.CssSelector("#mce_0_ifr")
                )
            );

            body = driver.FindElement(By.Id("tinymce"));
            Assert.That(body, Is.Not.Null);

            driver.SwitchTo().DefaultContent();
        }

        // --- COUNTING FRAMES ON A PAGE ---
        [Test]
        public void CountFramesOnPage()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/iframe");

            // Find all iframes on the current page
            var iframes = driver.FindElements(By.TagName("iframe"));
            TestContext.WriteLine($"Number of iframes on page: {iframes.Count}");

            // You can also use JavaScript to count frames
            long frameCount = (long)((IJavaScriptExecutor)driver)
                .ExecuteScript("return window.frames.length;");
            TestContext.WriteLine($"window.frames.length: {frameCount}");
        }
    }
}
```

---

## Code Example: Python

```python
# File: test_day11_frames_demo.py
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
    drv = webdriver.Chrome()
    drv.maximize_window()
    yield drv
    drv.quit()


# --- SWITCH TO FRAME BY ID/NAME ---
def test_switch_to_frame_by_name(driver):
    driver.get("https://the-internet.herokuapp.com/iframe")

    wait = WebDriverWait(driver, 10)

    # Wait for the frame and switch to it in one step
    wait.until(EC.frame_to_be_available_and_switch_to_it("mce_0_ifr"))

    # Now inside the iframe -- find the editor body
    editor_body = wait.until(EC.visibility_of_element_located((By.ID, "tinymce")))

    # Clear existing text and type new content
    editor_body.clear()
    editor_body.send_keys("Typed from Selenium inside an iframe!")

    editor_text = editor_body.text
    print(f"Editor text: {editor_text}")
    assert editor_text == "Typed from Selenium inside an iframe!"

    # Switch back to the main page
    driver.switch_to.default_content()

    # Now we can interact with main page elements
    heading = driver.find_element(By.TAG_NAME, "h3")
    assert "Editor" in heading.text


# --- SWITCH TO FRAME BY INDEX ---
def test_switch_to_frame_by_index(driver):
    driver.get("https://the-internet.herokuapp.com/iframe")

    wait = WebDriverWait(driver, 10)

    # Switch to the first iframe on the page (index 0)
    driver.switch_to.frame(0)

    editor_body = wait.until(
        EC.visibility_of_element_located((By.ID, "tinymce"))
    )

    text = editor_body.text
    print(f"Frame content: {text}")
    assert len(text) > 0

    driver.switch_to.default_content()


# --- SWITCH TO FRAME BY WEBELEMENT ---
def test_switch_to_frame_by_element(driver):
    driver.get("https://the-internet.herokuapp.com/iframe")

    wait = WebDriverWait(driver, 10)

    # Find the iframe element first
    iframe_element = wait.until(
        EC.visibility_of_element_located((By.ID, "mce_0_ifr"))
    )

    # Switch using the WebElement -- most reliable approach
    driver.switch_to.frame(iframe_element)

    editor_body = driver.find_element(By.ID, "tinymce")
    editor_body.clear()
    editor_body.send_keys("Switched by element reference")

    assert editor_body.text == "Switched by element reference"

    driver.switch_to.default_content()


# --- NESTED FRAMES ---
def test_handle_nested_frames(driver):
    driver.get("https://the-internet.herokuapp.com/nested_frames")

    # Page structure:
    # Main page
    #   +-- frame[name="frame-top"]
    #   |     +-- frame[name="frame-left"]
    #   |     +-- frame[name="frame-middle"]
    #   |     +-- frame[name="frame-right"]
    #   +-- frame[name="frame-bottom"]

    # Step 1: Switch to the top frame
    driver.switch_to.frame("frame-top")

    # Step 2: Switch to the middle frame (nested inside top)
    driver.switch_to.frame("frame-middle")

    # Read content inside frame-middle
    content = driver.find_element(By.ID, "content")
    print(f"Middle frame text: {content.text}")
    assert content.text == "MIDDLE"

    # Step 3: Go back to the parent frame (frame-top)
    driver.switch_to.parent_frame()

    # Step 4: Switch to frame-left (sibling of frame-middle)
    driver.switch_to.frame("frame-left")

    left_content = driver.find_element(By.TAG_NAME, "body")
    print(f"Left frame text: {left_content.text}")
    assert left_content.text == "LEFT"

    # Step 5: Go all the way back to the main page
    driver.switch_to.default_content()

    # Step 6: Switch to the bottom frame
    driver.switch_to.frame("frame-bottom")

    bottom_content = driver.find_element(By.TAG_NAME, "body")
    print(f"Bottom frame text: {bottom_content.text}")
    assert bottom_content.text == "BOTTOM"

    driver.switch_to.default_content()


# --- WAITING FOR FRAME ---
def test_wait_for_frame_and_switch(driver):
    driver.get("https://the-internet.herokuapp.com/iframe")

    wait = WebDriverWait(driver, 10)

    # frame_to_be_available_and_switch_to_it accepts name, id, index, or locator
    # By name/id:
    wait.until(EC.frame_to_be_available_and_switch_to_it("mce_0_ifr"))
    body = driver.find_element(By.ID, "tinymce")
    assert body is not None

    driver.switch_to.default_content()

    # By locator tuple:
    wait.until(
        EC.frame_to_be_available_and_switch_to_it((By.CSS_SELECTOR, "#mce_0_ifr"))
    )
    body = driver.find_element(By.ID, "tinymce")
    assert body is not None

    driver.switch_to.default_content()


# --- COUNTING FRAMES ---
def test_count_frames_on_page(driver):
    driver.get("https://the-internet.herokuapp.com/iframe")

    # Find all iframes using find_elements
    iframes = driver.find_elements(By.TAG_NAME, "iframe")
    print(f"Number of iframes on page: {len(iframes)}")

    # Using JavaScript
    frame_count = driver.execute_script("return window.frames.length;")
    print(f"window.frames.length: {frame_count}")


# --- HELPER: INTERACT WITH ELEMENT INSIDE FRAME ---
def test_frame_interaction_helper(driver):
    """Demonstrates a pattern for cleanly working with frames."""
    driver.get("https://the-internet.herokuapp.com/iframe")

    wait = WebDriverWait(driver, 10)

    # Pattern: switch in, do work, switch back
    text = interact_with_frame(
        driver, wait, "mce_0_ifr",
        action=lambda d, w: get_editor_text(d, w)
    )

    print(f"Got text from frame: {text}")
    assert len(text) > 0

    # Verify we are back on the main page
    heading = driver.find_element(By.TAG_NAME, "h3")
    assert "Editor" in heading.text


def interact_with_frame(driver, wait, frame_ref, action):
    """Switch into a frame, perform an action, switch back, return result."""
    wait.until(EC.frame_to_be_available_and_switch_to_it(frame_ref))
    try:
        result = action(driver, wait)
    finally:
        driver.switch_to.default_content()
    return result


def get_editor_text(driver, wait):
    """Read text from the TinyMCE editor body."""
    body = wait.until(EC.visibility_of_element_located((By.ID, "tinymce")))
    return body.text
```

---

## Step-by-Step Walkthrough

1. **Navigate to a page with iframes** -- The Internet Herokuapp provides pages with single iframes (editor) and nested frames.
2. **Identify the iframe** -- Inspect the page source to find `<iframe>` tags and note their `id`, `name`, or position.
3. **Wait and switch** -- Use `FrameToBeAvailableAndSwitchToIt` which waits for the frame to load and switches in a single step.
4. **Interact with elements** -- Once switched, all `FindElement` calls search within the iframe's DOM.
5. **Switch back** -- Call `DefaultContent()` to return to the main page, or `ParentFrame()` to go up one level.
6. **For nested frames** -- Switch level by level. You cannot jump directly from the main page to a deeply nested frame.

---

## Common Mistakes & How to Avoid Them

### 1. Trying to find an element inside a frame without switching first
```python
# BAD: element is inside an iframe, but driver context is the main page
driver.find_element(By.ID, "tinymce")  # NoSuchElementException!

# GOOD: switch to the frame first
driver.switch_to.frame("mce_0_ifr")
driver.find_element(By.ID, "tinymce")  # Works!
```

### 2. Forgetting to switch back before interacting with the main page
```csharp
// After working inside a frame, you MUST switch back
driver.SwitchTo().Frame("mce_0_ifr");
// ... do work in frame ...

// BAD: trying to find a main-page element while still in the frame
driver.FindElement(By.Id("main-page-element")); // NoSuchElementException!

// GOOD: switch back first
driver.SwitchTo().DefaultContent();
driver.FindElement(By.Id("main-page-element")); // Works!
```

### 3. Using the wrong index
Frame indices start at 0 and reflect DOM order, not visual order. If the page structure changes, indices break. Prefer using name/id or element reference.

### 4. Switching to a nested frame from the wrong level
```python
# BAD: trying to switch to a nested frame directly from main page
driver.switch_to.default_content()
driver.switch_to.frame("frame-middle")  # NoSuchFrameException!

# GOOD: switch level by level
driver.switch_to.frame("frame-top")     # First, enter the parent
driver.switch_to.frame("frame-middle")  # Then, enter the child
```

### 5. Not waiting for the frame to load
If the iframe is dynamically added to the page, switching immediately will fail. Always use `FrameToBeAvailableAndSwitchToIt`.

---

## Best Practices

1. **Always use `FrameToBeAvailableAndSwitchToIt`** instead of bare `SwitchTo().Frame()`. It handles timing.
2. **Prefer switching by element** over index or name. It is the most reliable method when IDs are dynamic.
3. **Always switch back to DefaultContent** in a finally block or teardown to avoid contaminating the next test.
4. **Create helper methods** that switch in, perform an action, and switch back. This prevents forgetting to switch back.
5. **Use `ParentFrame()`** for nested frames instead of `DefaultContent()` + re-navigating through all parent frames.
6. **Count frames** at the start if you are unsure of the page structure. Use `driver.find_elements(By.TAG_NAME, "iframe")`.

---

## Hands-On Exercise

**Goal:** Navigate the nested frames page and extract text from every frame.

1. Navigate to `https://the-internet.herokuapp.com/nested_frames`.
2. Extract text from each of the four frames: LEFT, MIDDLE, RIGHT, BOTTOM.
3. Store the text in a dictionary/map.
4. Print all frame texts.
5. Assert that all four values are correct.

**Expected output:**
```
Frame texts:
  frame-left: LEFT
  frame-middle: MIDDLE
  frame-right: RIGHT
  frame-bottom: BOTTOM

All frame assertions passed!
```

---

## Real-World Scenario

Payment forms frequently use iframes for security. Stripe, for example, places the credit card input inside an iframe so the merchant's JavaScript cannot access card data.

```python
# Switch to the Stripe card iframe
wait.until(
    EC.frame_to_be_available_and_switch_to_it(
        (By.CSS_SELECTOR, "iframe[name^='__privateStripeFrame']")
    )
)

# Now you can interact with the card number input inside the iframe
card_input = wait.until(
    EC.element_to_be_clickable((By.CSS_SELECTOR, "input[name='cardnumber']"))
)
card_input.send_keys("4242424242424242")

# Switch back to main page for expiry/CVC (may be in separate iframes)
driver.switch_to.default_content()
```

This is one of the most common iframe scenarios in real test automation work.

---

## Resources

- [Selenium Frames Documentation](https://www.selenium.dev/documentation/webdriver/interactions/frames/)
- [The Internet Herokuapp - iFrames](https://the-internet.herokuapp.com/iframe)
- [The Internet Herokuapp - Nested Frames](https://the-internet.herokuapp.com/nested_frames)
- [MDN: iframe Element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe)
