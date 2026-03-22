# Day 30: Final Project — Complete End-to-End Automation Framework

## Learning Objectives

By the end of this lesson, you will have built a complete, production-quality automation framework called **JobPortalAutomation** that includes:

- Page Object Model for all pages
- Config-driven execution (base URL, credentials, browser from config)
- Parallel execution support
- Screenshot capture on failure
- Report integration
- GitHub Actions workflow for CI/CD
- Complete runnable code in both C# and Python

---

## 1. Project Overview

### The Application Under Test

We are automating a **Job Portal** web application. The test scenarios cover the core user journeys:

1. **User Login** — valid credentials, invalid credentials, empty fields
2. **Job Search** — search with keyword, location filter, job type filter
3. **Resume Upload** — upload a file and verify it appears
4. **Job Application** — apply to a job posting and confirm submission
5. **Admin Workflow** — admin changes candidate status

Since we need a publicly available test site, all tests target `https://the-internet.herokuapp.com/` and map its pages to job portal concepts. The framework structure, patterns, and code are production-ready — you only need to swap the URLs and locators for your real application.

### Page Mapping

| Job Portal Page | Test Site Equivalent | URL Path |
|----------------|---------------------|----------|
| Login Page | Login Page | `/login` |
| Dashboard | Secure Area | `/secure` |
| Job Search | Form Authentication (search simulation) | `/login` |
| Resume Upload | File Upload | `/upload` |
| Job Application | Dynamic Loading (submit simulation) | `/dynamic_loading/1` |
| Admin Panel | Checkboxes (status toggles) | `/checkboxes` |

---

## 2. C# Complete Framework

### Folder Structure

```
JobPortalAutomation/
├── JobPortalAutomation.sln
├── .gitignore
├── .github/
│   └── workflows/
│       └── selenium-tests.yml
│
└── JobPortalAutomation.Tests/
    ├── JobPortalAutomation.Tests.csproj
    │
    ├── Config/
    │   └── appsettings.json
    │
    ├── Drivers/
    │   └── DriverFactory.cs
    │
    ├── Pages/
    │   ├── BasePage.cs
    │   ├── LoginPage.cs
    │   ├── DashboardPage.cs
    │   ├── JobSearchPage.cs
    │   ├── ResumeUploadPage.cs
    │   ├── JobApplicationPage.cs
    │   └── AdminPage.cs
    │
    ├── Tests/
    │   ├── BaseTest.cs
    │   ├── LoginTests.cs
    │   ├── JobSearchTests.cs
    │   ├── ResumeUploadTests.cs
    │   ├── JobApplicationTests.cs
    │   └── AdminWorkflowTests.cs
    │
    ├── Utils/
    │   ├── ConfigReader.cs
    │   ├── WaitHelper.cs
    │   ├── ScreenshotHelper.cs
    │   └── TestDataGenerator.cs
    │
    └── Reports/
        └── .gitkeep
```

### File: JobPortalAutomation.Tests.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <RootNamespace>JobPortalAutomation</RootNamespace>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.9.0" />
    <PackageReference Include="NUnit" Version="4.1.0" />
    <PackageReference Include="NUnit3TestAdapter" Version="4.5.0" />
    <PackageReference Include="Selenium.WebDriver" Version="4.18.1" />
    <PackageReference Include="Selenium.Support" Version="4.18.1" />
  </ItemGroup>
  <ItemGroup>
    <None Update="Config\appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
  </ItemGroup>
</Project>
```

### File: Config/appsettings.json

```json
{
    "BaseUrl": "https://the-internet.herokuapp.com",
    "Browser": "chrome",
    "Headless": "false",
    "DefaultTimeoutSeconds": "10",
    "GridUrl": "",
    "ValidUser": {
        "Username": "tomsmith",
        "Password": "SuperSecretPassword!"
    },
    "AdminUser": {
        "Username": "admin",
        "Password": "admin"
    },
    "AlertEmail": "qa-alerts@company.com"
}
```

### File: Utils/ConfigReader.cs

```csharp
using System;
using System.IO;
using System.Text.Json;

namespace JobPortalAutomation.Utils
{
    /// <summary>
    /// Reads configuration from appsettings.json.
    /// Environment variables override JSON values.
    /// </summary>
    public static class ConfigReader
    {
        private static readonly JsonDocument Config;

        static ConfigReader()
        {
            string path = Path.Combine(
                AppDomain.CurrentDomain.BaseDirectory,
                "Config", "appsettings.json"
            );

            if (File.Exists(path))
            {
                string json = File.ReadAllText(path);
                Config = JsonDocument.Parse(json);
            }
        }

        public static string BaseUrl =>
            Environment.GetEnvironmentVariable("BASE_URL")
            ?? GetString("BaseUrl")
            ?? "https://the-internet.herokuapp.com";

        public static string Browser =>
            Environment.GetEnvironmentVariable("BROWSER")
            ?? GetString("Browser")
            ?? "chrome";

        public static bool Headless
        {
            get
            {
                string env = Environment.GetEnvironmentVariable(
                    "SELENIUM_HEADLESS"
                );
                if (env != null)
                    return env.ToLower() == "true" || env == "1";
                return GetString("Headless")?.ToLower() == "true";
            }
        }

        public static int DefaultTimeout =>
            int.Parse(GetString("DefaultTimeoutSeconds") ?? "10");

        public static string GridUrl =>
            Environment.GetEnvironmentVariable("SELENIUM_GRID_URL")
            ?? GetString("GridUrl")
            ?? "";

        public static string ValidUsername =>
            GetNestedString("ValidUser", "Username") ?? "tomsmith";

        public static string ValidPassword =>
            GetNestedString("ValidUser", "Password")
            ?? "SuperSecretPassword!";

        public static string AdminUsername =>
            GetNestedString("AdminUser", "Username") ?? "admin";

        public static string AdminPassword =>
            GetNestedString("AdminUser", "Password") ?? "admin";

        private static string GetString(string key)
        {
            try
            {
                return Config?.RootElement.GetProperty(key).GetString();
            }
            catch { return null; }
        }

        private static string GetNestedString(
            string parent, string child)
        {
            try
            {
                return Config?.RootElement
                    .GetProperty(parent)
                    .GetProperty(child)
                    .GetString();
            }
            catch { return null; }
        }
    }
}
```

### File: Utils/WaitHelper.cs

```csharp
using OpenQA.Selenium;
using OpenQA.Selenium.Support.UI;
using System;
using System.Collections.ObjectModel;

namespace JobPortalAutomation.Utils
{
    /// <summary>
    /// Centralized explicit wait utilities.
    /// All waits use WebDriverWait — never Thread.Sleep.
    /// </summary>
    public class WaitHelper
    {
        private readonly WebDriverWait _wait;

        public WaitHelper(IWebDriver driver, int timeoutSeconds = 10)
        {
            _wait = new WebDriverWait(
                driver, TimeSpan.FromSeconds(timeoutSeconds)
            );
            // Ignore common transient exceptions during polling
            _wait.IgnoreExceptionTypes(
                typeof(NoSuchElementException),
                typeof(StaleElementReferenceException)
            );
        }

        public IWebElement WaitForVisible(By locator)
        {
            return _wait.Until(d =>
            {
                var el = d.FindElement(locator);
                return el.Displayed ? el : null;
            });
        }

        public IWebElement WaitForClickable(By locator)
        {
            return _wait.Until(d =>
            {
                var el = d.FindElement(locator);
                return (el.Displayed && el.Enabled) ? el : null;
            });
        }

        public bool WaitForInvisible(By locator)
        {
            return _wait.Until(d =>
            {
                try
                {
                    return !d.FindElement(locator).Displayed;
                }
                catch (NoSuchElementException) { return true; }
            });
        }

        public bool WaitForUrlContains(string urlPart)
        {
            return _wait.Until(d => d.Url.Contains(urlPart));
        }

        public IWebElement WaitForTextPresent(By locator, string text)
        {
            return _wait.Until(d =>
            {
                var el = d.FindElement(locator);
                return el.Text.Contains(text) ? el : null;
            });
        }

        public ReadOnlyCollection<IWebElement> WaitForElementCount(
            By locator, int minCount)
        {
            return _wait.Until(d =>
            {
                var elements = d.FindElements(locator);
                return elements.Count >= minCount ? elements : null;
            });
        }
    }
}
```

### File: Utils/ScreenshotHelper.cs

```csharp
using NUnit.Framework;
using OpenQA.Selenium;
using System;
using System.IO;

