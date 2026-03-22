# Selenium Quick Reference Cheat Sheet

> Side-by-side C# (NUnit) and Python (PyTest) syntax for every common operation.

---

## 1. Locator Strategies

| Strategy | C# | Python |
|----------|-----|--------|
| **ID** | `driver.FindElement(By.Id("myId"))` | `driver.find_element(By.ID, "myId")` |
| **Name** | `driver.FindElement(By.Name("myName"))` | `driver.find_element(By.NAME, "myName")` |
| **Class Name** | `driver.FindElement(By.ClassName("myClass"))` | `driver.find_element(By.CLASS_NAME, "myClass")` |
| **Tag Name** | `driver.FindElement(By.TagName("div"))` | `driver.find_element(By.TAG_NAME, "div")` |
| **Link Text** | `driver.FindElement(By.LinkText("Click Here"))` | `driver.find_element(By.LINK_TEXT, "Click Here")` |
| **Partial Link Text** | `driver.FindElement(By.PartialLinkText("Click"))` | `driver.find_element(By.PARTIAL_LINK_TEXT, "Click")` |
| **CSS Selector** | `driver.FindElement(By.CssSelector("div.info"))` | `driver.find_element(By.CSS_SELECTOR, "div.info")` |
| **XPath** | `driver.FindElement(By.XPath("//div[@id='x']"))` | `driver.find_element(By.XPATH, "//div[@id='x']")` |

**Find multiple elements** -- returns a list/collection:

| C# | Python |
|----|--------|
| `driver.FindElements(By.CssSelector(".item"))` | `driver.find_elements(By.CSS_SELECTOR, ".item")` |

**Required imports:**

```csharp
// C#
using OpenQA.Selenium;
```

```python
# Python
from selenium.webdriver.common.by import By
```

---

## 2. Wait Types

### Implicit Wait

Sets a global timeout for all `FindElement` calls. The driver polls the DOM until the element appears or the timeout expires.

```csharp
// C#
driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(10);
```

```python
# Python
driver.implicitly_wait(10)
```

**When to use:** Simple pages with predictable load times. Avoid mixing with explicit waits.

---

### Explicit Wait (WebDriverWait)

Waits for a specific condition before proceeding. More precise than implicit waits.

```csharp
// C#
using OpenQA.Selenium.Support.UI;

var wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
IWebElement element = wait.Until(
    SeleniumExtras.WaitHelpers.ExpectedConditions.ElementIsVisible(By.Id("myId"))
);
```

```python
# Python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(driver, 10)
element = wait.until(EC.visibility_of_element_located((By.ID, "myId")))
```

**When to use:** AJAX-loaded content, dynamic elements, any element that may not be immediately present.

---

### Fluent Wait

Explicit wait with custom polling interval and exception ignoring.

```csharp
// C#
var wait = new WebDriverWait(driver, TimeSpan.FromSeconds(30))
{
    PollingInterval = TimeSpan.FromMilliseconds(500)
};
wait.IgnoreExceptionTypes(typeof(NoSuchElementException));
IWebElement element = wait.Until(d => d.FindElement(By.Id("myId")));
```

```python
# Python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.common.exceptions import NoSuchElementException

wait = WebDriverWait(
    driver,
    timeout=30,
    poll_frequency=0.5,
    ignored_exceptions=[NoSuchElementException]
)
element = wait.until(lambda d: d.find_element(By.ID, "myId"))
```

**When to use:** Unstable elements, slow APIs, situations requiring fine-grained polling control.

---

### ExpectedConditions -- Full List

