# Day 19: Config Files — JSON/YAML for Environment-Driven Execution

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain why configuration should be externalized from test code
- Read configuration from `appsettings.json` in C# using `Microsoft.Extensions.Configuration`
- Read configuration from `config.yaml` or `config.json` in Python
- Create environment-specific configs (dev, staging, production)
- Store sensitive data using environment variables and `.env` files
- Set up `.gitignore` to prevent secrets from being committed

---

## 1. Why Externalize Configuration?

Hardcoded values like URLs, credentials, timeouts, and browser choices create problems:

- **Changing environments** requires editing source code
- **Secrets in code** get committed to version control
- **Different team members** need different paths and settings
- **CI/CD pipelines** need headless mode and different URLs

Configuration files solve all of these. Your test code reads settings from a file at runtime, and you swap the file to change behavior — no code changes needed.

### What to Externalize

| Setting | Example |
|---------|---------|
| Base URL | `https://staging.saucedemo.com` |
| Browser | `chrome`, `firefox`, `edge` |
| Headless mode | `true` / `false` |
| Timeouts | `10` seconds |
| Credentials | `standard_user` / `secret_sauce` |
| Screenshot path | `./screenshots` |
| Parallel thread count | `4` |

---

## 2. How It Works

### C# Approach: `appsettings.json`

.NET projects use `Microsoft.Extensions.Configuration` to read JSON config files. This is the same system used by ASP.NET Core — well-documented and widely understood.

### Python Approach: YAML or JSON

Python projects typically use:
- `config.yaml` with the `PyYAML` package (most readable)
- `config.json` with the built-in `json` module (no extra dependency)
- `.env` files with `python-dotenv` (for secrets)

---

## 3. Code Example: C# — appsettings.json Configuration

### Step 1: Install NuGet Packages

```bash
dotnet add package Microsoft.Extensions.Configuration
dotnet add package Microsoft.Extensions.Configuration.Json
dotnet add package Microsoft.Extensions.Configuration.EnvironmentVariables
```

### Step 2: Create appsettings.json

```json
// File: appsettings.json
{
    "TestSettings": {
        "BaseUrl": "https://www.saucedemo.com",
        "Browser": "chrome",
        "Headless": false,
        "TimeoutSeconds": 10,
        "ScreenshotPath": "./screenshots"
    },
    "Credentials": {
        "StandardUser": "standard_user",
        "Password": "secret_sauce"
    }
}
```

### Step 3: Create Environment-Specific Overrides

```json
// File: appsettings.staging.json
{
    "TestSettings": {
        "BaseUrl": "https://staging.saucedemo.com",
        "Headless": true
    }
}
```

```json
// File: appsettings.ci.json
{
    "TestSettings": {
        "BaseUrl": "https://staging.saucedemo.com",
        "Browser": "chrome",
        "Headless": true,
        "TimeoutSeconds": 30
    }
}
```

### Step 4: Set Copy to Output Directory

In your `.csproj`, ensure config files are copied to the build output:

```xml
<ItemGroup>
    <None Update="appsettings.json">
        <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
    <None Update="appsettings.*.json">
        <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
</ItemGroup>
```

### Step 5: Create the Configuration Reader

```csharp
// File: Config/TestConfiguration.cs
using Microsoft.Extensions.Configuration;
using System;
using System.IO;

namespace SeleniumTests.Config
{
    /// <summary>
    /// Reads test settings from appsettings.json and environment variables.
    /// Follows the Options pattern for strongly-typed configuration.
    /// </summary>
    public class TestConfiguration
    {
        private static IConfiguration _configuration;
        private static TestConfiguration _instance;

        // --- PROPERTIES (strongly typed) ---
        public string BaseUrl { get; private set; }
        public string Browser { get; private set; }
        public bool Headless { get; private set; }
        public int TimeoutSeconds { get; private set; }
        public string ScreenshotPath { get; private set; }
        public string StandardUser { get; private set; }
        public string Password { get; private set; }

        /// <summary>
        /// Singleton instance — configuration is loaded once and reused.
        /// </summary>
        public static TestConfiguration Instance
        {
            get
            {
                if (_instance == null)
                {
                    _instance = new TestConfiguration();
                }
                return _instance;
            }
        }

        private TestConfiguration()
        {
            // Determine the environment (default: "dev")
            // Set via environment variable: TEST_ENVIRONMENT=staging
            string environment = Environment.GetEnvironmentVariable("TEST_ENVIRONMENT") ?? "dev";

            // Build configuration with layered sources
            _configuration = new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                // Base settings (always loaded)
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: false)
                // Environment-specific overrides (optional — merges on top of base)
                .AddJsonFile($"appsettings.{environment}.json", optional: true, reloadOnChange: false)
                // Environment variables override everything (for CI/CD)
                .AddEnvironmentVariables(prefix: "TEST_")
                .Build();

            // Bind values to properties
            BaseUrl = _configuration["TestSettings:BaseUrl"] ?? "https://www.saucedemo.com";
            Browser = _configuration["TestSettings:Browser"] ?? "chrome";
            Headless = bool.Parse(_configuration["TestSettings:Headless"] ?? "false");
            TimeoutSeconds = int.Parse(_configuration["TestSettings:TimeoutSeconds"] ?? "10");
            ScreenshotPath = _configuration["TestSettings:ScreenshotPath"] ?? "./screenshots";
            StandardUser = _configuration["Credentials:StandardUser"] ?? "";
            Password = _configuration["Credentials:Password"] ?? "";
        }
    }
}
```

