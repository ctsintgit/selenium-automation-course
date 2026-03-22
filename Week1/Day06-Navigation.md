# Day 6: Navigation — Back, Forward, Refresh, URL, Title

## Learning Objectives

By the end of this lesson, you will be able to:

- Navigate to URLs, go back, go forward, and refresh pages using the Navigation API
- Read and verify the current URL and page title
- Manage browser cookies (add, get, delete)
- Use explicit waits after navigation to ensure pages are fully loaded
- Build tests that traverse multi-page flows and verify state at each step

---

## Core Concept Explanation

### The Navigation API

Selenium provides a dedicated navigation interface that mirrors browser toolbar actions:

| Browser Action | C# | Python |
|---------------|-----|--------|
| Go to URL | `driver.Navigate().GoToUrl(url)` | `driver.get(url)` |
| Back button | `driver.Navigate().Back()` | `driver.back()` |
| Forward button | `driver.Navigate().Forward()` | `driver.forward()` |
| Refresh | `driver.Navigate().Refresh()` | `driver.refresh()` |

### Page Information

| Information | C# | Python |
|------------|-----|--------|
| Current URL | `driver.Url` | `driver.current_url` |
| Page title | `driver.Title` | `driver.title` |
| Page source | `driver.PageSource` | `driver.page_source` |

### Cookie Management

Cookies are key-value pairs stored by the browser. They persist across page loads within a session and are commonly used for authentication tokens, session IDs, and user preferences.

| Operation | C# | Python |
|----------|-----|--------|
| Get all cookies | `driver.Manage().Cookies.AllCookies` | `driver.get_cookies()` |
| Get specific cookie | `driver.Manage().Cookies.GetCookieNamed(name)` | `driver.get_cookie(name)` |
| Add cookie | `driver.Manage().Cookies.AddCookie(cookie)` | `driver.add_cookie(cookie)` |
| Delete cookie | `driver.Manage().Cookies.DeleteCookieNamed(name)` | `driver.delete_cookie(name)` |
| Delete all cookies | `driver.Manage().Cookies.DeleteAllCookies()` | `driver.delete_all_cookies()` |

---

## How It Works (Technical Breakdown)

### Navigation Under the Hood

When you call `driver.Navigate().GoToUrl(url)`:

1. Selenium sends `POST /session/{id}/url` with the target URL
2. The browser driver instructs the browser to navigate
3. The call **blocks** until the page fires its `load` event (all resources downloaded)
4. If the page takes longer than `PageLoad` timeout, a `TimeoutException` is thrown

When you call `driver.Navigate().Back()`:

1. Selenium sends `POST /session/{id}/back`
2. The browser navigates to the previous entry in its history stack
3. The call blocks until the previous page's `load` event fires

### Important: Navigation vs URL Property

- `driver.Navigate().GoToUrl(url)` — **navigates** to the URL (blocks until loaded)
- `driver.Url` / `driver.current_url` — **reads** the current URL (returns immediately, does not navigate)

### Cookie Scope

Cookies are scoped to the domain. You can only set cookies for the domain you are currently on. To set a cookie for `example.com`, you must first navigate to `example.com`.

---

## Code Example: C# (Complete, Runnable)

