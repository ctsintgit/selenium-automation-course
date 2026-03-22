# Day 24: Screenshots on Failure + Test Evidence Collection

## Learning Objectives

By the end of this lesson, you will be able to:

- Take full-page screenshots programmatically in both C# and Python
- Capture element-level screenshots for targeted verification
- Implement automatic screenshot capture on test failure
- Name screenshots with timestamps and test names for easy identification
- Attach screenshots to HTML and Allure test reports
- Organize test evidence in a clean folder structure
- Understand video recording options for complex test debugging

---

## 1. Core Concept Explanation

### Why Capture Screenshots?

When a test fails in CI/CD at 3 AM, nobody is watching the browser. The test log might say "element not found," but that does not tell you why. A screenshot captures exactly what the browser was showing at the moment of failure:

- Was the page still loading?
- Did an unexpected popup appear?
- Was the element off-screen?
- Did the application show an error message?

Screenshots are **test evidence** — proof that your tests ran correctly in both passing and failing scenarios. Many QA teams require screenshots as part of their test reports for compliance and auditing.

### Types of Screenshots

| Type | What It Captures | When to Use |
|------|-----------------|-------------|
| Full page | The entire visible browser viewport | Test failures, page state verification |
| Element | A single DOM element cropped from the page | Visual regression, component testing |
| Full document | The entire scrollable page (browser-specific) | Long pages, documentation |

### Automatic vs. Manual Screenshots

- **Manual:** You explicitly call the screenshot method at specific points in your test.
- **Automatic:** A teardown hook checks if the test failed and captures a screenshot. This is the preferred approach because you never have to remember to add screenshot calls.

---

## 2. How It Works (Technical Breakdown)

### Screenshot Capture Flow

1. Your test code calls the screenshot API
2. Selenium sends a "take screenshot" command to the browser via WebDriver protocol
3. The browser captures the current viewport as a PNG image
4. The image data is base64-encoded and sent back to Selenium
5. Selenium decodes it and saves it as a file (or returns raw bytes)

### File Naming Strategy

Good screenshot names let you find the right image instantly:

```
TestResults/
  Screenshots/
    2026-03-21_14-30-45_LoginTest_ValidCredentials_PASSED.png
    2026-03-21_14-31-02_LoginTest_InvalidPassword_FAILED.png
    2026-03-21_14-31-15_SearchTest_EmptyQuery_FAILED.png
```

Pattern: `{date}_{time}_{testClass}_{testMethod}_{status}.png`

---

## 3. Code Example: C# (Complete, Runnable)