namespace JobPortalAutomation.Utils
{
    /// <summary>
    /// Screenshot capture with organized file naming.
    /// </summary>
    public static class ScreenshotHelper
    {
        private static readonly string BaseDir = Path.Combine(
            TestContext.CurrentContext.TestDirectory,
            "Reports", "Screenshots"
        );

        public static string CaptureScreenshot(
            IWebDriver driver, string label = "screenshot")
        {
            Directory.CreateDirectory(BaseDir);

            string testName = TestContext.CurrentContext.Test.Name;
            string timestamp = DateTime.Now.ToString(
                "yyyy-MM-dd_HH-mm-ss"
            );
            string fileName = SanitizeFileName(
                $"{timestamp}_{testName}_{label}.png"
            );
            string filePath = Path.Combine(BaseDir, fileName);

            try
            {
                var screenshot =
                    ((ITakesScreenshot)driver).GetScreenshot();
                screenshot.SaveAsFile(filePath);
                TestContext.AddTestAttachment(filePath, label);
                Console.WriteLine($"Screenshot: {filePath}");
                return filePath;
            }
            catch (Exception ex)
            {
                Console.WriteLine(
                    $"Screenshot failed: {ex.Message}"
                );
                return null;
            }
        }

        public static string CaptureElementScreenshot(
            IWebElement element, string label = "element")
        {
            Directory.CreateDirectory(BaseDir);

            string testName = TestContext.CurrentContext.Test.Name;
            string timestamp = DateTime.Now.ToString(
                "yyyy-MM-dd_HH-mm-ss"
            );
            string fileName = SanitizeFileName(
                $"{timestamp}_{testName}_{label}_element.png"
            );
            string filePath = Path.Combine(BaseDir, fileName);

            try
            {
                var screenshot =
                    ((ITakesScreenshot)element).GetScreenshot();
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

        private static string SanitizeFileName(string name)
        {
            foreach (char c in Path.GetInvalidFileNameChars())
            {
                name = name.Replace(c, '_');
            }
            return name;
        }
    }
}
```

### File: Utils/TestDataGenerator.cs

```csharp
using System;

namespace JobPortalAutomation.Utils
{
    /// <summary>
    /// Generates unique test data to avoid conflicts
    /// in parallel execution.
    /// </summary>
    public static class TestDataGenerator
    {
        private static readonly Random Random = new Random();

        public static string UniqueEmail()
        {
            string id = Guid.NewGuid().ToString("N").Substring(0, 8);
            return $"testuser_{id}@test.com";
        }

        public static string UniqueName()
        {
            string id = Guid.NewGuid().ToString("N").Substring(0, 6);
            return $"TestUser_{id}";
        }

        public static string UniqueJobTitle()
        {
            string[] titles = {
                "Software Engineer", "QA Analyst", "DevOps Engineer",
                "Product Manager", "Data Scientist", "UX Designer"
            };
            string id = Random.Next(1000, 9999).ToString();
            return $"{titles[Random.Next(titles.Length)]} #{id}";
        }

        public static string UniqueFileName()
        {
            string id = Guid.NewGuid().ToString("N").Substring(0, 8);
            return $"resume_{id}.pdf";
        }
    }
}
```

### File: Drivers/DriverFactory.cs

```csharp
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Firefox;
using OpenQA.Selenium.Edge;
using OpenQA.Selenium.Remote;
using System;

namespace JobPortalAutomation.Drivers
{
    /// <summary>
    /// Creates WebDriver instances based on configuration.
    /// Supports Chrome, Firefox, Edge — local and Grid.
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
                    var chromeOpts = GetChromeOptions(headless);
                    driver = string.IsNullOrEmpty(gridUrl)
                        ? new ChromeDriver(chromeOpts)
                        : new RemoteWebDriver(
                            new Uri(gridUrl), chromeOpts);
                    break;

                case "firefox":
                    var ffOpts = GetFirefoxOptions(headless);
                    driver = string.IsNullOrEmpty(gridUrl)
                        ? new FirefoxDriver(ffOpts)
                        : new RemoteWebDriver(
                            new Uri(gridUrl), ffOpts);
                    break;

                case "edge":
                    var edgeOpts = GetEdgeOptions(headless);
                    driver = string.IsNullOrEmpty(gridUrl)
                        ? new EdgeDriver(edgeOpts)
                        : new RemoteWebDriver(
                            new Uri(gridUrl), edgeOpts);
                    break;

                default:
                    throw new ArgumentException(
                        $"Unsupported browser: {browser}");
            }

            // Explicit waits only — never use implicit waits
            driver.Manage().Timeouts().ImplicitWait =
                TimeSpan.FromSeconds(0);
            driver.Manage().Timeouts().PageLoad =
                TimeSpan.FromSeconds(60);

            return driver;
        }

        private static ChromeOptions GetChromeOptions(bool headless)
        {
            var options = new ChromeOptions();
            options.AddArgument("--window-size=1920,1080");
            options.AddArgument("--disable-gpu");
            options.AddArgument("--no-sandbox");
            options.AddArgument("--disable-dev-shm-usage");
            if (headless)
                options.AddArgument("--headless=new");
            return options;
        }

        private static FirefoxOptions GetFirefoxOptions(bool headless)
        {
            var options = new FirefoxOptions();
            if (headless)
            {
                options.AddArgument("--headless");
                options.AddArgument("--width=1920");
                options.AddArgument("--height=1080");
            }
            return options;
        }

        private static EdgeOptions GetEdgeOptions(bool headless)
        {
            var options = new EdgeOptions();
            options.AddArgument("--window-size=1920,1080");
            if (headless)
                options.AddArgument("--headless=new");
            return options;
        }
    }
}
```

### File: Pages/BasePage.cs

```csharp
using OpenQA.Selenium;
using JobPortalAutomation.Utils;

namespace JobPortalAutomation.Pages
{
    /// <summary>
    /// Base class for all page objects.
    /// Contains shared functionality: navigation, waits, JS helpers.
    /// Page objects contain NO assertions — only element interactions.
    /// </summary>
    public abstract class BasePage
    {
        protected readonly IWebDriver Driver;
        protected readonly WaitHelper Wait;
        protected readonly string BaseUrl;

        protected BasePage(IWebDriver driver)
        {
            Driver = driver;
            Wait = new WaitHelper(driver, ConfigReader.DefaultTimeout);
            BaseUrl = ConfigReader.BaseUrl;
        }

        public string CurrentUrl => Driver.Url;
        public string PageTitle => Driver.Title;

        protected void NavigateTo(string path)
        {
            Driver.Navigate().GoToUrl($"{BaseUrl}{path}");
        }

        protected void ScrollTo(IWebElement element)
        {
            ((IJavaScriptExecutor)Driver).ExecuteScript(
                "arguments[0].scrollIntoView({block: 'center'});",
                element
            );
        }

        protected void JsClick(IWebElement element)
        {
            ((IJavaScriptExecutor)Driver).ExecuteScript(
                "arguments[0].click();", element
            );
        }

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

        protected string GetElementText(By locator)
        {
            try
            {
                return Driver.FindElement(locator).Text;
            }
            catch (NoSuchElementException)
            {
                return "";
            }
        }
    }
}
```

### File: Pages/LoginPage.cs

```csharp
using OpenQA.Selenium;

namespace JobPortalAutomation.Pages
{
    /// <summary>
    /// Login page — handles user authentication.
    /// Maps to: https://the-internet.herokuapp.com/login
    /// </summary>
    public class LoginPage : BasePage
    {
        // Locators — private, only used within this page
        private readonly By _usernameField = By.Id("username");
        private readonly By _passwordField = By.Id("password");
        private readonly By _loginButton =
            By.CssSelector("button[type='submit']");
        private readonly By _flashMessage = By.Id("flash");
        private readonly By _pageHeading = By.TagName("h2");

        public LoginPage(IWebDriver driver) : base(driver) { }

        /// <summary>Navigate to the login page.</summary>
        public LoginPage Open()
        {
            NavigateTo("/login");
            Wait.WaitForVisible(_usernameField);
            return this;
        }

        /// <summary>Enter username.</summary>
        public LoginPage EnterUsername(string username)
        {
            var field = Wait.WaitForVisible(_usernameField);
            field.Clear();
            field.SendKeys(username);
            return this;
        }

        /// <summary>Enter password.</summary>
        public LoginPage EnterPassword(string password)
        {
            var field = Wait.WaitForVisible(_passwordField);
            field.Clear();
            field.SendKeys(password);
            return this;
        }

