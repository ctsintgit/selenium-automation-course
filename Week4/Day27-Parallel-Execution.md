# Day 27: Parallel Execution — Multi-Browser, Multi-Thread

## Learning Objectives

By the end of this lesson, you will be able to:

- Understand why parallel execution is critical for large test suites
- Configure NUnit for parallel test execution in C# using `[Parallelizable]`
- Configure pytest for parallel execution using `pytest-xdist`
- Manage thread-safe WebDriver instances so parallel tests do not interfere
- Use `ThreadLocal<IWebDriver>` in C# and function-scoped fixtures in Python
- Run the same test suite across Chrome, Firefox, and Edge simultaneously
- Avoid common concurrency pitfalls like shared state and port conflicts

---

## 1. Core Concept Explanation

### Why Parallel Execution?

A test suite with 200 tests running sequentially at 10 seconds each takes **33 minutes**. Running them in parallel across 4 threads takes **~8 minutes**. On a Selenium Grid with 10 nodes, it drops to **~3 minutes**.

| Approach | 200 Tests @ 10s each |
|----------|---------------------|
| Sequential (1 thread) | 33 minutes |
| 4 threads | ~8.5 minutes |
| 10 threads (Grid) | ~3.5 minutes |
| 20 threads (Grid) | ~2 minutes |

### The Challenge: Thread Safety

Each test needs its own browser instance. If two tests share the same WebDriver, one test might navigate away from the page while another test is trying to click a button on it. The result: random, unpredictable failures called **flaky tests**.

**The rule: every thread gets its own WebDriver instance.**

### Parallel Strategies

| Strategy | Description |
|----------|-------------|
| **Parallel by test method** | Each test method runs in its own thread |
| **Parallel by test class** | Each test class runs in its own thread; tests within a class run sequentially |
| **Parallel by fixture** | Each unique fixture combination runs in its own thread |

For Selenium, **parallel by test class** or **parallel by test method** are both common. Parallel by class is safer because tests within a class may share setup logic.

---

## 2. How It Works (Technical Breakdown)

### C# (NUnit) Parallel Architecture

```
NUnit Test Runner
    |
    +-- Thread 1: TestClass_A.Test1() --> ChromeDriver instance 1
    |
    +-- Thread 2: TestClass_B.Test1() --> ChromeDriver instance 2
    |
    +-- Thread 3: TestClass_A.Test2() --> ChromeDriver instance 3
    |
    +-- Thread 4: TestClass_C.Test1() --> ChromeDriver instance 4
```

NUnit uses `[Parallelizable]` attributes and `[assembly: LevelOfParallelism(N)]` to control parallelism.

### Python (pytest-xdist) Parallel Architecture

```
pytest main process
    |
    +-- Worker gw0: test_a.py::test_1  --> Chrome instance 1
    |
    +-- Worker gw1: test_a.py::test_2  --> Chrome instance 2
    |
    +-- Worker gw2: test_b.py::test_1  --> Chrome instance 3
    |
    +-- Worker gw3: test_b.py::test_2  --> Chrome instance 4
```

pytest-xdist spawns separate worker processes, each with its own fixtures.

---

## 3. Code Example: C# (Complete, Runnable)

