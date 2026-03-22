# Day 15: NUnit (C#) and PyTest (Python) Setup and Structure

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain why a test framework is essential for Selenium automation
- Set up NUnit in a .NET 6/8 project with the correct NuGet packages
- Set up pytest in a Python project
- Write your first structured Selenium test using NUnit and pytest
- Run tests from the command line and your IDE
- Understand test discovery and naming conventions in both frameworks

---

## 1. Why Use a Test Framework with Selenium?

Selenium WebDriver is a browser automation library — it opens browsers, clicks buttons, and reads text. But it does not know what "pass" or "fail" means. A test framework provides:

- **Assertions**: Verify expected vs actual results (`Assert.AreEqual`, `assert`)
- **Test Discovery**: Automatically find and run all tests in your project
- **Reporting**: Summary of passed, failed, and skipped tests
- **Lifecycle Hooks**: Setup and teardown logic that runs before/after tests
- **Parallel Execution**: Run tests concurrently for faster feedback
- **Integration**: Works with CI/CD pipelines (GitHub Actions, Azure DevOps, Jenkins)

Without a test framework, you would write `if/else` blocks and `Console.WriteLine` statements to check results — fragile, verbose, and impossible to scale.

---

## 2. How It Works

### NUnit (C#)

NUnit is the most popular unit testing framework for .NET. It uses attributes (annotations) to mark classes and methods as tests.

**Key Components:**
- `[TestFixture]` — marks a class as containing tests
- `[Test]` — marks a method as a test case
- `Assert` — static class with comparison methods (`AreEqual`, `IsTrue`, `Contains`, etc.)
- **NUnit3TestAdapter** — allows Visual Studio and `dotnet test` to discover and run NUnit tests

### pytest (Python)

pytest is the de facto standard for Python testing. It uses naming conventions instead of decorators to discover tests.

**Key Components:**
- Files must start with `test_` or end with `_test.py`
- Functions must start with `test_`
- Use Python's built-in `assert` keyword — pytest enhances its output automatically
- No base class or decorator required for basic tests

---

## 3. NUnit Setup (C#)

### Step 1: Create the Project

```bash
dotnet new nunit -n SeleniumTests
cd SeleniumTests
```

This creates a project with NUnit already configured. If you prefer to add NUnit to an existing project:

```bash
dotnet add package NUnit
dotnet add package NUnit3TestAdapter
dotnet add package Microsoft.NET.Test.Sdk
```

### Step 2: Add Selenium

```bash
dotnet add package Selenium.WebDriver
```

ChromeDriver is managed automatically by Selenium 4.6+ via Selenium Manager — no separate NuGet package needed.

### Step 3: Verify the .csproj

Your project file should contain these package references:

```xml
<ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
    <PackageReference Include="NUnit" Version="4.*" />
    <PackageReference Include="NUnit3TestAdapter" Version="4.*" />
    <PackageReference Include="Selenium.WebDriver" Version="4.*" />
</ItemGroup>
```

---

## 4. pytest Setup (Python)

### Step 1: Create the Project

```bash
mkdir selenium_tests
cd selenium_tests
python -m venv venv
source venv/bin/activate   # Linux/Mac
venv\Scripts\activate      # Windows
```

### Step 2: Install Packages

```bash
pip install selenium pytest
```

### Step 3: Verify Installation

```bash
pytest --version
python -c "import selenium; print(selenium.__version__)"
```

---

## 5. Code Example: C# (NUnit)