| Condition | Description |
|-----------|-------------|
| `TitleIs(title)` / `title_is` | Title equals exact string |
| `TitleContains(text)` / `title_contains` | Title contains substring |
| `UrlContains(text)` / `url_contains` | URL contains substring |
| `UrlToBe(url)` / `url_to_be` | URL equals exact string |
| `PresenceOfAllElementsLocatedBy` / `presence_of_all_elements_located` | All matching elements exist in DOM |
| `ElementExists` / `presence_of_element_located` | Element exists in DOM (may not be visible) |
| `ElementIsVisible` / `visibility_of_element_located` | Element is visible on page |
| `InvisibilityOfElementLocated` / `invisibility_of_element_located` | Element is not visible |
| `ElementToBeClickable` / `element_to_be_clickable` | Element is visible and enabled |
| `StalenessOf` / `staleness_of` | Element is no longer in DOM |
| `ElementToBeSelected` / `element_to_be_selected` | Element (checkbox/option) is selected |
| `ElementSelectionStateToBe` / `element_selection_state_to_be` | Element selection matches expected |
| `FrameToBeAvailableAndSwitchToIt` / `frame_to_be_available_and_switch_to_it` | Frame is available; switches to it |
| `AlertIsPresent` / `alert_is_present` | JavaScript alert is present |
| `TextToBePresentInElement` / `text_to_be_present_in_element` | Element contains expected text |
| `TextToBePresentInElementValue` / `text_to_be_present_in_element_value` | Element value contains expected text |

---

## 3. Page Object Model (POM) Templates

### C# POM Template

```csharp
using OpenQA.Selenium;
using OpenQA.Selenium.Support.UI;
using SeleniumExtras.WaitHelpers;
using System;

namespace MyProject.Pages
{
    public class LoginPage
    {
        private readonly IWebDriver _driver;
        private readonly WebDriverWait _wait;

        // Locators
        private By UsernameField => By.Id("username");
        private By PasswordField => By.Id("password");
        private By LoginButton  => By.CssSelector("button[type='submit']");
        private By ErrorMessage => By.ClassName("error");

        // Constructor
        public LoginPage(IWebDriver driver)
        {
            _driver = driver;
            _wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
        }

        // Actions
        public LoginPage EnterUsername(string username)
        {
            _wait.Until(ExpectedConditions.ElementIsVisible(UsernameField))
                 .SendKeys(username);
            return this;
        }

        public LoginPage EnterPassword(string password)
        {
            _driver.FindElement(PasswordField).SendKeys(password);
            return this;
        }

        public DashboardPage ClickLogin()
        {
            _driver.FindElement(LoginButton).Click();
            return new DashboardPage(_driver);
        }

        public string GetErrorMessage()
        {
            return _wait.Until(ExpectedConditions.ElementIsVisible(ErrorMessage)).Text;
        }

        // Composite action
        public DashboardPage LoginAs(string username, string password)
        {
            EnterUsername(username);
            EnterPassword(password);
            return ClickLogin();
        }
    }
}
```

### Python POM Template

```python
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class LoginPage:
    # Locators
    USERNAME_FIELD = (By.ID, "username")
    PASSWORD_FIELD = (By.ID, "password")
    LOGIN_BUTTON   = (By.CSS_SELECTOR, "button[type='submit']")
    ERROR_MESSAGE  = (By.CLASS_NAME, "error")

    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)

    def enter_username(self, username):
        self.wait.until(EC.visibility_of_element_located(self.USERNAME_FIELD)) \
            .send_keys(username)
        return self

    def enter_password(self, password):
        self.driver.find_element(*self.PASSWORD_FIELD).send_keys(password)
        return self

    def click_login(self):
        self.driver.find_element(*self.LOGIN_BUTTON).click()
        from pages.dashboard_page import DashboardPage
        return DashboardPage(self.driver)

    def get_error_message(self):
        return self.wait.until(
            EC.visibility_of_element_located(self.ERROR_MESSAGE)
        ).text

    # Composite action
    def login_as(self, username, password):
        self.enter_username(username)
        self.enter_password(password)
        return self.click_login()
```

---

## 4. Common WebDriver Commands

### Navigation

| Action | C# | Python |
|--------|-----|--------|
| Open URL | `driver.Navigate().GoToUrl("https://...")` | `driver.get("https://...")` |
| Back | `driver.Navigate().Back()` | `driver.back()` |
| Forward | `driver.Navigate().Forward()` | `driver.forward()` |
| Refresh | `driver.Navigate().Refresh()` | `driver.refresh()` |
| Get current URL | `driver.Url` | `driver.current_url` |
| Get page title | `driver.Title` | `driver.title` |
| Get page source | `driver.PageSource` | `driver.page_source` |

