# Day 29: Framework Finalization — Folder Structure, Naming Conventions

## Learning Objectives

By the end of this lesson, you will be able to:

- Design a professional folder structure for Selenium test frameworks in both C# and Python
- Apply consistent naming conventions for tests, pages, utilities, and config files
- Create reusable utility classes: WaitHelper, ScreenshotHelper, ConfigReader
- Organize constants and enums for maintainable test data
- Write proper configuration files for different environments
- Set up `.gitignore` for test projects
- Document your framework with a clear README structure

---

## 1. Core Concept Explanation

### Why Folder Structure Matters

A new team member should be able to look at your project structure and immediately understand:
- Where to find tests
- Where to find page objects
- Where to add new tests
- Where configuration lives
- What utilities are available

A bad structure leads to duplicate code, lost files, and confusion. A good structure scales from 10 tests to 10,000 tests.

### The Three Pillars of Framework Organization

1. **Separation of Concerns** — Tests, pages, utilities, and config each have their own directory
2. **Naming Conventions** — Consistent, descriptive names that reveal intent
3. **Reusable Utilities** — Common operations (waits, screenshots, config reading) live in shared helper classes

---

## 2. Professional Folder Structure

### C# (.NET) Framework Structure

```
SeleniumFramework/
├── SeleniumFramework.sln                    # Solution file
├── .gitignore
├── README.md
│
├── SeleniumFramework.Tests/                  # Test project
│   ├── SeleniumFramework.Tests.csproj
│   │
│   ├── Config/
│   │   ├── appsettings.json                 # Environment config
│   │   ├── appsettings.staging.json         # Staging overrides
│   │   └── TestData.json                    # Test data (users, products)
│   │
│   ├── Constants/
│   │   ├── Urls.cs                          # URL constants
│   │   ├── TestUsers.cs                     # User credentials
│   │   └── Timeouts.cs                      # Timeout values
│   │
│   ├── Drivers/
│   │   └── DriverFactory.cs                 # Browser creation logic
│   │
│   ├── Extensions/
│   │   └── WebElementExtensions.cs          # Extension methods
│   │
│   ├── Pages/
│   │   ├── BasePage.cs                      # Common page methods
│   │   ├── LoginPage.cs
│   │   ├── DashboardPage.cs
│   │   ├── SearchPage.cs
│   │   └── Components/
│   │       ├── HeaderComponent.cs           # Shared UI components
│   │       └── FooterComponent.cs
│   │
│   ├── Tests/
│   │   ├── BaseTest.cs                      # SetUp/TearDown, screenshots
│   │   ├── LoginTests.cs
│   │   ├── SearchTests.cs
│   │   └── DashboardTests.cs
│   │
│   ├── Utils/
│   │   ├── WaitHelper.cs                    # Explicit wait utilities
│   │   ├── ScreenshotHelper.cs              # Screenshot capture
│   │   ├── ConfigReader.cs                  # Read appsettings.json
│   │   └── RandomDataGenerator.cs           # Generate test data
│   │
│   └── Reports/                             # Generated reports (gitignored)
│       └── .gitkeep
```

### Python Framework Structure

```
selenium_framework/
├── .gitignore
├── README.md
├── requirements.txt
├── pytest.ini                                # pytest configuration
├── conftest.py                               # Global fixtures
│
├── config/
│   ├── __init__.py
│   ├── settings.py                           # Environment config
│   ├── settings.staging.py                   # Staging overrides
│   └── test_data.json                        # Test data
│
├── constants/
│   ├── __init__.py
│   ├── urls.py                               # URL constants
│   ├── test_users.py                         # User credentials
│   └── timeouts.py                           # Timeout values
│
├── drivers/
│   ├── __init__.py
│   └── driver_factory.py                     # Browser creation
│
├── pages/
│   ├── __init__.py
│   ├── base_page.py                          # Common page methods
│   ├── login_page.py
│   ├── dashboard_page.py
│   ├── search_page.py
│   └── components/
│       ├── __init__.py
│       ├── header_component.py
│       └── footer_component.py
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                           # Test-specific fixtures
│   ├── test_login.py
│   ├── test_search.py
│   └── test_dashboard.py
│
├── utils/
│   ├── __init__.py
│   ├── wait_helper.py                        # Explicit wait utilities
│   ├── screenshot_helper.py                  # Screenshot capture
│   ├── config_reader.py                      # Read config files
│   └── random_data.py                        # Generate test data
│
└── reports/                                  # Generated (gitignored)
    └── .gitkeep
```