        /// <summary>Click the login button.</summary>
        public void ClickLogin()
        {
            Wait.WaitForClickable(_loginButton).Click();
        }

        /// <summary>
        /// Complete login flow. Returns DashboardPage on success.
        /// </summary>
        public DashboardPage LoginAs(string username, string password)
        {
            EnterUsername(username);
            EnterPassword(password);
            ClickLogin();
            return new DashboardPage(Driver);
        }

        /// <summary>
        /// Attempt login that is expected to fail.
        /// Returns self for further checks.
        /// </summary>
        public LoginPage LoginExpectingFailure(
            string username, string password)
        {
            EnterUsername(username);
            EnterPassword(password);
            ClickLogin();
            return this;
        }

        /// <summary>Get the flash message text.</summary>
        public string GetFlashMessage()
        {
            var flash = Wait.WaitForVisible(_flashMessage);
            return flash.Text;
        }

        /// <summary>Get the page heading text.</summary>
        public string GetHeading()
        {
            return Wait.WaitForVisible(_pageHeading).Text;
        }

        /// <summary>Check if username field is displayed.</summary>
        public bool IsUsernameFieldDisplayed()
        {
            return IsElementPresent(_usernameField);
        }
    }
}
```

### File: Pages/DashboardPage.cs

```csharp
using OpenQA.Selenium;

namespace JobPortalAutomation.Pages
{
    /// <summary>
    /// Dashboard / Secure Area page — shown after successful login.
    /// Maps to: https://the-internet.herokuapp.com/secure
    /// </summary>
    public class DashboardPage : BasePage
    {
        private readonly By _heading = By.TagName("h2");
        private readonly By _subheading = By.TagName("h4");
        private readonly By _flashMessage = By.Id("flash");
        private readonly By _logoutButton =
            By.CssSelector("a[href='/logout']");

        public DashboardPage(IWebDriver driver) : base(driver) { }

        /// <summary>Wait for the dashboard to fully load.</summary>
        public DashboardPage WaitForLoad()
        {
            Wait.WaitForUrlContains("/secure");
            Wait.WaitForVisible(_heading);
            return this;
        }

        /// <summary>Get the welcome heading.</summary>
        public string GetHeading()
        {
            return Wait.WaitForVisible(_heading).Text;
        }

        /// <summary>Get the flash/success message.</summary>
        public string GetFlashMessage()
        {
            return Wait.WaitForVisible(_flashMessage).Text;
        }

        /// <summary>Check if logout button is visible.</summary>
        public bool IsLogoutDisplayed()
        {
            return IsElementPresent(_logoutButton);
        }

        /// <summary>Click logout and return to login page.</summary>
        public LoginPage Logout()
        {
            Wait.WaitForClickable(_logoutButton).Click();
            return new LoginPage(Driver);
        }
    }
}
```

### File: Pages/JobSearchPage.cs

```csharp
using OpenQA.Selenium;
using OpenQA.Selenium.Support.UI;
using System.Collections.Generic;
using System.Linq;

namespace JobPortalAutomation.Pages
{
    /// <summary>
    /// Job Search page — search with keyword, location, and type filters.
    /// Simulated using the dropdown and form pages.
    /// Maps to: https://the-internet.herokuapp.com/dropdown (for filters)
    /// </summary>
    public class JobSearchPage : BasePage
    {
        private readonly By _dropdown = By.Id("dropdown");
        private readonly By _dropdownOptions =
            By.CssSelector("#dropdown option");

        public JobSearchPage(IWebDriver driver) : base(driver) { }

        /// <summary>Navigate to the search page.</summary>
        public JobSearchPage Open()
        {
            NavigateTo("/dropdown");
            Wait.WaitForVisible(_dropdown);
            return this;
        }

        /// <summary>Select a filter option by visible text.</summary>
        public JobSearchPage SelectFilter(string optionText)
        {
            var dropdown = Wait.WaitForClickable(_dropdown);
            var select = new SelectElement(dropdown);
            select.SelectByText(optionText);
            return this;
        }

        /// <summary>Select a filter option by value.</summary>
        public JobSearchPage SelectFilterByValue(string value)
        {
            var dropdown = Wait.WaitForClickable(_dropdown);
            var select = new SelectElement(dropdown);
            select.SelectByValue(value);
            return this;
        }

        /// <summary>Get the currently selected filter.</summary>
        public string GetSelectedFilter()
        {
            var dropdown = Wait.WaitForVisible(_dropdown);
            var select = new SelectElement(dropdown);
            return select.SelectedOption.Text;
        }

        /// <summary>Get all available filter options.</summary>
        public List<string> GetAllFilterOptions()
        {
            var options = Driver.FindElements(_dropdownOptions);
            return options.Select(o => o.Text).ToList();
        }

        /// <summary>Check if dropdown is displayed.</summary>
        public bool IsFilterDropdownDisplayed()
        {
            return IsElementPresent(_dropdown);
        }
    }
}
```

### File: Pages/ResumeUploadPage.cs

```csharp
using OpenQA.Selenium;

namespace JobPortalAutomation.Pages
{
    /// <summary>
    /// Resume Upload page — upload a file and verify it appears.
    /// Maps to: https://the-internet.herokuapp.com/upload
    /// </summary>
    public class ResumeUploadPage : BasePage
    {
        private readonly By _fileInput = By.Id("file-upload");
        private readonly By _uploadButton = By.Id("file-submit");
        private readonly By _uploadedFileName = By.Id("uploaded-files");
        private readonly By _dragDropArea = By.Id("drag-drop-upload");
        private readonly By _heading = By.TagName("h3");

        public ResumeUploadPage(IWebDriver driver) : base(driver) { }

        /// <summary>Navigate to the upload page.</summary>
        public ResumeUploadPage Open()
        {
            NavigateTo("/upload");
            Wait.WaitForVisible(_fileInput);
            return this;
        }

        /// <summary>
        /// Set the file path in the file input.
        /// Note: SendKeys to a file input sets the file path
        /// without opening a file dialog.
        /// </summary>
        public ResumeUploadPage SelectFile(string filePath)
        {
            var input = Driver.FindElement(_fileInput);
            input.SendKeys(filePath);
            return this;
        }

        /// <summary>Click the upload button.</summary>
        public ResumeUploadPage ClickUpload()
        {
            Wait.WaitForClickable(_uploadButton).Click();
            return this;
        }

        /// <summary>
        /// Upload a file end-to-end: select file and click upload.
        /// </summary>
        public ResumeUploadPage UploadFile(string filePath)
        {
            SelectFile(filePath);
            ClickUpload();
            return this;
        }

        /// <summary>Get the name of the uploaded file.</summary>
        public string GetUploadedFileName()
        {
            return Wait.WaitForVisible(_uploadedFileName).Text;
        }

        /// <summary>Get the page heading after upload.</summary>
        public string GetHeading()
        {
            return Wait.WaitForVisible(_heading).Text;
        }

        /// <summary>Check if upload was successful.</summary>
        public bool IsUploadSuccessful()
        {
            try
            {
                string heading = GetHeading();
                return heading.Contains("File Uploaded!");
            }
            catch
            {
                return false;
            }
        }
    }
}
```

### File: Pages/JobApplicationPage.cs

```csharp
using OpenQA.Selenium;

namespace JobPortalAutomation.Pages
{
    /// <summary>
    /// Job Application page — submit an application and confirm.
    /// Uses Dynamic Loading page to simulate async submission.
    /// Maps to: https://the-internet.herokuapp.com/dynamic_loading/1
    /// </summary>
    public class JobApplicationPage : BasePage
    {
        private readonly By _startButton =
            By.CssSelector("#start button");
        private readonly By _loadingIndicator = By.Id("loading");
        private readonly By _finishText =
            By.CssSelector("#finish h4");

        public JobApplicationPage(IWebDriver driver)
            : base(driver) { }

        /// <summary>Navigate to the application page.</summary>
        public JobApplicationPage Open()
        {
            NavigateTo("/dynamic_loading/1");
            Wait.WaitForVisible(_startButton);
            return this;
        }

        /// <summary>Click the submit/start button.</summary>
        public JobApplicationPage ClickSubmit()
        {
            Wait.WaitForClickable(_startButton).Click();
            return this;
        }

        /// <summary>Wait for submission to complete.</summary>
        public JobApplicationPage WaitForCompletion()
        {
            // Wait for loading indicator to disappear
            Wait.WaitForInvisible(_loadingIndicator);
            // Wait for result text to appear
            Wait.WaitForVisible(_finishText);
            return this;
        }

