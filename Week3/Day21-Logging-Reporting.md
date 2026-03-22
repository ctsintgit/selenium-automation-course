# Day 21: Logging and Reporting (ExtentReports / Allure) + Mini Project 3

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain why test reporting matters for stakeholders and debugging
- Set up ExtentReports in C# to generate HTML test reports
- Set up Allure in Python to generate interactive test reports
- Add screenshots to reports on test failure
- Implement logging with NLog (C#) and the logging module (Python)
- Build Mini Project 3: a complete mini framework with POM, config, data-driven tests, and reporting

---

## 1. Why Test Reporting Matters

Running tests in the terminal gives you pass/fail counts, but stakeholders need more:

- **Visual reports** — HTML reports with charts, timelines, and screenshots
- **Failure evidence** — Screenshots showing exactly what the browser displayed when a test failed
- **History** — Trends over time (are tests getting flakier?)
- **Categorization** — Group tests by feature, severity, or status
- **Logging** — Detailed step-by-step logs for debugging intermittent failures

---

## 2. How It Works

### C#: ExtentReports

ExtentReports is a popular open-source reporting library. It generates HTML reports with test details, screenshots, and status indicators.

**Flow:**
1. Create an `ExtentReports` instance at the start of the test run
2. Create an `ExtentTest` for each test method
3. Log steps (`test.Pass()`, `test.Fail()`, `test.Info()`)
4. Attach screenshots on failure
5. Flush the report at the end (writes the HTML file)

### Python: Allure

Allure is a test report framework that generates interactive HTML reports. It integrates with pytest via the `allure-pytest` plugin.

**Flow:**
1. Install `allure-pytest`
2. Run tests with `--alluredir=results`
3. Use decorators like `@allure.step`, `@allure.feature`, `@allure.severity`
4. Attach screenshots with `allure.attach`
5. Generate HTML with `allure serve results`

---

## 3. Code Example: C# — ExtentReports Setup

### Install NuGet Package

```bash
dotnet add package ExtentReports
```

### Report Manager

```csharp
// File: Reporting/ReportManager.cs
using AventStack.ExtentReports;
using AventStack.ExtentReports.Reporter;
using AventStack.ExtentReports.Reporter.Config;
using System;
using System.IO;

namespace SeleniumTests.Reporting
{
    /// <summary>
    /// Manages the ExtentReports lifecycle.
    /// Singleton — one report per test run.
    /// </summary>
    public static class ReportManager
    {
        private static ExtentReports _extent;
        private static string _reportPath;

        /// <summary>
        /// Initializes the report. Call once in [OneTimeSetUp] of the base class.
        /// </summary>
        public static ExtentReports GetInstance()
        {
            if (_extent == null)
            {
                // Create the report directory
                string reportDir = Path.Combine(
                    AppDomain.CurrentDomain.BaseDirectory, "Reports");
                Directory.CreateDirectory(reportDir);

                // Create the HTML file with a timestamp
                string timestamp = DateTime.Now.ToString("yyyyMMdd_HHmmss");
                _reportPath = Path.Combine(reportDir, $"TestReport_{timestamp}.html");

                // Configure the HTML reporter
                var htmlReporter = new ExtentSparkReporter(_reportPath);
                htmlReporter.Config.DocumentTitle = "Selenium Test Report";
                htmlReporter.Config.ReportName = "Sauce Demo Automation";
                htmlReporter.Config.Theme = Theme.Standard;

                // Create the ExtentReports instance
                _extent = new ExtentReports();
                _extent.AttachReporter(htmlReporter);
                _extent.AddSystemInfo("OS", Environment.OSVersion.ToString());
                _extent.AddSystemInfo("Machine", Environment.MachineName);
                _extent.AddSystemInfo("Browser", "Chrome");
            }
            return _extent;
        }

        /// <summary>
        /// Returns the report file path.
        /// </summary>
        public static string ReportPath => _reportPath;

        /// <summary>
        /// Flushes the report to disk. Call once in [OneTimeTearDown].
        /// </summary>
        public static void Flush()
        {
            _extent?.Flush();
        }
    }
}
```

### Updated Base Test with Reporting

```csharp
// File: Base/BaseTestWithReporting.cs
using AventStack.ExtentReports;
using NUnit.Framework;
using NUnit.Framework.Interfaces;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using SeleniumTests.Reporting;
using System;
using System.IO;

namespace SeleniumTests.Base
{
    [TestFixture]
    public abstract class BaseTestWithReporting
    {
        protected IWebDriver Driver { get; private set; }
        protected WebDriverWait Wait { get; private set; }

        // Each test gets its own ExtentTest for logging
        protected ExtentTest TestReport { get; private set; }

        private static ExtentReports _extent;

        [OneTimeSetUp]
        public void OneTimeSetUp()
        {
            // Initialize the report
            _extent = ReportManager.GetInstance();

            // Create the driver
            var options = new ChromeOptions();
            options.AddArgument("--start-maximized");
            Driver = new ChromeDriver(options);
            Wait = new WebDriverWait(Driver, TimeSpan.FromSeconds(10));
        }

        [SetUp]
        public void SetUp()
        {
            // Create a test entry in the report for this test method
            TestReport = _extent.CreateTest(TestContext.CurrentContext.Test.Name);
            TestReport.Info("Test started");

            Driver.Navigate().GoToUrl("https://www.saucedemo.com");
            TestReport.Info("Navigated to Sauce Demo");
        }

        [TearDown]
        public void TearDown()
        {
            var status = TestContext.CurrentContext.Result.Outcome.Status;
            var message = TestContext.CurrentContext.Result.Message ?? "";

            switch (status)
            {
                case TestStatus.Passed:
                    TestReport.Pass("Test passed");
                    break;

                case TestStatus.Failed:
                    // Log the failure message
                    TestReport.Fail($"Test failed: {message}");

                    // Capture and attach screenshot
                    string screenshotPath = CaptureScreenshot(
                        TestContext.CurrentContext.Test.Name);
                    if (screenshotPath != null)
                    {
                        TestReport.Fail("Screenshot",
                            MediaEntityBuilder.CreateScreenCaptureFromPath(
                                screenshotPath).Build());
                    }
                    break;

                case TestStatus.Skipped:
                    TestReport.Skip("Test was skipped");
                    break;
            }

            Driver.Manage().Cookies.DeleteAllCookies();
        }

        [OneTimeTearDown]
        public void OneTimeTearDown()
        {
            Driver?.Quit();

            // Flush writes the HTML report to disk
            ReportManager.Flush();
            Console.WriteLine($"Report saved to: {ReportManager.ReportPath}");
        }

        private string CaptureScreenshot(string testName)
        {
            try
            {
                var screenshot = ((ITakesScreenshot)Driver).GetScreenshot();
                string dir = Path.Combine(
                    AppDomain.CurrentDomain.BaseDirectory, "Reports", "Screenshots");
                Directory.CreateDirectory(dir);
                string path = Path.Combine(dir,
                    $"{testName}_{DateTime.Now:yyyyMMdd_HHmmss}.png");
                screenshot.SaveAsFile(path);
                return path;
            }
            catch
            {
                return null;
            }
        }
    }
}
```

### Test Using Reporting

```csharp
// File: Tests/LoginReportingTests.cs
using NUnit.Framework;
using OpenQA.Selenium;
using SeleniumTests.Base;

namespace SeleniumTests.Tests
{
    [TestFixture]
    public class LoginReportingTests : BaseTestWithReporting
    {
        [Test]
        public void ValidLogin_ShouldShowProducts()
        {
            TestReport.Info("Entering username: standard_user");
            Driver.FindElement(By.Id("user-name")).SendKeys("standard_user");

            TestReport.Info("Entering password");
            Driver.FindElement(By.Id("password")).SendKeys("secret_sauce");

            TestReport.Info("Clicking login button");
            Driver.FindElement(By.Id("login-button")).Click();

            Wait.Until(d => d.Url.Contains("inventory"));

            TestReport.Info("Verifying page title");
            var title = Driver.FindElement(By.ClassName("title")).Text;
            Assert.That(title, Is.EqualTo("Products"));
        }

        [Test]
        public void InvalidLogin_ShouldShowError()
        {
            TestReport.Info("Entering invalid credentials");
            Driver.FindElement(By.Id("user-name")).SendKeys("bad_user");
            Driver.FindElement(By.Id("password")).SendKeys("bad_pass");
            Driver.FindElement(By.Id("login-button")).Click();

            var error = Wait.Until(d =>
                d.FindElement(By.CssSelector("[data-test='error']")));

            TestReport.Info($"Error message: {error.Text}");
            Assert.That(error.Text, Does.Contain("Username and password"));
        }
    }
}
```

---

## 4. Code Example: Python — Allure Reporting

### Install Packages

```bash
pip install allure-pytest
```

You also need the Allure command-line tool to generate HTML:

```bash
# macOS
brew install allure

# Windows (scoop)
scoop install allure

# Or download from: https://github.com/allure-framework/allure2/releases
```

### conftest.py with Allure Integration

```python
# File: conftest.py
import pytest
import allure
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from datetime import datetime
import os


@pytest.fixture(scope="function")
def driver():
    """Creates a ChromeDriver for each test."""
    options = webdriver.ChromeOptions()
    options.add_argument("--start-maximized")
    browser = webdriver.Chrome(options=options)
    browser.get("https://www.saucedemo.com")

    yield browser

    browser.quit()


@pytest.fixture(scope="function")
def wait(driver):
    """Provides a WebDriverWait instance."""
    return WebDriverWait(driver, 10)


@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """
    Automatically attaches a screenshot to the Allure report
    when a test fails.
    """
    outcome = yield
    report = outcome.get_result()

    if report.when == "call" and report.failed:
        driver = item.funcargs.get("driver")
        if driver:
            # Take screenshot
            screenshot = driver.get_screenshot_as_png()
            allure.attach(
                screenshot,
                name=f"FAIL_{item.name}",
                attachment_type=allure.attachment_type.PNG
            )

            # Attach page source for debugging
            allure.attach(
                driver.page_source,
                name="Page Source",
                attachment_type=allure.attachment_type.HTML
            )
```

### Tests with Allure Decorators

```python
# File: tests/test_login_allure.py
import pytest
import allure
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


@allure.feature("Authentication")
@allure.story("Login")
class TestLoginAllure:
    """Login tests with Allure reporting integration."""

    @allure.title("Valid login redirects to inventory")
    @allure.severity(allure.severity_level.CRITICAL)
    def test_valid_login(self, driver, wait):
        with allure.step("Enter username"):
            driver.find_element(By.ID, "user-name").send_keys("standard_user")

        with allure.step("Enter password"):
            driver.find_element(By.ID, "password").send_keys("secret_sauce")

        with allure.step("Click login button"):
            driver.find_element(By.ID, "login-button").click()

        with allure.step("Verify redirect to inventory page"):
            wait.until(EC.url_contains("inventory"))
            assert "inventory" in driver.current_url

        with allure.step("Verify page title is Products"):
            title = driver.find_element(By.CLASS_NAME, "title").text
            assert title == "Products"

    @allure.title("Invalid login shows error message")
    @allure.severity(allure.severity_level.NORMAL)
    def test_invalid_login(self, driver, wait):
        with allure.step("Enter invalid credentials"):
            driver.find_element(By.ID, "user-name").send_keys("bad_user")
            driver.find_element(By.ID, "password").send_keys("bad_pass")

        with allure.step("Click login"):
            driver.find_element(By.ID, "login-button").click()

        with allure.step("Verify error message appears"):
            error = wait.until(
                EC.presence_of_element_located(
                    (By.CSS_SELECTOR, "[data-test='error']")
                )
            )
            assert "Username and password" in error.text

    @allure.title("Empty credentials show username required error")
    @allure.severity(allure.severity_level.MINOR)
    def test_empty_credentials(self, driver, wait):
        with allure.step("Click login without entering credentials"):
            driver.find_element(By.ID, "login-button").click()

        with allure.step("Verify 'Username is required' error"):
            error = wait.until(
                EC.presence_of_element_located(
                    (By.CSS_SELECTOR, "[data-test='error']")
                )
            )
            assert "Username is required" in error.text
```

### Running and Viewing Allure Reports

```bash
# Run tests and generate Allure results
pytest tests/ --alluredir=allure-results -v

# View the report in a browser (opens automatically)
allure serve allure-results

# Or generate a static HTML report
allure generate allure-results -o allure-report --clean
```

---

## 5. Logging

### C# — NLog Setup

```bash
dotnet add package NLog
dotnet add package NLog.Extensions.Logging
```

```xml
<!-- File: nlog.config -->
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <targets>
        <target name="file" xsi:type="File"
                fileName="logs/test-${shortdate}.log"
                layout="${longdate} | ${level:uppercase=true} | ${logger} | ${message}" />
        <target name="console" xsi:type="Console"
                layout="${time} | ${level:uppercase=true} | ${message}" />
    </targets>
    <rules>
        <logger name="*" minlevel="Info" writeTo="file,console" />
    </rules>
</nlog>
```

```csharp
// File: Base/LoggedBaseTest.cs (logging example)
using NLog;

public abstract class LoggedBaseTest : BaseTestWithReporting
{
    // Create a logger for each test class
    protected static readonly Logger Log = LogManager.GetCurrentClassLogger();

    [OneTimeSetUp]
    public new void OneTimeSetUp()
    {
        base.OneTimeSetUp();
        Log.Info("Test class started: {0}", GetType().Name);
    }

    [SetUp]
    public new void SetUp()
    {
        base.SetUp();
        Log.Info("Test started: {0}", TestContext.CurrentContext.Test.Name);
    }
}
```

### Python — logging Module

```python
# File: utils/logger.py
import logging
import os
from datetime import datetime


def get_logger(name: str) -> logging.Logger:
    """
    Creates a configured logger that writes to both console and file.
    """
    logger = logging.getLogger(name)

    # Avoid adding handlers multiple times
    if logger.handlers:
        return logger

    logger.setLevel(logging.DEBUG)

    # Console handler
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    console_format = logging.Formatter(
        "%(asctime)s | %(levelname)-8s | %(message)s",
        datefmt="%H:%M:%S"
    )
    console_handler.setFormatter(console_format)

    # File handler
    os.makedirs("logs", exist_ok=True)
    timestamp = datetime.now().strftime("%Y%m%d")
    file_handler = logging.FileHandler(f"logs/test-{timestamp}.log")
    file_handler.setLevel(logging.DEBUG)
    file_format = logging.Formatter(
        "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s"
    )
    file_handler.setFormatter(file_format)

    logger.addHandler(console_handler)
    logger.addHandler(file_handler)

    return logger
```

```python
# Usage in a page object
from utils.logger import get_logger

class LoginPage:
    log = get_logger("LoginPage")

    def login(self, username, password):
        self.log.info(f"Logging in as '{username}'")
        # ... selenium code ...
        self.log.info("Login completed")
```

---

## 6. Mini Project 3: Complete Framework

Build a complete test automation framework for Sauce Demo that combines everything from Week 3.

### Project Structure

```
MiniProject3/
├── CSharp/
│   ├── MiniProject3.csproj
│   ├── appsettings.json
│   ├── nlog.config
│   ├── Base/
│   │   └── BaseTest.cs
│   ├── Config/
│   │   └── TestConfiguration.cs
│   ├── Pages/
│   │   ├── LoginPage.cs
│   │   └── InventoryPage.cs
│   ├── Reporting/
│   │   └── ReportManager.cs
│   ├── TestData/
│   │   └── login_data.json
│   └── Tests/
│       ├── LoginTests.cs
│       └── InventoryTests.cs
│
├── Python/
│   ├── conftest.py
│   ├── pytest.ini
│   ├── config/
│   │   ├── config.yaml
│   │   └── config_reader.py
│   ├── pages/
│   │   ├── __init__.py
│   │   ├── login_page.py
│   │   └── inventory_page.py
│   ├── test_data/
│   │   └── login_data.json
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── test_login.py
│   │   └── test_inventory.py
│   └── utils/
│       ├── __init__.py
│       ├── data_loader.py
│       └── logger.py
```

### C# Mini Project — Key Files

```json
// File: appsettings.json
{
    "TestSettings": {
        "BaseUrl": "https://www.saucedemo.com",
        "Browser": "chrome",
        "Headless": false,
        "TimeoutSeconds": 10,
        "ScreenshotPath": "./Reports/Screenshots"
    },
    "Credentials": {
        "StandardUser": "standard_user",
        "Password": "secret_sauce"
    }
}
```

```json
// File: TestData/login_data.json
[
    { "username": "standard_user", "password": "secret_sauce", "shouldSucceed": true, "description": "Standard user" },
    { "username": "problem_user", "password": "secret_sauce", "shouldSucceed": true, "description": "Problem user" },
    { "username": "locked_out_user", "password": "secret_sauce", "shouldSucceed": false, "description": "Locked out user" },
    { "username": "invalid_user", "password": "wrong", "shouldSucceed": false, "description": "Invalid credentials" },
    { "username": "", "password": "", "shouldSucceed": false, "description": "Empty credentials" }
]
```

```csharp
// File: Tests/LoginTests.cs (Mini Project 3 - complete)
using NUnit.Framework;
using OpenQA.Selenium;
using SeleniumTests.Base;
using SeleniumTests.Config;
using SeleniumTests.Pages;
using System.Collections.Generic;
using System.IO;
using System.Text.Json;

namespace SeleniumTests.Tests
{
    [TestFixture]
    public class LoginTests : BaseTestWithReporting
    {
        private LoginPage loginPage;

        [SetUp]
        public void NavigateToLogin()
        {
            loginPage = new LoginPage(Driver);
        }

        // --- DATA-DRIVEN TESTS ---

        private static IEnumerable<TestCaseData> LoginData()
        {
            string path = Path.Combine(
                TestContext.CurrentContext.TestDirectory, "TestData", "login_data.json");
            string json = File.ReadAllText(path);
            var items = JsonSerializer.Deserialize<List<Dictionary<string, JsonElement>>>(json);

            foreach (var item in items)
            {
                yield return new TestCaseData(
                    item["username"].GetString(),
                    item["password"].GetString(),
                    item["shouldSucceed"].GetBoolean()
                ).SetName($"Login_{item["description"].GetString().Replace(" ", "_")}");
            }
        }

        [Test]
        [TestCaseSource(nameof(LoginData))]
        public void Login_DataDriven(string username, string password, bool shouldSucceed)
        {
            TestReport.Info($"Testing login with user: '{username}'");

            if (shouldSucceed)
            {
                var inventory = loginPage.Login(username, password);
                TestReport.Pass($"Login successful, title: {inventory.GetPageTitle()}");
                Assert.That(inventory.GetPageTitle(), Is.EqualTo("Products"));
            }
            else
            {
                loginPage.LoginExpectingError(username, password);
                string error = loginPage.GetErrorMessage();
                TestReport.Info($"Error shown: {error}");
                Assert.That(error, Is.Not.Empty, "Expected an error message");
            }
        }

        // --- INDIVIDUAL TESTS ---

        [Test]
        public void ValidLogin_ShouldShowSixProducts()
        {
            TestReport.Info("Logging in with standard_user");
            var config = TestConfiguration.Instance;
            var inventory = loginPage.Login(config.StandardUser, config.Password);

            int count = inventory.GetProductCount();
            TestReport.Info($"Product count: {count}");

            Assert.That(count, Is.EqualTo(6));
        }
    }
}
```

```csharp
// File: Tests/InventoryTests.cs
using NUnit.Framework;
using SeleniumTests.Base;
using SeleniumTests.Config;
using SeleniumTests.Pages;

namespace SeleniumTests.Tests
{
    [TestFixture]
    public class InventoryTests : BaseTestWithReporting
    {
        private InventoryPage inventoryPage;

        [SetUp]
        public void LoginFirst()
        {
            var config = TestConfiguration.Instance;
            var loginPage = new LoginPage(Driver);
            inventoryPage = loginPage.Login(config.StandardUser, config.Password);
        }

        [Test]
        public void Products_ShouldDisplayTitle()
        {
            TestReport.Info("Verifying page title");
            Assert.That(inventoryPage.GetPageTitle(), Is.EqualTo("Products"));
        }

        [Test]
        public void AddToCart_ShouldIncrementBadge()
        {
            TestReport.Info("Adding first product to cart");
            inventoryPage.AddFirstProductToCart();

            int cartCount = inventoryPage.GetCartItemCount();
            TestReport.Info($"Cart count: {cartCount}");

            Assert.That(cartCount, Is.EqualTo(1));
        }

        [Test]
        public void Products_ShouldHaveNames()
        {
            var names = inventoryPage.GetProductNames();
            TestReport.Info($"Found {names.Count} product names");

            Assert.That(names.Count, Is.EqualTo(6));
            Assert.That(names, Has.Some.Contain("Sauce Labs"));
        }
    }
}
```

### Python Mini Project — Key Files

```yaml
# File: config/config.yaml
test_settings:
  base_url: "https://www.saucedemo.com"
  browser: "chrome"
  headless: false
  timeout_seconds: 10

credentials:
  standard_user: "standard_user"
  password: "secret_sauce"
```

```ini
# File: pytest.ini
[pytest]
testpaths = tests
addopts = --alluredir=allure-results -v
markers =
    smoke: Quick smoke tests
    regression: Full regression tests
```

```python
# File: conftest.py (Mini Project 3 - complete)
import pytest
import allure
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from config.config_reader import TestConfiguration


@pytest.fixture(scope="session")
def config():
    return TestConfiguration()


@pytest.fixture(scope="function")
def driver(config):
    options = webdriver.ChromeOptions()
    if config.headless:
        options.add_argument("--headless")
    options.add_argument("--start-maximized")

    browser = webdriver.Chrome(options=options)
    browser.get(config.base_url)

    yield browser

    browser.quit()


@pytest.fixture(scope="function")
def wait(driver, config):
    return WebDriverWait(driver, config.timeout_seconds)


@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()

    if report.when == "call" and report.failed:
        driver = item.funcargs.get("driver")
        if driver:
            allure.attach(
                driver.get_screenshot_as_png(),
                name=f"FAIL_{item.name}",
                attachment_type=allure.attachment_type.PNG
            )
```

```python
# File: tests/test_login.py (Mini Project 3 - complete)
import pytest
import allure
from pages.login_page import LoginPage
from utils.data_loader import json_to_params_with_ids


# Load test data at module level
LOGIN_PARAMS, LOGIN_IDS = json_to_params_with_ids(
    "login_data.json",
    ["username", "password", "should_succeed"],
    "description"
)


@allure.feature("Authentication")
class TestLogin:

    @allure.story("Data-Driven Login")
    @allure.severity(allure.severity_level.CRITICAL)
    @pytest.mark.parametrize(
        "username, password, should_succeed",
        LOGIN_PARAMS,
        ids=LOGIN_IDS
    )
    def test_login_data_driven(self, driver, username, password, should_succeed):
        login_page = LoginPage(driver)

        if should_succeed:
            with allure.step(f"Login as '{username}'"):
                inventory = login_page.login(username, password)
            with allure.step("Verify inventory page loaded"):
                assert inventory.get_page_title() == "Products"
        else:
            with allure.step(f"Attempt login as '{username}' (expecting error)"):
                login_page.login_expecting_error(username, password)
            with allure.step("Verify error message displayed"):
                assert login_page.get_error_message() != ""

    @allure.story("Product Count")
    @allure.severity(allure.severity_level.NORMAL)
    def test_valid_login_shows_six_products(self, driver, config):
        login_page = LoginPage(driver)
        with allure.step("Login with configured credentials"):
            inventory = login_page.login(
                config.standard_user, config.password
            )
        with allure.step("Verify 6 products displayed"):
            assert inventory.get_product_count() == 6


@allure.feature("Inventory")
class TestInventory:

    @allure.story("Add to Cart")
    def test_add_product_to_cart(self, driver, config):
        login_page = LoginPage(driver)
        inventory = login_page.login(config.standard_user, config.password)

        with allure.step("Add first product to cart"):
            inventory.add_first_product_to_cart()

        with allure.step("Verify cart badge shows 1"):
            assert inventory.get_cart_item_count() == 1

    @allure.story("Product Names")
    def test_products_have_sauce_labs_items(self, driver, config):
        login_page = LoginPage(driver)
        inventory = login_page.login(config.standard_user, config.password)

        with allure.step("Get all product names"):
            names = inventory.get_product_names()

        with allure.step("Verify Sauce Labs products exist"):
            assert len(names) == 6
            assert any("Sauce Labs" in name for name in names)
```

### Running the Mini Project

```bash
# C#
dotnet test --verbosity normal
# Report generated at: Reports/TestReport_20260321_143000.html

# Python
pytest tests/ --alluredir=allure-results -v
allure serve allure-results
# Opens interactive HTML report in browser
```

---

## 7. Common Mistakes and How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `_extent.Flush()` (C#) | Report HTML file is empty | Always call `Flush()` in `[OneTimeTearDown]` |
| Not installing the Allure CLI | `allure serve` fails with "command not found" | Install via brew/scoop/download |
| Logging too much detail | Reports are cluttered and slow | Log key steps, not every line of code |
| Screenshots saved with relative paths | ExtentReports cannot find them | Use absolute paths or paths relative to the report |
| Not attaching screenshots in Python | Failures have no visual evidence | Use the `pytest_runtest_makereport` hook |
| Logging passwords | Security risk in shared reports | Never log sensitive values |

---

## 8. Best Practices

1. **Automate screenshot capture** — Use hooks/teardown so you never forget
2. **Log at the right level** — INFO for steps, DEBUG for details, ERROR for failures
3. **Keep reports in CI artifacts** — Archive HTML reports as build artifacts
4. **Clean old reports** — Use timestamps in filenames and periodically delete old ones
5. **Add system info** — Browser version, OS, environment name in the report header
6. **Use Allure categories** — `@allure.feature`, `@allure.story`, `@allure.severity` for organization

---

## 9. Hands-On Exercise

**Task:** Build Mini Project 3 end-to-end.

1. Set up the project structure (pages, tests, config, reporting)
2. Create page objects for LoginPage and InventoryPage
3. Create a config file with base URL, browser, and credentials
4. Create a test data JSON file with 5 login scenarios
5. Write data-driven login tests with reporting
6. Write 2 inventory tests with step logging
7. Run the suite and generate the HTML report

**Expected Results:**

```
8 tests total:
  5 data-driven login tests (3 pass, 2 pass with error verification)
  1 product count test
  1 add-to-cart test
  1 product names test

HTML report generated with:
  - Pass/fail summary
  - Screenshots for any failures
  - Step-by-step logs
  - System information
```

---

## 10. Real-World Scenario

A QA team at a fintech company runs 200 Selenium tests every night. Without reporting:

- A developer asks "did the login tests pass?" and the QA engineer scrolls through 2000 lines of terminal output
- A test failed two weeks ago but nobody noticed
- The PM asks for a testing status update and gets a vague "mostly passing"

With ExtentReports/Allure:

- The nightly run produces an HTML report emailed to the team
- Failed tests have screenshots showing exactly what went wrong
- The PM opens the report and sees "192/200 passed, 8 failed" with a chart
- Developers click on a failed test, see the screenshot, and fix the bug in minutes

---

## 11. Resources

- [ExtentReports .NET Documentation](https://www.extentreports.com/docs/versions/5/net/index.html)
- [Allure Framework](https://docs.qameta.io/allure/)
- [allure-pytest Plugin](https://pypi.org/project/allure-pytest/)
- [NLog Documentation](https://nlog-project.org/documentation/)
- [Python logging Module](https://docs.python.org/3/library/logging.html)
- [Sauce Demo Practice Site](https://www.saucedemo.com)