---

## 3. Naming Conventions

### File Naming

| Component | C# Convention | Python Convention | Example |
|-----------|--------------|-------------------|---------|
| Test class | `PascalCase` + `Tests` suffix | `snake_case` + `test_` prefix | `LoginTests.cs` / `test_login.py` |
| Page object | `PascalCase` + `Page` suffix | `snake_case` + `_page` suffix | `LoginPage.cs` / `login_page.py` |
| Utility | `PascalCase` + `Helper` suffix | `snake_case` + `_helper` suffix | `WaitHelper.cs` / `wait_helper.py` |
| Config | `PascalCase` / JSON | `snake_case` / JSON | `ConfigReader.cs` / `config_reader.py` |
| Constants | `PascalCase` | `UPPER_SNAKE_CASE` | `Urls.cs` / `urls.py` |

### Method/Function Naming

| Type | C# Convention | Python Convention | Example |
|------|--------------|-------------------|---------|
| Test method | `PascalCase`, descriptive | `snake_case`, `test_` prefix | `ValidLoginRedirectsToDashboard()` / `test_valid_login_redirects_to_dashboard()` |
| Page method | `PascalCase`, action verbs | `snake_case`, action verbs | `EnterUsername()` / `enter_username()` |
| Locator | `PascalCase` property | `UPPER_SNAKE_CASE` tuple | `UsernameField` / `USERNAME_FIELD` |

### Test Method Naming Pattern

Follow this pattern: **`{Action}_{Condition}_{ExpectedResult}`**

```csharp
// C#
public void Login_WithValidCredentials_RedirectsToDashboard() { }
public void Login_WithInvalidPassword_ShowsErrorMessage() { }
public void Search_WithEmptyQuery_ShowsAllResults() { }
```

```python
# Python
def test_login_with_valid_credentials_redirects_to_dashboard(): pass
def test_login_with_invalid_password_shows_error_message(): pass
def test_search_with_empty_query_shows_all_results(): pass
```

---

## 4. Code Example: C# Utility Classes (Complete, Runnable)