        /// <summary>Get the confirmation message.</summary>
        public string GetConfirmationMessage()
        {
            return Wait.WaitForVisible(_finishText).Text;
        }

        /// <summary>Full submit flow: click and wait.</summary>
        public string SubmitAndGetConfirmation()
        {
            ClickSubmit();
            WaitForCompletion();
            return GetConfirmationMessage();
        }

        /// <summary>Check if loading indicator is showing.</summary>
        public bool IsLoading()
        {
            return IsElementPresent(_loadingIndicator)
                && Driver.FindElement(_loadingIndicator).Displayed;
        }
    }
}
```

### File: Pages/AdminPage.cs

```csharp
using OpenQA.Selenium;
using System.Collections.ObjectModel;

namespace JobPortalAutomation.Pages
{
    /// <summary>
    /// Admin Panel — toggle candidate statuses.
    /// Uses Checkboxes page to simulate status toggles.
    /// Maps to: https://the-internet.herokuapp.com/checkboxes
    /// </summary>
    public class AdminPage : BasePage
    {
        private readonly By _checkboxes =
            By.CssSelector("input[type='checkbox']");
        private readonly By _heading = By.TagName("h3");

        public AdminPage(IWebDriver driver) : base(driver) { }

        /// <summary>Navigate to admin panel.</summary>
        public AdminPage Open()
        {
            NavigateTo("/checkboxes");
            Wait.WaitForVisible(_heading);
            return this;
        }

        /// <summary>Get all status checkboxes.</summary>
        public ReadOnlyCollection<IWebElement> GetAllCheckboxes()
        {
            return Driver.FindElements(_checkboxes);
        }

        /// <summary>Get checkbox count.</summary>
        public int GetCheckboxCount()
        {
            return Driver.FindElements(_checkboxes).Count;
        }

        /// <summary>Toggle a checkbox by index (0-based).</summary>
        public AdminPage ToggleCheckbox(int index)
        {
            var boxes = Driver.FindElements(_checkboxes);
            if (index < boxes.Count)
            {
                boxes[index].Click();
            }
            return this;
        }

        /// <summary>Check if a checkbox is selected.</summary>
        public bool IsCheckboxSelected(int index)
        {
            var boxes = Driver.FindElements(_checkboxes);
            return index < boxes.Count && boxes[index].Selected;
        }

        /// <summary>Set checkbox to a specific state.</summary>
        public AdminPage SetCheckboxState(int index, bool selected)
        {
            bool currentState = IsCheckboxSelected(index);
            if (currentState != selected)
            {
                ToggleCheckbox(index);
            }
            return this;
        }

        /// <summary>Get page heading.</summary>
        public string GetHeading()
        {
            return Wait.WaitForVisible(_heading).Text;
        }
    }
}
```

### File: Tests/BaseTest.cs

```csharp
using NUnit.Framework;
using NUnit.Framework.Interfaces;
using OpenQA.Selenium;
using JobPortalAutomation.Drivers;
using JobPortalAutomation.Utils;
using System;
using System.Threading;

// Enable parallel execution across the assembly
[assembly: LevelOfParallelism(4)]

namespace JobPortalAutomation.Tests
{
    /// <summary>
    /// Base class for all test fixtures.
    /// Handles driver lifecycle, screenshots, and common setup.
    /// </summary>
    public class BaseTest
    {
        // ThreadLocal for parallel execution safety
        private static readonly ThreadLocal<IWebDriver> ThreadDriver =
            new ThreadLocal<IWebDriver>();

        protected IWebDriver Driver
        {
            get => ThreadDriver.Value;
            private set => ThreadDriver.Value = value;
        }

        protected WaitHelper Wait { get; private set; }

        [SetUp]
        public virtual void SetUp()
        {
            Driver = DriverFactory.CreateDriver(
                browser: ConfigReader.Browser,
                headless: ConfigReader.Headless,
                gridUrl: string.IsNullOrEmpty(ConfigReader.GridUrl)
                    ? null : ConfigReader.GridUrl
            );

            Wait = new WaitHelper(Driver, ConfigReader.DefaultTimeout);

            Console.WriteLine(
                $"[{Thread.CurrentThread.ManagedThreadId}] " +
                $"Started: {TestContext.CurrentContext.Test.Name} " +
                $"({ConfigReader.Browser}" +
                $"{(ConfigReader.Headless ? " headless" : "")})"
            );
        }

        [TearDown]
        public virtual void TearDown()
        {
            var outcome = TestContext.CurrentContext.Result.Outcome;

            if (outcome.Status == TestStatus.Failed)
            {
                ScreenshotHelper.CaptureScreenshot(Driver, "FAILED");
                Console.WriteLine(
                    $"FAILED: {TestContext.CurrentContext.Test.Name}"
                );
                Console.WriteLine(
                    $"  Error: {outcome.Message}"
                );
            }

            Driver?.Quit();
            Driver = null;

            Console.WriteLine(
                $"[{Thread.CurrentThread.ManagedThreadId}] " +
                $"Finished: {TestContext.CurrentContext.Test.Name}"
            );
        }
    }
}
```

### File: Tests/LoginTests.cs

```csharp
using NUnit.Framework;
using JobPortalAutomation.Pages;
using JobPortalAutomation.Utils;

namespace JobPortalAutomation.Tests
{
    [TestFixture]
    [Parallelizable(ParallelScope.All)]
    public class LoginTests : BaseTest
    {
        [Test]
        public void Login_WithValidCredentials_RedirectsToDashboard()
        {
            var loginPage = new LoginPage(Driver).Open();

            var dashboard = loginPage.LoginAs(
                ConfigReader.ValidUsername,
                ConfigReader.ValidPassword
            );

            dashboard.WaitForLoad();

            Assert.That(dashboard.GetHeading(),
                Does.Contain("Secure Area"));
            Assert.That(dashboard.GetFlashMessage(),
                Does.Contain("You logged into"));
            Assert.That(dashboard.IsLogoutDisplayed(), Is.True);
        }

        [Test]
        public void Login_WithInvalidPassword_ShowsError()
        {
            var loginPage = new LoginPage(Driver).Open();

            loginPage.LoginExpectingFailure(
                ConfigReader.ValidUsername,
                "WrongPassword123!"
            );

            string message = loginPage.GetFlashMessage();
            Assert.That(message,
                Does.Contain("Your password is invalid"));
        }

        [Test]
        public void Login_WithInvalidUsername_ShowsError()
        {
            var loginPage = new LoginPage(Driver).Open();

            loginPage.LoginExpectingFailure(
                "nonexistentuser",
                ConfigReader.ValidPassword
            );

            string message = loginPage.GetFlashMessage();
            Assert.That(message,
                Does.Contain("Your username is invalid"));
        }

        [Test]
        public void Login_WithEmptyFields_ShowsError()
        {
            var loginPage = new LoginPage(Driver).Open();

            loginPage.ClickLogin();

            string message = loginPage.GetFlashMessage();
            Assert.That(message,
                Does.Contain("Your username is invalid"));
        }

        [Test]
        public void Login_ThenLogout_ReturnsToLoginPage()
        {
            var loginPage = new LoginPage(Driver).Open();

            var dashboard = loginPage.LoginAs(
                ConfigReader.ValidUsername,
                ConfigReader.ValidPassword
            );
            dashboard.WaitForLoad();

            var returnedLoginPage = dashboard.Logout();

            Assert.That(returnedLoginPage.GetFlashMessage(),
                Does.Contain("You logged out"));
            Assert.That(returnedLoginPage.IsUsernameFieldDisplayed(),
                Is.True);
        }
    }
}
```

### File: Tests/JobSearchTests.cs

```csharp
using NUnit.Framework;
using JobPortalAutomation.Pages;

namespace JobPortalAutomation.Tests
{
    [TestFixture]
    [Parallelizable(ParallelScope.All)]
    public class JobSearchTests : BaseTest
    {
        [Test]
        public void Search_FilterDropdownLoads_AllOptionsAvailable()
        {
            var searchPage = new JobSearchPage(Driver).Open();

            Assert.That(searchPage.IsFilterDropdownDisplayed(), Is.True);

            var options = searchPage.GetAllFilterOptions();
            Assert.That(options.Count, Is.GreaterThanOrEqualTo(2),
                "Should have at least 2 filter options");
        }

        [Test]
        public void Search_SelectFilter_ShowsSelectedOption()
        {
            var searchPage = new JobSearchPage(Driver).Open();

            searchPage.SelectFilter("Option 1");

            string selected = searchPage.GetSelectedFilter();
            Assert.That(selected, Is.EqualTo("Option 1"));
        }