### Step 6: Use Configuration in Base Test

```csharp
// File: Base/BaseTest.cs (updated to use config)
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Firefox;
using OpenQA.Selenium.Edge;
using OpenQA.Selenium.Support.UI;
using SeleniumTests.Config;
using System;

namespace SeleniumTests.Base
{
    [TestFixture]
    public abstract class BaseTest
    {
        protected IWebDriver Driver { get; private set; }
        protected WebDriverWait Wait { get; private set; }
        protected TestConfiguration Config => TestConfiguration.Instance;

        [OneTimeSetUp]
        public void OneTimeSetUp()
        {
            // Create driver based on config
            Driver = CreateDriver();
            Wait = new WebDriverWait(Driver, TimeSpan.FromSeconds(Config.TimeoutSeconds));
        }

        [SetUp]
        public void SetUp()
        {
            Driver.Navigate().GoToUrl(Config.BaseUrl);
        }

        [TearDown]
        public void TearDown()
        {
            if (TestContext.CurrentContext.Result.Outcome.Status ==
                NUnit.Framework.Interfaces.TestStatus.Failed)
            {
                TakeScreenshot(TestContext.CurrentContext.Test.Name);
            }
            Driver.Manage().Cookies.DeleteAllCookies();
        }

        [OneTimeTearDown]
        public void OneTimeTearDown()
        {
            Driver?.Quit();
        }

        private IWebDriver CreateDriver()
        {
            // Read browser choice from config
            switch (Config.Browser.ToLower())
            {
                case "firefox":
                    var ffOptions = new FirefoxOptions();
                    if (Config.Headless) ffOptions.AddArgument("--headless");
                    return new FirefoxDriver(ffOptions);

                case "edge":
                    var edgeOptions = new EdgeOptions();
                    if (Config.Headless) edgeOptions.AddArgument("--headless");
                    return new EdgeDriver(edgeOptions);

                case "chrome":
                default:
                    var chromeOptions = new ChromeOptions();
                    if (Config.Headless) chromeOptions.AddArgument("--headless");
                    chromeOptions.AddArgument("--start-maximized");
                    return new ChromeDriver(chromeOptions);
            }
        }

        private void TakeScreenshot(string testName)
        {
            try
            {
                var screenshot = ((ITakesScreenshot)Driver).GetScreenshot();
                string dir = Config.ScreenshotPath;
                System.IO.Directory.CreateDirectory(dir);
                string path = System.IO.Path.Combine(dir,
                    $"FAIL_{testName}_{DateTime.Now:yyyyMMdd_HHmmss}.png");
                screenshot.SaveAsFile(path);
                TestContext.AddTestAttachment(path);
            }
            catch { /* Swallow — screenshot failure should not fail the test */ }
        }
    }
}
```

---

## 4. Code Example: Python — YAML and JSON Configuration

### Step 1: Install Packages

```bash
pip install pyyaml python-dotenv
```

### Step 2: Create config.yaml

```yaml
# File: config/config.yaml
test_settings:
  base_url: "https://www.saucedemo.com"
  browser: "chrome"
  headless: false
  timeout_seconds: 10
  screenshot_path: "./screenshots"

credentials:
  standard_user: "standard_user"
  password: "secret_sauce"
```

### Step 3: Create Environment-Specific Configs

```yaml
# File: config/config.staging.yaml
test_settings:
  base_url: "https://staging.saucedemo.com"
  headless: true
  timeout_seconds: 20
```

```yaml
# File: config/config.ci.yaml
test_settings:
  base_url: "https://staging.saucedemo.com"
  browser: "chrome"
  headless: true
  timeout_seconds: 30
```

### Step 4: Create the Configuration Reader