```csharp
// File: Day24_Screenshots.cs
// Prerequisites:
//   dotnet new nunit -n Day24
//   cd Day24
//   dotnet add package Selenium.WebDriver
//   dotnet add package Selenium.Support

using NUnit.Framework;
using NUnit.Framework.Interfaces;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;
using System.IO;

namespace Day24
{
    // =============================================
    // BASE TEST CLASS WITH AUTO-SCREENSHOT ON FAILURE
    // =============================================
    public class BaseTest
    {
        protected IWebDriver driver;
        protected WebDriverWait wait;

        // Directory where screenshots will be saved
        private static readonly string ScreenshotDir = Path.Combine(
            TestContext.CurrentContext.TestDirectory,
            "TestResults",
            "Screenshots"
        );

        [SetUp]
        public void BaseSetUp()
        {
            var options = new ChromeOptions();
            options.AddArgument("--start-maximized");
            driver = new ChromeDriver(options);
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));

            // Create screenshot directory if it does not exist
            Directory.CreateDirectory(ScreenshotDir);
        }

        [TearDown]
        public void BaseTearDown()
        {
            // Check if the test failed BEFORE quitting the browser
            if (TestContext.CurrentContext.Result.Outcome.Status
                == TestStatus.Failed)
            {
                CaptureScreenshot("FAILED");
                Console.WriteLine("Test FAILED — screenshot captured.");
            }
            else
            {
                // Optional: capture on pass too for evidence
                // CaptureScreenshot("PASSED");
            }

            driver.Quit();
        }

        /// <summary>
        /// Captures a full-page screenshot with a descriptive filename.
        /// </summary>
        protected string CaptureScreenshot(string status)
        {
            try
            {
                // Build descriptive filename
                string testName = TestContext.CurrentContext.Test.Name;
                string timestamp = DateTime.Now.ToString("yyyy-MM-dd_HH-mm-ss");
                string fileName = $"{timestamp}_{testName}_{status}.png";
                string filePath = Path.Combine(ScreenshotDir, fileName);

                // Take screenshot
                Screenshot screenshot =
                    ((ITakesScreenshot)driver).GetScreenshot();
                screenshot.SaveAsFile(filePath);

                Console.WriteLine($"Screenshot saved: {filePath}");

                // Attach to NUnit test context (shows in test reports)
                TestContext.AddTestAttachment(filePath, "Screenshot");

                return filePath;
            }
            catch (Exception ex)
            {
                Console.WriteLine(
                    $"Failed to capture screenshot: {ex.Message}"
                );
                return null;
            }
        }

        /// <summary>
        /// Captures a screenshot of a specific element only.
        /// </summary>
        protected string CaptureElementScreenshot(
            IWebElement element, string label)
        {
            try
            {
                string timestamp = DateTime.Now.ToString("yyyy-MM-dd_HH-mm-ss");
                string testName = TestContext.CurrentContext.Test.Name;
                string fileName =
                    $"{timestamp}_{testName}_{label}_element.png";
                string filePath = Path.Combine(ScreenshotDir, fileName);

                // Element-level screenshot (Selenium 4 feature)
                Screenshot screenshot =
                    ((ITakesScreenshot)element).GetScreenshot();
                screenshot.SaveAsFile(filePath);

                Console.WriteLine($"Element screenshot saved: {filePath}");
                TestContext.AddTestAttachment(filePath, label);

                return filePath;
            }
            catch (Exception ex)
            {
                Console.WriteLine(
                    $"Failed to capture element screenshot: {ex.Message}"
                );
                return null;
            }
        }

        /// <summary>
        /// Highlights an element before taking its screenshot.
        /// Useful for debugging — shows exactly which element was targeted.
        /// </summary>
        protected string CaptureHighlightedScreenshot(
            IWebElement element, string label)
        {
            var js = (IJavaScriptExecutor)driver;

            // Save original style
            string originalStyle = element.GetAttribute("style") ?? "";

            // Highlight with red border
            js.ExecuteScript(
                "arguments[0].style.border = '3px solid red';" +
                "arguments[0].style.backgroundColor = '#ffffcc';",
                element
            );

            // Take full page screenshot showing the highlight
            string filePath = CaptureScreenshot(label);

            // Restore original style
            js.ExecuteScript(
                $"arguments[0].setAttribute('style', '{originalStyle}');",
                element
            );

            return filePath;
        }
    }

    // =============================================
    // TEST CLASS THAT INHERITS AUTO-SCREENSHOT
    // =============================================
    [TestFixture]
    public class ScreenshotDemoTests : BaseTest
    {
        [Test]
        public void TakeManualScreenshot()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            // Take a manual screenshot at any point
            CaptureScreenshot("login_page_loaded");

            Console.WriteLine("Manual screenshot taken successfully.");
        }

        [Test]
        public void TakeElementScreenshot()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            // Find a specific element
            IWebElement loginForm = driver.FindElement(By.Id("login"));

            // Take a screenshot of just that element
            CaptureElementScreenshot(loginForm, "login_form");

            Console.WriteLine("Element screenshot taken successfully.");
        }

        [Test]
        public void TakeHighlightedScreenshot()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            IWebElement usernameField = driver.FindElement(By.Id("username"));

            // Highlight and screenshot
            CaptureHighlightedScreenshot(usernameField, "username_highlighted");

            Console.WriteLine("Highlighted screenshot taken.");
        }

        [Test]
        public void TestThatWillFail()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            // This assertion will FAIL, triggering auto-screenshot
            IWebElement heading = driver.FindElement(By.TagName("h2"));
            Assert.That(heading.Text, Is.EqualTo("This Will Not Match"),
                "Intentional failure to demonstrate auto-screenshot");
        }

        [Test]
        public void FullPageScreenshotWithScroll()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/large"
            );

            // Standard screenshot only captures the viewport.
            // To get the full page, scroll and stitch or use Chrome DevTools.
            // Here we demonstrate taking screenshots at different scroll positions.

            var js = (IJavaScriptExecutor)driver;

            // Get page height
            long pageHeight = (long)js.ExecuteScript(
                "return document.body.scrollHeight;"
            );
            long viewportHeight = (long)js.ExecuteScript(
                "return window.innerHeight;"
            );

            Console.WriteLine($"Page height: {pageHeight}px");
            Console.WriteLine($"Viewport height: {viewportHeight}px");

            // Take screenshots at intervals
            int screenshotCount = 0;
            for (long pos = 0; pos < pageHeight; pos += viewportHeight)
            {
                js.ExecuteScript($"window.scrollTo(0, {pos});");
                CaptureScreenshot($"scroll_position_{screenshotCount}");
                screenshotCount++;
            }

            Console.WriteLine(
                $"Captured {screenshotCount} scroll screenshots."
            );
        }
    }

    // =============================================
    // ENHANCED BASE TEST WITH REPORT INTEGRATION
    // =============================================
    // This shows how you would structure it for ExtentReports.
    // Install: dotnet add package ExtentReports
    // (Commented out to keep the example self-contained)

    /*
    public class ReportingBaseTest
    {
        protected IWebDriver driver;
        private static ExtentReports extent;
        private ExtentTest test;

        [OneTimeSetUp]
        public void GlobalSetUp()
        {
            var htmlReporter = new ExtentSparkReporter(
                "TestResults/TestReport.html"
            );
            extent = new ExtentReports();
            extent.AttachReporter(htmlReporter);
        }

        [SetUp]
        public void SetUp()
        {
            driver = new ChromeDriver();
            test = extent.CreateTest(
                TestContext.CurrentContext.Test.Name
            );
        }

        [TearDown]
        public void TearDown()
        {
            var status = TestContext.CurrentContext.Result.Outcome.Status;

            if (status == TestStatus.Failed)
            {
                // Capture screenshot
                var screenshot = ((ITakesScreenshot)driver).GetScreenshot();
                string base64 = screenshot.AsBase64EncodedString;

                // Attach to ExtentReports as base64 (embedded in HTML)
                test.Fail("Test failed",
                    MediaEntityBuilder
                        .CreateScreenCaptureFromBase64String(base64)
                        .Build()
                );

                test.Fail(
                    TestContext.CurrentContext.Result.Message
                );
            }
            else
            {
                test.Pass("Test passed");
            }

            driver.Quit();
        }

        [OneTimeTearDown]
        public void GlobalTearDown()
        {
            extent.Flush();
        }
    }
    */
}
```

