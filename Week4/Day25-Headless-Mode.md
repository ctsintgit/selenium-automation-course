# Day 25: Headless Mode — Chrome and Firefox Headless Execution

## Learning Objectives

By the end of this lesson, you will be able to:

- Understand what headless mode is and why it matters for CI/CD
- Configure Chrome headless mode using Selenium 4 options in C# and Python
- Configure Firefox headless mode
- Handle headless-specific quirks like viewport size and font rendering
- Take screenshots in headless mode (yes, it works)
- Compare performance between headed and headless execution
- Troubleshoot common headless-only failures

---

## 1. Core Concept Explanation

### What Is Headless Mode?

Headless mode runs the browser **without a visible window**. The browser engine still loads pages, executes JavaScript, renders CSS, and processes network requests — it just does not draw pixels to a screen.

Think of it as a browser running in a dark room. Everything works exactly the same, but nobody is watching.

### Why Use Headless Mode?

| Reason | Explanation |
|--------|-------------|
| **CI/CD servers** | Build servers typically have no monitor or GUI. Headless is the only option. |
| **Speed** | No screen rendering means ~20-30% faster execution |
| **Resource efficiency** | Lower CPU and memory usage without GPU rendering |
| **Parallel execution** | Run many browser instances without overwhelming the display server |
| **Docker containers** | Containers usually have no X server or display |

### When NOT to Use Headless Mode

- **Debugging test failures** — you need to see what is happening
- **Visual regression testing** — subtle rendering differences between headed and headless
- **Tests that depend on window focus** — some OS-level focus behaviors differ
- **First-time test development** — always develop with a visible browser

### The "--headless=new" Flag

Selenium 4 / Chrome 109+ introduced `--headless=new`, replacing the old `--headless` flag. The new headless mode is a full browser that simply does not create a window, whereas the old mode used a separate rendering pipeline with known differences.

**Always use `--headless=new`** (not `--headless`). The old mode is deprecated.

---

## 2. How It Works (Technical Breakdown)

### Architecture Difference

**Headed mode:**
```
Your Test --> WebDriver Protocol --> Browser Process --> GPU --> Screen
```

**Headless mode:**
```
Your Test --> WebDriver Protocol --> Browser Process --> Virtual Framebuffer
```

In headless mode, the browser renders to an in-memory framebuffer instead of the screen. This framebuffer is what screenshots capture.

### Default Viewport Size

In headed mode, the browser opens with a default window size (often 1024x768 or whatever the OS provides). In headless mode, Chrome defaults to **800x600** unless you specify otherwise.

This smaller viewport can cause:
- Elements that appear side-by-side in headed mode to stack vertically
- Responsive layouts switching to mobile view
- "Element not interactable" errors because elements are off-screen

**Always set viewport size explicitly in headless mode.**

---

## 3. Code Example: C# (Complete, Runnable)