```python
# File: config/config_reader.py
import os
import yaml
import json
from pathlib import Path


class TestConfiguration:
    """
    Reads test configuration from YAML files with environment-based overrides.

    Priority (highest wins):
    1. Environment variables (TEST_BASE_URL, TEST_BROWSER, etc.)
    2. Environment-specific config file (config.staging.yaml)
    3. Base config file (config.yaml)
    """

    _instance = None

    def __new__(cls):
        """Singleton — only one instance is ever created."""
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._load_config()
        return cls._instance

    def _load_config(self):
        """Load base config, merge environment overrides, apply env vars."""
        config_dir = Path(__file__).parent

        # Load base config
        base_path = config_dir / "config.yaml"
        with open(base_path, "r") as f:
            config = yaml.safe_load(f)

        # Load environment-specific overrides
        env = os.getenv("TEST_ENVIRONMENT", "dev")
        env_path = config_dir / f"config.{env}.yaml"
        if env_path.exists():
            with open(env_path, "r") as f:
                env_config = yaml.safe_load(f)
            config = self._deep_merge(config, env_config)

        # Store the merged config
        settings = config.get("test_settings", {})
        credentials = config.get("credentials", {})

        # Apply environment variables (highest priority)
        self.base_url = os.getenv("TEST_BASE_URL", settings.get("base_url", ""))
        self.browser = os.getenv("TEST_BROWSER", settings.get("browser", "chrome"))
        self.headless = os.getenv("TEST_HEADLESS", str(settings.get("headless", False))).lower() == "true"
        self.timeout_seconds = int(os.getenv("TEST_TIMEOUT", settings.get("timeout_seconds", 10)))
        self.screenshot_path = settings.get("screenshot_path", "./screenshots")
        self.standard_user = os.getenv("TEST_USER", credentials.get("standard_user", ""))
        self.password = os.getenv("TEST_PASSWORD", credentials.get("password", ""))

    @staticmethod
    def _deep_merge(base: dict, override: dict) -> dict:
        """Recursively merge override into base."""
        result = base.copy()
        for key, value in override.items():
            if key in result and isinstance(result[key], dict) and isinstance(value, dict):
                result[key] = TestConfiguration._deep_merge(result[key], value)
            else:
                result[key] = value
        return result
```

### Step 5: Use Configuration in conftest.py

```python
# File: conftest.py (updated to use config)
import pytest
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from config.config_reader import TestConfiguration
import os
from datetime import datetime


@pytest.fixture(scope="session")
def config():
    """Provides the test configuration to all tests."""
    return TestConfiguration()


@pytest.fixture(scope="function")
def driver(config):
    """Creates a WebDriver based on configuration."""
    browser = config.browser.lower()

    if browser == "firefox":
        options = webdriver.FirefoxOptions()
        if config.headless:
            options.add_argument("--headless")
        browser_driver = webdriver.Firefox(options=options)

    elif browser == "edge":
        options = webdriver.EdgeOptions()
        if config.headless:
            options.add_argument("--headless")
        browser_driver = webdriver.Edge(options=options)

    else:  # Default: chrome
        options = webdriver.ChromeOptions()
        if config.headless:
            options.add_argument("--headless")
        options.add_argument("--start-maximized")
        browser_driver = webdriver.Chrome(options=options)

    # Navigate to the base URL
    browser_driver.get(config.base_url)

    yield browser_driver

    browser_driver.quit()


@pytest.fixture(scope="function")
def wait(driver, config):
    """Provides a WebDriverWait configured from settings."""
    return WebDriverWait(driver, config.timeout_seconds)
```

### Step 6: Write Tests Using Config

```python
# File: tests/test_config_driven.py
import pytest
from pages.login_page import LoginPage


class TestConfigDriven:
    """Tests that read credentials and URLs from configuration."""

    def test_login_with_configured_credentials(self, driver, config):
        """Uses credentials from config.yaml instead of hardcoded values."""
        login_page = LoginPage(driver)
        inventory_page = login_page.login(
            config.standard_user,
            config.password
        )
        assert inventory_page.get_page_title() == "Products"

    def test_base_url_matches_config(self, driver, config):
        """Verifies the driver navigated to the configured base URL."""
        assert config.base_url in driver.current_url
```

---

## 5. Storing Sensitive Data

### Option 1: Environment Variables

```bash
# Set before running tests
export TEST_BASE_URL="https://staging.example.com"
export TEST_USER="admin"
export TEST_PASSWORD="s3cret"

# Then run tests
pytest -v
```

### Option 2: .env File (Python)

```bash
# File: .env (NEVER commit this file)
TEST_BASE_URL=https://staging.example.com
TEST_USER=admin
TEST_PASSWORD=s3cret
```

```python
# Load .env in conftest.py
from dotenv import load_dotenv
load_dotenv()  # Reads .env and sets environment variables
```

### Option 3: User Secrets (C#)

```bash
dotnet user-secrets init
dotnet user-secrets set "Credentials:Password" "s3cret"
```

---