```csharp
// File: Day27_ParallelExecution.cs
// Prerequisites:
//   dotnet new nunit -n Day27
//   cd Day27
//   dotnet add package Selenium.WebDriver
//   dotnet add package Selenium.Support
//
// Run: dotnet test -- NUnit.NumberOfTestWorkers=4

using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Firefox;
using OpenQA.Selenium.Edge;
using OpenQA.Selenium.Support.UI;
using System;
using System.Threading;

// Set the maximum number of parallel threads for the entire assembly
[assembly: LevelOfParallelism(4)]

namespace Day27
{
    // =============================================
    // THREAD-SAFE BASE TEST CLASS
    // =============================================
    public class ParallelBaseTest
    {
        // ThreadLocal ensures each thread gets its own driver instance.
        // Without this, parallel tests would share one driver and crash.
        private static readonly ThreadLocal<IWebDriver> ThreadDriver =
            new ThreadLocal<IWebDriver>();

        protected IWebDriver Driver
        {
            get => ThreadDriver.Value;
            set => ThreadDriver.Value = value;
        }

        protected WebDriverWait Wait { get; set; }

        protected IWebDriver CreateChromeDriver()
        {
            var options = new ChromeOptions();
            options.AddArgument("--window-size=1920,1080");
            // Headless recommended for parallel to reduce resource usage
            options.AddArgument("--headless=new");
            options.AddArgument("--disable-gpu");
            return new ChromeDriver(options);
        }

        protected IWebDriver CreateFirefoxDriver()
        {
            var options = new FirefoxOptions();
            options.AddArgument("--headless");
            options.AddArgument("--width=1920");
            options.AddArgument("--height=1080");
            return new FirefoxDriver(options);
        }

        protected IWebDriver CreateEdgeDriver()
        {
            var options = new EdgeOptions();
            options.AddArgument("--headless=new");
            options.AddArgument("--window-size=1920,1080");
            return new EdgeDriver(options);
        }

        [SetUp]
        public virtual void SetUp()
        {
            Driver = CreateChromeDriver();
            Wait = new WebDriverWait(Driver, TimeSpan.FromSeconds(10));

            Console.WriteLine(
                $"[Thread {Thread.CurrentThread.ManagedThreadId}] " +
                $"Started: {TestContext.CurrentContext.Test.Name}"
            );
        }

        [TearDown]
        public virtual void TearDown()
        {
            Console.WriteLine(
                $"[Thread {Thread.CurrentThread.ManagedThreadId}] " +
                $"Finished: {TestContext.CurrentContext.Test.Name}"
            );

            Driver?.Quit();
            Driver = null;
        }
    }

    // =============================================
    // PARALLEL TEST CLASS: ALL METHODS IN PARALLEL
    // =============================================
    [TestFixture]
    [Parallelizable(ParallelScope.All)]
    // ParallelScope.All = all test methods in this class run in parallel
    public class ParallelLoginTests : ParallelBaseTest
    {
        [Test]
        public void ValidLogin()
        {
            Driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            Driver.FindElement(By.Id("username")).SendKeys("tomsmith");
            Driver.FindElement(By.Id("password"))
                .SendKeys("SuperSecretPassword!");
            Driver.FindElement(
                By.CssSelector("button[type='submit']")
            ).Click();

            IWebElement flash = Wait.Until(
                d => d.FindElement(By.Id("flash"))
            );

            Assert.That(flash.Text, Does.Contain("You logged into"));
            Console.WriteLine(
                $"[Thread {Thread.CurrentThread.ManagedThreadId}] " +
                "Valid login passed"
            );
        }

        [Test]
        public void InvalidLogin()
        {
            Driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            Driver.FindElement(By.Id("username")).SendKeys("wronguser");
            Driver.FindElement(By.Id("password")).SendKeys("wrongpass");
            Driver.FindElement(
                By.CssSelector("button[type='submit']")
            ).Click();

            IWebElement flash = Wait.Until(
                d => d.FindElement(By.Id("flash"))
            );

            Assert.That(flash.Text, Does.Contain("Your username is invalid"));
            Console.WriteLine(
                $"[Thread {Thread.CurrentThread.ManagedThreadId}] " +
                "Invalid login passed"
            );
        }

        [Test]
        public void EmptyCredentials()
        {
            Driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            // Click login without entering anything
            Driver.FindElement(
                By.CssSelector("button[type='submit']")
            ).Click();

            IWebElement flash = Wait.Until(
                d => d.FindElement(By.Id("flash"))
            );

            Assert.That(flash.Text, Does.Contain("Your username is invalid"));
            Console.WriteLine(
                $"[Thread {Thread.CurrentThread.ManagedThreadId}] " +
                "Empty credentials passed"
            );
        }
    }

    // =============================================
    // PARALLEL AT FIXTURE LEVEL
    // =============================================
    [TestFixture]
    [Parallelizable(ParallelScope.Fixtures)]
    // ParallelScope.Fixtures = this class runs in parallel with OTHER classes,
    // but tests WITHIN this class run sequentially
    public class ParallelNavigationTests : ParallelBaseTest
    {
        [Test]
        public void NavigateToCheckboxes()
        {
            Driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/checkboxes"
            );

            var checkboxes = Driver.FindElements(By.CssSelector("input[type='checkbox']"));
            Assert.That(checkboxes.Count, Is.GreaterThan(0));

            Console.WriteLine(
                $"[Thread {Thread.CurrentThread.ManagedThreadId}] " +
                $"Found {checkboxes.Count} checkboxes"
            );
        }

        [Test]
        public void NavigateToDropdown()
        {
            Driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/dropdown"
            );

            IWebElement dropdown = Driver.FindElement(By.Id("dropdown"));
            Assert.That(dropdown.Displayed, Is.True);

            Console.WriteLine(
                $"[Thread {Thread.CurrentThread.ManagedThreadId}] " +
                "Dropdown page loaded"
            );
        }
    }

    // =============================================
    // CROSS-BROWSER PARALLEL TESTS
    // =============================================
    [TestFixture]
    [Parallelizable(ParallelScope.All)]
    public class CrossBrowserTests
    {
        // Each test method creates its own driver (different browser)
        // No shared state — fully independent

        [Test]
        public void TestOnChrome()
        {
            var options = new ChromeOptions();
            options.AddArgument("--headless=new");
            options.AddArgument("--window-size=1920,1080");
            var driver = new ChromeDriver(options);

            try
            {
                driver.Navigate().GoToUrl(
                    "https://the-internet.herokuapp.com/"
                );

                Assert.That(driver.Title, Does.Contain("Internet"));
                Console.WriteLine(
                    $"[Chrome - Thread {Thread.CurrentThread.ManagedThreadId}] " +
                    "Passed"
                );
            }
            finally
            {
                driver.Quit();
            }
        }

        [Test]
        public void TestOnFirefox()
        {
            var options = new FirefoxOptions();
            options.AddArgument("--headless");
            var driver = new FirefoxDriver(options);

            try
            {
                driver.Navigate().GoToUrl(
                    "https://the-internet.herokuapp.com/"
                );

                Assert.That(driver.Title, Does.Contain("Internet"));
                Console.WriteLine(
                    $"[Firefox - Thread {Thread.CurrentThread.ManagedThreadId}] " +
                    "Passed"
                );
            }
            finally
            {
                driver.Quit();
            }
        }

        [Test]
        public void TestOnEdge()
        {
            var options = new EdgeOptions();
            options.AddArgument("--headless=new");
            options.AddArgument("--window-size=1920,1080");
            var driver = new EdgeDriver(options);

            try
            {
                driver.Navigate().GoToUrl(
                    "https://the-internet.herokuapp.com/"
                );

                Assert.That(driver.Title, Does.Contain("Internet"));
                Console.WriteLine(
                    $"[Edge - Thread {Thread.CurrentThread.ManagedThreadId}] " +
                    "Passed"
                );
            }
            finally
            {
                driver.Quit();
            }
        }
    }

    // =============================================
    // DATA-DRIVEN PARALLEL TESTS
    // =============================================
    [TestFixture]
    [Parallelizable(ParallelScope.All)]
    public class DataDrivenParallelTests : ParallelBaseTest
    {
        [TestCase("checkboxes", "Checkboxes")]
        [TestCase("dropdown", "Dropdown")]
        [TestCase("login", "Login")]
        [TestCase("inputs", "Inputs")]
        [TestCase("key_presses", "Key Presses")]
        public void NavigateAndVerifyTitle(string path, string expectedHeading)
        {
            Driver.Navigate().GoToUrl(
                $"https://the-internet.herokuapp.com/{path}"
            );

            IWebElement heading = Wait.Until(
                d => d.FindElement(By.TagName("h3"))
            );

            Assert.That(heading.Text, Does.Contain(expectedHeading));

            Console.WriteLine(
                $"[Thread {Thread.CurrentThread.ManagedThreadId}] " +
                $"/{path} -> {heading.Text}"
            );
        }
    }
}
```