```csharp
// File: Day25_HeadlessMode.cs
// Prerequisites:
//   dotnet new nunit -n Day25
//   cd Day25
//   dotnet add package Selenium.WebDriver
//   dotnet add package Selenium.Support

using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Firefox;
using OpenQA.Selenium.Support.UI;
using System;
using System.Diagnostics;
using System.IO;

namespace Day25
{
    [TestFixture]
    public class ChromeHeadlessTests
    {
        private IWebDriver driver;
        private WebDriverWait wait;

        [SetUp]
        public void SetUp()
        {
            var options = new ChromeOptions();

            // Enable headless mode (new headless, Chrome 109+)
            options.AddArgument("--headless=new");

            // CRITICAL: Set window size explicitly
            // Without this, Chrome headless defaults to 800x600
            options.AddArgument("--window-size=1920,1080");

            // Recommended headless arguments
            options.AddArgument("--disable-gpu");
            options.AddArgument("--no-sandbox");
            options.AddArgument("--disable-dev-shm-usage");

            // Optional: disable images for faster execution
            // options.AddArgument("--blink-settings=imagesEnabled=false");

            driver = new ChromeDriver(options);
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        [Test]
        public void BasicHeadlessNavigation()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            string title = driver.Title;
            Console.WriteLine($"Page title: {title}");

            Assert.That(title, Does.Contain("Internet"));
            Console.WriteLine("Headless navigation successful!");
        }

        [Test]
        public void HeadlessScreenshotStillWorks()
        {
            // Screenshots work in headless mode — the browser renders
            // to an in-memory framebuffer
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            string screenshotDir = Path.Combine(
                TestContext.CurrentContext.TestDirectory,
                "TestResults", "Screenshots"
            );
            Directory.CreateDirectory(screenshotDir);

            string filePath = Path.Combine(
                screenshotDir, "headless_screenshot.png"
            );

            Screenshot screenshot =
                ((ITakesScreenshot)driver).GetScreenshot();
            screenshot.SaveAsFile(filePath);

            Assert.That(File.Exists(filePath), Is.True);
            Assert.That(new FileInfo(filePath).Length, Is.GreaterThan(0));
            Console.WriteLine($"Headless screenshot saved: {filePath}");
        }

        [Test]
        public void VerifyViewportSize()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            var js = (IJavaScriptExecutor)driver;

            long width = (long)js.ExecuteScript(
                "return window.innerWidth;"
            );
            long height = (long)js.ExecuteScript(
                "return window.innerHeight;"
            );

            Console.WriteLine($"Viewport: {width}x{height}");

            // Should be close to 1920x1080 (minus scrollbar)
            Assert.That(width, Is.GreaterThanOrEqualTo(1900));
            Assert.That(height, Is.GreaterThanOrEqualTo(1000));
        }

        [Test]
        public void HeadlessFormInteraction()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            // All normal Selenium interactions work in headless mode
            IWebElement username = driver.FindElement(By.Id("username"));
            IWebElement password = driver.FindElement(By.Id("password"));
            IWebElement loginBtn = driver.FindElement(
                By.CssSelector("button[type='submit']")
            );

            username.SendKeys("tomsmith");
            password.SendKeys("SuperSecretPassword!");
            loginBtn.Click();

            // Wait for success message
            IWebElement flash = wait.Until(d =>
            {
                var el = d.FindElement(By.Id("flash"));
                return el.Displayed ? el : null;
            });

            Console.WriteLine($"Flash message: {flash.Text}");
            Assert.That(flash.Text, Does.Contain("You logged into"));
        }

        [Test]
        public void HeadlessJavaScriptExecution()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            var js = (IJavaScriptExecutor)driver;

            // JS execution works identically in headless mode
            string userAgent = (string)js.ExecuteScript(
                "return navigator.userAgent;"
            );

            Console.WriteLine($"User agent: {userAgent}");

            // Note: headless Chrome may include "HeadlessChrome" in UA
            // Some sites block headless browsers based on this
            // You can override the user agent:
            // options.AddArgument("--user-agent=Mozilla/5.0 ...");
        }
    }

    [TestFixture]
    public class FirefoxHeadlessTests
    {
        private IWebDriver driver;

        [SetUp]
        public void SetUp()
        {
            var options = new FirefoxOptions();

            // Firefox headless mode
            options.AddArgument("--headless");

            // Set window size
            options.AddArgument("--width=1920");
            options.AddArgument("--height=1080");

            driver = new FirefoxDriver(options);
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        [Test]
        public void FirefoxHeadlessNavigation()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            string title = driver.Title;
            Console.WriteLine($"Firefox headless title: {title}");

            Assert.That(title, Does.Contain("Internet"));
        }
    }

    [TestFixture]
    public class PerformanceComparison
    {
        [Test]
        public void CompareHeadedVsHeadless()
        {
            string url = "https://the-internet.herokuapp.com/login";

            // --- HEADLESS RUN ---
            var headlessOptions = new ChromeOptions();
            headlessOptions.AddArgument("--headless=new");
            headlessOptions.AddArgument("--window-size=1920,1080");
            headlessOptions.AddArgument("--disable-gpu");

            var headlessStopwatch = Stopwatch.StartNew();

            var headlessDriver = new ChromeDriver(headlessOptions);
            headlessDriver.Navigate().GoToUrl(url);

            var headlessWait = new WebDriverWait(
                headlessDriver, TimeSpan.FromSeconds(10)
            );
            headlessWait.Until(d => d.FindElement(By.Id("username")));

            headlessDriver.FindElement(By.Id("username"))
                .SendKeys("tomsmith");
            headlessDriver.FindElement(By.Id("password"))
                .SendKeys("SuperSecretPassword!");
            headlessDriver.FindElement(
                By.CssSelector("button[type='submit']")
            ).Click();

            headlessWait.Until(d =>
            {
                try { return d.FindElement(By.Id("flash")).Displayed; }
                catch { return false; }
            });

            headlessStopwatch.Stop();
            long headlessMs = headlessStopwatch.ElapsedMilliseconds;
            headlessDriver.Quit();

            // --- HEADED RUN ---
            var headedOptions = new ChromeOptions();
            headedOptions.AddArgument("--window-size=1920,1080");

            var headedStopwatch = Stopwatch.StartNew();

            var headedDriver = new ChromeDriver(headedOptions);
            headedDriver.Navigate().GoToUrl(url);

            var headedWait = new WebDriverWait(
                headedDriver, TimeSpan.FromSeconds(10)
            );
            headedWait.Until(d => d.FindElement(By.Id("username")));

            headedDriver.FindElement(By.Id("username"))
                .SendKeys("tomsmith");
            headedDriver.FindElement(By.Id("password"))
                .SendKeys("SuperSecretPassword!");
            headedDriver.FindElement(
                By.CssSelector("button[type='submit']")
            ).Click();

            headedWait.Until(d =>
            {
                try { return d.FindElement(By.Id("flash")).Displayed; }
                catch { return false; }
            });

            headedStopwatch.Stop();
            long headedMs = headedStopwatch.ElapsedMilliseconds;
            headedDriver.Quit();

            // --- COMPARISON ---
            Console.WriteLine($"Headless time: {headlessMs}ms");
            Console.WriteLine($"Headed time:   {headedMs}ms");
            Console.WriteLine(
                $"Difference:    {headedMs - headlessMs}ms " +
                $"({((double)(headedMs - headlessMs) / headedMs * 100):F1}%)"
            );
        }
    }

    [TestFixture]
    public class HeadlessConfigurableTests
    {
        // Pattern: read headless flag from environment variable
        // so CI runs headless, local development runs headed

        private IWebDriver driver;

        [SetUp]
        public void SetUp()
        {
            var options = new ChromeOptions();
            options.AddArgument("--window-size=1920,1080");

            // Check environment variable
            string headless = Environment.GetEnvironmentVariable(
                "SELENIUM_HEADLESS"
            );
            bool isHeadless = headless?.ToLower() == "true"
                           || headless == "1";

            if (isHeadless)
            {
                options.AddArgument("--headless=new");
                options.AddArgument("--disable-gpu");
                options.AddArgument("--no-sandbox");
                Console.WriteLine("Running in HEADLESS mode");
            }
            else
            {
                Console.WriteLine("Running in HEADED mode");
            }

            driver = new ChromeDriver(options);
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        [Test]
        public void ConfigDrivenHeadlessTest()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            Assert.That(driver.Title, Does.Contain("Internet"));
            Console.WriteLine(
                $"Test passed in " +
                $"{(driver is ChromeDriver ? "Chrome" : "unknown")} mode"
            );
        }
    }
}
```