## 6. .gitignore for Config Security

```gitignore
# File: .gitignore (add these lines)

# Never commit files with real credentials
.env
appsettings.local.json
config/config.local.yaml

# Keep example files (with placeholder values) in version control
# appsettings.example.json
# config/config.example.yaml
```

Create an example file with placeholder values that IS committed:

```yaml
# File: config/config.example.yaml (committed to git)
test_settings:
  base_url: "https://www.saucedemo.com"
  browser: "chrome"
  headless: false
  timeout_seconds: 10

credentials:
  standard_user: "YOUR_USERNAME_HERE"
  password: "YOUR_PASSWORD_HERE"
```

---

## 7. Step-by-Step Walkthrough

### C# Configuration Loading Order

1. `TestConfiguration` constructor runs
2. Reads `TEST_ENVIRONMENT` env var (e.g., "staging"), defaults to "dev"
3. Loads `appsettings.json` (base settings)
4. Loads `appsettings.staging.json` (overrides matching keys)
5. Reads environment variables with `TEST_` prefix (overrides everything)
6. Properties like `BaseUrl`, `Browser`, `Headless` are set

### Python Configuration Loading Order

1. `TestConfiguration()` singleton is created
2. Reads `TEST_ENVIRONMENT` env var, defaults to "dev"
3. Loads `config/config.yaml` (base settings)
4. Loads `config/config.staging.yaml` and deep-merges (if it exists)
5. Checks environment variables `TEST_BASE_URL`, `TEST_BROWSER`, etc. (highest priority)
6. Attributes like `base_url`, `browser`, `headless` are set

---

## 8. Common Mistakes and How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Committing `.env` or `appsettings.local.json` | Passwords in version control | Add to `.gitignore` before the first commit |
| Hardcoding `Directory.GetCurrentDirectory()` | Path differs between IDE and CLI | Use the config builder's `SetBasePath` correctly |
| Not setting `CopyToOutputDirectory` (C#) | JSON file not found at runtime | Set to `PreserveNewest` in `.csproj` |
| Forgetting to install `PyYAML` | `ModuleNotFoundError: No module named 'yaml'` | `pip install pyyaml` |
| Using `config.json` for secrets in Python | JSON does not support comments — harder to maintain | Use YAML for config, `.env` for secrets |
| Not providing defaults | Tests crash if a config key is missing | Always provide fallback values |

---

## 9. Best Practices

1. **Layer your configuration** — Base file for defaults, environment files for overrides, env vars for secrets
2. **Never commit secrets** — Use `.env`, user secrets, or CI/CD secret stores
3. **Provide example files** — Ship `config.example.yaml` so new team members know what to fill in
4. **Use strongly typed config** — Parse values into proper types (`int`, `bool`) in the config reader, not in tests
5. **Singleton pattern** — Load config once, reuse everywhere
6. **Document all config keys** — Add comments in the example file explaining each setting

---

## 10. Hands-On Exercise

**Task:** Make your test framework fully configurable.

1. Create `appsettings.json` (C#) or `config.yaml` (Python) with base URL, browser, headless, timeout, and credentials
2. Create a staging override file that changes the URL and enables headless mode
3. Create a configuration reader class (singleton)
4. Update `BaseTest` / `conftest.py` to use the config
5. Run tests with `TEST_ENVIRONMENT=staging` and verify the staging URL is used

**Expected Output (normal run):**

```
[Config] Environment: dev
[Config] Base URL: https://www.saucedemo.com
[Config] Browser: chrome
[Config] Headless: False
PASSED test_login_with_configured_credentials
```

**Expected Output (staging run):**

```bash
TEST_ENVIRONMENT=staging pytest -v
# [Config] Environment: staging
# [Config] Base URL: https://staging.saucedemo.com
# [Config] Headless: True
```

---

## 11. Real-World Scenario

A QA team runs their Selenium tests in three environments:

- **Dev laptops** — Chrome, visible browser, `localhost:3000`, short timeouts
- **CI pipeline (GitHub Actions)** — Chrome headless, staging URL, longer timeouts
- **Nightly regression** — Chrome headless, production URL, screenshots on failure

With externalized configuration, the same test code runs everywhere. The CI pipeline sets `TEST_ENVIRONMENT=ci` and `TEST_HEADLESS=true` as environment variables, and the config reader picks up the right values automatically. No code changes, no special branches, no conditionals in tests.

---

## 12. Resources

- [Microsoft.Extensions.Configuration](https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration)
- [PyYAML Documentation](https://pyyaml.org/wiki/PyYAMLDocumentation)
- [python-dotenv](https://pypi.org/project/python-dotenv/)
- [The Twelve-Factor App — Config](https://12factor.net/config)
- [.NET User Secrets](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets)