---

## 4. Code Example: Python (Complete, Runnable)

```python
# File: test_day27_parallel_execution.py
# Prerequisites:
#   pip install selenium pytest pytest-xdist
#
# Run parallel:
#   pytest test_day27_parallel_execution.py -n 4 -v
#   pytest test_day27_parallel_execution.py -n auto -v  (auto-detect CPU cores)

import os
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options as ChromeOptions
from selenium.webdriver.firefox.options import Options as FirefoxOptions
from selenium.webdriver.edge.options import Options as EdgeOptions
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


# =============================================
# FIXTURES (THREAD-SAFE BY DEFAULT IN PYTEST)
# =============================================
# pytest-xdist runs each worker in a separate PROCESS,
# so function-scoped fixtures are naturally isolated.

@pytest.fixture
def driver():
    """
    Function-scoped fixture = each test gets its own driver.
    pytest-xdist workers are separate processes, so there
    is no shared state. This is thread-safe by default.
    """
    options = ChromeOptions()
    options.add_argument("--headless=new")
    options.add_argument("--window-size=1920,1080")
    options.add_argument("--disable-gpu")

    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


@pytest.fixture
def wait(driver):
    """Explicit wait tied to the driver fixture."""
    return WebDriverWait(driver, 10)


# =============================================
# PARALLEL LOGIN TESTS
# =============================================
# Run with: pytest -n 3 -v
# Each of these 3 tests runs in a separate worker process

class TestParallelLogin:

    def test_valid_login(self, driver, wait):
        """Test login with valid credentials."""
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
        worker = os.environ.get("PYTEST_XDIST_WORKER", "main")
        print(f"[{worker}] Valid login passed")

    def test_invalid_login(self, driver, wait):
        """Test login with invalid credentials."""
        driver.get("https://the-internet.herokuapp.com/login")

        driver.find_element(By.ID, "username").send_keys("wronguser")
        driver.find_element(By.ID, "password").send_keys("wrongpass")
        driver.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        ).click()

        flash = wait.until(
            EC.visibility_of_element_located((By.ID, "flash"))
        )

        assert "Your username is invalid" in flash.text
        worker = os.environ.get("PYTEST_XDIST_WORKER", "main")
        print(f"[{worker}] Invalid login passed")

    def test_empty_credentials(self, driver, wait):
        """Test login with empty fields."""
        driver.get("https://the-internet.herokuapp.com/login")

        driver.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        ).click()

        flash = wait.until(
            EC.visibility_of_element_located((By.ID, "flash"))
        )

        assert "Your username is invalid" in flash.text
        worker = os.environ.get("PYTEST_XDIST_WORKER", "main")
        print(f"[{worker}] Empty credentials passed")


# =============================================
# CROSS-BROWSER PARALLEL TESTS
# =============================================

@pytest.fixture(params=["chrome", "firefox", "edge"])
def cross_browser_driver(request):
    """
    Parametrized fixture — creates a different browser for each param.
    With pytest-xdist, each browser runs in its own worker process.
    """
    browser = request.param

    if browser == "chrome":
        options = ChromeOptions()
        options.add_argument("--headless=new")
        options.add_argument("--window-size=1920,1080")
        drv = webdriver.Chrome(options=options)
    elif browser == "firefox":
        options = FirefoxOptions()
        options.add_argument("--headless")
        options.add_argument("--width=1920")
        options.add_argument("--height=1080")
        drv = webdriver.Firefox(options=options)
    elif browser == "edge":
        options = EdgeOptions()
        options.add_argument("--headless=new")
        options.add_argument("--window-size=1920,1080")
        drv = webdriver.Edge(options=options)
    else:
        raise ValueError(f"Unknown browser: {browser}")

    drv._browser_name = browser
    yield drv
    drv.quit()


class TestCrossBrowser:
    """
    Every test in this class runs 3 times: Chrome, Firefox, Edge.
    With -n 3, all three run simultaneously.
    """

    def test_homepage_title(self, cross_browser_driver):
        """Verify homepage loads correctly on all browsers."""
        cross_browser_driver.get("https://the-internet.herokuapp.com/")

        assert "Internet" in cross_browser_driver.title
        print(
            f"[{cross_browser_driver._browser_name}] "
            f"Title: {cross_browser_driver.title}"
        )

    def test_login_form_exists(self, cross_browser_driver):
        """Verify login page has required elements on all browsers."""
        cross_browser_driver.get(
            "https://the-internet.herokuapp.com/login"
        )

        username = cross_browser_driver.find_element(By.ID, "username")
        password = cross_browser_driver.find_element(By.ID, "password")
        submit = cross_browser_driver.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        )

        assert username.is_displayed()
        assert password.is_displayed()
        assert submit.is_displayed()
        print(
            f"[{cross_browser_driver._browser_name}] "
            "Login form elements verified"
        )


# =============================================
# DATA-DRIVEN PARALLEL TESTS
# =============================================

@pytest.mark.parametrize("path,expected_heading", [
    ("checkboxes", "Checkboxes"),
    ("dropdown", "Dropdown"),
    ("login", "Login"),
    ("inputs", "Inputs"),
    ("key_presses", "Key Presses"),
    ("hovers", "Hovers"),
    ("drag_and_drop", "Drag and Drop"),
    ("dynamic_loading", "Dynamically Loaded"),
])
def test_page_headings(driver, wait, path, expected_heading):
    """
    Data-driven test that verifies page headings.
    With -n auto, each parametrized case runs in its own worker.
    """
    driver.get(f"https://the-internet.herokuapp.com/{path}")

    heading = wait.until(
        EC.presence_of_element_located((By.TAG_NAME, "h3"))
    )

    assert expected_heading in heading.text
    worker = os.environ.get("PYTEST_XDIST_WORKER", "main")
    print(f"[{worker}] /{path} -> {heading.text}")


# =============================================
# SEQUENTIAL VS PARALLEL TIMING
# =============================================

class TestTimingComparison:
    """
    Run this class both ways to see the difference:

    Sequential: pytest test_day27.py::TestTimingComparison -v
    Parallel:   pytest test_day27.py::TestTimingComparison -n 5 -v
    """

    def test_page_1(self, driver, wait):
        driver.get("https://the-internet.herokuapp.com/login")
        wait.until(EC.presence_of_element_located((By.ID, "username")))
        assert driver.title is not None

    def test_page_2(self, driver, wait):
        driver.get("https://the-internet.herokuapp.com/checkboxes")
        wait.until(EC.presence_of_element_located(
            (By.CSS_SELECTOR, "input[type='checkbox']")
        ))
        assert driver.title is not None

    def test_page_3(self, driver, wait):
        driver.get("https://the-internet.herokuapp.com/dropdown")
        wait.until(EC.presence_of_element_located((By.ID, "dropdown")))
        assert driver.title is not None

    def test_page_4(self, driver, wait):
        driver.get("https://the-internet.herokuapp.com/inputs")
        wait.until(EC.presence_of_element_located(
            (By.CSS_SELECTOR, "input[type='number']")
        ))
        assert driver.title is not None

    def test_page_5(self, driver, wait):
        driver.get("https://the-internet.herokuapp.com/key_presses")
        wait.until(EC.presence_of_element_located((By.ID, "target")))
        assert driver.title is not None


# =============================================
# CONFTEST.PY FOR PARALLEL PROJECTS
# =============================================
# For a real project, create conftest.py:

CONFTEST_EXAMPLE = '''
# conftest.py
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options


@pytest.fixture(scope="function")
def driver():
    """
    Function scope is REQUIRED for parallel execution.
    Session or module scope shares the driver across tests,
    which is NOT thread-safe.
    """
    options = Options()
    options.add_argument("--headless=new")
    options.add_argument("--window-size=1920,1080")
    options.add_argument("--disable-gpu")

    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


# pytest.ini or pyproject.toml:
# [tool.pytest.ini_options]
# addopts = "-n auto --dist=loadscope"
'''
```