```csharp
// =============================================
// File: Utils/ConfigReader.cs
// =============================================
using System;
using System.IO;
using System.Text.Json;

namespace SeleniumFramework.Utils
{
    /// <summary>
    /// Reads configuration from appsettings.json.
    /// Supports environment-specific overrides.
    /// </summary>
    public static class ConfigReader
    {
        private static JsonDocument _config;

        static ConfigReader()
        {
            // Determine environment
            string env = Environment.GetEnvironmentVariable(
                "TEST_ENVIRONMENT"
            ) ?? "development";

            // Load base config
            string basePath = Path.Combine(
                AppDomain.CurrentDomain.BaseDirectory,
                "Config", "appsettings.json"
            );

            if (File.Exists(basePath))
            {
                string json = File.ReadAllText(basePath);
                _config = JsonDocument.Parse(json);
            }

            // Load environment override if it exists
            string envPath = Path.Combine(
                AppDomain.CurrentDomain.BaseDirectory,
                "Config", $"appsettings.{env}.json"
            );

            if (File.Exists(envPath))
            {
                string json = File.ReadAllText(envPath);
                _config = JsonDocument.Parse(json);
            }
        }

        public static string GetBaseUrl()
        {
            return GetValue("BaseUrl")
                ?? "https://the-internet.herokuapp.com";
        }

        public static string GetBrowser()
        {
            return GetValue("Browser") ?? "chrome";
        }

        public static bool IsHeadless()
        {
            string env = Environment.GetEnvironmentVariable(
                "SELENIUM_HEADLESS"
            );
            if (env != null) return env.ToLower() == "true";
            return bool.Parse(GetValue("Headless") ?? "false");
        }

        public static int GetDefaultTimeout()
        {
            return int.Parse(GetValue("DefaultTimeoutSeconds") ?? "10");
        }

        private static string GetValue(string key)
        {
            try
            {
                return _config?.RootElement.GetProperty(key).GetString();
            }
            catch
            {
                return null;
            }
        }
    }
}

// =============================================
// File: Utils/WaitHelper.cs
// =============================================
using OpenQA.Selenium;
using OpenQA.Selenium.Support.UI;
using System;

namespace SeleniumFramework.Utils
{
    /// <summary>
    /// Reusable explicit wait methods. Never use Thread.Sleep.
    /// </summary>
    public class WaitHelper
    {
        private readonly IWebDriver _driver;
        private readonly WebDriverWait _wait;

        public WaitHelper(IWebDriver driver, int timeoutSeconds = 10)
        {
            _driver = driver;
            _wait = new WebDriverWait(
                driver, TimeSpan.FromSeconds(timeoutSeconds)
            );
        }

        /// <summary>Wait until element is visible and return it.</summary>
        public IWebElement WaitForVisible(By locator)
        {
            return _wait.Until(d =>
            {
                try
                {
                    var el = d.FindElement(locator);
                    return el.Displayed ? el : null;
                }
                catch (NoSuchElementException)
                {
                    return null;
                }
            });
        }

        /// <summary>Wait until element is clickable and return it.</summary>
        public IWebElement WaitForClickable(By locator)
        {
            return _wait.Until(d =>
            {
                try
                {
                    var el = d.FindElement(locator);
                    return (el.Displayed && el.Enabled) ? el : null;
                }
                catch (NoSuchElementException)
                {
                    return null;
                }
            });
        }

        /// <summary>Wait until element disappears from the page.</summary>
        public bool WaitForInvisible(By locator)
        {
            return _wait.Until(d =>
            {
                try
                {
                    return !d.FindElement(locator).Displayed;
                }
                catch (NoSuchElementException)
                {
                    return true;
                }
            });
        }

        /// <summary>Wait until URL contains the expected text.</summary>
        public bool WaitForUrlContains(string urlPart)
        {
            return _wait.Until(d => d.Url.Contains(urlPart));
        }

        /// <summary>Wait until page title contains the expected text.</summary>
        public bool WaitForTitleContains(string titlePart)
        {
            return _wait.Until(d => d.Title.Contains(titlePart));
        }

        /// <summary>Wait for element text to contain expected value.</summary>
        public IWebElement WaitForTextPresent(By locator, string text)
        {
            return _wait.Until(d =>
            {
                try
                {
                    var el = d.FindElement(locator);
                    return el.Text.Contains(text) ? el : null;
                }
                catch (NoSuchElementException)
                {
                    return null;
                }
            });
        }
    }
}

// =============================================
// File: Utils/ScreenshotHelper.cs
// =============================================
using NUnit.Framework;
using OpenQA.Selenium;
using System;
using System.IO;

namespace SeleniumFramework.Utils
{
    /// <summary>
    /// Centralized screenshot capture with naming and organization.
    /// </summary>
    public static class ScreenshotHelper
    {
        private static readonly string BaseDir = Path.Combine(
            TestContext.CurrentContext.TestDirectory,
            "Reports", "Screenshots"
        );

        /// <summary>Capture full page screenshot.</summary>
        public static string Capture(
            IWebDriver driver, string label = "screenshot")
        {
            Directory.CreateDirectory(BaseDir);

            string testName = TestContext.CurrentContext.Test.Name;
            string timestamp = DateTime.Now.ToString("yyyy-MM-dd_HH-mm-ss");
            string fileName = $"{timestamp}_{testName}_{label}.png";
            string filePath = Path.Combine(BaseDir, fileName);

            try
            {
                var screenshot = ((ITakesScreenshot)driver).GetScreenshot();
                screenshot.SaveAsFile(filePath);
                TestContext.AddTestAttachment(filePath, label);
                return filePath;
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Screenshot failed: {ex.Message}");
                return null;
            }
        }

        /// <summary>Capture screenshot of a specific element.</summary>
        public static string CaptureElement(
            IWebElement element, string label = "element")
        {
            Directory.CreateDirectory(BaseDir);

            string testName = TestContext.CurrentContext.Test.Name;
            string timestamp = DateTime.Now.ToString("yyyy-MM-dd_HH-mm-ss");
            string fileName = $"{timestamp}_{testName}_{label}_element.png";
            string filePath = Path.Combine(BaseDir, fileName);

            try
            {
                var screenshot = ((ITakesScreenshot)element).GetScreenshot();
                screenshot.SaveAsFile(filePath);
                TestContext.AddTestAttachment(filePath, label);
                return filePath;
            }
            catch (Exception ex)
            {
                Console.WriteLine(
                    $"Element screenshot failed: {ex.Message}"
                );
                return null;
            }
        }
    }
}

// =============================================
// File: Drivers/DriverFactory.cs
// =============================================
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Firefox;
using OpenQA.Selenium.Edge;
using OpenQA.Selenium.Remote;
using System;

namespace SeleniumFramework.Drivers
{
    /// <summary>
    /// Creates WebDriver instances based on configuration.
    /// Supports local and remote (Grid) execution.
    /// </summary>
    public static class DriverFactory
    {
        public static IWebDriver CreateDriver(
            string browser = "chrome",
            bool headless = false,
            string gridUrl = null)
        {
            IWebDriver driver;

            switch (browser.ToLower())
            {
                case "chrome":
                    var chromeOptions = new ChromeOptions();
                    chromeOptions.AddArgument("--window-size=1920,1080");
                    chromeOptions.AddArgument("--disable-gpu");
                    chromeOptions.AddArgument("--no-sandbox");
                    chromeOptions.AddArgument("--disable-dev-shm-usage");
                    if (headless)
                        chromeOptions.AddArgument("--headless=new");

                    driver = string.IsNullOrEmpty(gridUrl)
                        ? new ChromeDriver(chromeOptions)
                        : new RemoteWebDriver(
                            new Uri(gridUrl), chromeOptions);
                    break;

                case "firefox":
                    var ffOptions = new FirefoxOptions();
                    if (headless)
                        ffOptions.AddArgument("--headless");

                    driver = string.IsNullOrEmpty(gridUrl)
                        ? new FirefoxDriver(ffOptions)
                        : new RemoteWebDriver(
                            new Uri(gridUrl), ffOptions);
                    break;

                case "edge":
                    var edgeOptions = new EdgeOptions();
                    edgeOptions.AddArgument("--window-size=1920,1080");
                    if (headless)
                        edgeOptions.AddArgument("--headless=new");

                    driver = string.IsNullOrEmpty(gridUrl)
                        ? new EdgeDriver(edgeOptions)
                        : new RemoteWebDriver(
                            new Uri(gridUrl), edgeOptions);
                    break;

                default:
                    throw new ArgumentException(
                        $"Unsupported browser: {browser}"
                    );
            }

            driver.Manage().Timeouts().ImplicitWait =
                TimeSpan.FromSeconds(0); // We use explicit waits only

            return driver;
        }
    }
}

// =============================================
// File: Pages/BasePage.cs
// =============================================
using OpenQA.Selenium;
using SeleniumFramework.Utils;

namespace SeleniumFramework.Pages
{
    /// <summary>
    /// Base class for all page objects. Provides common
    /// functionality like waits, navigation, and element helpers.
    /// </summary>
    public abstract class BasePage
    {
        protected readonly IWebDriver Driver;
        protected readonly WaitHelper Wait;

        protected BasePage(IWebDriver driver)
        {
            Driver = driver;
            Wait = new WaitHelper(driver);
        }

        /// <summary>Current page URL.</summary>
        public string CurrentUrl => Driver.Url;

        /// <summary>Current page title.</summary>
        public string PageTitle => Driver.Title;

        /// <summary>Navigate to a URL.</summary>
        protected void NavigateTo(string url)
        {
            Driver.Navigate().GoToUrl(url);
        }

        /// <summary>Scroll to an element using JS.</summary>
        protected void ScrollToElement(IWebElement element)
        {
            ((IJavaScriptExecutor)Driver).ExecuteScript(
                "arguments[0].scrollIntoView({block: 'center'});",
                element
            );
        }

        /// <summary>Check if element is present on page.</summary>
        protected bool IsElementPresent(By locator)
        {
            try
            {
                Driver.FindElement(locator);
                return true;
            }
            catch (NoSuchElementException)
            {
                return false;
            }
        }
    }
}

// =============================================
// File: Tests/BaseTest.cs
// =============================================
using NUnit.Framework;
using NUnit.Framework.Interfaces;
using OpenQA.Selenium;
using SeleniumFramework.Drivers;
using SeleniumFramework.Utils;

namespace SeleniumFramework.Tests
{
    /// <summary>
    /// Base class for all test fixtures. Handles driver lifecycle,
    /// screenshots on failure, and common setup.
    /// </summary>
    public class BaseTest
    {
        protected IWebDriver Driver;
        protected WaitHelper Wait;

        [SetUp]
        public virtual void SetUp()
        {
            string browser = ConfigReader.GetBrowser();
            bool headless = ConfigReader.IsHeadless();

            Driver = DriverFactory.CreateDriver(browser, headless);
            Wait = new WaitHelper(
                Driver, ConfigReader.GetDefaultTimeout()
            );
        }

        [TearDown]
        public virtual void TearDown()
        {
            if (TestContext.CurrentContext.Result.Outcome.Status
                == TestStatus.Failed)
            {
                ScreenshotHelper.Capture(Driver, "FAILED");
            }

            Driver?.Quit();
        }
    }
}

// =============================================
// File: Config/appsettings.json
// =============================================
/*
{
    "BaseUrl": "https://the-internet.herokuapp.com",
    "Browser": "chrome",
    "Headless": "false",
    "DefaultTimeoutSeconds": "10",
    "GridUrl": "",
    "TestUsers": {
        "ValidUser": {
            "Username": "tomsmith",
            "Password": "SuperSecretPassword!"
        },
        "AdminUser": {
            "Username": "admin",
            "Password": "admin123"
        }
    }
}
*/

// =============================================
// File: Constants/Urls.cs
// =============================================
namespace SeleniumFramework.Constants
{
    public static class Urls
    {
        public const string Login = "/login";
        public const string Dashboard = "/secure";
        public const string Checkboxes = "/checkboxes";
        public const string Dropdown = "/dropdown";
    }
}

// =============================================
// File: Constants/Timeouts.cs
// =============================================
namespace SeleniumFramework.Constants
{
    public static class Timeouts
    {
        public const int Short = 5;
        public const int Default = 10;
        public const int Long = 30;
        public const int PageLoad = 60;
    }
}
```