```csharp
// File: GoogleSearchTests.cs
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;

namespace SeleniumTests
{
    [TestFixture]
    public class GoogleSearchTests
    {
        // Declare the driver at the class level so all tests can use it
        private IWebDriver driver;

        [SetUp]
        public void SetUp()
        {
            // Selenium 4.6+ automatically manages ChromeDriver via Selenium Manager
            var options = new ChromeOptions();
            // options.AddArgument("--headless"); // Uncomment to run without visible browser
            driver = new ChromeDriver(options);

            // Set an implicit wait as a safety net (explicit waits are preferred)
            driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(5);
        }

        [Test]
        public void GoogleTitle_ShouldContainGoogle()
        {
            // Arrange: navigate to Google
            driver.Navigate().GoToUrl("https://www.google.com");

            // Act: get the page title
            string title = driver.Title;

            // Assert: verify the title contains "Google"
            Assert.That(title, Does.Contain("Google"),
                "Expected page title to contain 'Google'");
        }

        [Test]
        public void GoogleSearch_ShouldReturnResults()
        {
            // Arrange
            driver.Navigate().GoToUrl("https://www.google.com");

            // Act: find the search box, type a query, submit
            var searchBox = driver.FindElement(By.Name("q"));
            searchBox.SendKeys("Selenium WebDriver");
            searchBox.SendKeys(Keys.Enter);

            // Wait for results to load
            var wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
            wait.Until(d => d.FindElement(By.Id("search")));

            // Assert: results container exists
            var results = driver.FindElement(By.Id("search"));
            Assert.That(results.Displayed, Is.True,
                "Expected search results to be displayed");
        }

        [TearDown]
        public void TearDown()
        {
            // Always quit the driver to close the browser and free resources
            if (driver != null)
            {
                driver.Quit();
                driver = null;
            }
        }
    }
}
```

### Running from Command Line

```bash
dotnet test
dotnet test --filter "GoogleTitle_ShouldContainGoogle"
dotnet test --verbosity normal
```

### Running from Visual Studio

1. Open Test Explorer (Test > Test Explorer)
2. Click "Run All" or right-click a specific test

---

## 6. Code Example: Python (pytest)

```python
# File: test_google_search.py
import pytest
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class TestGoogleSearch:
    """Test suite for Google search functionality."""

    def setup_method(self):
        """Runs before each test method — creates a fresh browser."""
        options = webdriver.ChromeOptions()
        # options.add_argument("--headless")  # Uncomment for headless mode
        # Selenium 4.6+ automatically manages ChromeDriver
        self.driver = webdriver.Chrome(options=options)

    def test_google_title_contains_google(self):
        """Verify the Google homepage title contains 'Google'."""
        # Arrange: navigate to Google
        self.driver.get("https://www.google.com")

        # Act: get the title
        title = self.driver.title

        # Assert: check the title
        assert "Google" in title, f"Expected 'Google' in title, got '{title}'"

    def test_google_search_returns_results(self):
        """Verify that searching on Google returns results."""
        # Arrange
        self.driver.get("https://www.google.com")

        # Act: type a query and press Enter
        search_box = self.driver.find_element(By.NAME, "q")
        search_box.send_keys("Selenium WebDriver")
        search_box.send_keys(Keys.ENTER)

        # Wait for results
        wait = WebDriverWait(self.driver, 10)
        results = wait.until(
            EC.presence_of_element_located((By.ID, "search"))
        )

        # Assert: results are displayed
        assert results.is_displayed(), "Expected search results to be visible"

    def teardown_method(self):
        """Runs after each test method — closes the browser."""
        if self.driver:
            self.driver.quit()
```

### Running from Command Line

```bash
pytest test_google_search.py -v
pytest test_google_search.py::TestGoogleSearch::test_google_title_contains_google
pytest -v --tb=short
```

### Running from VS Code / PyCharm

- **VS Code**: Install the Python extension, open the Testing sidebar, click play
- **PyCharm**: Right-click the test file or function, select "Run"

---

## 7. Step-by-Step Walkthrough

### NUnit Flow

1. `dotnet test` scans for assemblies containing `[TestFixture]` classes
2. For each `[Test]` method, NUnit calls `[SetUp]` first
3. The test method runs — assertions either pass or throw an exception
4. `[TearDown]` runs regardless of pass/fail
5. Results are collected and displayed in the console or Test Explorer