---

## 4. Code Example: Python (Complete, Runnable)

```python
# File: test_day24_screenshots.py
# Prerequisites:
#   pip install selenium pytest
#   Optional: pip install allure-pytest (for Allure reports)

import os
import pytest
from datetime import datetime
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


# =============================================
# SCREENSHOT DIRECTORY SETUP
# =============================================
SCREENSHOT_DIR = os.path.join(
    os.path.dirname(__file__), "TestResults", "Screenshots"
)
os.makedirs(SCREENSHOT_DIR, exist_ok=True)


# =============================================
# CONFTEST.PY HOOKS FOR AUTO-SCREENSHOT
# =============================================
# In a real project, this code goes in conftest.py.
# For this lesson, we include it here for completeness.

@pytest.fixture
def driver():
    """Set up Chrome browser."""
    options = Options()
    options.add_argument("--start-maximized")
    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


@pytest.fixture
def wait(driver):
    """Explicit wait instance."""
    return WebDriverWait(driver, 10)


@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """
    Pytest hook that runs after each test phase (setup, call, teardown).
    Captures a screenshot when a test fails.
    """
    outcome = yield
    report = outcome.get_result()

    # Only act on the 'call' phase (the actual test, not setup/teardown)
    if report.when == "call" and report.failed:
        # Get the driver from the test's fixtures
        driver = item.funcargs.get("driver")
        if driver:
            timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
            test_name = item.name
            filename = f"{timestamp}_{test_name}_FAILED.png"
            filepath = os.path.join(SCREENSHOT_DIR, filename)

            try:
                driver.save_screenshot(filepath)
                print(f"\nScreenshot saved: {filepath}")

                # If using Allure, attach the screenshot
                try:
                    import allure
                    allure.attach.file(
                        filepath,
                        name=f"Failure: {test_name}",
                        attachment_type=allure.attachment_type.PNG
                    )
                except ImportError:
                    pass  # Allure not installed — skip

            except Exception as e:
                print(f"Failed to capture screenshot: {e}")


# =============================================
# HELPER FUNCTIONS
# =============================================

def take_screenshot(driver, name="screenshot"):
    """
    Take a full-page screenshot with timestamp.
    Returns the file path.
    """
    timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
    filename = f"{timestamp}_{name}.png"
    filepath = os.path.join(SCREENSHOT_DIR, filename)

    driver.save_screenshot(filepath)
    print(f"Screenshot saved: {filepath}")
    return filepath


def take_element_screenshot(driver, element, name="element"):
    """
    Take a screenshot of a specific element.
    Selenium 4 supports element-level screenshots.
    """
    timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
    filename = f"{timestamp}_{name}_element.png"
    filepath = os.path.join(SCREENSHOT_DIR, filename)

    # Element screenshot returns PNG bytes
    element.screenshot(filepath)
    print(f"Element screenshot saved: {filepath}")
    return filepath


def take_highlighted_screenshot(driver, element, name="highlighted"):
    """
    Highlight an element with a red border, then take a full screenshot.
    Useful for showing exactly which element was being interacted with.
    """
    # Save original style
    original_style = element.get_attribute("style") or ""

    # Add highlight
    driver.execute_script(
        "arguments[0].style.border = '3px solid red';"
        "arguments[0].style.backgroundColor = '#ffffcc';",
        element
    )

    # Take screenshot
    filepath = take_screenshot(driver, name)

    # Restore original style
    driver.execute_script(
        f"arguments[0].setAttribute('style', '{original_style}');",
        element
    )

    return filepath


# =============================================
# TEST CLASSES
# =============================================

class TestManualScreenshots:

    def test_take_full_page_screenshot(self, driver):
        """Take a manual screenshot of the full viewport."""
        driver.get("https://the-internet.herokuapp.com/login")

        filepath = take_screenshot(driver, "login_page")

        assert os.path.exists(filepath), "Screenshot file should exist"
        assert os.path.getsize(filepath) > 0, "Screenshot should not be empty"
        print(f"Full page screenshot: {filepath}")

    def test_take_element_screenshot(self, driver):
        """Take a screenshot of just one element."""
        driver.get("https://the-internet.herokuapp.com/login")

        login_form = driver.find_element(By.ID, "login")

        filepath = take_element_screenshot(driver, login_form, "login_form")

        assert os.path.exists(filepath)
        print(f"Element screenshot: {filepath}")

    def test_take_highlighted_screenshot(self, driver):
        """Highlight an element and take a screenshot."""
        driver.get("https://the-internet.herokuapp.com/login")

        username_field = driver.find_element(By.ID, "username")

        filepath = take_highlighted_screenshot(
            driver, username_field, "username_field"
        )

        assert os.path.exists(filepath)
        print(f"Highlighted screenshot: {filepath}")


class TestAutoScreenshotOnFailure:

    def test_intentional_failure(self, driver):
        """
        This test WILL FAIL, which triggers the auto-screenshot hook.
        Check the TestResults/Screenshots folder after running.
        """
        driver.get("https://the-internet.herokuapp.com/login")

        heading = driver.find_element(By.TAG_NAME, "h2")

        # This assertion is intentionally wrong
        assert heading.text == "This Will Not Match", \
            "Intentional failure to demonstrate auto-screenshot"

    def test_passing_test(self, driver):
        """This test passes — no screenshot should be taken."""
        driver.get("https://the-internet.herokuapp.com/login")

        heading = driver.find_element(By.TAG_NAME, "h2")
        assert "Login" in heading.text


class TestScrollingScreenshots:

    def test_full_page_via_scrolling(self, driver):
        """
        Capture the full scrollable page by taking screenshots
        at different scroll positions.
        """
        driver.get("https://the-internet.herokuapp.com/large")

        # Get page dimensions
        page_height = driver.execute_script(
            "return document.body.scrollHeight;"
        )
        viewport_height = driver.execute_script(
            "return window.innerHeight;"
        )

        print(f"Page height: {page_height}px")
        print(f"Viewport height: {viewport_height}px")

        # Scroll and capture
        position = 0
        count = 0
        while position < page_height:
            driver.execute_script(f"window.scrollTo(0, {position});")
            take_screenshot(driver, f"scroll_pos_{count}")
            position += viewport_height
            count += 1

        print(f"Captured {count} scroll screenshots.")


class TestScreenshotAsBase64:

    def test_get_screenshot_as_base64(self, driver):
        """
        Get screenshot as base64 string — useful for embedding
        directly in HTML reports without saving to disk.
        """
        driver.get("https://the-internet.herokuapp.com/login")

        # Get base64-encoded screenshot
        base64_screenshot = driver.get_screenshot_as_base64()

        # This string can be embedded in HTML:
        # <img src="data:image/png;base64,{base64_screenshot}" />

        assert len(base64_screenshot) > 0
        print(f"Base64 screenshot length: {len(base64_screenshot)} chars")

    def test_get_screenshot_as_bytes(self, driver):
        """Get screenshot as raw PNG bytes."""
        driver.get("https://the-internet.herokuapp.com/login")

        # Get raw PNG bytes
        png_bytes = driver.get_screenshot_as_png()

        # Verify it is a valid PNG (starts with PNG magic bytes)
        assert png_bytes[:4] == b'\x89PNG'
        print(f"Screenshot size: {len(png_bytes)} bytes")


# =============================================
# CONFTEST.PY FOR A REAL PROJECT
# =============================================
# In a real project, create a conftest.py file with this content:

CONFTEST_EXAMPLE = """
# conftest.py
import os
import pytest
from datetime import datetime
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

SCREENSHOT_DIR = os.path.join(os.path.dirname(__file__),
                              "TestResults", "Screenshots")
os.makedirs(SCREENSHOT_DIR, exist_ok=True)


@pytest.fixture(scope="function")
def driver():
    options = Options()
    options.add_argument("--start-maximized")
    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()

    if report.when == "call" and report.failed:
        driver = item.funcargs.get("driver")
        if driver:
            timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
            filename = f"{timestamp}_{item.name}_FAILED.png"
            filepath = os.path.join(SCREENSHOT_DIR, filename)

            try:
                driver.save_screenshot(filepath)
                print(f"\\nFailure screenshot: {filepath}")
            except Exception as e:
                print(f"Screenshot failed: {e}")
"""
```

