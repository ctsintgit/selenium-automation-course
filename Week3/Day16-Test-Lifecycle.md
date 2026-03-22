# Day 16: Test Lifecycle — Setup, Teardown, Fixtures

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain the test lifecycle and why setup/teardown matters
- Use NUnit lifecycle attributes: `[OneTimeSetUp]`, `[SetUp]`, `[TearDown]`, `[OneTimeTearDown]`
- Use pytest fixtures with different scopes (`function`, `class`, `module`, `session`)
- Create a reusable WebDriver fixture in `conftest.py`
- Build a base test class in C# that all test classes inherit from
- Ensure proper driver cleanup so browser processes never leak

---

## 1. Core Concept: The Test Lifecycle

Every test follows a predictable lifecycle:

```
[One-time setup for the entire class]
    [Setup for test 1]
        [Run test 1]
    [Teardown for test 1]
    [Setup for test 2]
        [Run test 2]
    [Teardown for test 2]
[One-time teardown for the entire class]
```

**Why does this matter for Selenium?**

- **Setup**: Create the WebDriver, navigate to a starting URL, log in
- **Teardown**: Quit the driver, close the browser, release resources
- **One-time setup**: Start a single browser session shared across tests (faster)
- **One-time teardown**: Final cleanup

The goal is to avoid repeating browser creation code in every single test method.

---

## 2. How It Works

### NUnit Lifecycle Attributes

| Attribute | When It Runs | Typical Use |
|-----------|-------------|-------------|
| `[OneTimeSetUp]` | Once before all tests in the class | Create shared resources |
| `[SetUp]` | Before each test method | Navigate to starting page, reset state |
| `[TearDown]` | After each test method | Take screenshot on failure, clear cookies |
| `[OneTimeTearDown]` | Once after all tests in the class | Quit the driver |

### pytest Fixture Scopes

| Scope | When It Runs | Typical Use |
|-------|-------------|-------------|
| `function` (default) | Before/after each test function | Fresh driver per test |
| `class` | Before/after each test class | Shared driver within a class |
| `module` | Before/after each test file | Shared driver within a file |
| `session` | Before/after the entire test run | Single driver for all tests |

---

## 3. Code Example: C# (NUnit) — Base Test Class

```csharp
// File: Base/BaseTest.cs
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;

namespace SeleniumTests.Base
{
    /// <summary>
    /// Base class for all test classes. Provides WebDriver lifecycle management.
    /// Every test class should inherit from this class.
    /// </summary>
    [TestFixture]
    public abstract class BaseTest
    {
        // Protected so subclasses can access the driver and wait
        protected IWebDriver Driver { get; private set; }
        protected WebDriverWait Wait { get; private set; }

        // Base URL — can be overridden by subclasses or read from config
        protected virtual string BaseUrl => "https://www.saucedemo.com";

        [OneTimeSetUp]
        public void OneTimeSetUp()
        {
            // Runs ONCE before all tests in the class
            // Create the ChromeDriver — Selenium 4.6+ manages the driver binary
            var options = new ChromeOptions();
            options.AddArgument("--start-maximized");
            // options.AddArgument("--headless");

            Driver = new ChromeDriver(options);
            Wait = new WebDriverWait(Driver, TimeSpan.FromSeconds(10));

            Console.WriteLine("[OneTimeSetUp] Browser launched");
        }

        [SetUp]
        public void SetUp()
        {
            // Runs BEFORE each test method
            // Navigate to the base URL so each test starts from a known state
            Driver.Navigate().GoToUrl(BaseUrl);
            Console.WriteLine($"[SetUp] Navigated to {BaseUrl}");
        }

        [TearDown]
        public void TearDown()
        {
            // Runs AFTER each test method
            // Check if the test failed and take a screenshot
            if (TestContext.CurrentContext.Result.Outcome.Status ==
                NUnit.Framework.Interfaces.TestStatus.Failed)
            {
                TakeScreenshot(TestContext.CurrentContext.Test.Name);
            }

            // Clear cookies to reset session state between tests
            Driver.Manage().Cookies.DeleteAllCookies();
            Console.WriteLine("[TearDown] Cookies cleared");
        }

        [OneTimeTearDown]
        public void OneTimeTearDown()
        {
            // Runs ONCE after all tests in the class
            if (Driver != null)
            {
                Driver.Quit();
                Driver = null;
                Console.WriteLine("[OneTimeTearDown] Browser closed");
            }
        }

        /// <summary>
        /// Takes a screenshot and saves it to the TestResults folder.
        /// </summary>
        protected void TakeScreenshot(string testName)
        {
            try
            {
                var screenshot = ((ITakesScreenshot)Driver).GetScreenshot();
                string fileName = $"FAIL_{testName}_{DateTime.Now:yyyyMMdd_HHmmss}.png";
                string path = System.IO.Path.Combine(
                    TestContext.CurrentContext.WorkDirectory, fileName);
                screenshot.SaveAsFile(path);
                TestContext.AddTestAttachment(path, "Failure Screenshot");
                Console.WriteLine($"[Screenshot] Saved to {path}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"[Screenshot] Failed: {ex.Message}");
            }
        }
    }
}
```