### Element Interaction

| Action | C# | Python |
|--------|-----|--------|
| Click | `element.Click()` | `element.click()` |
| Type text | `element.SendKeys("text")` | `element.send_keys("text")` |
| Clear field | `element.Clear()` | `element.clear()` |
| Get text | `element.Text` | `element.text` |
| Get attribute | `element.GetAttribute("href")` | `element.get_attribute("href")` |
| Get CSS value | `element.GetCssValue("color")` | `element.value_of_css_property("color")` |
| Is displayed? | `element.Displayed` | `element.is_displayed()` |
| Is enabled? | `element.Enabled` | `element.is_enabled()` |
| Is selected? | `element.Selected` | `element.is_selected()` |
| Submit form | `element.Submit()` | `element.submit()` |
| Get tag name | `element.TagName` | `element.tag_name` |
| Get size | `element.Size` | `element.size` |
| Get location | `element.Location` | `element.location` |

### Dropdown (Select)

```csharp
// C#
using OpenQA.Selenium.Support.UI;

var select = new SelectElement(driver.FindElement(By.Id("dropdown")));
select.SelectByText("Option 1");
select.SelectByValue("1");
select.SelectByIndex(0);
var allOptions = select.Options;
var selected = select.SelectedOption;
```

```python
# Python
from selenium.webdriver.support.ui import Select

select = Select(driver.find_element(By.ID, "dropdown"))
select.select_by_visible_text("Option 1")
select.select_by_value("1")
select.select_by_index(0)
all_options = select.options
selected = select.first_selected_option
```

### Windows and Tabs

| Action | C# | Python |
|--------|-----|--------|
| Current window handle | `driver.CurrentWindowHandle` | `driver.current_window_handle` |
| All window handles | `driver.WindowHandles` | `driver.window_handles` |
| Switch to window | `driver.SwitchTo().Window(handle)` | `driver.switch_to.window(handle)` |
| Open new tab | `driver.SwitchTo().NewWindow(WindowType.Tab)` | `driver.switch_to.new_window("tab")` |
| Close current tab | `driver.Close()` | `driver.close()` |
| Quit browser | `driver.Quit()` | `driver.quit()` |

### Frames and iFrames

| Action | C# | Python |
|--------|-----|--------|
| Switch by index | `driver.SwitchTo().Frame(0)` | `driver.switch_to.frame(0)` |
| Switch by name/id | `driver.SwitchTo().Frame("frameName")` | `driver.switch_to.frame("frameName")` |
| Switch by element | `driver.SwitchTo().Frame(element)` | `driver.switch_to.frame(element)` |
| Switch to parent | `driver.SwitchTo().ParentFrame()` | `driver.switch_to.parent_frame()` |
| Switch to default | `driver.SwitchTo().DefaultContent()` | `driver.switch_to.default_content()` |

### Alerts

| Action | C# | Python |
|--------|-----|--------|
| Switch to alert | `driver.SwitchTo().Alert()` | `driver.switch_to.alert` |
| Accept (OK) | `driver.SwitchTo().Alert().Accept()` | `driver.switch_to.alert.accept()` |
| Dismiss (Cancel) | `driver.SwitchTo().Alert().Dismiss()` | `driver.switch_to.alert.dismiss()` |
| Get alert text | `driver.SwitchTo().Alert().Text` | `driver.switch_to.alert.text` |
| Send text to prompt | `driver.SwitchTo().Alert().SendKeys("text")` | `driver.switch_to.alert.send_keys("text")` |

### Cookies

| Action | C# | Python |
|--------|-----|--------|
| Get all cookies | `driver.Manage().Cookies.AllCookies` | `driver.get_cookies()` |
| Get cookie by name | `driver.Manage().Cookies.GetCookieNamed("name")` | `driver.get_cookie("name")` |
| Add cookie | `driver.Manage().Cookies.AddCookie(new Cookie("k","v"))` | `driver.add_cookie({"name":"k","value":"v"})` |
| Delete cookie | `driver.Manage().Cookies.DeleteCookieNamed("name")` | `driver.delete_cookie("name")` |
| Delete all | `driver.Manage().Cookies.DeleteAllCookies()` | `driver.delete_all_cookies()` |