```csharp
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;
using System.Linq;

namespace SeleniumCourse;

[TestFixture]
public class Day06_Navigation
{
    private IWebDriver driver;
    private WebDriverWait wait;

    [SetUp]
    public void Setup()
    {
        var options = new ChromeOptions();
        driver = new ChromeDriver(options);
        wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));

        // Set page load timeout to 30 seconds
        driver.Manage().Timeouts().PageLoad = TimeSpan.FromSeconds(30);
    }

    [Test]
    public void NavigateToUrl()
    {
        // Navigate to the first page
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

        // Read and verify the URL
        string currentUrl = driver.Url;
        Console.WriteLine($"Current URL: {currentUrl}");
        Assert.That(currentUrl, Does.Contain("the-internet.herokuapp.com"));

        // Read and verify the title
        string title = driver.Title;
        Console.WriteLine($"Page title: {title}");
        Assert.That(title, Is.EqualTo("The Internet"));
    }

    [Test]
    public void NavigateBackAndForward()
    {
        // Step 1: Go to the home page
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");
        string homePage = driver.Url;
        Console.WriteLine($"Step 1 - Home: {homePage}");

        // Step 2: Navigate to the login page by clicking a link
        IWebElement loginLink = driver.FindElement(
            By.CssSelector("a[href='/login']"));
        loginLink.Click();

        // Wait for the login page to load
        wait.Until(d => d.Url.Contains("/login"));
        string loginPage = driver.Url;
        Console.WriteLine($"Step 2 - Login: {loginPage}");
        Assert.That(loginPage, Does.Contain("/login"));

        // Step 3: Go back to the home page
        driver.Navigate().Back();

        // Wait for the home page URL
        wait.Until(d => d.Url == homePage);
        Console.WriteLine($"Step 3 - After Back: {driver.Url}");
        Assert.That(driver.Url, Is.EqualTo(homePage));

        // Step 4: Go forward to the login page again
        driver.Navigate().Forward();

        // Wait for the login page URL
        wait.Until(d => d.Url.Contains("/login"));
        Console.WriteLine($"Step 4 - After Forward: {driver.Url}");
        Assert.That(driver.Url, Does.Contain("/login"));
    }

    [Test]
    public void RefreshPage()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/add_remove_elements/");

        // Add an element
        IWebElement addButton = driver.FindElement(
            By.CssSelector("button[onclick='addElement()']"));
        addButton.Click();

        // Verify element was added
        var deleteBefore = driver.FindElements(By.CssSelector(".added-manually"));
        Console.WriteLine($"Before refresh: {deleteBefore.Count} delete button(s)");
        Assert.That(deleteBefore.Count, Is.EqualTo(1));

        // Refresh the page — this resets dynamically added elements
        driver.Navigate().Refresh();

        // Wait for the page to reload
        wait.Until(d => d.FindElement(By.CssSelector("button[onclick='addElement()']")));

        // Verify the dynamically added element is gone
        var deleteAfter = driver.FindElements(By.CssSelector(".added-manually"));
        Console.WriteLine($"After refresh: {deleteAfter.Count} delete button(s)");
        Assert.That(deleteAfter.Count, Is.EqualTo(0));
    }

    [Test]
    public void ReadPageProperties()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // URL
        string url = driver.Url;
        Console.WriteLine($"URL: {url}");

        // Title
        string title = driver.Title;
        Console.WriteLine($"Title: {title}");

        // Page source (HTML)
        string source = driver.PageSource;
        Console.WriteLine($"Page source length: {source.Length} characters");
        Assert.That(source, Does.Contain("<h2>Login Page</h2>"));

        // Verify URL components
        Assert.That(url, Does.StartWith("https://"));
        Assert.That(url, Does.EndWith("/login"));
    }

    [Test]
    public void MultiPageFlowWithVerification()
    {
        // Simulate a user journey through multiple pages

        // Page 1: Home
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");
        Assert.That(driver.Title, Is.EqualTo("The Internet"));
        Console.WriteLine("Page 1: Home - OK");

        // Page 2: Login page
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");
        wait.Until(d => d.FindElement(By.Id("username")));
        Assert.That(driver.Url, Does.Contain("/login"));
        Console.WriteLine("Page 2: Login - OK");

        // Page 3: Perform login
        driver.FindElement(By.Id("username")).SendKeys("tomsmith");
        driver.FindElement(By.Id("password")).SendKeys("SuperSecretPassword!");
        driver.FindElement(By.CssSelector("button.radius")).Click();

        // Wait for redirect to secure area
        wait.Until(d => d.Url.Contains("/secure"));
        Assert.That(driver.Url, Does.Contain("/secure"));
        Console.WriteLine("Page 3: Secure Area - OK");

        // Page 4: Logout
        IWebElement logoutButton = wait.Until(d =>
            d.FindElement(By.CssSelector("a[href='/logout']")));
        logoutButton.Click();

        // Wait for redirect back to login
        wait.Until(d => d.Url.Contains("/login"));
        Assert.That(driver.Url, Does.Contain("/login"));
        Console.WriteLine("Page 4: Back to Login - OK");
    }

    [Test]
    public void ManageCookies()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

        // Get all existing cookies
        var allCookies = driver.Manage().Cookies.AllCookies;
        Console.WriteLine($"Initial cookies: {allCookies.Count}");

        foreach (var cookie in allCookies)
        {
            Console.WriteLine($"  {cookie.Name} = {cookie.Value}");
        }

        // Add a custom cookie
        Cookie newCookie = new Cookie("test_cookie", "selenium_value");
        driver.Manage().Cookies.AddCookie(newCookie);

        // Retrieve the cookie we just added
        Cookie retrieved = driver.Manage().Cookies.GetCookieNamed("test_cookie");
        Console.WriteLine($"Retrieved cookie: {retrieved.Name} = {retrieved.Value}");
        Assert.That(retrieved.Value, Is.EqualTo("selenium_value"));

        // Delete the specific cookie
        driver.Manage().Cookies.DeleteCookieNamed("test_cookie");

        // Verify it is gone
        Cookie deleted = driver.Manage().Cookies.GetCookieNamed("test_cookie");
        Assert.That(deleted, Is.Null);
        Console.WriteLine("Cookie deleted successfully");

        // Delete all cookies
        driver.Manage().Cookies.DeleteAllCookies();
        var remaining = driver.Manage().Cookies.AllCookies;
        Console.WriteLine($"Cookies after delete all: {remaining.Count}");
        Assert.That(remaining.Count, Is.EqualTo(0));
    }

    [Test]
    public void CookieWithProperties()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

        // Add a cookie with full properties
        Cookie detailedCookie = new Cookie(
            name: "session_pref",
            value: "dark_mode",
            domain: "the-internet.herokuapp.com",
            path: "/",
            expiry: DateTime.Now.AddDays(7)
        );
        driver.Manage().Cookies.AddCookie(detailedCookie);

        // Retrieve and inspect
        Cookie retrieved = driver.Manage().Cookies.GetCookieNamed("session_pref");
        Console.WriteLine($"Name: {retrieved.Name}");
        Console.WriteLine($"Value: {retrieved.Value}");
        Console.WriteLine($"Domain: {retrieved.Domain}");
        Console.WriteLine($"Path: {retrieved.Path}");
        Console.WriteLine($"Expiry: {retrieved.Expiry}");

        Assert.That(retrieved.Value, Is.EqualTo("dark_mode"));
    }

    [TearDown]
    public void Teardown()
    {
        driver.Quit();
    }
}
```

