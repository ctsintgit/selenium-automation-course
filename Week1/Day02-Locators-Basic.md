# Day 2: Locators — ID, Name, ClassName, TagName

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain what locators are and why choosing the right one matters
- Use `FindElement` and `FindElements` to locate single and multiple elements
- Write locators using `By.Id`, `By.Name`, `By.ClassName`, and `By.TagName`
- Inspect elements using Chrome DevTools to determine their attributes
- Choose the best locator strategy for a given element

---

## Core Concept Explanation

### What Are Locators?

A **locator** is a strategy Selenium uses to find HTML elements on a web page. Before you can click a button, type into a field, or read text, you must first tell Selenium *which* element you mean. Locators are the address system for the DOM.

### The Locator Strategies

Selenium provides eight locator strategies. Today we cover the four simplest ones:

| Strategy | HTML Attribute | Example HTML | Locator |
|----------|---------------|--------------|---------|
| **ID** | `id` | `<input id="username">` | `By.Id("username")` |
| **Name** | `name` | `<input name="email">` | `By.Name("email")` |
| **ClassName** | `class` | `<div class="alert">` | `By.ClassName("alert")` |
| **TagName** | element tag | `<h1>Title</h1>` | `By.TagName("h1")` |

### FindElement vs FindElements

| Method | Returns | When no match found |
|--------|---------|-------------------|
| `FindElement` | A single `IWebElement` / `WebElement` | Throws `NoSuchElementException` |
| `FindElements` | A list of elements (can be empty) | Returns an empty list — no exception |

Use `FindElement` when you expect exactly one match. Use `FindElements` when there may be zero or many matches.

---

## How It Works (Technical Breakdown)

### How Selenium Finds Elements

1. Your code calls `driver.FindElement(By.Id("username"))`
2. Selenium sends an HTTP POST to the browser driver:
   ```
   POST /session/{sessionId}/element
   Body: {"using": "css selector", "value": "#username"}
   ```
   Note: Selenium 4 internally converts `By.Id` to a CSS selector (`#username`) before sending the request, because the W3C protocol only supports CSS and XPath natively.
3. The browser driver queries the DOM and returns the element reference
4. Selenium wraps the reference in an `IWebElement` / `WebElement` object

### How Chrome DevTools Helps

To find an element's ID, name, or class:

1. Open Chrome and navigate to the page
2. Right-click the element you want to locate
3. Select **Inspect** (or press `F12`)
4. The Elements panel highlights the HTML for that element
5. Read the attributes: `id="..."`, `name="..."`, `class="..."`

You can also press `Ctrl+F` in the Elements panel and type a CSS selector or XPath to test it before writing code.

---

## Code Example: C# (Complete, Runnable)

