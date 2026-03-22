# 30-Day Selenium Automation Testing Course

## Dual-Track: C# (NUnit) + Python (PyTest)

Master Selenium WebDriver from scratch in 30 days. Every lesson includes complete code examples in both C# and Python, so you can follow whichever track fits your background -- or learn both side by side.

---

## Prerequisites

Before starting this course, you should have:

- **General**: Basic understanding of HTML, CSS, and how web pages work (DOM structure, forms, links)
- **For C# Track**: Visual Studio 2019+ or VS Code with C# extension, .NET 6+ SDK installed, basic C# syntax knowledge (variables, loops, classes, methods)
- **For Python Track**: Python 3.8+ installed, a code editor (VS Code, PyCharm), basic Python syntax knowledge (variables, loops, functions, classes)
- **Browser**: Google Chrome (latest) or Mozilla Firefox (latest) installed
- **OS**: Windows 10/11, macOS, or Linux

No prior Selenium or test automation experience is required.

---

## How to Use This Course

1. **Pick your track** -- C# (NUnit) or Python (PyTest) -- or follow both.
2. **Complete one day per day.** Each lesson is designed to take 1-2 hours including practice exercises.
3. **Type the code yourself.** Do not copy-paste. Muscle memory matters for automation.
4. **Run every example** against the practice websites listed below.
5. **Use the Cheat Sheets** as quick-lookup references while coding.
6. **Build the Final Project** in Week 4 to tie everything together.

---

## Table of Contents

### Week 1 -- Foundations

| Day | Topic |
|-----|-------|
| [Day 01](Week1/Day01-Setup-And-First-Script.md) | Setup and First Script |
| [Day 02](Week1/Day02-Locators-Basic.md) | Locators: ID, Name, ClassName, TagName |
| [Day 03](Week1/Day03-XPath.md) | XPath (Absolute vs Relative) |
| [Day 04](Week1/Day04-CSS-Selectors.md) | CSS Selectors |
| [Day 05](Week1/Day05-Browser-Actions.md) | Browser Actions: Click, SendKeys, Clear, Submit |
| [Day 06](Week1/Day06-Navigation.md) | Navigation: Back, Forward, Refresh, URL, Title |
| [Day 07](Week1/Day07-Dropdowns-Checkboxes-Radio.md) | Dropdowns, Checkboxes, and Radio Buttons |

### Week 2 -- Synchronization & Complex UI

| Day | Topic |
|-----|-------|
| [Day 08](Week2/Day08-Implicit-vs-Explicit-Wait.md) | Implicit vs. Explicit Wait |
| [Day 09](Week2/Day09-Fluent-Wait.md) | Fluent Wait and ExpectedConditions |
| [Day 10](Week2/Day10-Alerts.md) | Alerts: Accept, Dismiss, GetText, SendKeys |
| [Day 11](Week2/Day11-Frames.md) | Frames and iFrames |
| [Day 12](Week2/Day12-Windows-Tabs.md) | Multiple Windows and Tabs |
| [Day 13](Week2/Day13-Dynamic-Elements.md) | Dynamic Elements: AJAX, Lazy Loading, Stale Elements |
| [Day 14](Week2/Day14-File-Upload-Download.md) | File Upload and Download |

### Week 3 -- Test Framework & Page Object Model

| Day | Topic |
|-----|-------|
| [Day 15](Week3/Day15-NUnit-PyTest-Setup.md) | NUnit / PyTest Setup |
| [Day 16](Week3/Day16-Test-Lifecycle.md) | Test Lifecycle: Setup, Teardown, Fixtures |
| [Day 17](Week3/Day17-Page-Object-Model.md) | Page Object Model (POM) |
| [Day 18](Week3/Day18-Page-Factory.md) | Page Factory Pattern |
| [Day 19](Week3/Day19-Config-Files.md) | Config Files: JSON/YAML for Environment-Driven Execution |
| [Day 20](Week3/Day20-Data-Driven-Testing.md) | Data-Driven Testing |
| [Day 21](Week3/Day21-Logging-Reporting.md) | Logging and Reporting |

### Week 4 -- Advanced + Enterprise

