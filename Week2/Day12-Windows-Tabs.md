# Day 12: Multiple Windows and Tabs -- Switching by Handle

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain the window handle concept in Selenium
- Get the current window handle and all open window handles
- Switch between multiple browser windows or tabs
- Open a new tab or window using Selenium 4's NewWindow API
- Close windows and switch back to the original
- Handle popups and new windows opened by link clicks

---

## Core Concept Explanation

Every browser window and tab has a unique **window handle** -- a string identifier assigned by the browser. Selenium can only interact with one window at a time. To interact with a different window or tab, you must switch the driver's focus using the handle.

Key properties:

| Property/Method | C# | Python | Returns |
|----------------|-----|--------|---------|
| Current handle | `driver.CurrentWindowHandle` | `driver.current_window_handle` | `string` -- handle of focused window |
| All handles | `driver.WindowHandles` | `driver.window_handles` | `ReadOnlyCollection<string>` / `list` |
| Switch to window | `driver.SwitchTo().Window(handle)` | `driver.switch_to.window(handle)` | void |
| Open new tab (Selenium 4) | `driver.SwitchTo().NewWindow(WindowType.Tab)` | `driver.switch_to.new_window("tab")` | Switches to new tab |
| Open new window (Selenium 4) | `driver.SwitchTo().NewWindow(WindowType.Window)` | `driver.switch_to.new_window("window")` | Switches to new window |
| Close current window | `driver.Close()` | `driver.close()` | void |
| Quit all windows | `driver.Quit()` | `driver.quit()` | void |

**Important:** `driver.Close()` closes only the currently focused window. After closing, you must switch to another open window, otherwise subsequent commands will fail with `NoSuchWindowException`.

---

## How It Works

1. When the browser starts, there is one window with one handle.
2. When a link opens a new tab/window (via `target="_blank"` or JavaScript), a new handle is created.
3. **The driver stays focused on the original window** even after a new one opens. You must explicitly switch.
4. `driver.WindowHandles` returns all open handles. The new handle is typically the one not in your original set.
5. After switching, all Selenium commands apply to the newly focused window.
6. When done, close the window with `driver.Close()` and switch back to the original handle.

### Selenium 4 NewWindow API

Selenium 4 added `SwitchTo().NewWindow()` which creates a new empty tab or window and automatically switches focus to it. This is cleaner than using JavaScript `window.open()`.

---

## Code Example: C#