---

## 5. Step-by-Step Walkthrough

### C# NUnit Parallel Setup

1. Add `[assembly: LevelOfParallelism(4)]` at the top of any file to set max threads
2. Add `[Parallelizable(ParallelScope.All)]` to test fixtures you want parallelized
3. Use `ThreadLocal<IWebDriver>` or create drivers in `[SetUp]` to ensure isolation
4. Run with `dotnet test -- NUnit.NumberOfTestWorkers=4`

### Python pytest-xdist Setup

1. Install: `pip install pytest-xdist`
2. Use function-scoped fixtures (the default) for driver creation
3. Run with: `pytest -n 4 -v` (4 workers) or `pytest -n auto` (auto-detect cores)
4. Each worker is a separate process, so isolation is automatic

### Cross-Browser Parallel Testing

1. Create a parametrized fixture with browser names
2. Inside the fixture, create the appropriate driver based on the parameter
3. Each parametrized variant runs independently
4. With `-n 3` (Python) or `LevelOfParallelism(3)` (C#), all three browsers run simultaneously

---

## 6. Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Sharing WebDriver across threads | Random failures, wrong pages | Use `ThreadLocal<>` (C#) or function-scoped fixtures (Python) |
| Using static/class-level driver | Multiple tests use same driver | Make driver instance-level |
| Session-scoped fixture with xdist | All tests in a session share one driver | Use `scope="function"` |
| Not closing browsers in teardown | Resource leak, port exhaustion | Always `driver.Quit()` in teardown/finally |
| Too many parallel threads | Machine runs out of RAM/CPU | Start with 4, increase gradually |
| Shared test data (same user/record) | Tests conflict with each other | Use unique test data per test |
| File I/O conflicts | Multiple tests write to same file | Include thread/worker ID in filenames |

---

## 7. Best Practices

1. **Start with a small parallel count (2-4)** and increase gradually. Monitor CPU and RAM usage.

2. **Use headless mode for parallel tests.** It reduces resource usage significantly.

3. **Make tests independent.** No test should depend on the result or side effects of another test.

4. **Use unique test data.** If tests share a database, use different user accounts or generate unique records.

5. **Keep fixture scope as narrow as possible.** Function scope is safest for parallel execution.

6. **Log the thread/worker ID** in test output so you can trace which worker ran which test.

7. **Run parallel tests on CI first.** Your local machine may not have enough resources for high parallelism.

8. **Use Selenium Grid for large-scale parallel execution.** Local machines top out at 4-8 browser instances.

---

## 8. Hands-On Exercise

### Task: Run Same Tests on 3 Browsers Simultaneously

**Objective:** Create a test suite that runs on Chrome, Firefox, and Edge in parallel.

**Steps:**
1. Create a parametrized fixture/test source with 3 browsers
2. Write 3 tests: login, checkbox toggle, dropdown select
3. Run all 9 test executions (3 tests x 3 browsers) in parallel
4. Verify all pass on all browsers

**Run Commands:**
```bash
# Python
pytest test_day27.py -n 3 -v

# C#
dotnet test -- NUnit.NumberOfTestWorkers=3
```

**Expected Output:**
```
[chrome]  test_login          PASSED
[firefox] test_login          PASSED
[edge]    test_login          PASSED
[chrome]  test_checkbox       PASSED
[firefox] test_checkbox       PASSED
[edge]    test_checkbox       PASSED
[chrome]  test_dropdown       PASSED
[firefox] test_dropdown       PASSED
[edge]    test_dropdown       PASSED

9 passed in 12.5 seconds (vs 45.0 seconds sequential)
```

---

## 9. Real-World Scenario

### Scenario: Nightly Regression Suite on Grid

```python
# conftest.py for a 500-test regression suite
import os
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options


GRID_URL = os.environ.get(
    "SELENIUM_GRID_URL", "http://selenium-grid:4444"
)


@pytest.fixture(scope="function")
def driver():
    """
    Each test gets its own remote browser on the Grid.
    With 20 Grid nodes and pytest -n 20, all 500 tests
    complete in ~5 minutes instead of ~80 minutes.
    """
    options = Options()
    options.add_argument("--headless=new")
    options.add_argument("--window-size=1920,1080")

    drv = webdriver.Remote(
        command_executor=GRID_URL,
        options=options
    )
    yield drv
    drv.quit()


# Run: pytest tests/ -n 20 --dist=loadscope -v
# --dist=loadscope groups tests by module,
# so tests in the same file go to the same worker.
```

---

## 10. Resources

- [NUnit Parallelizable Attribute](https://docs.nunit.org/articles/nunit/writing-tests/attributes/parallelizable.html)
- [NUnit LevelOfParallelism](https://docs.nunit.org/articles/nunit/writing-tests/attributes/levelofparallelism.html)
- [pytest-xdist Documentation](https://pytest-xdist.readthedocs.io/)
- [Selenium Grid for Parallel Testing](https://www.selenium.dev/documentation/grid/)
- [Thread Safety in Selenium](https://www.selenium.dev/documentation/webdriver/troubleshooting/threads/)