---

## 5. Code Example: Python Utility Classes (Complete, Runnable)

```python
# =============================================
# File: config/settings.py
# =============================================
"""
Central configuration for the test framework.
Reads from environment variables with sensible defaults.
"""
import os
import json


class Settings:
    """Framework configuration. Env vars override defaults."""

    BASE_URL = os.environ.get(
        "BASE_URL", "https://the-internet.herokuapp.com"
    )
    BROWSER = os.environ.get("BROWSER", "chrome")
    HEADLESS = os.environ.get(
        "SELENIUM_HEADLESS", "false"
    ).lower() in ("true", "1", "yes")
    DEFAULT_TIMEOUT = int(os.environ.get("DEFAULT_TIMEOUT", "10"))
    GRID_URL = os.environ.get("SELENIUM_GRID_URL", "")
    SCREENSHOT_DIR = os.path.join(
        os.path.dirname(os.path.dirname(__file__)),
        "reports", "screenshots"
    )

    @classmethod
    def load_test_data(cls, filename="test_data.json"):
        """Load test data from JSON file."""
        filepath = os.path.join(
            os.path.dirname(__file__), filename
        )
        with open(filepath, "r") as f:
            return json.load(f)


# =============================================
# File: constants/urls.py
# =============================================
"""URL path constants."""


class Urls:
    LOGIN = "/login"
    DASHBOARD = "/secure"
    CHECKBOXES = "/checkboxes"
    DROPDOWN = "/dropdown"
    KEY_PRESSES = "/key_presses"


# =============================================
# File: constants/test_users.py
# =============================================
"""Test user credentials."""


class TestUsers:
    VALID_USER = {
        "username": "tomsmith",
        "password": "SuperSecretPassword!"
    }
    INVALID_USER = {
        "username": "wronguser",
        "password": "wrongpass"
    }


# =============================================
# File: constants/timeouts.py
# =============================================
"""Timeout constants in seconds."""


class Timeouts:
    SHORT = 5
    DEFAULT = 10
    LONG = 30
    PAGE_LOAD = 60


# =============================================
# File: drivers/driver_factory.py
# =============================================
"""
Browser creation factory. Supports local and Grid execution.
"""
from selenium import webdriver
from selenium.webdriver.chrome.options import Options as ChromeOptions
from selenium.webdriver.firefox.options import Options as FirefoxOptions
from selenium.webdriver.edge.options import Options as EdgeOptions


class DriverFactory:

    @staticmethod
    def create_driver(browser="chrome", headless=False, grid_url=None):
        """
        Create a WebDriver instance.

        Args:
            browser: "chrome", "firefox", or "edge"
            headless: Run without GUI
            grid_url: Selenium Grid URL (None for local)

        Returns:
            WebDriver instance
        """
        if browser == "chrome":
            options = ChromeOptions()
            options.add_argument("--window-size=1920,1080")
            options.add_argument("--disable-gpu")
            options.add_argument("--no-sandbox")
            options.add_argument("--disable-dev-shm-usage")
            if headless:
                options.add_argument("--headless=new")

            if grid_url:
                return webdriver.Remote(
                    command_executor=grid_url,
                    options=options
                )
            return webdriver.Chrome(options=options)

        elif browser == "firefox":
            options = FirefoxOptions()
            if headless:
                options.add_argument("--headless")
                options.add_argument("--width=1920")
                options.add_argument("--height=1080")

            if grid_url:
                return webdriver.Remote(
                    command_executor=grid_url,
                    options=options
                )
            return webdriver.Firefox(options=options)

        elif browser == "edge":
            options = EdgeOptions()
            options.add_argument("--window-size=1920,1080")
            if headless:
                options.add_argument("--headless=new")

            if grid_url:
                return webdriver.Remote(
                    command_executor=grid_url,
                    options=options
                )
            return webdriver.Edge(options=options)

        raise ValueError(f"Unsupported browser: {browser}")


# =============================================
# File: utils/wait_helper.py
# =============================================
"""
Reusable explicit wait methods. Never use time.sleep().
"""
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import (
    NoSuchElementException,
    TimeoutException,
)


class WaitHelper:

    def __init__(self, driver, timeout=10):
        self.driver = driver
        self.wait = WebDriverWait(driver, timeout)

    def wait_for_visible(self, locator):
        """Wait until element is visible and return it."""
        return self.wait.until(
            EC.visibility_of_element_located(locator)
        )

    def wait_for_clickable(self, locator):
        """Wait until element is clickable and return it."""
        return self.wait.until(
            EC.element_to_be_clickable(locator)
        )

    def wait_for_invisible(self, locator):
        """Wait until element disappears."""
        return self.wait.until(
            EC.invisibility_of_element_located(locator)
        )

    def wait_for_url_contains(self, url_part):
        """Wait until URL contains the expected text."""
        return self.wait.until(EC.url_contains(url_part))

    def wait_for_title_contains(self, title_part):
        """Wait until page title contains expected text."""
        return self.wait.until(EC.title_contains(title_part))

    def wait_for_text_present(self, locator, text):
        """Wait for element to contain specific text."""
        return self.wait.until(
            EC.text_to_be_present_in_element(locator, text)
        )

    def wait_for_element_count(self, locator, count):
        """Wait until a specific number of elements are present."""
        return self.wait.until(
            lambda d: len(d.find_elements(*locator)) >= count
        )


# =============================================
# File: utils/screenshot_helper.py
# =============================================
"""
Screenshot capture and organization.
"""
import os
from datetime import datetime
from config.settings import Settings


class ScreenshotHelper:

    @staticmethod
    def capture(driver, name="screenshot"):
        """Take a full-page screenshot."""
        os.makedirs(Settings.SCREENSHOT_DIR, exist_ok=True)

        timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
        filename = f"{timestamp}_{name}.png"
        filepath = os.path.join(Settings.SCREENSHOT_DIR, filename)

        try:
            driver.save_screenshot(filepath)
            print(f"Screenshot: {filepath}")
            return filepath
        except Exception as e:
            print(f"Screenshot failed: {e}")
            return None

    @staticmethod
    def capture_element(element, name="element"):
        """Take a screenshot of a specific element."""
        os.makedirs(Settings.SCREENSHOT_DIR, exist_ok=True)

        timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
        filename = f"{timestamp}_{name}_element.png"
        filepath = os.path.join(Settings.SCREENSHOT_DIR, filename)

        try:
            element.screenshot(filepath)
            print(f"Element screenshot: {filepath}")
            return filepath
        except Exception as e:
            print(f"Element screenshot failed: {e}")
            return None


# =============================================
# File: pages/base_page.py
# =============================================
"""
Base class for all page objects.
"""
from selenium.webdriver.common.by import By
from selenium.common.exceptions import NoSuchElementException
from utils.wait_helper import WaitHelper


class BasePage:

    def __init__(self, driver):
        self.driver = driver
        self.wait = WaitHelper(driver)

    @property
    def current_url(self):
        return self.driver.current_url

    @property
    def page_title(self):
        return self.driver.title

    def navigate_to(self, url):
        self.driver.get(url)

    def scroll_to_element(self, element):
        self.driver.execute_script(
            "arguments[0].scrollIntoView({block: 'center'});",
            element
        )

    def is_element_present(self, locator):
        try:
            self.driver.find_element(*locator)
            return True
        except NoSuchElementException:
            return False


# =============================================
# File: conftest.py (root level)
# =============================================
"""
Global pytest fixtures and hooks.
"""
import os
import pytest
from datetime import datetime
from drivers.driver_factory import DriverFactory
from config.settings import Settings
from utils.wait_helper import WaitHelper


@pytest.fixture(scope="function")
def driver():
    """Create a browser driver for each test."""
    drv = DriverFactory.create_driver(
        browser=Settings.BROWSER,
        headless=Settings.HEADLESS,
        grid_url=Settings.GRID_URL or None
    )
    yield drv
    drv.quit()


@pytest.fixture
def wait(driver):
    """Explicit wait helper."""
    return WaitHelper(driver, Settings.DEFAULT_TIMEOUT)


@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Auto-capture screenshot on test failure."""
    outcome = yield
    report = outcome.get_result()

    if report.when == "call" and report.failed:
        driver = item.funcargs.get("driver")
        if driver:
            os.makedirs(Settings.SCREENSHOT_DIR, exist_ok=True)
            timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
            filepath = os.path.join(
                Settings.SCREENSHOT_DIR,
                f"{timestamp}_{item.name}_FAILED.png"
            )
            try:
                driver.save_screenshot(filepath)
            except Exception:
                pass


# =============================================
# File: pytest.ini
# =============================================
"""
[pytest]
testpaths = tests
addopts = -v --tb=short
markers =
    smoke: Quick smoke tests
    regression: Full regression suite
    slow: Tests that take more than 30 seconds
"""


# =============================================
# File: .gitignore (for Python Selenium projects)
# =============================================
GITIGNORE_CONTENT = """
# Python
__pycache__/
*.py[cod]
*.egg-info/
.eggs/
dist/
build/
*.egg

# Virtual environment
venv/
.venv/
env/

# IDE
.idea/
.vscode/
*.suo
*.user

# Test outputs
reports/
TestResults/
*.html
allure-results/

# OS
.DS_Store
Thumbs.db

# Selenium
*.log
geckodriver.log
chromedriver.log

# Environment
.env
*.env
"""
```