```csharp
// File: Day12_WindowsDemo.cs
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
using System.Linq;

namespace Day12
{
    [TestFixture]
    public class WindowsDemo
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

        // --- BASIC WINDOW SWITCHING ---
        [Test]
        public void SwitchBetweenWindows()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/windows");

            // Store the original window handle
            string originalWindow = driver.CurrentWindowHandle;
            TestContext.WriteLine($"Original window handle: {originalWindow}");

            // Verify we start with one window
            Assert.That(driver.WindowHandles.Count, Is.EqualTo(1));

            // Click the link that opens a new window/tab
            driver.FindElement(By.LinkText("Click Here")).Click();

            // Wait until there are 2 windows
            wait.Until(d => d.WindowHandles.Count == 2);

            // Find the new window handle
            string newWindow = driver.WindowHandles
                .First(handle => handle != originalWindow);

            TestContext.WriteLine($"New window handle: {newWindow}");

            // Switch to the new window
            driver.SwitchTo().Window(newWindow);

            // Verify we are on the new page
            wait.Until(ExpectedConditions.TitleContains("New Window"));
            IWebElement heading = driver.FindElement(By.TagName("h3"));
            Assert.That(heading.Text, Is.EqualTo("New Window"));
            TestContext.WriteLine($"New window title: {driver.Title}");

            // Close the new window
            driver.Close();

            // Switch back to the original window
            driver.SwitchTo().Window(originalWindow);

            // Verify we are back on the original page
            Assert.That(driver.Title, Does.Contain("The Internet"));
            TestContext.WriteLine("Successfully switched back to original window");
        }

        // --- SELENIUM 4: OPEN NEW TAB ---
        [Test]
        public void OpenNewTabSelenium4()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");
            string originalWindow = driver.CurrentWindowHandle;

            // Selenium 4: open a new tab and switch to it automatically
            driver.SwitchTo().NewWindow(WindowType.Tab);

            // We are now in the new empty tab
            Assert.That(driver.WindowHandles.Count, Is.EqualTo(2));

            // Navigate to a different page in the new tab
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/tables");

            IWebElement heading = wait.Until(
                ExpectedConditions.ElementIsVisible(By.TagName("h3"))
            );
            TestContext.WriteLine($"New tab title: {driver.Title}");
            Assert.That(heading.Text, Does.Contain("Data Tables"));

            // Close the new tab
            driver.Close();

            // Switch back to original
            driver.SwitchTo().Window(originalWindow);
            Assert.That(driver.Title, Does.Contain("The Internet"));
        }

        // --- SELENIUM 4: OPEN NEW WINDOW ---
        [Test]
        public void OpenNewWindowSelenium4()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");
            string originalWindow = driver.CurrentWindowHandle;

            // Selenium 4: open a new browser window (not a tab)
            driver.SwitchTo().NewWindow(WindowType.Window);

            // Navigate in the new window
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

            IWebElement heading = wait.Until(
                ExpectedConditions.ElementIsVisible(By.TagName("h2"))
            );
            TestContext.WriteLine($"New window page: {heading.Text}");

            // Close the new window
            driver.Close();

            // Switch back
            driver.SwitchTo().Window(originalWindow);
        }

        // --- HANDLING MULTIPLE WINDOWS ---
        [Test]
        public void HandleMultipleWindows()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/windows");
            string originalWindow = driver.CurrentWindowHandle;

            // Open multiple windows by clicking the link several times
            // (Each click may or may not open a new window depending on the site)
            // We will use NewWindow to guarantee multiple windows
            driver.SwitchTo().NewWindow(WindowType.Tab);
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/tables");

            driver.SwitchTo().NewWindow(WindowType.Tab);
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

            TestContext.WriteLine($"Total windows open: {driver.WindowHandles.Count}");
            Assert.That(driver.WindowHandles.Count, Is.EqualTo(3));

            // Iterate through all windows and print their titles
            IList<string> allHandles = driver.WindowHandles;
            foreach (string handle in allHandles)
            {
                driver.SwitchTo().Window(handle);
                TestContext.WriteLine($"  Handle: {handle}, Title: {driver.Title}");
            }

            // Close all windows except the original
            foreach (string handle in allHandles)
            {
                if (handle != originalWindow)
                {
                    driver.SwitchTo().Window(handle);
                    driver.Close();
                }
            }

            // Switch back to original
            driver.SwitchTo().Window(originalWindow);
            Assert.That(driver.WindowHandles.Count, Is.EqualTo(1));
            TestContext.WriteLine("All extra windows closed, back to original");
        }

        // --- WAITING FOR A NEW WINDOW TO OPEN ---
        [Test]
        public void WaitForNewWindow()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/windows");

            string originalWindow = driver.CurrentWindowHandle;
            IList<string> handlesBefore = driver.WindowHandles;

            // Click the link that opens a new window
            driver.FindElement(By.LinkText("Click Here")).Click();

            // Wait for a new window handle to appear
            wait.Until(d => d.WindowHandles.Count > handlesBefore.Count);

            // Find the new handle by comparing before and after
            string newHandle = driver.WindowHandles
                .First(h => !handlesBefore.Contains(h));

            driver.SwitchTo().Window(newHandle);

            // Verify
            wait.Until(ExpectedConditions.TitleContains("New Window"));
            Assert.That(driver.Title, Does.Contain("New Window"));

            // Clean up
            driver.Close();
            driver.SwitchTo().Window(originalWindow);
        }
    }
}
```

---

## Code Example: Python

