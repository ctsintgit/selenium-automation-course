# Day 28: CI/CD Integration — GitHub Actions Pipeline for Selenium

## Learning Objectives

By the end of this lesson, you will be able to:

- Understand why CI/CD is essential for test automation
- Create a GitHub Actions workflow that runs Selenium tests automatically
- Install Chrome and ChromeDriver in a CI environment
- Run tests in headless mode on CI servers
- Publish test results and screenshots as build artifacts
- Use matrix strategy to test across multiple browsers and OS versions
- Debug CI failures using logs and artifacts

---

## 1. Core Concept Explanation

### What Is CI/CD?

**CI (Continuous Integration):** Automatically build and test code every time someone pushes a change. If tests fail, the team knows immediately.

**CD (Continuous Delivery/Deployment):** Automatically deploy code that passes all tests to staging or production.

For test automation, CI is the critical part. Your Selenium tests run automatically on every pull request, blocking merges if tests fail.

### Why CI/CD for Selenium Tests?

| Without CI/CD | With CI/CD |
|--------------|-----------|
| "Did anyone run the tests?" | Tests run automatically on every push |
| Tests run on one developer's machine | Tests run on a clean, consistent environment |
| Failures discovered days later | Failures discovered in minutes |
| "It works on my machine" | Works on the CI machine = works everywhere |
| Manual test reports | Automated reports and artifacts |

### GitHub Actions Basics

GitHub Actions is a CI/CD platform built into GitHub. Key concepts:

| Concept | Description |
|---------|-------------|
| **Workflow** | A YAML file in `.github/workflows/` that defines automation |
| **Trigger** | What starts the workflow: push, pull_request, schedule, manual |
| **Job** | A set of steps that run on a single machine |
| **Step** | A single command or action within a job |
| **Runner** | The machine that executes the job (ubuntu-latest, windows-latest, macos-latest) |
| **Artifact** | Files saved from a job (test reports, screenshots) |
| **Matrix** | Run the same job with different configurations (browsers, OS) |

---

## 2. How It Works (Technical Breakdown)

### Workflow Execution Flow

```
Developer pushes code to GitHub
        |
        v
GitHub detects .github/workflows/*.yml
        |
        v
Workflow triggers (on: push, pull_request)
        |
        v
GitHub provisions a runner (fresh Ubuntu/Windows VM)
        |
        v
Steps execute sequentially:
  1. Checkout code
  2. Install language runtime (.NET / Python)
  3. Install Chrome browser
  4. Install dependencies (NuGet / pip)
  5. Run tests (headless)
  6. Upload artifacts (reports, screenshots)
        |
        v
Results appear on PR / commit status check
```

### Runner Environment

GitHub-hosted runners come pre-installed with many tools. On `ubuntu-latest`:
- Google Chrome (latest stable)
- ChromeDriver (matching version)
- .NET SDK 6.0 and 8.0
- Python 3.x
- Firefox
- Edge

This means you often do not need to install Chrome manually on Ubuntu runners.

---

## 3. Code Example: C# GitHub Actions Workflow

```yaml
# File: .github/workflows/selenium-tests-csharp.yml
# This workflow runs C# Selenium tests on every push and PR

name: C# Selenium Tests

# When to run
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  # Manual trigger from GitHub UI
  workflow_dispatch:

jobs:
  selenium-tests:
    runs-on: ubuntu-latest
    # Timeout prevents hung tests from running forever
    timeout-minutes: 30

    steps:
      # Step 1: Check out the repository code
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Set up .NET SDK
      - name: Set up .NET 8
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      # Step 3: Verify Chrome is available
      # (Ubuntu runners have Chrome pre-installed)
      - name: Verify Chrome installation
        run: |
          google-chrome --version
          which google-chrome

      # Step 4: Restore NuGet packages
      - name: Restore dependencies
        run: dotnet restore
        working-directory: ./SeleniumTests

      # Step 5: Build the project
      - name: Build
        run: dotnet build --no-restore
        working-directory: ./SeleniumTests

      # Step 6: Run tests
      - name: Run Selenium tests
        run: |
          dotnet test \
            --no-build \
            --verbosity normal \
            --logger "trx;LogFileName=test-results.trx" \
            --results-directory ./TestResults
        working-directory: ./SeleniumTests
        env:
          # Tell tests to run headless
          SELENIUM_HEADLESS: "true"

      # Step 7: Upload test results as artifact
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()  # Upload even if tests failed
        with:
          name: test-results-csharp
          path: ./SeleniumTests/TestResults/
          retention-days: 30

      # Step 8: Upload screenshots (if any were captured)
      - name: Upload screenshots
        uses: actions/upload-artifact@v4
        if: failure()  # Only upload screenshots when tests fail
        with:
          name: failure-screenshots-csharp
          path: ./SeleniumTests/TestResults/Screenshots/
          retention-days: 14

      # Step 9: Publish test report
      - name: Publish test report
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: C# Test Results
          path: './SeleniumTests/TestResults/test-results.trx'
          reporter: dotnet-trx
```