### Screenshots

```csharp
// C# -- full page
var screenshot = ((ITakesScreenshot)driver).GetScreenshot();
screenshot.SaveAsFile("screenshot.png");

// C# -- specific element
var elemShot = ((ITakesScreenshot)element).GetScreenshot();
elemShot.SaveAsFile("element.png");
```

```python
# Python -- full page
driver.save_screenshot("screenshot.png")

# Python -- specific element
element.screenshot("element.png")
```

### JavaScript Execution

```csharp
// C#
IJavaScriptExecutor js = (IJavaScriptExecutor)driver;
js.ExecuteScript("window.scrollTo(0, document.body.scrollHeight)");
string title = (string)js.ExecuteScript("return document.title");
js.ExecuteScript("arguments[0].click()", element);
js.ExecuteScript("arguments[0].scrollIntoView(true)", element);
```

```python
# Python
driver.execute_script("window.scrollTo(0, document.body.scrollHeight)")
title = driver.execute_script("return document.title")
driver.execute_script("arguments[0].click()", element)
driver.execute_script("arguments[0].scrollIntoView(true)", element)
```

### Browser Options

```csharp
// C# -- Chrome
var options = new ChromeOptions();
options.AddArgument("--headless=new");
options.AddArgument("--start-maximized");
options.AddArgument("--disable-notifications");
options.AddArgument("--incognito");
options.AddArgument("--window-size=1920,1080");
var driver = new ChromeDriver(options);
```

```python
# Python -- Chrome
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_argument("--headless=new")
options.add_argument("--start-maximized")
options.add_argument("--disable-notifications")
options.add_argument("--incognito")
options.add_argument("--window-size=1920,1080")
driver = webdriver.Chrome(options=options)
```

### Actions (Mouse and Keyboard)

```csharp
// C#
using OpenQA.Selenium.Interactions;

var actions = new Actions(driver);
actions.MoveToElement(element).Perform();                    // Hover
actions.DoubleClick(element).Perform();                      // Double-click
actions.ContextClick(element).Perform();                     // Right-click
actions.DragAndDrop(source, target).Perform();               // Drag and drop
actions.ClickAndHold(element).MoveByOffset(100, 0).Release().Perform(); // Slider
actions.KeyDown(Keys.Control).SendKeys("a").KeyUp(Keys.Control).Perform(); // Select all
```

```python
# Python
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.common.keys import Keys

actions = ActionChains(driver)
actions.move_to_element(element).perform()                   # Hover
actions.double_click(element).perform()                      # Double-click
actions.context_click(element).perform()                     # Right-click
actions.drag_and_drop(source, target).perform()              # Drag and drop
actions.click_and_hold(element).move_by_offset(100, 0).release().perform()  # Slider
actions.key_down(Keys.CONTROL).send_keys("a").key_up(Keys.CONTROL).perform()  # Select all
```

---

## 5. XPath Cheat Sheet

### Basic Patterns

| XPath | Selects |
|-------|---------|
| `//tag` | All `<tag>` elements anywhere |
| `/html/body/div` | Absolute path from root |
| `//div[@id='main']` | `<div>` with `id="main"` |
| `//input[@name='email']` | `<input>` with `name="email"` |
| `//a[@class='active']` | `<a>` with exact `class="active"` |
| `//button[text()='Submit']` | `<button>` with exact text |
| `//button[contains(text(),'Sub')]` | `<button>` whose text contains "Sub" |
| `//div[contains(@class,'card')]` | `<div>` whose class contains "card" |
| `//input[starts-with(@id,'user')]` | `<input>` whose id starts with "user" |
| `//*[@id='myId']` | Any element with `id="myId"` |

### Axes