---

## Code Example: Python (Complete, Runnable)

```python
import pytest
from datetime import datetime, timedelta
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


@pytest.fixture
def driver():
    """Create a ChromeDriver, yield it, then quit."""
    options = Options()
    drv = webdriver.Chrome(options=options)
    drv.set_page_load_timeout(30)
    yield drv
    drv.quit()


def test_navigate_to_url(driver):
    """Navigate to a URL and verify title and URL."""
    driver.get("https://the-internet.herokuapp.com/")

    current_url = driver.current_url
    print(f"Current URL: {current_url}")
    assert "the-internet.herokuapp.com" in current_url

    title = driver.title
    print(f"Page title: {title}")
    assert title == "The Internet"


def test_navigate_back_and_forward(driver):
    """Use back() and forward() to traverse browser history."""
    # Step 1: Go to home page
    driver.get("https://the-internet.herokuapp.com/")
    home_url = driver.current_url
    print(f"Step 1 - Home: {home_url}")

    # Step 2: Click a link to go to the login page
    login_link = driver.find_element(By.CSS_SELECTOR, "a[href='/login']")
    login_link.click()

    wait = WebDriverWait(driver, 10)
    wait.until(EC.url_contains("/login"))
    login_url = driver.current_url
    print(f"Step 2 - Login: {login_url}")
    assert "/login" in login_url

    # Step 3: Go back
    driver.back()
    wait.until(EC.url_to_be(home_url))
    print(f"Step 3 - After back: {driver.current_url}")
    assert driver.current_url == home_url

    # Step 4: Go forward
    driver.forward()
    wait.until(EC.url_contains("/login"))
    print(f"Step 4 - After forward: {driver.current_url}")
    assert "/login" in driver.current_url


def test_refresh_page(driver):
    """Refresh the page and verify dynamic content is reset."""
    driver.get("https://the-internet.herokuapp.com/add_remove_elements/")

    # Add an element
    add_button = driver.find_element(
        By.CSS_SELECTOR, "button[onclick='addElement()']"
    )
    add_button.click()

    delete_before = driver.find_elements(By.CSS_SELECTOR, ".added-manually")
    print(f"Before refresh: {len(delete_before)} delete button(s)")
    assert len(delete_before) == 1

    # Refresh the page
    driver.refresh()

    # Wait for page to reload
    wait = WebDriverWait(driver, 10)
    wait.until(
        EC.presence_of_element_located(
            (By.CSS_SELECTOR, "button[onclick='addElement()']")
        )
    )

    # Dynamically added element should be gone
    delete_after = driver.find_elements(By.CSS_SELECTOR, ".added-manually")
    print(f"After refresh: {len(delete_after)} delete button(s)")
    assert len(delete_after) == 0


def test_read_page_properties(driver):
    """Read URL, title, and page source."""
    driver.get("https://the-internet.herokuapp.com/login")

    url = driver.current_url
    print(f"URL: {url}")

    title = driver.title
    print(f"Title: {title}")

    source = driver.page_source
    print(f"Page source length: {len(source)} characters")
    assert "<h2>Login Page</h2>" in source

    assert url.startswith("https://")
    assert url.endswith("/login")


def test_multi_page_flow(driver):
    """Navigate through a complete user journey."""
    wait = WebDriverWait(driver, 10)

    # Page 1: Home
    driver.get("https://the-internet.herokuapp.com/")
    assert driver.title == "The Internet"
    print("Page 1: Home - OK")

    # Page 2: Login
    driver.get("https://the-internet.herokuapp.com/login")
    wait.until(EC.presence_of_element_located((By.ID, "username")))
    assert "/login" in driver.current_url
    print("Page 2: Login - OK")

    # Page 3: Perform login
    driver.find_element(By.ID, "username").send_keys("tomsmith")
    driver.find_element(By.ID, "password").send_keys("SuperSecretPassword!")
    driver.find_element(By.CSS_SELECTOR, "button.radius").click()

    wait.until(EC.url_contains("/secure"))
    assert "/secure" in driver.current_url
    print("Page 3: Secure Area - OK")

    # Page 4: Logout
    logout_button = wait.until(
        EC.element_to_be_clickable((By.CSS_SELECTOR, "a[href='/logout']"))
    )
    logout_button.click()

    wait.until(EC.url_contains("/login"))
    assert "/login" in driver.current_url
    print("Page 4: Back to Login - OK")


def test_manage_cookies(driver):
    """Add, retrieve, and delete cookies."""
    driver.get("https://the-internet.herokuapp.com/")

    # Get all existing cookies
    all_cookies = driver.get_cookies()
    print(f"Initial cookies: {len(all_cookies)}")
    for cookie in all_cookies:
        print(f"  {cookie['name']} = {cookie['value']}")

    # Add a custom cookie
    driver.add_cookie({"name": "test_cookie", "value": "selenium_value"})

    # Retrieve the cookie
    retrieved = driver.get_cookie("test_cookie")
    print(f"Retrieved cookie: {retrieved['name']} = {retrieved['value']}")
    assert retrieved["value"] == "selenium_value"

    # Delete the specific cookie
    driver.delete_cookie("test_cookie")

    # Verify it is gone
    deleted = driver.get_cookie("test_cookie")
    assert deleted is None
    print("Cookie deleted successfully")

    # Delete all cookies
    driver.delete_all_cookies()
    remaining = driver.get_cookies()
    print(f"Cookies after delete all: {len(remaining)}")
    assert len(remaining) == 0


def test_cookie_with_properties(driver):
    """Add a cookie with domain, path, and expiry."""
    driver.get("https://the-internet.herokuapp.com/")

    # Add a cookie with full properties
    expiry_timestamp = int((datetime.now() + timedelta(days=7)).timestamp())
    driver.add_cookie({
        "name": "session_pref",
        "value": "dark_mode",
        "domain": "the-internet.herokuapp.com",
        "path": "/",
        "expiry": expiry_timestamp,
    })

    # Retrieve and inspect
    retrieved = driver.get_cookie("session_pref")
    print(f"Name: {retrieved['name']}")
    print(f"Value: {retrieved['value']}")
    print(f"Domain: {retrieved['domain']}")
    print(f"Path: {retrieved['path']}")
    print(f"Expiry: {retrieved.get('expiry')}")

    assert retrieved["value"] == "dark_mode"
```