```csharp
// File: Tests/LoginTests.cs
using NUnit.Framework;
using OpenQA.Selenium;
using SeleniumTests.Base;

namespace SeleniumTests.Tests
{
    [TestFixture]
    public class LoginTests : BaseTest
    {
        [Test]
        public void ValidLogin_ShouldRedirectToInventory()
        {
            // Driver and Wait are inherited from BaseTest
            // SetUp already navigated to the login page

            Driver.FindElement(By.Id("user-name")).SendKeys("standard_user");
            Driver.FindElement(By.Id("password")).SendKeys("secret_sauce");
            Driver.FindElement(By.Id("login-button")).Click();

            // Wait for the inventory page to load
            Wait.Until(d => d.Url.Contains("inventory"));

            Assert.That(Driver.Url, Does.Contain("inventory"),
                "Expected URL to contain 'inventory' after login");
        }

        [Test]
        public void InvalidLogin_ShouldShowError()
        {
            Driver.FindElement(By.Id("user-name")).SendKeys("invalid_user");
            Driver.FindElement(By.Id("password")).SendKeys("wrong_password");
            Driver.FindElement(By.Id("login-button")).Click();

            var errorMessage = Wait.Until(d =>
                d.FindElement(By.CssSelector("[data-test='error']")));

            Assert.That(errorMessage.Text, Does.Contain("Username and password"),
                "Expected error message about credentials");
        }
    }
}
```

---

## 4. Code Example: Python (pytest) — conftest.py and Fixtures