---

## 5. Step-by-Step Walkthrough

### Setting Up Auto-Screenshot on Failure

#### C# (NUnit)

1. Create a `BaseTest` class that all test classes inherit from.
2. In `[TearDown]`, check `TestContext.CurrentContext.Result.Outcome.Status`.
3. If the status is `TestStatus.Failed`, call `GetScreenshot()` before `driver.Quit()`.
4. Save with a descriptive filename including timestamp and test name.

#### Python (pytest)

1. Create a `conftest.py` file in your test root directory.
2. Implement the `pytest_runtest_makereport` hook.
3. Check `report.when == "call"` and `report.failed`.
4. Access the driver from `item.funcargs.get("driver")`.
5. Call `driver.save_screenshot(filepath)`.

### Taking Element Screenshots

1. Find the element normally with any locator.
2. Cast it to `ITakesScreenshot` (C#) or call `.screenshot()` (Python).
3. The resulting image is cropped to just that element's bounding rectangle.

---

## 6. Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Calling `driver.Quit()` before screenshot | Browser is closed, screenshot fails | Always take screenshot BEFORE quit |
| Not creating output directory | `FileNotFoundException` / `FileNotFoundError` | Use `Directory.CreateDirectory()` / `os.makedirs(exist_ok=True)` |
| Hardcoding screenshot path | Fails on different machines | Use relative paths from test directory |
| Screenshot in try block only | Missed when exception occurs | Put screenshot in catch/finally or teardown hook |
| Overwriting same filename | Previous screenshots lost | Include timestamp in filename |
| Not wrapping screenshot in try/catch | Screenshot failure crashes test teardown | Always wrap in try/catch |

---

## 7. Best Practices

1. **Always capture screenshots in teardown, not in the test.** Teardown runs even when the test throws an exception.

2. **Include timestamp, test name, and status in the filename.** This makes screenshots self-documenting.

3. **Wrap screenshot capture in try/catch.** If the browser crashed, the screenshot call will also fail — do not let it mask the real error.

4. **Use base64 for HTML reports.** Embedding screenshots as base64 in HTML reports means the report is a single self-contained file.

5. **Clean up old screenshots periodically.** Screenshots accumulate fast. Add a cleanup step to your CI pipeline.

6. **Take screenshots at key milestones, not just failures.** For compliance testing, capture evidence at login, data entry, and submission steps.

7. **Set viewport size explicitly** for consistent screenshots across environments:
   ```python
   driver.set_window_size(1920, 1080)
   ```

---

## 8. Hands-On Exercise

### Task: Implement Auto-Screenshot-on-Failure in a Base Test Class

**Objective:** Create a reusable base test class that automatically captures screenshots when any test fails.

**Requirements:**
1. Screenshots saved to `TestResults/Screenshots/` directory
2. Filename format: `{timestamp}_{testName}_{PASSED|FAILED}.png`
3. Screenshot taken before browser closes
4. Element screenshot helper method available
5. At least one test that intentionally fails to verify the mechanism

**Expected Output:**
```
PASSED: test_passing_test
FAILED: test_intentional_failure
  Screenshot saved: TestResults/Screenshots/2026-03-21_14-30-02_test_intentional_failure_FAILED.png

Results: 1 passed, 1 failed
Check TestResults/Screenshots/ for failure evidence.
```

---

## 9. Real-World Scenario

### Scenario: E-Commerce Checkout Test with Evidence Trail

A payment processing test needs screenshots at every step for audit compliance:

```python
class TestCheckoutEvidence:
    """
    Captures screenshots at each checkout step for audit trail.
    Even passing tests get evidence screenshots.
    """

    def test_complete_checkout(self, driver, wait):
        evidence_dir = os.path.join(SCREENSHOT_DIR, "checkout_evidence")
        os.makedirs(evidence_dir, exist_ok=True)

        # Step 1: Login
        driver.get("https://shop.example.com/login")
        driver.find_element(By.ID, "email").send_keys("test@test.com")
        driver.find_element(By.ID, "password").send_keys("password123")
        driver.find_element(By.ID, "login-btn").click()
        wait.until(EC.url_contains("/dashboard"))
        take_screenshot(driver, "step1_login_complete")

        # Step 2: Add item to cart
        driver.get("https://shop.example.com/products/widget-123")
        driver.find_element(By.ID, "add-to-cart").click()
        wait.until(EC.text_to_be_present_in_element(
            (By.ID, "cart-count"), "1"
        ))
        take_screenshot(driver, "step2_item_added")

        # Step 3: Enter shipping info
        driver.find_element(By.ID, "checkout-btn").click()
        # ... fill shipping form ...
        take_screenshot(driver, "step3_shipping_entered")

        # Step 4: Payment
        # ... fill payment form ...
        take_screenshot(driver, "step4_payment_entered")

        # Step 5: Confirm order
        driver.find_element(By.ID, "place-order").click()
        confirmation = wait.until(
            EC.visibility_of_element_located((By.ID, "confirmation-number"))
        )
        take_screenshot(driver, "step5_order_confirmed")

        # Final evidence: element screenshot of confirmation number
        take_element_screenshot(
            driver, confirmation, "confirmation_number"
        )

        print(f"Order confirmed: {confirmation.text}")
        print(f"Evidence saved to: {evidence_dir}")
```

---

## 10. Resources

- [Selenium Screenshots Documentation](https://www.selenium.dev/documentation/webdriver/interactions/screenshots/)
- [NUnit TestContext API](https://docs.nunit.org/articles/nunit/writing-tests/TestContext.html)
- [pytest Hooks Reference](https://docs.pytest.org/en/latest/reference/reference.html#hooks)
- [Allure Report - Python Integration](https://allurereport.org/docs/pytest/)
- [ExtentReports for .NET](https://www.extentreports.com/docs/versions/5/net/)
- [Video Recording with Selenoid](https://aerokube.com/selenoid/latest/#_video_recording)