---

## 4. Code Example: Python (Complete, Runnable)

```python
# File: test_day25_headless_mode.py
# Prerequisites:
#   pip install selenium pytest

import os
import time
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options as ChromeOptions
from selenium.webdriver.firefox.options import Options as FirefoxOptions
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


# =============================================
# FIXTURES
# =============================================

@pytest.fixture
def headless_chrome():
    """Chrome in headless mode."""
    options = ChromeOptions()

    # Enable new headless mode (Chrome 109+)
    options.add_argument("--headless=new")

    # CRITICAL: Set viewport size explicitly
    # Chrome headless defaults to 800x600 without this
    options.add_argument("--window-size=1920,1080")

    # Recommended arguments for headless stability
    options.add_argument("--disable-gpu")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")

    driver = webdriver.Chrome(options=options)
    yield driver
    driver.quit()


@pytest.fixture
def headless_firefox():
    """Firefox in headless mode."""
    options = FirefoxOptions()
    options.add_argument("--headless")
    options.add_argument("--width=1920")
    options.add_argument("--height=1080")

    driver = webdriver.Firefox(options=options)
    yield driver
    driver.quit()


@pytest.fixture
def headed_chrome():
    """Chrome in regular headed mode (for comparison)."""
    options = ChromeOptions()
    options.add_argument("--window-size=1920,1080")
    driver = webdriver.Chrome(options=options)
    yield driver
    driver.quit()


@pytest.fixture
def config_driven_driver():
    """
    Read headless setting from environment variable.
    Set SELENIUM_HEADLESS=true in CI, leave unset for local dev.
    """
    options = ChromeOptions()
    options.add_argument("--window-size=1920,1080")

    is_headless = os.environ.get("SELENIUM_HEADLESS", "").lower() in (
        "true", "1", "yes"
    )

    if is_headless:
        options.add_argument("--headless=new")
        options.add_argument("--disable-gpu")
        options.add_argument("--no-sandbox")
        print("Running in HEADLESS mode")
    else:
        print("Running in HEADED mode")

    driver = webdriver.Chrome(options=options)
    yield driver
    driver.quit()


# =============================================
# CHROME HEADLESS TESTS
# =============================================

class TestChromeHeadless:

    def test_basic_navigation(self, headless_chrome):
        """Basic page navigation works in headless mode."""
        headless_chrome.get("https://the-internet.herokuapp.com/")

        title = headless_chrome.title
        print(f"Page title: {title}")

        assert "Internet" in title
        print("Headless navigation successful!")

    def test_screenshot_in_headless(self, headless_chrome):
        """Screenshots work in headless mode."""
        headless_chrome.get("https://the-internet.herokuapp.com/login")

        screenshot_dir = os.path.join(
            os.path.dirname(__file__), "TestResults", "Screenshots"
        )
        os.makedirs(screenshot_dir, exist_ok=True)

        filepath = os.path.join(screenshot_dir, "headless_screenshot.png")
        headless_chrome.save_screenshot(filepath)

        assert os.path.exists(filepath)
        assert os.path.getsize(filepath) > 0
        print(f"Headless screenshot saved: {filepath}")

    def test_verify_viewport_size(self, headless_chrome):
        """Verify the viewport is set correctly in headless mode."""
        headless_chrome.get("https://the-internet.herokuapp.com/")

        width = headless_chrome.execute_script(
            "return window.innerWidth;"
        )
        height = headless_chrome.execute_script(
            "return window.innerHeight;"
        )

        print(f"Viewport: {width}x{height}")

        # Should be close to 1920x1080
        assert width >= 1900, f"Width {width} is too small"
        assert height >= 1000, f"Height {height} is too small"

    def test_form_interaction(self, headless_chrome):
        """Form interactions work normally in headless mode."""
        headless_chrome.get("https://the-internet.herokuapp.com/login")
        wait = WebDriverWait(headless_chrome, 10)

        headless_chrome.find_element(By.ID, "username").send_keys(
            "tomsmith"
        )
        headless_chrome.find_element(By.ID, "password").send_keys(
            "SuperSecretPassword!"
        )
        headless_chrome.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        ).click()

        flash = wait.until(
            EC.visibility_of_element_located((By.ID, "flash"))
        )

        print(f"Flash message: {flash.text}")
        assert "You logged into" in flash.text

    def test_user_agent_in_headless(self, headless_chrome):
        """Check what user agent headless Chrome reports."""
        headless_chrome.get("https://the-internet.herokuapp.com/")

        user_agent = headless_chrome.execute_script(
            "return navigator.userAgent;"
        )

        print(f"User agent: {user_agent}")
        # New headless mode should NOT contain "HeadlessChrome"
        # (unlike old --headless which did)


# =============================================
# FIREFOX HEADLESS TESTS
# =============================================

class TestFirefoxHeadless:

    def test_firefox_headless_navigation(self, headless_firefox):
        """Firefox headless mode works too."""
        headless_firefox.get("https://the-internet.herokuapp.com/")

        title = headless_firefox.title
        print(f"Firefox headless title: {title}")

        assert "Internet" in title


# =============================================
# PERFORMANCE COMPARISON
# =============================================

class TestPerformance:

    def test_compare_headed_vs_headless(self):
        """
        Compare execution speed between headed and headless modes.
        Note: This test creates its own drivers for fair comparison.
        """
        url = "https://the-internet.herokuapp.com/login"

        # --- HEADLESS RUN ---
        headless_options = ChromeOptions()
        headless_options.add_argument("--headless=new")
        headless_options.add_argument("--window-size=1920,1080")
        headless_options.add_argument("--disable-gpu")

        headless_start = time.perf_counter()

        headless_driver = webdriver.Chrome(options=headless_options)
        headless_driver.get(url)
        headless_wait = WebDriverWait(headless_driver, 10)
        headless_wait.until(
            EC.presence_of_element_located((By.ID, "username"))
        )

        headless_driver.find_element(By.ID, "username").send_keys(
            "tomsmith"
        )
        headless_driver.find_element(By.ID, "password").send_keys(
            "SuperSecretPassword!"
        )
        headless_driver.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        ).click()

        headless_wait.until(
            EC.visibility_of_element_located((By.ID, "flash"))
        )

        headless_time = time.perf_counter() - headless_start
        headless_driver.quit()

        # --- HEADED RUN ---
        headed_options = ChromeOptions()
        headed_options.add_argument("--window-size=1920,1080")

        headed_start = time.perf_counter()

        headed_driver = webdriver.Chrome(options=headed_options)
        headed_driver.get(url)
        headed_wait = WebDriverWait(headed_driver, 10)
        headed_wait.until(
            EC.presence_of_element_located((By.ID, "username"))
        )

        headed_driver.find_element(By.ID, "username").send_keys(
            "tomsmith"
        )
        headed_driver.find_element(By.ID, "password").send_keys(
            "SuperSecretPassword!"
        )
        headed_driver.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        ).click()

        headed_wait.until(
            EC.visibility_of_element_located((By.ID, "flash"))
        )

        headed_time = time.perf_counter() - headed_start
        headed_driver.quit()

        # --- RESULTS ---
        print(f"Headless time: {headless_time:.2f}s")
        print(f"Headed time:   {headed_time:.2f}s")
        diff = headed_time - headless_time
        pct = (diff / headed_time) * 100
        print(f"Difference:    {diff:.2f}s ({pct:.1f}% faster headless)")


# =============================================
# CONFIGURABLE HEADLESS (CI/CD PATTERN)
# =============================================

class TestConfigDriven:

    def test_environment_driven_headless(self, config_driven_driver):
        """
        Run this test with:
          SELENIUM_HEADLESS=true pytest test_day25.py::TestConfigDriven
        or without the env var for headed mode.
        """
        config_driven_driver.get("https://the-internet.herokuapp.com/")
        assert "Internet" in config_driven_driver.title


# =============================================
# HEADLESS-SPECIFIC ISSUES AND FIXES
# =============================================

class TestHeadlessQuirks:

    def test_disable_images_for_speed(self):
        """Disable image loading in headless mode for faster tests."""
        options = ChromeOptions()
        options.add_argument("--headless=new")
        options.add_argument("--window-size=1920,1080")

        # Disable images via Chrome preferences
        prefs = {"profile.managed_default_content_settings.images": 2}
        options.add_experimental_option("prefs", prefs)

        driver = webdriver.Chrome(options=options)
        driver.get("https://the-internet.herokuapp.com/")

        title = driver.title
        print(f"Title (no images): {title}")
        assert "Internet" in title

        driver.quit()

    def test_custom_user_agent(self):
        """Override user agent to avoid headless detection."""
        options = ChromeOptions()
        options.add_argument("--headless=new")
        options.add_argument("--window-size=1920,1080")

        # Set a regular Chrome user agent
        options.add_argument(
            "--user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
            "AppleWebKit/537.36 (KHTML, like Gecko) "
            "Chrome/120.0.0.0 Safari/537.36"
        )

        driver = webdriver.Chrome(options=options)
        driver.get("https://the-internet.herokuapp.com/")

        ua = driver.execute_script("return navigator.userAgent;")
        print(f"Custom UA: {ua}")

        assert "HeadlessChrome" not in ua
        driver.quit()

    def test_download_file_in_headless(self):
        """Configure file downloads in headless mode."""
        download_dir = os.path.join(
            os.path.dirname(__file__), "downloads"
        )
        os.makedirs(download_dir, exist_ok=True)

        options = ChromeOptions()
        options.add_argument("--headless=new")
        options.add_argument("--window-size=1920,1080")

        # Configure download directory
        prefs = {
            "download.default_directory": download_dir,
            "download.prompt_for_download": False,
            "download.directory_upgrade": True,
        }
        options.add_experimental_option("prefs", prefs)

        driver = webdriver.Chrome(options=options)

        # Navigate to a page with downloadable files
        driver.get("https://the-internet.herokuapp.com/download")

        print(f"Download directory configured: {download_dir}")
        print("Download page loaded in headless mode.")

        driver.quit()
```