---

## 6. Step-by-Step Walkthrough

### Organizing an Existing Project

1. **Identify all files** and categorize them: test, page, utility, config
2. **Create the directory structure** as shown above
3. **Move files** into their correct directories
4. **Update imports/references** after moving files
5. **Extract common code** into utility classes (waits, screenshots, config)
6. **Create a BaseTest/conftest** with shared setup/teardown
7. **Create a BasePage** with common page object methods
8. **Rename files** to follow naming conventions
9. **Add `.gitignore`** to exclude reports, screenshots, and IDE files
10. **Verify everything still runs** after reorganization

---

## 7. Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Flat structure (all files in one folder) | Cannot find anything at 50+ files | Use the directory structure above |
| Test logic in page objects | Pages become untestable, hard to reuse | Pages contain only element interactions, no assertions |
| Assertions in page objects | Couples page to specific test expectations | Return values from pages, assert in tests |
| Hardcoded values everywhere | Changing a URL means editing 50 files | Use constants and config files |
| Copy-pasted wait logic | Inconsistent timeout handling | Create WaitHelper class |
| No base test class | Duplicate setup/teardown in every test | Create BaseTest with common lifecycle |

---

## 8. Best Practices

1. **Page objects should have NO assertions.** They describe what you can do on a page. Tests decide what is correct.