---

## Step-by-Step Walkthrough

### How `Back()` and `Forward()` work:

The browser maintains a **history stack** — a list of URLs you have visited in order.

```
History:  [Home] --> [Login] --> [Secure]
                                  ^
                              (current)
```

- `Back()` moves the pointer left: current becomes `[Login]`
- `Forward()` moves the pointer right: current becomes `[Secure]`
- `GoToUrl(newUrl)` adds a new entry and clears forward history

### Why Refresh() matters in testing:

`Refresh()` is useful when:

- Testing that server-side state persists across page reloads
- Verifying that client-side state (like dynamically added DOM elements) is reset
- Testing caching behavior
- Recovering from a JavaScript error that corrupted the page

### Cookie lifecycle in tests:

1. Navigate to the domain first (you cannot set cookies before navigation)
2. Add cookies with `AddCookie()` / `add_cookie()`
3. Refresh or navigate to let the server see the new cookies
4. Read cookies to verify state
5. Delete cookies in teardown to prevent state leakage between tests

---

## Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Not waiting after `Back()`/`Forward()` | Page may not be fully loaded when you access elements | Use `wait.Until(d => d.Url.Contains(...))` after navigation |
| Setting cookies before navigating | `InvalidCookieDomainException` — browser has no domain context | Navigate to the domain first, then set cookies |
| Confusing `driver.Url` (read) with `GoToUrl` (navigate) | `driver.Url` does not navigate anywhere | Use `GoToUrl()` / `get()` to navigate |
| Not handling page load timeout | Test hangs forever on a slow page | Set `driver.Manage().Timeouts().PageLoad` |
| Using `driver.Url` for assertions without waiting | URL may not have changed yet after a click | Use `wait.Until(EC.url_contains(...))` |
| Checking `driver.Title` on SPA pages | Title may not update on single-page apps | Use URL checks or element presence instead |