        [Test]
        public void Search_SelectDifferentFilter_UpdatesSelection()
        {
            var searchPage = new JobSearchPage(Driver).Open();

            // Select first filter
            searchPage.SelectFilter("Option 1");
            Assert.That(searchPage.GetSelectedFilter(),
                Is.EqualTo("Option 1"));

            // Change to second filter
            searchPage.SelectFilter("Option 2");
            Assert.That(searchPage.GetSelectedFilter(),
                Is.EqualTo("Option 2"));
        }

        [Test]
        public void Search_SelectByValue_WorksCorrectly()
        {
            var searchPage = new JobSearchPage(Driver).Open();

            searchPage.SelectFilterByValue("1");

            string selected = searchPage.GetSelectedFilter();
            Assert.That(selected, Is.EqualTo("Option 1"));
        }
    }
}
```

### File: Tests/ResumeUploadTests.cs

```csharp
using NUnit.Framework;
using JobPortalAutomation.Pages;
using System.IO;

namespace JobPortalAutomation.Tests
{
    [TestFixture]
    [Parallelizable(ParallelScope.All)]
    public class ResumeUploadTests : BaseTest
    {
        private string _testFilePath;

        [SetUp]
        public override void SetUp()
        {
            base.SetUp();

            // Create a temporary test file for upload
            _testFilePath = Path.Combine(
                Path.GetTempPath(), "test_resume.txt"
            );
            File.WriteAllText(_testFilePath,
                "Test resume content for automation testing."
            );
        }

        [TearDown]
        public override void TearDown()
        {
            // Clean up temp file
            if (File.Exists(_testFilePath))
                File.Delete(_testFilePath);

            base.TearDown();
        }

        [Test]
        public void Upload_ValidFile_ShowsSuccessMessage()
        {
            var uploadPage = new ResumeUploadPage(Driver).Open();

            uploadPage.UploadFile(_testFilePath);

            Assert.That(uploadPage.IsUploadSuccessful(), Is.True);
            Assert.That(uploadPage.GetHeading(),
                Is.EqualTo("File Uploaded!"));
        }

        [Test]
        public void Upload_ValidFile_ShowsFileName()
        {
            var uploadPage = new ResumeUploadPage(Driver).Open();

            uploadPage.UploadFile(_testFilePath);

            string uploadedName = uploadPage.GetUploadedFileName();
            Assert.That(uploadedName,
                Does.Contain("test_resume.txt"));
        }
    }
}
```

### File: Tests/JobApplicationTests.cs

```csharp
using NUnit.Framework;
using JobPortalAutomation.Pages;

namespace JobPortalAutomation.Tests
{
    [TestFixture]
    [Parallelizable(ParallelScope.All)]
    public class JobApplicationTests : BaseTest
    {
        [Test]
        public void Application_Submit_ShowsConfirmation()
        {
            var appPage = new JobApplicationPage(Driver).Open();

            string confirmation = appPage.SubmitAndGetConfirmation();

            Assert.That(confirmation, Is.Not.Empty,
                "Should show a confirmation message");
            Assert.That(confirmation, Does.Contain("Hello World!"));
        }

        [Test]
        public void Application_ShowsLoadingDuringSubmission()
        {
            var appPage = new JobApplicationPage(Driver).Open();

            appPage.ClickSubmit();

            // The loading indicator should appear briefly
            // Then disappear when done
            appPage.WaitForCompletion();

            string result = appPage.GetConfirmationMessage();
            Assert.That(result, Is.Not.Empty);
        }
    }
}
```

### File: Tests/AdminWorkflowTests.cs

```csharp
using NUnit.Framework;
using JobPortalAutomation.Pages;

namespace JobPortalAutomation.Tests
{
    [TestFixture]
    [Parallelizable(ParallelScope.All)]
    public class AdminWorkflowTests : BaseTest
    {
        [Test]
        public void Admin_PageLoads_ShowsCheckboxes()
        {
            var adminPage = new AdminPage(Driver).Open();

            Assert.That(adminPage.GetHeading(),
                Does.Contain("Checkboxes"));
            Assert.That(adminPage.GetCheckboxCount(),
                Is.GreaterThan(0));
        }

        [Test]
        public void Admin_ToggleStatus_ChangesCheckboxState()
        {
            var adminPage = new AdminPage(Driver).Open();

            bool initialState = adminPage.IsCheckboxSelected(0);

            adminPage.ToggleCheckbox(0);

            bool newState = adminPage.IsCheckboxSelected(0);
            Assert.That(newState, Is.Not.EqualTo(initialState),
                "Checkbox state should have toggled");
        }

        [Test]
        public void Admin_SetStatusApproved_CheckboxIsSelected()
        {
            var adminPage = new AdminPage(Driver).Open();

            // Set first checkbox to "approved" (checked)
            adminPage.SetCheckboxState(0, true);

            Assert.That(adminPage.IsCheckboxSelected(0), Is.True,
                "Checkbox should be selected (approved)");
        }

        [Test]
        public void Admin_SetStatusRejected_CheckboxIsDeselected()
        {
            var adminPage = new AdminPage(Driver).Open();

            // Set first checkbox to "rejected" (unchecked)
            adminPage.SetCheckboxState(0, false);

            Assert.That(adminPage.IsCheckboxSelected(0), Is.False,
                "Checkbox should be deselected (rejected)");
        }

        [Test]
        public void Admin_MultipleStatusChanges_AllPersist()
        {
            var adminPage = new AdminPage(Driver).Open();

            // Set both checkboxes to specific states
            adminPage.SetCheckboxState(0, true);
            adminPage.SetCheckboxState(1, false);

            Assert.That(adminPage.IsCheckboxSelected(0), Is.True);
            Assert.That(adminPage.IsCheckboxSelected(1), Is.False);
        }
    }
}
```

### File: .github/workflows/selenium-tests.yml

```yaml
name: JobPortal Selenium Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    strategy:
      fail-fast: false
      matrix:
        browser: [chrome]

    steps:
      - uses: actions/checkout@v4

      - name: Set up .NET 8
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Verify Chrome
        run: google-chrome --version

      - name: Restore dependencies
        run: dotnet restore
        working-directory: ./JobPortalAutomation.Tests

      - name: Build
        run: dotnet build --no-restore
        working-directory: ./JobPortalAutomation.Tests

      - name: Run tests
        run: |
          dotnet test --no-build \
            --verbosity normal \
            --logger "trx;LogFileName=results.trx" \
            --results-directory ./TestResults \
            -- NUnit.NumberOfTestWorkers=4
        working-directory: ./JobPortalAutomation.Tests
        env:
          SELENIUM_HEADLESS: "true"
          BROWSER: ${{ matrix.browser }}

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ matrix.browser }}
          path: ./JobPortalAutomation.Tests/TestResults/

      - name: Upload failure screenshots
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: screenshots-${{ matrix.browser }}
          path: ./JobPortalAutomation.Tests/Reports/Screenshots/
```

---

## 3. Python Complete Framework

### Folder Structure

```
job_portal_automation/
├── .gitignore
├── requirements.txt
├── pytest.ini
├── conftest.py
│
├── .github/
│   └── workflows/
│       └── selenium-tests.yml
│
├── config/
│   ├── __init__.py
│   ├── settings.py
│   └── test_data.json
│
├── drivers/
│   ├── __init__.py
│   └── driver_factory.py
│
├── pages/
│   ├── __init__.py
│   ├── base_page.py
│   ├── login_page.py
│   ├── dashboard_page.py
│   ├── job_search_page.py
│   ├── resume_upload_page.py
│   ├── job_application_page.py
│   └── admin_page.py
│
├── tests/
│   ├── __init__.py
│   ├── test_login.py
│   ├── test_job_search.py
│   ├── test_resume_upload.py
│   ├── test_job_application.py
│   └── test_admin_workflow.py
│
├── utils/
│   ├── __init__.py
│   ├── wait_helper.py
│   ├── screenshot_helper.py
│   └── test_data_generator.py
│
└── reports/
    └── .gitkeep
```

### File: requirements.txt

```
selenium==4.18.1
pytest==8.1.1
pytest-html==4.1.1
pytest-xdist==3.5.0
```

### File: pytest.ini

```ini
[pytest]
testpaths = tests
addopts = -v --tb=short
markers =
    smoke: Quick smoke tests
    regression: Full regression suite
```

### File: config/settings.py

```python
"""Central configuration. Environment variables override defaults."""
import os
import json