```python
# File: test_day12_windows_demo.py
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


# --- BASIC WINDOW SWITCHING ---
def test_switch_between_windows(driver):
    driver.get("https://the-internet.herokuapp.com/windows")

    # Store the original window handle
    original_window = driver.current_window_handle
    print(f"Original window handle: {original_window}")

    # Verify we start with one window
    assert len(driver.window_handles) == 1

    # Click the link that opens a new window/tab
    driver.find_element(By.LINK_TEXT, "Click Here").click()

    # Wait until there are 2 windows
    wait = WebDriverWait(driver, 10)
    wait.until(EC.number_of_windows_to_be(2))

    # Find the new window handle
    new_window = [h for h in driver.window_handles if h != original_window][0]
    print(f"New window handle: {new_window}")

    # Switch to the new window
    driver.switch_to.window(new_window)

    # Verify we are on the new page
    wait.until(EC.title_contains("New Window"))
    heading = driver.find_element(By.TAG_NAME, "h3")
    assert heading.text == "New Window"
    print(f"New window title: {driver.title}")

    # Close the new window
    driver.close()

    # Switch back to the original window
    driver.switch_to.window(original_window)

    # Verify we are back
    assert "The Internet" in driver.title
    print("Successfully switched back to original window")


# --- SELENIUM 4: OPEN NEW TAB ---
def test_open_new_tab_selenium4(driver):
    driver.get("https://the-internet.herokuapp.com/")
    original_window = driver.current_window_handle

    # Selenium 4: open a new tab and switch to it automatically
    driver.switch_to.new_window("tab")

    assert len(driver.window_handles) == 2

    # Navigate in the new tab
    driver.get("https://the-internet.herokuapp.com/tables")

    wait = WebDriverWait(driver, 10)
    heading = wait.until(EC.visibility_of_element_located((By.TAG_NAME, "h3")))
    print(f"New tab title: {driver.title}")
    assert "Data Tables" in heading.text

    # Close the new tab
    driver.close()

    # Switch back
    driver.switch_to.window(original_window)
    assert "The Internet" in driver.title


# --- SELENIUM 4: OPEN NEW WINDOW ---
def test_open_new_window_selenium4(driver):
    driver.get("https://the-internet.herokuapp.com/")
    original_window = driver.current_window_handle

    # Selenium 4: open a new browser window
    driver.switch_to.new_window("window")

    driver.get("https://the-internet.herokuapp.com/login")

    wait = WebDriverWait(driver, 10)
    heading = wait.until(EC.visibility_of_element_located((By.TAG_NAME, "h2")))
    print(f"New window page: {heading.text}")

    driver.close()
    driver.switch_to.window(original_window)


# --- HANDLING MULTIPLE WINDOWS ---
def test_handle_multiple_windows(driver):
    driver.get("https://the-internet.herokuapp.com/windows")
    original_window = driver.current_window_handle

    # Open multiple tabs
    driver.switch_to.new_window("tab")
    driver.get("https://the-internet.herokuapp.com/tables")

    driver.switch_to.new_window("tab")
    driver.get("https://the-internet.herokuapp.com/login")

    print(f"Total windows open: {len(driver.window_handles)}")
    assert len(driver.window_handles) == 3

    # Iterate through all windows and print titles
    for handle in driver.window_handles:
        driver.switch_to.window(handle)
        print(f"  Handle: {handle}, Title: {driver.title}")

    # Close all windows except the original
    for handle in driver.window_handles[:]:  # Copy the list since we modify it
        if handle != original_window:
            driver.switch_to.window(handle)
            driver.close()

    # Switch back to original
    driver.switch_to.window(original_window)
    assert len(driver.window_handles) == 1
    print("All extra windows closed, back to original")


# --- WAITING FOR A NEW WINDOW ---
def test_wait_for_new_window(driver):
    driver.get("https://the-internet.herokuapp.com/windows")

    original_window = driver.current_window_handle
    handles_before = set(driver.window_handles)

    # Click the link that opens a new window
    driver.find_element(By.LINK_TEXT, "Click Here").click()

    # Wait for a new window handle to appear
    wait = WebDriverWait(driver, 10)
    wait.until(EC.number_of_windows_to_be(len(handles_before) + 1))

    # Find the new handle by set difference
    new_handle = (set(driver.window_handles) - handles_before).pop()

    driver.switch_to.window(new_handle)

    wait.until(EC.title_contains("New Window"))
    assert "New Window" in driver.title

    driver.close()
    driver.switch_to.window(original_window)


# --- HELPER: WINDOW CONTEXT MANAGER ---
def test_window_context_pattern(driver):
    """Demonstrates a clean pattern for working with new windows."""
    driver.get("https://the-internet.herokuapp.com/windows")
    original_window = driver.current_window_handle

    # Click link to open new window
    driver.find_element(By.LINK_TEXT, "Click Here").click()

    wait = WebDriverWait(driver, 10)
    wait.until(EC.number_of_windows_to_be(2))

    # Use the helper to work in the new window
    new_title = work_in_new_window(driver, original_window)
    print(f"New window title was: {new_title}")

    # We are automatically back to the original window
    assert "The Internet" in driver.title


def work_in_new_window(driver, original_handle):
    """Switch to the new window, do work, close it, switch back."""
    new_handle = [h for h in driver.window_handles if h != original_handle][0]

    driver.switch_to.window(new_handle)
    try:
        wait = WebDriverWait(driver, 10)
        wait.until(EC.title_contains("New Window"))
        title = driver.title
        return title
    finally:
        driver.close()
        driver.switch_to.window(original_handle)


# --- OPENING A LINK IN A NEW TAB WITH KEYBOARD ---
def test_get_window_size_and_position(driver):
    driver.get("https://the-internet.herokuapp.com/")

    # Open a new tab
    driver.switch_to.new_window("tab")
    driver.get("https://the-internet.herokuapp.com/tables")

    # Get window size and position
    size = driver.get_window_size()
    position = driver.get_window_position()

    print(f"Window size: {size['width']}x{size['height']}")
    print(f"Window position: ({position['x']}, {position['y']})")

    # Resize the window
    driver.set_window_size(800, 600)
    new_size = driver.get_window_size()
    assert new_size["width"] == 800

    driver.close()
    original = driver.window_handles[0]
    driver.switch_to.window(original)
```