```csharp
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;
using System.Collections.Generic;

namespace SeleniumCourse;

[TestFixture]
public class Day02_BasicLocators
{
    private IWebDriver driver;

    [SetUp]
    public void Setup()
    {
        var options = new ChromeOptions();
        driver = new ChromeDriver(options);
        driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(5);
    }

    [Test]
    public void FindElementById()
    {
        // Navigate to the login page
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Find the username input by its ID attribute
        // HTML: <input type="text" id="username" ... >
        IWebElement usernameField = driver.FindElement(By.Id("username"));

        // Type text into the field
        usernameField.SendKeys("tomsmith");

        // Verify the value was entered
        string enteredValue = usernameField.GetAttribute("value");
        Console.WriteLine($"Username field value: {enteredValue}");
        Assert.That(enteredValue, Is.EqualTo("tomsmith"));
    }

    [Test]
    public void FindElementByName()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Find the password field by its name attribute
        // HTML: <input type="password" name="password" ... >
        IWebElement passwordField = driver.FindElement(By.Name("password"));

        passwordField.SendKeys("SuperSecretPassword!");

        // Verify the field accepted input
        string enteredValue = passwordField.GetAttribute("value");
        Console.WriteLine($"Password field value: {enteredValue}");
        Assert.That(enteredValue, Is.EqualTo("SuperSecretPassword!"));
    }

    [Test]
    public void FindElementByClassName()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Find the login button by its class name
        // HTML: <button class="radius" type="submit">...</button>
        IWebElement loginButton = driver.FindElement(By.ClassName("radius"));

        // Verify the button text
        string buttonText = loginButton.Text;
        Console.WriteLine($"Button text: {buttonText}");
        Assert.That(buttonText, Does.Contain("Login"));
    }

    [Test]
    public void FindElementByTagName()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Find the h2 heading by tag name
        // HTML: <h2>Login Page</h2>
        IWebElement heading = driver.FindElement(By.TagName("h2"));

        string headingText = heading.Text;
        Console.WriteLine($"Page heading: {headingText}");
        Assert.That(headingText, Is.EqualTo("Login Page"));
    }

    [Test]
    public void FindMultipleElements()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/checkboxes");

        // FindElements returns ALL matching elements as a list
        // HTML: <input type="checkbox"> (two of them on this page)
        IList<IWebElement> checkboxes = driver.FindElements(By.TagName("input"));

        Console.WriteLine($"Found {checkboxes.Count} checkboxes");
        Assert.That(checkboxes.Count, Is.EqualTo(2));

        // Access individual elements by index
        IWebElement firstCheckbox = checkboxes[0];
        IWebElement secondCheckbox = checkboxes[1];

        Console.WriteLine($"First checkbox selected: {firstCheckbox.Selected}");
        Console.WriteLine($"Second checkbox selected: {secondCheckbox.Selected}");
    }

    [Test]
    public void FindElements_ReturnsEmptyList_WhenNoMatch()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // FindElements with a non-existent class returns an empty list
        // This does NOT throw an exception
        IList<IWebElement> nonExistent = driver.FindElements(By.ClassName("does-not-exist"));

        Console.WriteLine($"Elements found: {nonExistent.Count}");
        Assert.That(nonExistent.Count, Is.EqualTo(0));
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
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By


@pytest.fixture
def driver():
    """Create a ChromeDriver, yield it to the test, then quit."""
    options = Options()
    drv = webdriver.Chrome(options=options)
    drv.implicitly_wait(5)
    yield drv
    drv.quit()


def test_find_element_by_id(driver):
    """Find the username input field by its ID attribute."""
    driver.get("https://the-internet.herokuapp.com/login")

    # HTML: <input type="text" id="username" ... >
    username_field = driver.find_element(By.ID, "username")

    # Type text into the field
    username_field.send_keys("tomsmith")

    # Verify the value
    entered_value = username_field.get_attribute("value")
    print(f"Username field value: {entered_value}")
    assert entered_value == "tomsmith"


def test_find_element_by_name(driver):
    """Find the password field by its name attribute."""
    driver.get("https://the-internet.herokuapp.com/login")

    # HTML: <input type="password" name="password" ... >
    password_field = driver.find_element(By.NAME, "password")

    password_field.send_keys("SuperSecretPassword!")

    entered_value = password_field.get_attribute("value")
    print(f"Password field value: {entered_value}")
    assert entered_value == "SuperSecretPassword!"


def test_find_element_by_class_name(driver):
    """Find the login button by its class name."""
    driver.get("https://the-internet.herokuapp.com/login")

    # HTML: <button class="radius" type="submit">...</button>
    login_button = driver.find_element(By.CLASS_NAME, "radius")

    button_text = login_button.text
    print(f"Button text: {button_text}")
    assert "Login" in button_text


def test_find_element_by_tag_name(driver):
    """Find the heading element by its tag name."""
    driver.get("https://the-internet.herokuapp.com/login")

    # HTML: <h2>Login Page</h2>
    heading = driver.find_element(By.TAG_NAME, "h2")

    heading_text = heading.text
    print(f"Page heading: {heading_text}")
    assert heading_text == "Login Page"


def test_find_multiple_elements(driver):
    """Use find_elements to get a list of all matching elements."""
    driver.get("https://the-internet.herokuapp.com/checkboxes")

    # find_elements returns a list of all matching elements
    checkboxes = driver.find_elements(By.TAG_NAME, "input")

    print(f"Found {len(checkboxes)} checkboxes")
    assert len(checkboxes) == 2

    # Access individual elements by index
    first_checkbox = checkboxes[0]
    second_checkbox = checkboxes[1]

    print(f"First checkbox selected: {first_checkbox.is_selected()}")
    print(f"Second checkbox selected: {second_checkbox.is_selected()}")


def test_find_elements_returns_empty_list(driver):
    """find_elements returns an empty list when no match is found."""
    driver.get("https://the-internet.herokuapp.com/login")

    # This does NOT raise an exception — it returns []
    non_existent = driver.find_elements(By.CLASS_NAME, "does-not-exist")

    print(f"Elements found: {len(non_existent)}")
    assert len(non_existent) == 0
```

---

## Step-by-Step Walkthrough

### How to inspect an element in Chrome DevTools:

1. Open Chrome and navigate to `https://the-internet.herokuapp.com/login`
2. Right-click on the username text field
3. Click **Inspect** — DevTools opens with the element highlighted
4. In the Elements panel, you see:
   ```html
   <input type="text" name="username" id="username">
   ```
5. This element has **both** an `id` and a `name` — you can use either
6. To test your locator, press `Ctrl+F` in the Elements panel and type `#username` (CSS for ID) — it should highlight exactly one match

### Choosing a locator for the login button:

1. Right-click the **Login** button and inspect it
2. You see: `<button class="radius" type="submit"><i class="fa fa-2x fa-sign-in"> Login</i></button>`
3. It has no `id` or `name`, but has `class="radius"`
4. Use `By.ClassName("radius")`
5. Note: if multiple elements share the same class, `FindElement` returns the first one — use `FindElements` to get all of them

---

## Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using `By.ClassName` with multiple classes | `By.ClassName("btn primary")` throws an error | `ClassName` accepts only a single class name. Use CSS selector `By.CssSelector(".btn.primary")` for multiple classes |
| Assuming `FindElement` waits for the element | Without an implicit or explicit wait, `FindElement` fails instantly if the element has not loaded | Set an implicit wait or use explicit waits (covered in Week 2) |
| Confusing `By.Id` in C# vs Python | C#: `By.Id("x")`, Python: `By.ID, "x"` | Note: Python uses `By.ID` (uppercase) as the first arg, the value as the second |
| Using `FindElement` when multiple matches exist | Returns only the first match — may not be the one you want | Use `FindElements` and index into the list, or use a more specific locator |
| Catching `NoSuchElementException` instead of fixing the locator | Hides real bugs | Fix the locator or use `FindElements` if the element is optional |

---

## Best Practices

1. **Prefer ID locators** — IDs should be unique on a page, making them the most reliable locator. If an element has an ID, use it.

2. **Name is a good second choice** — especially for form fields, the `name` attribute is stable and meaningful.

3. **Avoid TagName for specific elements** — `By.TagName("div")` matches dozens of elements. Only use it when you want all elements of a type (e.g., all `<input>` fields).

4. **Use FindElements defensively** — when you are unsure if an element exists, `FindElements` returns an empty list instead of throwing an exception. Check the count before accessing elements.

5. **Keep locators in constants or Page Objects** — do not scatter string literals throughout your tests. This makes maintenance easier when the UI changes.

```csharp
// Good: locator defined once
private readonly By UsernameField = By.Id("username");

// Then used in tests
driver.FindElement(UsernameField).SendKeys("tomsmith");
```

```python
# Good: locator defined once
USERNAME_FIELD = (By.ID, "username")

# Then used in tests
driver.find_element(*USERNAME_FIELD).send_keys("tomsmith")
```

---

## Hands-On Exercise

### Task

Navigate to `https://the-internet.herokuapp.com/login` and write a test that:

1. Finds the username field **by ID** and types `"tomsmith"`
2. Finds the password field **by Name** and types `"SuperSecretPassword!"`
3. Finds the login button **by ClassName** and clicks it
4. After the page redirects, finds the flash message **by ID** (`"flash"`) and reads its text
5. Asserts that the flash message contains `"You logged into a secure area!"`
6. Finds the logout button **by TagName** — look for an `<a>` tag inside the content area, or by class

### Expected Output

```
Username entered: tomsmith
Password entered: SuperSecretPassword!
Clicked login button
Flash message: You logged into a secure area!
Test passed!
```

---

## Real-World Scenario

You are testing a company intranet application. The login page has a username field with `id="emp-username"`, a password field with `name="password"`, and a submit button with `class="btn-login"`. There is also a "Remember Me" checkbox with `id="remember"`.

Your test:

1. Finds the username field by ID — fast and reliable
2. Finds the password field by Name — IDs are not always present on every field
3. Finds the checkbox by ID and checks `Selected` / `is_selected()` before clicking
4. Finds the button by ClassName and clicks it
5. After login, finds a welcome message by TagName (`<h1>`) and verifies the text

This is a real pattern used in thousands of production test suites. The locator strategies you learned today are the foundation of every Selenium test.

---

## Resources

- [Selenium Locator Strategies](https://www.selenium.dev/documentation/webdriver/elements/locators/)
- [Finding Web Elements](https://www.selenium.dev/documentation/webdriver/elements/finders/)
- [Chrome DevTools — Inspect Elements](https://developer.chrome.com/docs/devtools/dom/)
- [Practice Site: The Internet](https://the-internet.herokuapp.com/)
- [HTML Attributes Reference (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes)