class Settings:
    BASE_URL = os.environ.get(
        "BASE_URL", "https://the-internet.herokuapp.com"
    )
    BROWSER = os.environ.get("BROWSER", "chrome")
    HEADLESS = os.environ.get(
        "SELENIUM_HEADLESS", "false"
    ).lower() in ("true", "1", "yes")
    DEFAULT_TIMEOUT = int(os.environ.get("DEFAULT_TIMEOUT", "10"))
    GRID_URL = os.environ.get("SELENIUM_GRID_URL", "")

    # Paths
    PROJECT_ROOT = os.path.dirname(os.path.dirname(__file__))
    SCREENSHOT_DIR = os.path.join(PROJECT_ROOT, "reports", "screenshots")
    CONFIG_DIR = os.path.dirname(__file__)

    # Credentials
    VALID_USERNAME = os.environ.get("VALID_USERNAME", "tomsmith")
    VALID_PASSWORD = os.environ.get(
        "VALID_PASSWORD", "SuperSecretPassword!"
    )

    @classmethod
    def load_test_data(cls):
        path = os.path.join(cls.CONFIG_DIR, "test_data.json")
        if os.path.exists(path):
            with open(path, "r") as f:
                return json.load(f)
        return {}
```

### File: config/test_data.json

```json
{
    "valid_user": {
        "username": "tomsmith",
        "password": "SuperSecretPassword!"
    },
    "invalid_user": {
        "username": "wronguser",
        "password": "wrongpass"
    },
    "job_filters": ["Option 1", "Option 2"],
    "test_resume_content": "Test resume for automation."
}
```

### File: drivers/driver_factory.py

```python
"""Browser creation factory supporting local and Grid execution."""
from selenium import webdriver
from selenium.webdriver.chrome.options import Options as ChromeOptions
from selenium.webdriver.firefox.options import Options as FirefoxOptions
from selenium.webdriver.edge.options import Options as EdgeOptions


class DriverFactory:

    @staticmethod
    def create_driver(browser="chrome", headless=False, grid_url=None):
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
                    command_executor=grid_url, options=options
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
                    command_executor=grid_url, options=options
                )
            return webdriver.Firefox(options=options)

        elif browser == "edge":
            options = EdgeOptions()
            options.add_argument("--window-size=1920,1080")
            if headless:
                options.add_argument("--headless=new")
            if grid_url:
                return webdriver.Remote(
                    command_executor=grid_url, options=options
                )
            return webdriver.Edge(options=options)

        raise ValueError(f"Unsupported browser: {browser}")
```

### File: utils/wait_helper.py

```python
"""Reusable explicit wait methods. Never use time.sleep()."""
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import (
    NoSuchElementException,
    StaleElementReferenceException,
)


class WaitHelper:

    def __init__(self, driver, timeout=10):
        self.driver = driver
        self.wait = WebDriverWait(
            driver, timeout,
            ignored_exceptions=[
                NoSuchElementException,
                StaleElementReferenceException,
            ]
        )

    def wait_for_visible(self, locator):
        return self.wait.until(
            EC.visibility_of_element_located(locator)
        )

    def wait_for_clickable(self, locator):
        return self.wait.until(
            EC.element_to_be_clickable(locator)
        )

    def wait_for_invisible(self, locator):
        return self.wait.until(
            EC.invisibility_of_element_located(locator)
        )

    def wait_for_url_contains(self, url_part):
        return self.wait.until(EC.url_contains(url_part))

    def wait_for_title_contains(self, title_part):
        return self.wait.until(EC.title_contains(title_part))

    def wait_for_text_present(self, locator, text):
        return self.wait.until(
            EC.text_to_be_present_in_element(locator, text)
        )

    def wait_for_element_count(self, locator, min_count):
        return self.wait.until(
            lambda d: len(d.find_elements(*locator)) >= min_count
        )
```

### File: utils/screenshot_helper.py

```python
"""Screenshot capture with organized naming."""
import os
from datetime import datetime
from config.settings import Settings


class ScreenshotHelper:

    @staticmethod
    def capture(driver, name="screenshot"):
        os.makedirs(Settings.SCREENSHOT_DIR, exist_ok=True)
        timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
        # Sanitize the name for safe filenames
        safe_name = "".join(
            c if c.isalnum() or c in "-_" else "_" for c in name
        )
        filename = f"{timestamp}_{safe_name}.png"
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
        os.makedirs(Settings.SCREENSHOT_DIR, exist_ok=True)
        timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
        filename = f"{timestamp}_{name}_element.png"
        filepath = os.path.join(Settings.SCREENSHOT_DIR, filename)
        try:
            element.screenshot(filepath)
            return filepath
        except Exception as e:
            print(f"Element screenshot failed: {e}")
            return None
```

### File: utils/test_data_generator.py

```python
"""Generate unique test data for parallel test safety."""
import uuid
import random


class TestDataGenerator:

    @staticmethod
    def unique_email():
        uid = uuid.uuid4().hex[:8]
        return f"testuser_{uid}@test.com"

    @staticmethod
    def unique_name():
        uid = uuid.uuid4().hex[:6]
        return f"TestUser_{uid}"

    @staticmethod
    def unique_job_title():
        titles = [
            "Software Engineer", "QA Analyst",
            "DevOps Engineer", "Product Manager",
            "Data Scientist", "UX Designer",
        ]
        num = random.randint(1000, 9999)
        return f"{random.choice(titles)} #{num}"

    @staticmethod
    def unique_filename():
        uid = uuid.uuid4().hex[:8]
        return f"resume_{uid}.pdf"
```

### File: pages/base_page.py

```python
"""Base class for all page objects."""
from selenium.webdriver.common.by import By
from selenium.common.exceptions import NoSuchElementException
from utils.wait_helper import WaitHelper
from config.settings import Settings


class BasePage:
    """
    Common functionality for all pages.
    Page objects contain NO assertions — only interactions.
    """

    def __init__(self, driver):
        self.driver = driver
        self.wait = WaitHelper(driver, Settings.DEFAULT_TIMEOUT)
        self.base_url = Settings.BASE_URL

    @property
    def current_url(self):
        return self.driver.current_url

    @property
    def page_title(self):
        return self.driver.title

    def navigate_to(self, path):
        self.driver.get(f"{self.base_url}{path}")

    def scroll_to(self, element):
        self.driver.execute_script(
            "arguments[0].scrollIntoView({block: 'center'});",
            element
        )

    def js_click(self, element):
        self.driver.execute_script(
            "arguments[0].click();", element
        )

    def is_element_present(self, locator):
        try:
            self.driver.find_element(*locator)
            return True
        except NoSuchElementException:
            return False

    def get_element_text(self, locator):
        try:
            return self.driver.find_element(*locator).text
        except NoSuchElementException:
            return ""
```

### File: pages/login_page.py

```python
"""Login page object."""
from selenium.webdriver.common.by import By
from pages.base_page import BasePage
from pages.dashboard_page import DashboardPage


class LoginPage(BasePage):
    """Login page — handles user authentication."""

    # Locators
    USERNAME_FIELD = (By.ID, "username")
    PASSWORD_FIELD = (By.ID, "password")
    LOGIN_BUTTON = (By.CSS_SELECTOR, "button[type='submit']")
    FLASH_MESSAGE = (By.ID, "flash")
    PAGE_HEADING = (By.TAG_NAME, "h2")

    def open(self):
        self.navigate_to("/login")
        self.wait.wait_for_visible(self.USERNAME_FIELD)
        return self

    def enter_username(self, username):
        field = self.wait.wait_for_visible(self.USERNAME_FIELD)
        field.clear()
        field.send_keys(username)
        return self

    def enter_password(self, password):
        field = self.wait.wait_for_visible(self.PASSWORD_FIELD)
        field.clear()
        field.send_keys(password)
        return self

    def click_login(self):
        self.wait.wait_for_clickable(self.LOGIN_BUTTON).click()

    def login_as(self, username, password):
        """Full login flow. Returns DashboardPage."""
        self.enter_username(username)
        self.enter_password(password)
        self.click_login()
        return DashboardPage(self.driver)

    def login_expecting_failure(self, username, password):
        """Login expected to fail. Returns self."""
        self.enter_username(username)
        self.enter_password(password)
        self.click_login()
        return self

    def get_flash_message(self):
        flash = self.wait.wait_for_visible(self.FLASH_MESSAGE)
        return flash.text

    def get_heading(self):
        return self.wait.wait_for_visible(self.PAGE_HEADING).text

    def is_username_field_displayed(self):
        return self.is_element_present(self.USERNAME_FIELD)