### pytest Flow

1. `pytest` discovers files matching `test_*.py` or `*_test.py`
2. Inside those files, it finds functions/methods starting with `test_`
3. `setup_method` runs before each test (if defined)
4. The test function runs — `assert` statements either pass or raise `AssertionError`
5. `teardown_method` runs regardless of pass/fail
6. Results are printed to the terminal

---

## 8. Common Mistakes and How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `[Test]` attribute (C#) | Method is never discovered or run | Always annotate test methods with `[Test]` |
| Test method not starting with `test_` (Python) | pytest silently skips it | Follow the `test_` prefix convention |
| Not installing NUnit3TestAdapter | `dotnet test` finds zero tests | Add the adapter NuGet package |
| Using `Thread.Sleep` instead of explicit waits | Slow, flaky tests | Use `WebDriverWait` with expected conditions |
| Not calling `driver.Quit()` in teardown | Browser processes pile up, memory leaks | Always quit in teardown, even on failure |
| Hardcoding absolute paths to ChromeDriver | Breaks on other machines | Let Selenium 4.6+ manage drivers automatically |
| Making tests depend on each other | One failure cascades to all | Each test should be independent and self-contained |

---

## 9. Best Practices

1. **One assertion per test** (when practical) — makes failures easy to diagnose
2. **Descriptive test names** — `GoogleSearch_ShouldReturnResults` tells you what and why
3. **Arrange-Act-Assert pattern** — structure every test in three clear sections
4. **Independent tests** — no test should rely on another test running first
5. **Clean state** — create a fresh driver in setup, quit in teardown
6. **Use explicit waits** — `WebDriverWait` with conditions, never `Thread.Sleep`
7. **Run tests in CI** — integrate with GitHub Actions or Azure DevOps early

---

## 10. Hands-On Exercise

**Task:** Convert a simple automation script into a proper test suite.

Create a test class with three tests for the Sauce Demo site (`https://www.saucedemo.com`):

1. `test_valid_login` — Login with `standard_user` / `secret_sauce`, assert the URL contains `inventory`
2. `test_invalid_login` — Login with `locked_out_user` / `secret_sauce`, assert an error message appears
3. `test_empty_credentials` — Click login without entering anything, assert an error message appears

**Expected Output (pytest):**

```
test_saucedemo.py::TestSauceDemo::test_valid_login PASSED
test_saucedemo.py::TestSauceDemo::test_invalid_login PASSED
test_saucedemo.py::TestSauceDemo::test_empty_credentials PASSED
========================= 3 passed in 12.34s =========================
```

**Expected Output (NUnit):**

```
Passed!  - Failed:     0, Passed:     3, Skipped:     0, Total:     3
```

---

## 11. Real-World Scenario

In a real project, you might start writing Selenium scripts as standalone console apps. The moment you have more than two or three scripts, you face problems:

- How do you know which scripts passed or failed?
- How do you run only the login tests?
- How do you generate a report for stakeholders?
- How do you integrate with your CI/CD pipeline?

A test framework solves all of these. At companies, QA teams typically:

1. Create an NUnit or pytest project from day one
2. Organize tests by feature (login, checkout, search)
3. Run the full suite on every pull request via CI
4. Generate reports with pass/fail counts and screenshots of failures

This is the foundation you are building in Week 3.

---

## 12. Resources

- [NUnit Documentation](https://docs.nunit.org/)
- [pytest Documentation](https://docs.pytest.org/en/stable/)
- [Selenium 4 Documentation](https://www.selenium.dev/documentation/)
- [NUnit Assertions Reference](https://docs.nunit.org/articles/nunit/writing-tests/assertions/assertions.html)
- [pytest Assertions](https://docs.pytest.org/en/stable/how-to/assert.html)
- [Sauce Demo Practice Site](https://www.saucedemo.com)