### Complete C# Test Project for CI

```csharp
// File: SeleniumTests/Tests/CIReadyTests.cs
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;
using System.IO;

namespace SeleniumTests.Tests
{
    [TestFixture]
    public class CIReadyTests
    {
        private IWebDriver driver;
        private WebDriverWait wait;

        [SetUp]
        public void SetUp()
        {
            var options = new ChromeOptions();

            // ALWAYS run headless in CI
            // Check env var so local dev can run headed
            string headless = Environment.GetEnvironmentVariable(
                "SELENIUM_HEADLESS"
            );
            if (headless?.ToLower() == "true" || headless == "1")
            {
                options.AddArgument("--headless=new");
            }

            // CI-essential arguments
            options.AddArgument("--window-size=1920,1080");
            options.AddArgument("--disable-gpu");
            options.AddArgument("--no-sandbox");
            options.AddArgument("--disable-dev-shm-usage");

            driver = new ChromeDriver(options);
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(15));
        }

        [TearDown]
        public void TearDown()
        {
            // Screenshot on failure
            if (TestContext.CurrentContext.Result.Outcome.Status
                == NUnit.Framework.Interfaces.TestStatus.Failed)
            {
                try
                {
                    string dir = Path.Combine(
                        TestContext.CurrentContext.TestDirectory,
                        "TestResults", "Screenshots"
                    );
                    Directory.CreateDirectory(dir);

                    string file = Path.Combine(
                        dir,
                        $"{DateTime.Now:yyyy-MM-dd_HH-mm-ss}_" +
                        $"{TestContext.CurrentContext.Test.Name}_FAILED.png"
                    );

                    ((ITakesScreenshot)driver).GetScreenshot()
                        .SaveAsFile(file);

                    TestContext.AddTestAttachment(file);
                    Console.WriteLine($"Screenshot: {file}");
                }
                catch (Exception ex)
                {
                    Console.WriteLine(
                        $"Screenshot failed: {ex.Message}"
                    );
                }
            }

            driver?.Quit();
        }

        [Test]
        public void HomepageLoads()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            Assert.That(driver.Title, Does.Contain("Internet"));
        }

        [Test]
        public void LoginWithValidCredentials()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            driver.FindElement(By.Id("username")).SendKeys("tomsmith");
            driver.FindElement(By.Id("password"))
                .SendKeys("SuperSecretPassword!");
            driver.FindElement(
                By.CssSelector("button[type='submit']")
            ).Click();

            var flash = wait.Until(
                d => d.FindElement(By.Id("flash"))
            );

            Assert.That(flash.Text, Does.Contain("You logged into"));
        }

        [Test]
        public void CheckboxToggle()
        {
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/checkboxes"
            );

            var checkboxes = driver.FindElements(
                By.CssSelector("input[type='checkbox']")
            );

            Assert.That(checkboxes.Count, Is.GreaterThan(0));

            // Toggle first checkbox
            checkboxes[0].Click();
            Assert.That(checkboxes[0].Selected, Is.True);
        }
    }
}
```

### C# Project File (.csproj)

```xml
<!-- File: SeleniumTests/SeleniumTests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.9.0" />
    <PackageReference Include="NUnit" Version="4.1.0" />
    <PackageReference Include="NUnit3TestAdapter" Version="4.5.0" />
    <PackageReference Include="Selenium.WebDriver" Version="4.18.1" />
    <PackageReference Include="Selenium.Support" Version="4.18.1" />
  </ItemGroup>
</Project>
```

---

## 4. Code Example: Python GitHub Actions Workflow

```yaml
# File: .github/workflows/selenium-tests-python.yml
# This workflow runs Python Selenium tests on every push and PR

name: Python Selenium Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  selenium-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      # Step 1: Checkout
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Set up Python
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'  # Cache pip packages for faster builds

      # Step 3: Verify Chrome
      - name: Verify Chrome installation
        run: google-chrome --version

      # Step 4: Install Python dependencies
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      # Step 5: Run tests
      - name: Run Selenium tests
        run: |
          pytest tests/ \
            -v \
            --tb=short \
            --junitxml=TestResults/test-results.xml \
            --html=TestResults/report.html \
            --self-contained-html \
            -n auto
        env:
          SELENIUM_HEADLESS: "true"

      # Step 6: Upload test results
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-python
          path: TestResults/
          retention-days: 30

      # Step 7: Upload screenshots on failure
      - name: Upload failure screenshots
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: failure-screenshots-python
          path: TestResults/Screenshots/
          retention-days: 14

      # Step 8: Publish test report
      - name: Publish test report
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Python Test Results
          path: TestResults/test-results.xml
          reporter: java-junit
```