2. **One page object per page (or significant component).** Do not put 5 pages in one file.

3. **Config should support multiple environments** (dev, staging, production) without code changes.

4. **Tests should be readable as documentation.** Someone unfamiliar with the code should understand what is being tested.

5. **Keep test data separate from test logic.** Use JSON files, CSV files, or constants classes.

6. **Version control your framework.** Use `.gitignore` to exclude generated files but include everything else.

---

## 9. Hands-On Exercise

### Task: Reorganize All Previous Code into Proper Framework Structure

**Objective:** Take the code from Days 22-28 and organize it into a professional framework.

**Steps:**
1. Create the complete directory structure
2. Create `DriverFactory` that supports Chrome, Firefox, Edge, local, and Grid
3. Create `WaitHelper` with all common wait methods
4. Create `ScreenshotHelper` with auto-capture on failure
5. Create `ConfigReader`/`Settings` that reads from config files
6. Create `BasePage` with common page operations
7. Create `BaseTest`/`conftest.py` with driver lifecycle management
8. Move at least 3 test classes into the new structure
9. Verify all tests still pass

---

## 10. Real-World Scenario

### Scenario: Enterprise Framework for 20-Person QA Team

```
CompanyAutomation/
├── README.md                    # Getting started, architecture docs
├── CONTRIBUTING.md              # How to add new tests
├── docker-compose.yml           # Grid setup
├── .github/
│   └── workflows/
│       ├── smoke-tests.yml      # Runs on every PR
│       ├── regression.yml       # Runs nightly
│       └── release-tests.yml    # Runs before deployment
│
├── src/                         # Framework core (shared package)
│   ├── pages/
│   ├── utils/
│   ├── drivers/
│   └── config/
│
├── tests/
│   ├── smoke/                   # 50 critical tests, 5 min
│   ├── regression/              # 500 tests, 30 min
│   │   ├── auth/
│   │   ├── search/
│   │   ├── checkout/
│   │   └── admin/
│   └── performance/             # Load-related UI tests
│
└── data/
    ├── test_users.json
    ├── test_products.csv
    └── fixtures/                 # Database seed data
```

---

## 11. Resources

- [Selenium Best Practices](https://www.selenium.dev/documentation/test_practices/)
- [Page Object Model](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/)
- [NUnit Project Structure](https://docs.nunit.org/articles/nunit/getting-started/installation.html)
- [pytest Project Layout](https://docs.pytest.org/en/latest/goodpractices.html)
- [Python Project Structure](https://docs.python-guide.org/writing/structure/)
- [.gitignore Templates](https://github.com/github/gitignore)