| Axis | Example | Selects |
|------|---------|---------|
| `parent` | `//span/parent::div` | Parent `<div>` of `<span>` |
| `child` | `//div/child::p` | Direct `<p>` children of `<div>` |
| `ancestor` | `//input/ancestor::form` | Ancestor `<form>` of `<input>` |
| `descendant` | `//div/descendant::a` | All `<a>` descendants of `<div>` |
| `following-sibling` | `//h2/following-sibling::p` | `<p>` siblings after `<h2>` |
| `preceding-sibling` | `//h2/preceding-sibling::p` | `<p>` siblings before `<h2>` |
| `following` | `//h2/following::p` | All `<p>` after `<h2>` in document |
| `preceding` | `//h2/preceding::p` | All `<p>` before `<h2>` in document |

### Functions

| Function | Example |
|----------|---------|
| `text()` | `//a[text()='Home']` |
| `contains()` | `//div[contains(@class,'active')]` |
| `starts-with()` | `//input[starts-with(@id,'txt')]` |
| `normalize-space()` | `//span[normalize-space()='Hello']` |
| `not()` | `//input[not(@disabled)]` |
| `last()` | `(//tr)[last()]` |
| `position()` | `(//li)[position()<=3]` |
| `count()` | `//ul[count(li)>5]` |
| `string-length()` | `//input[string-length(@value)>0]` |

### Logical Operators

| Operator | Example |
|----------|---------|
| `and` | `//input[@type='text' and @name='email']` |
| `or` | `//input[@type='text' or @type='email']` |
| `\|` (union) | `//h1 \| //h2` (selects both) |

### Index-Based Selection

```xpath
(//div[@class='item'])[1]      # First matching element
(//div[@class='item'])[last()] # Last matching element
(//tr)[position() > 1]         # All rows except header
```

---

## 6. CSS Selector Cheat Sheet

### Basic Selectors

| Selector | Example | Selects |
|----------|---------|---------|
| `#id` | `#username` | Element with `id="username"` |
| `.class` | `.btn-primary` | Elements with class `btn-primary` |
| `tag` | `input` | All `<input>` elements |
| `tag#id` | `input#email` | `<input>` with `id="email"` |
| `tag.class` | `div.container` | `<div>` with class `container` |
| `.class1.class2` | `.btn.active` | Elements with both classes |
| `[attr]` | `[disabled]` | Elements with `disabled` attribute |
| `[attr='val']` | `[type='submit']` | Attribute equals value |
| `[attr*='val']` | `[class*='card']` | Attribute contains "card" |
| `[attr^='val']` | `[id^='user']` | Attribute starts with "user" |
| `[attr$='val']` | `[src$='.png']` | Attribute ends with ".png" |

### Combinators

| Combinator | Example | Selects |
|------------|---------|---------|
| `A B` | `div p` | All `<p>` inside `<div>` (any depth) |
| `A > B` | `div > p` | Direct `<p>` children of `<div>` |
| `A + B` | `h2 + p` | First `<p>` immediately after `<h2>` |
| `A ~ B` | `h2 ~ p` | All `<p>` siblings after `<h2>` |

### Pseudo-Classes

| Pseudo | Example | Selects |
|--------|---------|---------|
| `:first-child` | `li:first-child` | First `<li>` in its parent |
| `:last-child` | `li:last-child` | Last `<li>` in its parent |
| `:nth-child(n)` | `tr:nth-child(2)` | Second `<tr>` |
| `:nth-child(odd)` | `tr:nth-child(odd)` | Odd rows |
| `:nth-of-type(n)` | `p:nth-of-type(2)` | Second `<p>` of its type |
| `:not(sel)` | `input:not([disabled])` | Enabled inputs |
| `:checked` | `input:checked` | Checked checkboxes/radios |
| `:enabled` | `input:enabled` | Enabled inputs |
| `:disabled` | `input:disabled` | Disabled inputs |

---