```python
# File: conftest.py
# This file is automatically discovered by pytest.
# Fixtures defined here are available to ALL test files in the same directory.

import pytest
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from datetime import datetime
import os


@pytest.fixture(scope="function")
def driver():
    """
    Creates a fresh ChromeDriver for each test function.
    Scope='function' means a new browser opens for every test.
    The 'yield' keyword separates setup from teardown.
    """
    # --- SETUP ---
    options = webdriver.ChromeOptions()
    options.add_argument("--start-maximized")
    # options.add_argument("--headless")

    # Selenium 4.6+ manages ChromeDriver automatically
    browser = webdriver.Chrome(options=options)
    print("\n[Setup] Browser launched")

    # Yield gives the driver to the test — everything after yield is teardown
    yield browser

    # --- TEARDOWN ---
    browser.quit()
    print("[Teardown] Browser closed")


@pytest.fixture(scope="class")
def class_driver(request):
    """
    Creates ONE ChromeDriver shared across all tests in a class.
    Faster than scope='function' but tests must not depend on order.
    """
    options = webdriver.ChromeOptions()
    options.add_argument("--start-maximized")

    browser = webdriver.Chrome(options=options)
    print("\n[ClassSetup] Browser launched")

    # Attach the driver to the test class so all methods can access it
    request.cls.driver = browser
    request.cls.wait = WebDriverWait(browser, 10)

    yield browser

    browser.quit()
    print("[ClassTeardown] Browser closed")


@pytest.fixture(scope="session")
def session_driver():
    """
    Creates ONE ChromeDriver shared across the ENTIRE test session.
    Maximum speed, but use carefully — state leaks between tests.
    """
    options = webdriver.ChromeOptions()
    options.add_argument("--start-maximized")

    browser = webdriver.Chrome(options=options)
    print("\n[SessionSetup] Browser launched")

    yield browser

    browser.quit()
    print("[SessionTeardown] Browser closed")


@pytest.fixture(autouse=True)
def navigate_to_base(driver):
    """
    autouse=True means this fixture runs automatically for every test
    that uses the 'driver' fixture. Navigates to the base URL.
    """
    driver.get("https://www.saucedemo.com")
    print("[Setup] Navigated to Sauce Demo")

    yield

    # After each test, clear cookies
    driver.delete_all_cookies()
    print("[Teardown] Cookies cleared")


def take_screenshot(driver, test_name):
    """Utility function to capture a screenshot on failure."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = f"FAIL_{test_name}_{timestamp}.png"
    filepath = os.path.join("screenshots", filename)
    os.makedirs("screenshots", exist_ok=True)
    driver.save_screenshot(filepath)
    print(f"[Screenshot] Saved to {filepath}")
    return filepath


@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """
    pytest hook that takes a screenshot when a test fails.
    This runs automatically — no decorator needed on tests.
    """
    outcome = yield
    report = outcome.get_result()

    if report.when == "call" and report.failed:
        driver = item.funcargs.get("driver")
        if driver:
            take_screenshot(driver, item.name)
```

```python
# File: test_login.py
import pytest
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class TestLogin:
    """Login tests using the 'driver' fixture from conftest.py."""

    def test_valid_login(self, driver):
        """Valid credentials should redirect to inventory page."""
        # conftest already navigated to the login page

        driver.find_element(By.ID, "user-name").send_keys("standard_user")
        driver.find_element(By.ID, "password").send_keys("secret_sauce")
        driver.find_element(By.ID, "login-button").click()

        # Explicit wait for the URL to change
        wait = WebDriverWait(driver, 10)
        wait.until(EC.url_contains("inventory"))

        assert "inventory" in driver.current_url

    def test_invalid_login(self, driver):
        """Invalid credentials should show an error message."""
        driver.find_element(By.ID, "user-name").send_keys("invalid_user")
        driver.find_element(By.ID, "password").send_keys("wrong_pass")
        driver.find_element(By.ID, "login-button").click()

        wait = WebDriverWait(driver, 10)
        error = wait.until(
            EC.presence_of_element_located((By.CSS_SELECTOR, "[data-test='error']"))
        )

        assert "Username and password" in error.text

    def test_empty_credentials(self, driver):
        """Empty credentials should show an error message."""
        driver.find_element(By.ID, "login-button").click()

        wait = WebDriverWait(driver, 10)
        error = wait.until(
            EC.presence_of_element_located((By.CSS_SELECTOR, "[data-test='error']"))
        )

        assert "Username is required" in error.text
```

---

## 5. Step-by-Step Walkthrough

### C# Execution Order (LoginTests)

```
1. [OneTimeSetUp]   → ChromeDriver created, browser opens
2. [SetUp]          → Navigate to https://www.saucedemo.com
3. ValidLogin test  → Runs assertions
4. [TearDown]       → Check for failure, clear cookies
5. [SetUp]          → Navigate to https://www.saucedemo.com
6. InvalidLogin test → Runs assertions
7. [TearDown]       → Check for failure, clear cookies
8. [OneTimeTearDown] → driver.Quit(), browser closes
```

### Python Execution Order (TestLogin)

```
1. driver fixture setup      → Chrome launched
2. navigate_to_base setup    → Navigate to Sauce Demo
3. test_valid_login           → Runs assertions
4. navigate_to_base teardown  → Clear cookies
5. driver fixture teardown    → Chrome quit
6. driver fixture setup       → NEW Chrome launched (scope=function)
7. navigate_to_base setup     → Navigate to Sauce Demo
8. test_invalid_login          → Runs assertions
9. navigate_to_base teardown  → Clear cookies
10. driver fixture teardown   → Chrome quit
```

