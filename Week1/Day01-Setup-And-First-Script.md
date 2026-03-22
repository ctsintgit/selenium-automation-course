# Day 1: Selenium Architecture, Components, Setup & First Script

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain what Selenium is and identify its core components (WebDriver, Grid, IDE)
- Describe how WebDriver communicates with browsers via the W3C WebDriver protocol
- Set up a Selenium project in both C# (.NET 6/8) and Python
- Write and run your first browser automation script
- Use Selenium Manager for automatic driver management (no manual ChromeDriver downloads)

---

## Core Concept Explanation

### What Is Selenium?

Selenium is an open-source framework for automating web browsers. It lets you write code that controls a real browser — clicking buttons, filling forms, reading text — just like a human user would. Selenium is the industry standard for browser-based test automation.

### Selenium Components

| Component | Purpose |
|-----------|---------|
| **Selenium WebDriver** | The core library. Provides a programming API to control browsers directly. This is what you will use daily. |
| **Selenium Grid** | Runs tests in parallel across multiple machines and browsers. Used in CI/CD pipelines at scale. |
| **Selenium IDE** | A browser extension for record-and-playback. Good for quick prototyping, not for production tests. |

This course focuses on **Selenium WebDriver** — the component that matters for real automation work.

---

## How It Works (Technical Breakdown)

### The WebDriver Architecture

```
Your Test Code  --->  WebDriver API  --->  Browser Driver  --->  Browser
   (C#/Python)        (HTTP Client)      (chromedriver.exe)    (Chrome)
```

1. **Your test code** calls the WebDriver API (e.g., `driver.FindElement(By.Id("login"))`)
2. The WebDriver API translates that call into an **HTTP request** following the **W3C WebDriver protocol**
3. The HTTP request is sent to the **browser driver** (chromedriver for Chrome, geckodriver for Firefox), which is a small executable running as a local server
4. The browser driver translates the HTTP command into **browser-native instructions** and sends them to the actual browser
5. The browser performs the action and returns the result back through the same chain

### W3C WebDriver Protocol

The W3C WebDriver protocol is a REST API standard. For example:

- `POST /session` — start a new browser session
- `POST /session/{id}/url` — navigate to a URL
- `POST /session/{id}/element` — find an element
- `DELETE /session/{id}` — close the browser session

You never call these endpoints directly — the Selenium library handles it for you.

### Selenium Manager (Automatic Driver Management)

Starting with Selenium 4.6, **Selenium Manager** is built into the Selenium libraries. It automatically:

- Detects which browser version you have installed
- Downloads the matching browser driver executable
- Places it in a cache directory and configures the path

This means you no longer need to manually download chromedriver or manage its version. Your code simply creates a `ChromeDriver` instance and Selenium Manager handles the rest.

---

## Code Example: C# (Complete, Runnable)

### Project Setup

Open a terminal and run these commands:

```bash
# Create a new NUnit test project
dotnet new nunit -n SeleniumCourse -f net8.0
cd SeleniumCourse

# Install Selenium WebDriver (includes Selenium Manager)
dotnet add package Selenium.WebDriver --version 4.27.0

# Verify packages
dotnet restore
```

### Test File: `Day01_FirstScript.cs`

```csharp
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;

namespace SeleniumCourse;

[TestFixture]
public class Day01_FirstScript
{
    // Declare the driver at the class level so all tests can use it
    private IWebDriver driver;

    [SetUp]
    public void Setup()
    {
        // ChromeOptions lets you configure Chrome behavior
        var options = new ChromeOptions();

        // Optional: run in headless mode (no visible browser window)
        // options.AddArgument("--headless=new");

        // Create the ChromeDriver — Selenium Manager handles driver download
        driver = new ChromeDriver(options);

        // Set a default timeout for page loads
        driver.Manage().Timeouts().PageLoad = TimeSpan.FromSeconds(30);
    }

    [Test]
    public void OpenBrowserAndVerifyTitle()
    {
        // Navigate to a website
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

        // Read the page title
        string title = driver.Title;

        // Print the title to the console
        Console.WriteLine($"Page title is: {title}");

        // Assert that the title matches what we expect
        Assert.That(title, Is.EqualTo("The Internet"));
    }

    [Test]
    public void NavigateAndReadContent()
    {
        // Navigate to a specific page
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Read the page heading text
        IWebElement heading = driver.FindElement(By.TagName("h2"));
        string headingText = heading.Text;

        Console.WriteLine($"Heading text is: {headingText}");

        // Verify the heading
        Assert.That(headingText, Is.EqualTo("Login Page"));
    }

    [TearDown]
    public void Teardown()
    {
        // Always close the browser after the test
        // Quit() closes all windows and ends the WebDriver session
        driver.Quit();
    }
}
```

### Run the tests:

```bash
dotnet test --logger "console;verbosity=detailed"
```

---

## Code Example: Python (Complete, Runnable)

### Project Setup

```bash
# Create project directory
mkdir selenium_course
cd selenium_course

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install packages
pip install selenium==4.27.0 pytest
```

### Test File: `test_day01_first_script.py`