---

## Best Practices

1. **Always wait after navigation** — whether you navigate with `GoToUrl`, `Back`, `Forward`, or by clicking a link, wait for a condition before interacting with the new page.

2. **Use URL assertions for page verification** — `Assert.That(driver.Url, Does.Contain("/expected-path"))` is more reliable than checking the title, which may be the same across pages.

3. **Set page load timeout** — protect your tests from hanging forever on slow or broken pages:
   ```csharp
   driver.Manage().Timeouts().PageLoad = TimeSpan.FromSeconds(30);
   ```

4. **Clean up cookies between tests** — if your tests set cookies, delete them in teardown to prevent state leakage.

5. **Use `GoToUrl` for direct navigation, clicks for user flow** — when testing a specific page, navigate directly. When testing a user journey, click through the UI naturally.

---

## Hands-On Exercise

### Task

Write a test that simulates a user journey:

1. Navigate to `https://the-internet.herokuapp.com/`
2. Record the home page URL and title
3. Click the "Checkboxes" link to navigate to the checkboxes page
4. Verify the URL changed to contain `/checkboxes`
5. Go back to the home page
6. Verify you are back (URL matches the recorded home URL)
7. Go forward to the checkboxes page
8. Verify you are on the checkboxes page again
9. Refresh the page
10. Verify the page is still the checkboxes page
11. Add a cookie `"visited" = "checkboxes"` and verify it was stored
12. Navigate to the home page and verify the cookie persists

### Expected Output

```
Home URL: https://the-internet.herokuapp.com/
Home title: The Internet
Navigated to checkboxes: https://the-internet.herokuapp.com/checkboxes
After back: https://the-internet.herokuapp.com/
After forward: https://the-internet.herokuapp.com/checkboxes
After refresh: https://the-internet.herokuapp.com/checkboxes
Cookie set: visited = checkboxes
After navigating home, cookie still exists: visited = checkboxes
```

---

## Real-World Scenario

You are testing an e-commerce checkout flow. The user must navigate through:

1. **Product listing** -> click a product
2. **Product detail** -> click "Add to Cart"
3. **Cart page** -> click "Checkout"
4. **Shipping form** -> fill in details, click "Continue"
5. **Payment form** -> fill in details, click "Place Order"
6. **Confirmation page** -> verify order number

At each step, you verify the URL changes correctly. If the user clicks "Back" from the payment page, they should return to the shipping form with their data preserved. Your test:

```python
# After filling shipping form and clicking Continue:
wait.until(EC.url_contains("/checkout/payment"))

# Go back to verify shipping data is preserved
driver.back()
wait.until(EC.url_contains("/checkout/shipping"))

# Verify the name field still has the user's input
name_field = driver.find_element(By.ID, "shipping-name")
assert name_field.get_attribute("value") == "Jane Smith"
```

The navigation API lets you test the exact same Back/Forward behavior that real users perform.

---

## Resources

- [Selenium Navigation](https://www.selenium.dev/documentation/webdriver/interactions/navigation/)
- [Selenium Cookies](https://www.selenium.dev/documentation/webdriver/interactions/cookies/)
- [Browser Timeouts](https://www.selenium.dev/documentation/webdriver/drivers/options/#timeouts)
- [WebDriverWait — Expected Conditions](https://www.selenium.dev/documentation/webdriver/support_features/expected_conditions/)
- [Practice Site: The Internet](https://the-internet.herokuapp.com/)