Notice that with `scope="function"`, Python creates a brand new browser for each test. This is slower but guarantees complete isolation.

---

## 6. Common Mistakes and How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Creating driver in `[SetUp]` and quitting in `[TearDown]` | Opens/closes browser for every test — very slow | Use `[OneTimeSetUp]`/`[OneTimeTearDown]` for the driver |
| Forgetting to quit the driver | Zombie browser processes accumulate | Always quit in teardown; use try/finally or yield |
| Using `scope="session"` carelessly | Login cookies from test 1 affect test 2 | Clear cookies between tests or use `scope="function"` |
| Not handling test failures in teardown | Screenshots never captured | Use NUnit's `TestContext` or pytest hooks |
| Making the base class `public` but not `abstract` | NUnit tries to instantiate it as a test class | Mark base classes as `abstract` |
| Putting `conftest.py` in the wrong directory | Fixtures are not discovered | Place it in the test root or the directory where tests live |

---

## 7. Best Practices

1. **Use a base test class (C#)** — All test classes inherit from `BaseTest`, which manages the driver lifecycle. Change the driver setup in one place, and all tests benefit.

2. **Use `conftest.py` (Python)** — Define fixtures once, share them across all test files. Never import `conftest.py` — pytest discovers it automatically.

3. **Choose the right scope** — `function` scope is safest (full isolation), `class` or `session` scope is faster. Pick based on how independent your tests need to be.

4. **Screenshot on failure** — Capture evidence automatically. In C#, use `TestContext.CurrentContext.Result.Outcome`. In Python, use the `pytest_runtest_makereport` hook.

5. **Never rely on test execution order** — Tests can run in any order. Each test should set up its own preconditions.

6. **Clean up state between tests** — At minimum, clear cookies. For full isolation, navigate back to the starting page.

---

## 8. Hands-On Exercise

**Task:** Build a reusable test foundation for Sauce Demo.

### C# Requirements

1. Create `BaseTest.cs` with `[OneTimeSetUp]`, `[SetUp]`, `[TearDown]`, `[OneTimeTearDown]`
2. Create `LoginTests.cs` inheriting from `BaseTest` with three tests
3. Create `InventoryTests.cs` inheriting from `BaseTest` with one test that logs in and verifies the product count

### Python Requirements

1. Create `conftest.py` with a `driver` fixture (scope=function) and auto-navigate fixture
2. Create `test_login.py` with three login tests
3. Create `test_inventory.py` with one test that verifies product count after login

**Expected Output:**

```
test_login.py::TestLogin::test_valid_login PASSED
test_login.py::TestLogin::test_invalid_login PASSED
test_login.py::TestLogin::test_empty_credentials PASSED
test_inventory.py::TestInventory::test_product_count PASSED
========================= 4 passed in 18.56s =========================
```

---

## 9. Real-World Scenario

In production test suites, you often see layered base classes:

```
BaseTest (driver lifecycle)
  └── AuthenticatedBaseTest (login in OneTimeSetUp)
        └── AdminTests (tests that require admin login)
        └── UserTests (tests that require regular user login)
```

This avoids repeating login steps in every test. The `AuthenticatedBaseTest` logs in during `[OneTimeSetUp]` and all subclass tests run in an already-authenticated session.

In Python, you achieve the same with fixture composition — a `logged_in_driver` fixture that depends on the `driver` fixture.

---

## 10. Resources

- [NUnit SetUp and TearDown](https://docs.nunit.org/articles/nunit/writing-tests/attributes/setup.html)
- [pytest Fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
- [pytest conftest.py](https://docs.pytest.org/en/stable/reference/fixtures.html#conftest-py-sharing-fixtures-across-tests)
- [Selenium 4 WebDriver Lifecycle](https://www.selenium.dev/documentation/webdriver/getting_started/)