| Day | Topic |
|-----|-------|
| [Day 22](Week4/Day22-JavaScriptExecutor.md) | JavaScriptExecutor |
| [Day 23](Week4/Day23-Shadow-DOM.md) | Shadow DOM |
| [Day 24](Week4/Day24-Screenshots.md) | Screenshots on Failure + Test Evidence |
| [Day 25](Week4/Day25-Headless-Mode.md) | Headless Mode: Chrome and Firefox |
| [Day 26](Week4/Day26-Selenium-Grid.md) | Selenium Grid: Hub/Node Architecture |
| [Day 27](Week4/Day27-Parallel-Execution.md) | Parallel Execution: Multi-Browser, Multi-Thread |
| [Day 28](Week4/Day28-CICD-Integration.md) | CI/CD Integration: GitHub Actions |
| [Day 29](Week4/Day29-Framework-Finalization.md) | Framework Finalization: Structure and Conventions |
| [Day 30](Week4/Day30-Final-Project.md) | Final Project: Complete E2E Framework |

### Reference Materials

| Resource | Description |
|----------|-------------|
| [Quick Reference](Cheat-Sheet/Quick-Reference.md) | Side-by-side C# / Python syntax for all common operations |
| [Glossary](Cheat-Sheet/Glossary.md) | Definitions of every Selenium term you will encounter |

---

## Setup Instructions

### C# Track (NUnit)

1. **Install .NET SDK** (6.0 or later) from [https://dotnet.microsoft.com/download](https://dotnet.microsoft.com/download).

2. **Create a test project**:
   ```bash
   dotnet new nunit -n SeleniumCourse
   cd SeleniumCourse
   ```

3. **Add Selenium packages**:
   ```bash
   dotnet add package Selenium.WebDriver
   dotnet add package Selenium.Support
   dotnet add package WebDriverManager
   ```

4. **Verify** -- create a simple test that opens a browser:
   ```csharp
   using NUnit.Framework;
   using OpenQA.Selenium;
   using OpenQA.Selenium.Chrome;
   using WebDriverManager;
   using WebDriverManager.DriverConfigs.Impl;

   [TestFixture]
   public class SetupCheck
   {
       private IWebDriver driver;

       [SetUp]
       public void Setup()
       {
           new DriverManager().SetUpDriver(new ChromeConfig());
           driver = new ChromeDriver();
       }

       [Test]
       public void BrowserOpens()
       {
           driver.Navigate().GoToUrl("https://www.google.com");
           Assert.That(driver.Title, Does.Contain("Google"));
       }

       [TearDown]
       public void Teardown()
       {
           driver.Quit();
       }
   }
   ```

5. **Run**: `dotnet test`

### Python Track (PyTest)

1. **Install Python 3.8+** from [https://www.python.org/downloads/](https://www.python.org/downloads/).

2. **Create a virtual environment**:
   ```bash
   python -m venv venv
   source venv/bin/activate   # Linux/macOS
   venv\Scripts\activate      # Windows
   ```

3. **Install packages**:
   ```bash
   pip install selenium pytest webdriver-manager
   ```

4. **Verify** -- create `test_setup.py`:
   ```python
   import pytest
   from selenium import webdriver
   from selenium.webdriver.chrome.service import Service
   from webdriver_manager.chrome import ChromeDriverManager

   @pytest.fixture
   def driver():
       service = Service(ChromeDriverManager().install())
       drv = webdriver.Chrome(service=service)
       yield drv
       drv.quit()

   def test_browser_opens(driver):
       driver.get("https://www.google.com")
       assert "Google" in driver.title
   ```

5. **Run**: `pytest test_setup.py -v`

---

## Recommended Practice Websites

These sites are purpose-built for Selenium practice and will not break when you automate against them:

| Website | URL | Good For |
|---------|-----|----------|
| The Internet (Heroku) | [https://the-internet.herokuapp.com](https://the-internet.herokuapp.com) | All core concepts: login, forms, alerts, frames, uploads, dynamic loading |
| Sauce Demo | [https://www.saucedemo.com](https://www.saucedemo.com) | E-commerce flow: login, product selection, cart, checkout |
| DemoQA | [https://demoqa.com](https://demoqa.com) | Forms, widgets, interactions, book store API |
| Automation Exercise | [https://automationexercise.com](https://automationexercise.com) | Full e-commerce test scenarios with an API |
| Practice Automation | [https://practice-automation.com](https://practice-automation.com) | Sliders, popups, modals, calendars |
| UI Testing Playground | [http://uitestingplayground.com](http://uitestingplayground.com) | Tricky scenarios: dynamic IDs, AJAX, hidden layers |
| Selenium Playground (LambdaTest) | [https://www.lambdatest.com/selenium-playground](https://www.lambdatest.com/selenium-playground) | Input forms, tables, alerts, drag-and-drop |
| OrangeHRM Demo | [https://opensource-demo.orangehrmlive.com](https://opensource-demo.orangehrmlive.com) | Real-world HR app login and navigation |

---

Happy automating!