## 7. NUnit Attributes (C#)

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `[TestFixture]` | Marks a class as containing tests | `[TestFixture] public class LoginTests { }` |
| `[Test]` | Marks a method as a test | `[Test] public void ValidLogin() { }` |
| `[SetUp]` | Runs before each test | Initialize driver |
| `[TearDown]` | Runs after each test | Quit driver |
| `[OneTimeSetUp]` | Runs once before all tests in fixture | Load test data |
| `[OneTimeTearDown]` | Runs once after all tests in fixture | Clean up resources |
| `[TestCase("a","b")]` | Parameterized test | `[TestCase("admin","pass")] public void Login(string u, string p)` |
| `[TestCaseSource]` | External data source for params | `[TestCaseSource(nameof(TestData))]` |
| `[Category("Smoke")]` | Categorize tests for filtering | Run with `--where "cat==Smoke"` |
| `[Ignore("reason")]` | Skip a test | Temporarily disable |
| `[Order(1)]` | Control test execution order | Use sparingly |
| `[Retry(3)]` | Retry flaky test up to N times | `[Retry(2)]` |
| `[Timeout(5000)]` | Max milliseconds for a test | Fail if too slow |
| `[Parallelizable]` | Allow parallel execution | `[Parallelizable(ParallelScope.All)]` |
| `[Description("...")]` | Human-readable description | Appears in reports |

### Common Assertions

```csharp
Assert.That(actual, Is.EqualTo(expected));
Assert.That(title, Does.Contain("Dashboard"));
Assert.That(items, Has.Count.EqualTo(5));
Assert.That(element.Displayed, Is.True);
Assert.That(value, Is.GreaterThan(0));
Assert.That(list, Is.Not.Empty);
Assert.That(text, Does.StartWith("Hello"));
Assert.That(text, Does.Match(@"\d{3}-\d{4}"));
Assert.Throws<NoSuchElementException>(() => driver.FindElement(By.Id("missing")));
```

---

## 8. PyTest Markers and Fixtures (Python)

### Markers

```python
import pytest

@pytest.mark.smoke
def test_login():
    """Run with: pytest -m smoke"""
    pass

@pytest.mark.skip(reason="Not implemented yet")
def test_feature():
    pass

@pytest.mark.skipif(sys.platform == "win32", reason="Linux only")
def test_linux_feature():
    pass

@pytest.mark.xfail(reason="Known bug #123")
def test_known_issue():
    pass

@pytest.mark.parametrize("username,password", [
    ("admin", "admin123"),
    ("user", "user123"),
    ("invalid", "wrong"),
])
def test_login_scenarios(username, password):
    pass
```

Register custom markers in `pytest.ini`:

```ini
[pytest]
markers =
    smoke: Smoke tests
    regression: Regression tests
```

### Fixtures

```python
import pytest
from selenium import webdriver

# Basic fixture -- function scope (default, runs per test)
@pytest.fixture
def driver():
    drv = webdriver.Chrome()
    drv.maximize_window()
    yield drv
    drv.quit()

# Session-scoped fixture -- one browser for all tests
@pytest.fixture(scope="session")
def browser():
    drv = webdriver.Chrome()
    yield drv
    drv.quit()

# Fixture with parameters
@pytest.fixture(params=["chrome", "firefox"])
def multi_browser(request):
    if request.param == "chrome":
        drv = webdriver.Chrome()
    else:
        drv = webdriver.Firefox()
    yield drv
    drv.quit()
```

### conftest.py Patterns

Place `conftest.py` in your test directory. Fixtures defined here are automatically available to all tests in that directory and subdirectories.

```python
# conftest.py
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

@pytest.fixture(scope="function")
def driver(request):
    options = Options()
    if request.config.getoption("--headless", default=False):
        options.add_argument("--headless=new")
    drv = webdriver.Chrome(options=options)
    drv.maximize_window()
    drv.implicitly_wait(10)
    yield drv
    drv.quit()

@pytest.fixture
def login_page(driver):
    from pages.login_page import LoginPage
    driver.get("https://example.com/login")
    return LoginPage(driver)

def pytest_addoption(parser):
    parser.addoption("--headless", action="store_true", help="Run in headless mode")
```

### Common Assertions

```python
assert driver.title == "Dashboard"
assert "Welcome" in driver.page_source
assert element.is_displayed()
assert len(items) == 5
assert element.text.startswith("Hello")

# With pytest.raises for expected exceptions
with pytest.raises(NoSuchElementException):
    driver.find_element(By.ID, "missing")
```

---