---

## 5. Step-by-Step Walkthrough

### Setting Up Chrome Headless

1. Create `ChromeOptions` (C#) or `Options` (Python)
2. Add `--headless=new` argument
3. Add `--window-size=1920,1080` to set viewport
4. Add `--disable-gpu` and `--no-sandbox` for CI stability
5. Pass options to `ChromeDriver` / `webdriver.Chrome()`
6. All subsequent Selenium commands work identically

### Setting Up Firefox Headless

1. Create `FirefoxOptions`
2. Add `--headless` argument (Firefox does not use `=new`)
3. Set `--width=1920` and `--height=1080` separately
4. Pass options to `FirefoxDriver` / `webdriver.Firefox()`

### Making It Configurable

1. Read an environment variable (`SELENIUM_HEADLESS`)
2. If `true`, add headless arguments
3. If absent or `false`, run headed
4. Set the variable in CI config, leave unset for local development

---

## 6. Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Not setting window size | 800x600 viewport breaks responsive layouts | Always add `--window-size=1920,1080` |
| Using old `--headless` flag | Different rendering, deprecated | Use `--headless=new` |
| Assuming headless = faster by default | Driver startup time dominates short tests | Headless benefit shows in large test suites |
| Not handling file downloads | Default download behavior differs in headless | Configure download directory via preferences |
| Forgetting `--no-sandbox` in Docker | Chrome crashes on startup | Add `--no-sandbox` and `--disable-dev-shm-usage` |
| Tests pass headed but fail headless | Usually viewport size or timing issue | Set viewport, increase wait timeouts slightly |

---

## 7. Best Practices

1. **Always set `--window-size` explicitly.** This eliminates the most common class of headless-only failures.

2. **Use `--headless=new` for Chrome.** The new headless mode is nearly identical to headed mode.

3. **Make headless configurable via environment variable.** Developers run headed locally; CI runs headless automatically.

4. **Take screenshots on failure even in headless mode.** They work perfectly and are your only window into what happened.

5. **Add `--disable-dev-shm-usage` for Docker/CI.** Shared memory in Docker containers is often too small for Chrome.

6. **Run your full test suite both headed and headless** at least once before relying on headless-only. Catch any mode-specific issues early.

7. **Do not hard-code headless mode.** Use a config file or environment variable so you can switch easily.

---

## 8. Hands-On Exercise

### Task: Run Full Test Suite Headless and Compare Execution Time

**Objective:** Run the same set of 5 tests in both headed and headless mode, then compare total execution time.

**Steps:**
1. Create 5 tests that interact with `https://the-internet.herokuapp.com/`:
   - Navigate to login page and verify title
   - Log in with valid credentials
   - Navigate to dropdown page and select an option
   - Navigate to checkboxes page and toggle a checkbox
   - Navigate to key presses page and type a key
2. Run all 5 tests in headed mode and record total time
3. Run all 5 tests in headless mode and record total time
4. Print comparison

**Expected Output:**
```
Headed mode:   12.4 seconds (5 tests)
Headless mode:  8.7 seconds (5 tests)
Speedup:        29.8%
```

---

## 9. Real-World Scenario

### Scenario: CI/CD Pipeline with Headless Testing

```yaml
# .github/workflows/selenium-tests.yml
name: Selenium Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Chrome
        run: |
          wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
          echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" | sudo tee /etc/apt/sources.list.d/google-chrome.list
          sudo apt-get update
          sudo apt-get install -y google-chrome-stable

      - name: Install dependencies
        run: pip install selenium pytest

      - name: Run tests (headless)
        env:
          SELENIUM_HEADLESS: "true"
        run: pytest tests/ -v --tb=short
```

---

## 10. Resources

- [Chrome Headless Mode Documentation](https://developer.chrome.com/docs/chromium/new-headless)
- [Selenium ChromeOptions](https://www.selenium.dev/documentation/webdriver/browsers/chrome/)
- [Selenium FirefoxOptions](https://www.selenium.dev/documentation/webdriver/browsers/firefox/)
- [Chrome Flags List](https://peter.sh/experiments/chromium-command-line-switches/)
- [Running Chrome in Docker](https://github.com/nicholasgasior/docker-chrome)