### requirements.txt

```
# File: requirements.txt
selenium==4.18.1
pytest==8.1.1
pytest-html==4.1.1
pytest-xdist==3.5.0
```

### Complete Python Test for CI

```python
# File: tests/test_ci_ready.py
import os
import pytest
from datetime import datetime
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


SCREENSHOT_DIR = os.path.join(
    os.path.dirname(os.path.dirname(__file__)),
    "TestResults", "Screenshots"
)


@pytest.fixture
def driver():
    """Create a Chrome driver, headless if SELENIUM_HEADLESS is set."""
    options = Options()
    options.add_argument("--window-size=1920,1080")

    # Check environment variable for headless mode
    is_headless = os.environ.get(
        "SELENIUM_HEADLESS", ""
    ).lower() in ("true", "1", "yes")

    if is_headless:
        options.add_argument("--headless=new")

    # CI-essential arguments
    options.add_argument("--disable-gpu")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")

    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


@pytest.fixture
def wait(driver):
    return WebDriverWait(driver, 15)


# Auto-screenshot on failure (conftest.py hook)
@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()

    if report.when == "call" and report.failed:
        driver = item.funcargs.get("driver")
        if driver:
            os.makedirs(SCREENSHOT_DIR, exist_ok=True)
            timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
            filepath = os.path.join(
                SCREENSHOT_DIR,
                f"{timestamp}_{item.name}_FAILED.png"
            )
            try:
                driver.save_screenshot(filepath)
                print(f"\nScreenshot: {filepath}")
            except Exception:
                pass


# =============================================
# TESTS
# =============================================

class TestHomepage:

    def test_homepage_loads(self, driver):
        driver.get("https://the-internet.herokuapp.com/")
        assert "Internet" in driver.title

    def test_homepage_has_links(self, driver, wait):
        driver.get("https://the-internet.herokuapp.com/")
        links = driver.find_elements(By.CSS_SELECTOR, "ul li a")
        assert len(links) > 10


class TestLogin:

    def test_valid_login(self, driver, wait):
        driver.get("https://the-internet.herokuapp.com/login")

        driver.find_element(By.ID, "username").send_keys("tomsmith")
        driver.find_element(By.ID, "password").send_keys(
            "SuperSecretPassword!"
        )
        driver.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        ).click()

        flash = wait.until(
            EC.visibility_of_element_located((By.ID, "flash"))
        )
        assert "You logged into" in flash.text

    def test_invalid_login(self, driver, wait):
        driver.get("https://the-internet.herokuapp.com/login")

        driver.find_element(By.ID, "username").send_keys("wrong")
        driver.find_element(By.ID, "password").send_keys("wrong")
        driver.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        ).click()

        flash = wait.until(
            EC.visibility_of_element_located((By.ID, "flash"))
        )
        assert "Your username is invalid" in flash.text


class TestCheckboxes:

    def test_toggle_checkbox(self, driver):
        driver.get("https://the-internet.herokuapp.com/checkboxes")

        checkboxes = driver.find_elements(
            By.CSS_SELECTOR, "input[type='checkbox']"
        )
        assert len(checkboxes) > 0

        checkboxes[0].click()
        assert checkboxes[0].is_selected()
```

---

## 5. Matrix Strategy: Multi-Browser, Multi-OS

```yaml
# File: .github/workflows/matrix-tests.yml
name: Cross-Browser Matrix Tests

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    runs-on: ${{ matrix.os }}
    timeout-minutes: 30

    strategy:
      fail-fast: false  # Do not cancel other jobs if one fails
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.10', '3.11', '3.12']
        # This creates 9 job combinations (3 OS x 3 Python versions)

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install Chrome (Ubuntu)
        if: matrix.os == 'ubuntu-latest'
        run: google-chrome --version

      - name: Install dependencies
        run: pip install selenium pytest

      - name: Run tests
        run: pytest tests/ -v --tb=short
        env:
          SELENIUM_HEADLESS: "true"

      - name: Upload results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: results-${{ matrix.os }}-py${{ matrix.python-version }}
          path: TestResults/
```

---

## 6. Step-by-Step Walkthrough

### Creating Your First Workflow