```

### File: pages/dashboard_page.py

```python
"""Dashboard / Secure Area page object."""
from selenium.webdriver.common.by import By
from pages.base_page import BasePage


class DashboardPage(BasePage):
    """Shown after successful login."""

    HEADING = (By.TAG_NAME, "h2")
    SUBHEADING = (By.TAG_NAME, "h4")
    FLASH_MESSAGE = (By.ID, "flash")
    LOGOUT_BUTTON = (By.CSS_SELECTOR, "a[href='/logout']")

    def wait_for_load(self):
        self.wait.wait_for_url_contains("/secure")
        self.wait.wait_for_visible(self.HEADING)
        return self

    def get_heading(self):
        return self.wait.wait_for_visible(self.HEADING).text

    def get_flash_message(self):
        return self.wait.wait_for_visible(self.FLASH_MESSAGE).text

    def is_logout_displayed(self):
        return self.is_element_present(self.LOGOUT_BUTTON)

    def logout(self):
        self.wait.wait_for_clickable(self.LOGOUT_BUTTON).click()
        from pages.login_page import LoginPage
        return LoginPage(self.driver)
```

### File: pages/job_search_page.py

```python
"""Job Search page object (uses dropdown page)."""
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select
from pages.base_page import BasePage


class JobSearchPage(BasePage):
    """Search with filters using dropdown selections."""

    DROPDOWN = (By.ID, "dropdown")
    DROPDOWN_OPTIONS = (By.CSS_SELECTOR, "#dropdown option")

    def open(self):
        self.navigate_to("/dropdown")
        self.wait.wait_for_visible(self.DROPDOWN)
        return self

    def select_filter(self, option_text):
        dropdown = self.wait.wait_for_clickable(self.DROPDOWN)
        select = Select(dropdown)
        select.select_by_visible_text(option_text)
        return self

    def select_filter_by_value(self, value):
        dropdown = self.wait.wait_for_clickable(self.DROPDOWN)
        select = Select(dropdown)
        select.select_by_value(value)
        return self

    def get_selected_filter(self):
        dropdown = self.wait.wait_for_visible(self.DROPDOWN)
        select = Select(dropdown)
        return select.first_selected_option.text

    def get_all_filter_options(self):
        options = self.driver.find_elements(*self.DROPDOWN_OPTIONS)
        return [o.text for o in options]

    def is_filter_dropdown_displayed(self):
        return self.is_element_present(self.DROPDOWN)
```

### File: pages/resume_upload_page.py

```python
"""Resume Upload page object."""
from selenium.webdriver.common.by import By
from pages.base_page import BasePage


class ResumeUploadPage(BasePage):
    """Upload a file and verify it appears."""

    FILE_INPUT = (By.ID, "file-upload")
    UPLOAD_BUTTON = (By.ID, "file-submit")
    UPLOADED_FILENAME = (By.ID, "uploaded-files")
    HEADING = (By.TAG_NAME, "h3")

    def open(self):
        self.navigate_to("/upload")
        self.wait.wait_for_visible(self.FILE_INPUT)
        return self

    def select_file(self, file_path):
        file_input = self.driver.find_element(*self.FILE_INPUT)
        file_input.send_keys(file_path)
        return self

    def click_upload(self):
        self.wait.wait_for_clickable(self.UPLOAD_BUTTON).click()
        return self

    def upload_file(self, file_path):
        self.select_file(file_path)
        self.click_upload()
        return self

    def get_uploaded_filename(self):
        return self.wait.wait_for_visible(self.UPLOADED_FILENAME).text

    def get_heading(self):
        return self.wait.wait_for_visible(self.HEADING).text

    def is_upload_successful(self):
        try:
            return "File Uploaded!" in self.get_heading()
        except Exception:
            return False
```

### File: pages/job_application_page.py

```python
"""Job Application page object (uses Dynamic Loading)."""
from selenium.webdriver.common.by import By
from pages.base_page import BasePage


class JobApplicationPage(BasePage):
    """Submit application and wait for confirmation."""

    START_BUTTON = (By.CSS_SELECTOR, "#start button")
    LOADING_INDICATOR = (By.ID, "loading")
    FINISH_TEXT = (By.CSS_SELECTOR, "#finish h4")

    def open(self):
        self.navigate_to("/dynamic_loading/1")
        self.wait.wait_for_visible(self.START_BUTTON)
        return self

    def click_submit(self):
        self.wait.wait_for_clickable(self.START_BUTTON).click()
        return self

    def wait_for_completion(self):
        self.wait.wait_for_invisible(self.LOADING_INDICATOR)
        self.wait.wait_for_visible(self.FINISH_TEXT)
        return self

    def get_confirmation_message(self):
        return self.wait.wait_for_visible(self.FINISH_TEXT).text

    def submit_and_get_confirmation(self):
        self.click_submit()
        self.wait_for_completion()
        return self.get_confirmation_message()

    def is_loading(self):
        return self.is_element_present(self.LOADING_INDICATOR)
```

### File: pages/admin_page.py

```python
"""Admin Panel page object (uses Checkboxes page)."""
from selenium.webdriver.common.by import By
from pages.base_page import BasePage


class AdminPage(BasePage):
    """Toggle candidate statuses via checkboxes."""

    CHECKBOXES = (By.CSS_SELECTOR, "input[type='checkbox']")
    HEADING = (By.TAG_NAME, "h3")

    def open(self):
        self.navigate_to("/checkboxes")
        self.wait.wait_for_visible(self.HEADING)
        return self

    def get_all_checkboxes(self):
        return self.driver.find_elements(*self.CHECKBOXES)

    def get_checkbox_count(self):
        return len(self.driver.find_elements(*self.CHECKBOXES))

    def toggle_checkbox(self, index):
        boxes = self.driver.find_elements(*self.CHECKBOXES)
        if index < len(boxes):
            boxes[index].click()
        return self

    def is_checkbox_selected(self, index):
        boxes = self.driver.find_elements(*self.CHECKBOXES)
        return index < len(boxes) and boxes[index].is_selected()

    def set_checkbox_state(self, index, selected):
        current = self.is_checkbox_selected(index)
        if current != selected:
            self.toggle_checkbox(index)
        return self

    def get_heading(self):
        return self.wait.wait_for_visible(self.HEADING).text
```

### File: conftest.py (root)

```python
"""Global fixtures and hooks for the test framework."""
import os
import pytest
from datetime import datetime
from drivers.driver_factory import DriverFactory
from config.settings import Settings
from utils.wait_helper import WaitHelper


@pytest.fixture(scope="function")
def driver():
    """Create a browser driver for each test function."""
    drv = DriverFactory.create_driver(
        browser=Settings.BROWSER,
        headless=Settings.HEADLESS,
        grid_url=Settings.GRID_URL or None
    )
    yield drv
    drv.quit()


@pytest.fixture
def wait(driver):
    """Explicit wait helper bound to the driver."""
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
            safe_name = "".join(
                c if c.isalnum() or c in "-_" else "_"
                for c in item.name
            )
            filepath = os.path.join(
                Settings.SCREENSHOT_DIR,
                f"{timestamp}_{safe_name}_FAILED.png"
            )
            try:
                driver.save_screenshot(filepath)
                print(f"\nFailure screenshot: {filepath}")
            except Exception:
                pass
```

### File: tests/test_login.py

```python
"""Login page tests."""
import pytest
from pages.login_page import LoginPage
from config.settings import Settings


class TestLogin:

    @pytest.mark.smoke
    def test_valid_login_redirects_to_dashboard(self, driver):
        login_page = LoginPage(driver).open()

        dashboard = login_page.login_as(
            Settings.VALID_USERNAME,
            Settings.VALID_PASSWORD
        )
        dashboard.wait_for_load()

        assert "Secure Area" in dashboard.get_heading()
        assert "You logged into" in dashboard.get_flash_message()
        assert dashboard.is_logout_displayed()

    def test_invalid_password_shows_error(self, driver):
        login_page = LoginPage(driver).open()

        login_page.login_expecting_failure(
            Settings.VALID_USERNAME,
            "WrongPassword123!"
        )

        message = login_page.get_flash_message()
        assert "Your password is invalid" in message

    def test_invalid_username_shows_error(self, driver):
        login_page = LoginPage(driver).open()

        login_page.login_expecting_failure(
            "nonexistentuser",
            Settings.VALID_PASSWORD
        )

        message = login_page.get_flash_message()
        assert "Your username is invalid" in message

    def test_empty_fields_shows_error(self, driver):
        login_page = LoginPage(driver).open()
        login_page.click_login()

        message = login_page.get_flash_message()
        assert "Your username is invalid" in message

    @pytest.mark.smoke
    def test_login_then_logout_returns_to_login(self, driver):
        login_page = LoginPage(driver).open()

        dashboard = login_page.login_as(
            Settings.VALID_USERNAME,
            Settings.VALID_PASSWORD
        )
        dashboard.wait_for_load()

        returned_login = dashboard.logout()

        assert "You logged out" in returned_login.get_flash_message()
        assert returned_login.is_username_field_displayed()