---

## Step-by-Step Walkthrough

1. **Store the original handle** -- Before anything opens a new window, save `CurrentWindowHandle` / `current_window_handle`. You will need this to switch back.
2. **Trigger the new window** -- Click a link with `target="_blank"`, or use `SwitchTo().NewWindow()` to open a blank tab/window.
3. **Wait for the new handle** -- Use `wait.Until(d => d.WindowHandles.Count == 2)` or `EC.number_of_windows_to_be(2)`. Do not assume the handle appears instantly.
4. **Find the new handle** -- Compare `WindowHandles` against the original handle. The new one is the difference.
5. **Switch to the new window** -- `driver.SwitchTo().Window(newHandle)` moves focus.
6. **Interact with the new window** -- All commands now apply to the new window.
7. **Close and switch back** -- `driver.Close()` closes the current window. Then `driver.SwitchTo().Window(originalHandle)` returns focus.

---

## Common Mistakes & How to Avoid Them

### 1. Not waiting for the new window to open
```python
# BAD: new window may not exist yet
driver.find_element(By.LINK_TEXT, "Click Here").click()
new_handle = driver.window_handles[1]  # IndexError!

# GOOD: wait for the window count to increase
wait.until(EC.number_of_windows_to_be(2))
```

### 2. Forgetting to switch back after closing a window
```csharp
// BAD: after Close(), driver has no focused window
driver.Close();
driver.FindElement(By.Id("something")); // NoSuchWindowException!

// GOOD: switch back immediately after closing
driver.Close();
driver.SwitchTo().Window(originalWindow);
```

### 3. Using Close() when you mean Quit()
`Close()` closes only the current window. `Quit()` closes all windows and ends the driver session. In teardown, always use `Quit()`.

### 4. Assuming window handle order
The order of handles in `WindowHandles` is not guaranteed. Never rely on index position. Use set difference to find new handles.

### 5. Confusing windows and frames
Windows and tabs are top-level browser contexts with separate handles. Frames are embedded documents within a single page. Use `SwitchTo().Window()` for windows and `SwitchTo().Frame()` for frames.

---

## Best Practices

1. **Always store the original handle** before opening new windows.
2. **Use set difference** to find new handles instead of relying on list order.
3. **Wait for the expected number of windows** before switching.
4. **Close and switch back in a finally block** to prevent handle leaks in tests.
5. **Use Selenium 4's NewWindow API** instead of JavaScript `window.open()` -- it is cleaner and switches automatically.
6. **Keep track of all handles** if you open multiple windows. Store them in a dictionary with meaningful names.

---

## Hands-On Exercise

**Goal:** Automate a multi-window workflow.

1. Navigate to `https://the-internet.herokuapp.com/windows`.
2. Store the original window handle.
3. Click "Click Here" to open a new window.
4. Wait for the new window and switch to it.
5. Verify the new window displays "New Window".
6. Open a third tab using Selenium 4 NewWindow and navigate to `https://the-internet.herokuapp.com/tables`.
7. Verify you now have 3 windows.
8. Print the title of each window.
9. Close all windows except the original.
10. Verify only one window remains.

**Expected output:**
```
Original window: The Internet
Window 2: New Window
Window 3: The Internet (tables page)
Total windows: 3
Closing extra windows...
Windows remaining: 1
Back on original page: The Internet
```

---

## Real-World Scenario

Consider an email application where clicking a link in an email opens the linked page in a new tab:

```python
# Reading email in a web mail client
driver.get("https://mail.example.com/inbox")

# Click a link inside an email body
original = driver.current_window_handle
email_link = driver.find_element(By.CSS_SELECTOR, ".email-body a")
email_link.click()

# Wait for the linked page to open in a new tab
wait = WebDriverWait(driver, 10)
wait.until(EC.number_of_windows_to_be(2))

# Switch to the new tab and verify the destination
new_tab = [h for h in driver.window_handles if h != original][0]
driver.switch_to.window(new_tab)

# Verify we landed on the expected page
wait.until(EC.url_contains("example.com/promo"))
print(f"Link opened: {driver.current_url}")

# Close the tab and return to the inbox
driver.close()
driver.switch_to.window(original)
```

---

## Resources

- [Selenium Windows Documentation](https://www.selenium.dev/documentation/webdriver/interactions/windows/)
- [The Internet Herokuapp - Multiple Windows](https://the-internet.herokuapp.com/windows)
- [Selenium 4 New Window API](https://www.selenium.dev/documentation/webdriver/interactions/windows/#create-new-window-or-new-tab-and-switch)