```python
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By


@pytest.fixture
def driver():
    """
    Pytest fixture that creates a ChromeDriver before each test
    and quits it after the test completes.
    Selenium Manager handles driver download automatically.
    """
    # Configure Chrome options
    options = Options()

    # Optional: run headless (no visible window)
    # options.add_argument("--headless=new")

    # Create the driver — Selenium Manager handles chromedriver
    drv = webdriver.Chrome(options=options)

    # Set page load timeout
    drv.set_page_load_timeout(30)

    # yield gives the driver to the test, then runs cleanup after
    yield drv

    # Cleanup: close all browser windows and end the session
    drv.quit()


def test_open_browser_and_verify_title(driver):
    """Open a website and verify the page title."""
    # Navigate to the website
    driver.get("https://the-internet.herokuapp.com/")

    # Read the page title
    title = driver.title

    # Print the title
    print(f"Page title is: {title}")

    # Assert the title matches
    assert title == "The Internet"


def test_navigate_and_read_content(driver):
    """Navigate to a page and read element content."""
    # Navigate to the login page
    driver.get("https://the-internet.herokuapp.com/login")

    # Find the heading element by tag name
    heading = driver.find_element(By.TAG_NAME, "h2")
    heading_text = heading.text

    print(f"Heading text is: {heading_text}")

    # Verify the heading text
    assert heading_text == "Login Page"
```

### Run the tests:

```bash
pytest test_day01_first_script.py -v -s
```

---

## Step-by-Step Walkthrough

### What happens when you run the C# test `OpenBrowserAndVerifyTitle`:

1. **`[SetUp]`** runs first. `new ChromeDriver(options)` triggers Selenium Manager, which checks your Chrome version, downloads the matching chromedriver if needed, starts the chromedriver process, and connects to it.

2. **`driver.Navigate().GoToUrl(...)`** sends an HTTP POST to chromedriver with the URL. Chromedriver tells Chrome to navigate there. The call blocks until the page finishes loading.

3. **`driver.Title`** sends an HTTP GET to chromedriver asking for the page title. Chromedriver queries Chrome's DOM and returns the title string.

4. **`Assert.That(...)`** checks the value. If it does not match, NUnit marks the test as failed.

5. **`[TearDown]`** runs after the test. `driver.Quit()` sends a DELETE request to close the session, which shuts down Chrome and the chromedriver process.

### What happens in the Python test:

The flow is identical. The `@pytest.fixture` with `yield` is equivalent to `[SetUp]` + `[TearDown]`. Code before `yield` is setup, code after `yield` is teardown.

---

## Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `driver.Quit()` | Browser processes pile up, consuming memory | Always use `[TearDown]` (C#) or a fixture with `yield` (Python) |
| Using `driver.Close()` instead of `driver.Quit()` | `Close()` only closes the current tab; the driver process may linger | Use `Quit()` to end the entire session |
| Manually downloading chromedriver | Version mismatches cause crashes when Chrome auto-updates | Let Selenium Manager handle it (Selenium 4.6+) |
| Not setting timeouts | Tests hang forever on slow pages | Set `PageLoad` and `ImplicitWait` timeouts |
| Hardcoding chromedriver path | Breaks on other machines | Remove path arguments; Selenium Manager resolves it |

---

## Best Practices

1. **Always use `Quit()` in teardown** — this guarantees the browser and driver process are cleaned up, even if the test fails.

2. **Use a fixture or setup/teardown pattern** — never create the driver inside the test method itself. Separation of setup and test logic keeps code clean.

3. **Let Selenium Manager handle drivers** — avoid third-party packages like WebDriverManager unless you have a specific reason. Selenium 4.6+ includes automatic driver management.

4. **Run headless in CI/CD** — add `--headless=new` when running on servers without a display. Remove it during development so you can see what the browser does.

5. **One driver per test** — creating a fresh browser for each test prevents state leakage between tests.

---

## Hands-On Exercise

### Task

Write a test that does the following:

1. Opens Chrome and navigates to `https://the-internet.herokuapp.com/`
2. Prints the page title
3. Asserts the title is `"The Internet"`
4. Navigates to `https://the-internet.herokuapp.com/status_codes`
5. Prints the new page title
6. Asserts the current URL contains `"status_codes"`
7. Closes the browser

### Expected Output

```
Page title is: The Internet
New page title is: The Internet
Current URL contains 'status_codes': True
Test passed!
```

---

## Real-World Scenario

Imagine you work at a company that deploys a web application every week. Before each deployment, someone manually opens the site, checks that the login page loads, verifies the title, and clicks through a few pages. This takes 30 minutes and is error-prone.

With Selenium, you write a test that does all of this in 10 seconds. It runs automatically in your CI/CD pipeline after every build. If the login page is broken, the test fails and blocks the deployment before any customer sees the problem.

Today's script — opening a browser, navigating to a URL, and checking the title — is the first building block of that automated safety net.

---

## Resources

- [Selenium Official Documentation](https://www.selenium.dev/documentation/)
- [Selenium WebDriver Getting Started](https://www.selenium.dev/documentation/webdriver/getting_started/)
- [Selenium Manager (Automatic Driver Management)](https://www.selenium.dev/documentation/selenium_manager/)
- [W3C WebDriver Specification](https://www.w3.org/TR/webdriver2/)
- [NUnit Documentation](https://docs.nunit.org/)
- [Pytest Documentation](https://docs.pytest.org/)
- [Practice Site: The Internet](https://the-internet.herokuapp.com/)