```

### File: tests/test_job_search.py

```python
"""Job search / filter tests."""
import pytest
from pages.job_search_page import JobSearchPage


class TestJobSearch:

    def test_filter_dropdown_loads(self, driver):
        search_page = JobSearchPage(driver).open()

        assert search_page.is_filter_dropdown_displayed()
        options = search_page.get_all_filter_options()
        assert len(options) >= 2

    def test_select_filter_shows_selected(self, driver):
        search_page = JobSearchPage(driver).open()
        search_page.select_filter("Option 1")

        assert search_page.get_selected_filter() == "Option 1"

    def test_change_filter_updates_selection(self, driver):
        search_page = JobSearchPage(driver).open()

        search_page.select_filter("Option 1")
        assert search_page.get_selected_filter() == "Option 1"

        search_page.select_filter("Option 2")
        assert search_page.get_selected_filter() == "Option 2"

    def test_select_by_value(self, driver):
        search_page = JobSearchPage(driver).open()
        search_page.select_filter_by_value("1")

        assert search_page.get_selected_filter() == "Option 1"
```

### File: tests/test_resume_upload.py

```python
"""Resume upload tests."""
import os
import tempfile
import pytest
from pages.resume_upload_page import ResumeUploadPage


@pytest.fixture
def test_file():
    """Create a temporary file for upload testing."""
    path = os.path.join(tempfile.gettempdir(), "test_resume.txt")
    with open(path, "w") as f:
        f.write("Test resume content for automation.")
    yield path
    if os.path.exists(path):
        os.remove(path)


class TestResumeUpload:

    def test_upload_valid_file_shows_success(self, driver, test_file):
        upload_page = ResumeUploadPage(driver).open()
        upload_page.upload_file(test_file)

        assert upload_page.is_upload_successful()
        assert upload_page.get_heading() == "File Uploaded!"

    def test_upload_valid_file_shows_filename(self, driver, test_file):
        upload_page = ResumeUploadPage(driver).open()
        upload_page.upload_file(test_file)

        assert "test_resume.txt" in upload_page.get_uploaded_filename()
```

### File: tests/test_job_application.py

```python
"""Job application submission tests."""
import pytest
from pages.job_application_page import JobApplicationPage


class TestJobApplication:

    @pytest.mark.smoke
    def test_submit_shows_confirmation(self, driver):
        app_page = JobApplicationPage(driver).open()
        confirmation = app_page.submit_and_get_confirmation()

        assert confirmation, "Should show a confirmation message"
        assert "Hello World!" in confirmation

    def test_loading_appears_during_submission(self, driver):
        app_page = JobApplicationPage(driver).open()
        app_page.click_submit()
        app_page.wait_for_completion()

        result = app_page.get_confirmation_message()
        assert result
```

### File: tests/test_admin_workflow.py

```python
"""Admin panel / status workflow tests."""
import pytest
from pages.admin_page import AdminPage


class TestAdminWorkflow:

    def test_page_loads_with_checkboxes(self, driver):
        admin_page = AdminPage(driver).open()

        assert "Checkboxes" in admin_page.get_heading()
        assert admin_page.get_checkbox_count() > 0

    def test_toggle_status_changes_state(self, driver):
        admin_page = AdminPage(driver).open()
        initial = admin_page.is_checkbox_selected(0)

        admin_page.toggle_checkbox(0)

        assert admin_page.is_checkbox_selected(0) != initial

    def test_set_status_approved(self, driver):
        admin_page = AdminPage(driver).open()
        admin_page.set_checkbox_state(0, True)

        assert admin_page.is_checkbox_selected(0) is True

    def test_set_status_rejected(self, driver):
        admin_page = AdminPage(driver).open()
        admin_page.set_checkbox_state(0, False)

        assert admin_page.is_checkbox_selected(0) is False

    def test_multiple_status_changes_persist(self, driver):
        admin_page = AdminPage(driver).open()

        admin_page.set_checkbox_state(0, True)
        admin_page.set_checkbox_state(1, False)

        assert admin_page.is_checkbox_selected(0) is True
        assert admin_page.is_checkbox_selected(1) is False
```

### File: .github/workflows/selenium-tests.yml (Python)

```yaml
name: JobPortal Selenium Tests (Python)

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Verify Chrome
        run: google-chrome --version

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: |
          pytest tests/ \
            -v --tb=short \
            --junitxml=reports/results.xml \
            -n auto
        env:
          SELENIUM_HEADLESS: "true"

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: reports/

      - name: Upload failure screenshots
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: failure-screenshots
          path: reports/screenshots/
```

### File: .gitignore

```
# Python
__pycache__/
*.py[cod]
*.egg-info/
.eggs/
dist/
build/

# Virtual environment
venv/
.venv/

# IDE
.idea/
.vscode/
*.suo
*.user
*.vs/

# .NET
bin/
obj/
*.nupkg

# Test outputs
reports/screenshots/
reports/*.html
reports/*.xml
TestResults/
allure-results/

# Selenium
*.log
geckodriver.log
chromedriver.log

# OS
.DS_Store
Thumbs.db

# Environment
.env
```

---

## 4. Running the Framework

### C# Commands

```bash
# Restore and build
cd JobPortalAutomation.Tests
dotnet restore
dotnet build

# Run all tests
dotnet test -v normal

# Run headless
SELENIUM_HEADLESS=true dotnet test

# Run with parallel workers
dotnet test -- NUnit.NumberOfTestWorkers=4

# Run specific test class
dotnet test --filter "FullyQualifiedName~LoginTests"

# Run smoke tests only
dotnet test --filter "TestCategory=Smoke"
```

### Python Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Run all tests
pytest

# Run headless
SELENIUM_HEADLESS=true pytest

# Run in parallel
pytest -n auto

# Run specific test file
pytest tests/test_login.py

# Run smoke tests only
pytest -m smoke

# Run with HTML report
pytest --html=reports/report.html --self-contained-html

# Run on Firefox
BROWSER=firefox pytest
```

---

## 5. Framework Summary

### What This Framework Includes

| Feature | Implementation |
|---------|---------------|
| **Page Object Model** | 6 page objects with clean locator separation |
| **Config-driven** | JSON config + environment variable overrides |
| **Parallel execution** | ThreadLocal (C#) / function-scoped fixtures (Python) |
| **Screenshot on failure** | Auto-capture in teardown/hook |
| **Explicit waits only** | WaitHelper class, no Thread.Sleep |
| **Multi-browser** | DriverFactory supports Chrome, Firefox, Edge |
| **Grid support** | RemoteWebDriver via config |
| **CI/CD** | GitHub Actions workflow with artifacts |
| **Test isolation** | Each test gets its own browser |
| **Reusable utilities** | WaitHelper, ScreenshotHelper, ConfigReader |

### Test Count

| Test File | Test Count |
|-----------|-----------|
| LoginTests | 5 |
| JobSearchTests | 4 |
| ResumeUploadTests | 2 |
| JobApplicationTests | 2 |
| AdminWorkflowTests | 5 |
| **Total** | **18 tests** |

### Next Steps

To adapt this framework for your real application:

1. **Replace URLs** in `Settings`/`appsettings.json` with your application's URLs
2. **Update locators** in page objects to match your application's elements
3. **Add more page objects** as you automate new pages
4. **Add more test classes** following the same patterns
5. **Configure CI/CD** with your actual GitHub repository
6. **Add Allure/ExtentReports** for rich HTML reports
7. **Add API helpers** for test data setup/teardown

---

## 6. Resources

- [Selenium Documentation](https://www.selenium.dev/documentation/)
- [NUnit Documentation](https://docs.nunit.org/)
- [pytest Documentation](https://docs.pytest.org/)
- [Page Object Model Best Practices](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Docker Selenium](https://github.com/SeleniumHQ/docker-selenium)
- [The Internet - Test App](https://the-internet.herokuapp.com/)