1. **Create the directory:** `.github/workflows/` in your repository root
2. **Create the YAML file:** `selenium-tests.yml`
3. **Define the trigger:** `on: [push, pull_request]`
4. **Define the job:** `runs-on: ubuntu-latest`
5. **Add steps:** checkout, setup runtime, install deps, run tests, upload artifacts
6. **Commit and push.** GitHub automatically detects the workflow and runs it.

### Viewing Results

1. Go to your repository on GitHub
2. Click the **Actions** tab
3. Click on the workflow run
4. See pass/fail status for each job
5. Download artifacts (screenshots, reports) from the job summary

### Debugging Failures

1. Click on the failed step to expand its logs
2. Look for assertion errors, element-not-found errors, timeouts
3. Download the screenshot artifact to see what the browser was showing
4. Check if the test passes locally with `SELENIUM_HEADLESS=true`

---

## 7. Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Not running headless in CI | "no display" error on Linux runners | Add `--headless=new` via env var |
| Missing `--no-sandbox` | Chrome crashes on Linux CI | Always add `--no-sandbox` in CI |
| Missing `--disable-dev-shm-usage` | Chrome out-of-memory crashes | Always add this flag in CI |
| Hardcoded file paths | Paths differ between local and CI | Use relative paths or env vars |
| Not using `if: always()` on artifacts | Artifacts only upload on success | Use `if: always()` to upload even on failure |
| Tests too slow for CI timeout | Workflow times out | Set reasonable timeout, use parallel execution |
| Not caching dependencies | Slow builds | Use `cache: 'pip'` or `cache: 'nuget'` |

---

## 8. Best Practices

1. **Run tests on every pull request.** This catches issues before they reach the main branch.

2. **Use `if: always()` for test result uploads.** You need artifacts most when tests fail.

3. **Set a workflow timeout.** Prevent hung tests from running (and billing) for hours.

4. **Use `fail-fast: false` in matrix builds.** One browser failing should not cancel the other browser tests.

5. **Cache dependencies.** Save 30-60 seconds per build by caching pip/NuGet packages.

6. **Pin action versions.** Use `actions/checkout@v4`, not `actions/checkout@main`.

7. **Use environment variables for configuration.** `SELENIUM_HEADLESS=true` should switch to headless mode.

8. **Store secrets properly.** Use GitHub Secrets for URLs, credentials, API keys — never hardcode them.

---

## 8. Hands-On Exercise

### Task: Create a Working GitHub Actions Workflow

**Objective:** Create a workflow that runs your Selenium tests automatically on push.

**Steps:**
1. Create `.github/workflows/selenium-tests.yml` in your project
2. Configure it to run on push to main and on pull requests
3. Set up Python 3.11 (or .NET 8)
4. Install Chrome and dependencies
5. Run tests in headless mode
6. Upload test results and screenshots as artifacts
7. Push to GitHub and verify the workflow runs

**Verification:**
- Go to Actions tab on GitHub
- Click on the workflow run
- All tests should pass (green checkmark)
- Download the test-results artifact

---

## 9. Real-World Scenario

### Scenario: Complete CI Pipeline with Notifications

```yaml
# .github/workflows/full-pipeline.yml
name: Full Test Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    # Run nightly at 2 AM UTC
    - cron: '0 2 * * *'

jobs:
  # Job 1: Unit Tests (fast, no browser)
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install pytest
      - run: pytest tests/unit/ -v

  # Job 2: Selenium Tests (depends on unit tests passing)
  selenium-tests:
    needs: unit-tests  # Only run if unit tests pass
    runs-on: ubuntu-latest
    timeout-minutes: 30

    strategy:
      fail-fast: false
      matrix:
        browser: [chrome, firefox]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run Selenium tests
        run: |
          pytest tests/e2e/ \
            -v --tb=short \
            -n 4 \
            --junitxml=TestResults/results-${{ matrix.browser }}.xml
        env:
          SELENIUM_HEADLESS: "true"
          BROWSER: ${{ matrix.browser }}

      - name: Upload results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: results-${{ matrix.browser }}
          path: TestResults/

  # Job 3: Notify on failure
  notify:
    needs: [unit-tests, selenium-tests]
    runs-on: ubuntu-latest
    if: failure()
    steps:
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Selenium tests FAILED on ${{ github.ref }}. Check: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 10. Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [actions/setup-python](https://github.com/actions/setup-python)
- [actions/setup-dotnet](https://github.com/actions/setup-dotnet)
- [actions/upload-artifact](https://github.com/actions/upload-artifact)
- [dorny/test-reporter](https://github.com/dorny/test-reporter)
- [GitHub Actions Runners - Pre-installed Software](https://github.com/actions/runner-images)
