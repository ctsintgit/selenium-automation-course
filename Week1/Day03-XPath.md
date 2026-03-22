# Day 3: XPath (Absolute vs Relative) with Real Examples

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain what XPath is and how it traverses the DOM tree
- Distinguish between absolute and relative XPath and know when to use each
- Write XPath expressions using attributes, text, and functions
- Use XPath axes (parent, child, sibling, ancestor) to navigate between related elements
- Combine multiple conditions with `and` / `or` operators
- Test XPath expressions in Chrome DevTools before writing code

---

## Core Concept Explanation

### What Is XPath?

**XPath** (XML Path Language) is a query language for selecting nodes in an XML or HTML document. Think of it as a file path for the DOM tree. Just as `/home/user/documents/file.txt` navigates a file system, an XPath like `//div[@class='login']/form/input[@id='username']` navigates the HTML tree to find a specific element.

### Why XPath Matters

XPath is the most powerful locator strategy in Selenium. While ID and Name are simpler, they only work when those attributes exist. XPath can find any element in any situation:

- Elements without IDs or names
- Elements identified by their text content
- Elements identified by their relationship to other elements (parent, sibling)
- Elements matching partial attribute values

---

## How It Works (Technical Breakdown)

### Absolute XPath

An absolute XPath starts from the root of the document and specifies every level:

```
/html/body/div[1]/div[2]/form/input[1]
```

- Starts with a single `/` (root)
- Specifies every parent-child relationship
- **Fragile** — any change to the page structure breaks it
- **Never use absolute XPath in production tests**

### Relative XPath

A relative XPath starts from anywhere in the document using `//`:

```
//input[@id='username']
```

- Starts with `//` (search anywhere in the document)
- Matches elements based on attributes, text, or position
- **Robust** — survives most page structure changes
- **Always use relative XPath**

### XPath Syntax Reference

| Syntax | Meaning | Example |
|--------|---------|---------|
| `//` | Search anywhere in document | `//input` (all input elements) |
| `/` | Direct child | `//form/input` (input directly inside form) |
| `[@attr='value']` | Attribute match | `//input[@type='text']` |
| `text()` | Match by visible text | `//a[text()='Click Here']` |
| `contains()` | Partial match | `//div[contains(@class, 'error')]` |
| `starts-with()` | Prefix match | `//input[starts-with(@id, 'user')]` |
| `and` / `or` | Combine conditions | `//input[@type='text' and @name='email']` |
| `[n]` | Index (1-based) | `//ul/li[3]` (third li element) |

### XPath Axes

Axes let you navigate from one element to related elements:

| Axis | Direction | Example |
|------|-----------|---------|
| `parent::` | One level up | `//input[@id='email']/parent::div` |
| `child::` | One level down (default) | `//form/child::input` (same as `//form/input`) |
| `ancestor::` | All levels up | `//input[@id='email']/ancestor::form` |
| `following-sibling::` | Same-level, after | `//label[text()='Email']/following-sibling::input` |
| `preceding-sibling::` | Same-level, before | `//input[@id='email']/preceding-sibling::label` |
| `descendant::` | All levels down | `//form/descendant::input` (any depth) |

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
public class Day03_XPath
{
    private IWebDriver driver;
    private WebDriverWait wait;

    [SetUp]
    public void Setup()
    {
        var options = new ChromeOptions();
        driver = new ChromeDriver(options);
        wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
    }

    [Test]
    public void BasicXPath_AttributeMatch()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // XPath: find input where id attribute equals 'username'
        IWebElement username = driver.FindElement(
            By.XPath("//input[@id='username']"));
        username.SendKeys("tomsmith");

        // XPath: find input where type attribute equals 'password'
        IWebElement password = driver.FindElement(
            By.XPath("//input[@type='password']"));
        password.SendKeys("SuperSecretPassword!");

        Console.WriteLine("Fields populated using attribute-match XPaths");
        Assert.That(username.GetAttribute("value"), Is.EqualTo("tomsmith"));
    }

    [Test]
    public void XPath_TextMatch()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // XPath: find element by its visible text content
        IWebElement heading = driver.FindElement(
            By.XPath("//h2[text()='Login Page']"));

        Console.WriteLine($"Found heading: {heading.Text}");
        Assert.That(heading.Text, Is.EqualTo("Login Page"));
    }

    [Test]
    public void XPath_ContainsFunction()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // contains() matches partial attribute values
        // Useful when classes are dynamic: "btn btn-primary active"
        IWebElement button = driver.FindElement(
            By.XPath("//button[contains(@class, 'radius')]"));

        Console.WriteLine($"Button text: {button.Text}");
        Assert.That(button.Text, Does.Contain("Login"));

        // contains() on text content
        IWebElement subheading = driver.FindElement(
            By.XPath("//*[contains(text(), 'logged into')]").Equals(null)
                ? By.XPath("//h4[contains(text(), 'login')]")
                : By.XPath("//h4"));

        // A simpler contains on text example:
        IWebElement pageText = driver.FindElement(
            By.XPath("//*[contains(text(), 'Username')]"));
        Console.WriteLine($"Found element containing 'Username': {pageText.TagName}");
    }

    [Test]
    public void XPath_StartsWithFunction()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // starts-with() matches the beginning of an attribute value
        // Useful for dynamically generated IDs like "user_12345"
        IWebElement usernameField = driver.FindElement(
            By.XPath("//input[starts-with(@id, 'user')]"));

        usernameField.SendKeys("tomsmith");
        Console.WriteLine($"Found field starting with 'user': {usernameField.GetAttribute("id")}");
        Assert.That(usernameField.GetAttribute("id"), Is.EqualTo("username"));
    }

    [Test]
    public void XPath_MultipleConditions()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Combine conditions with 'and'
        IWebElement usernameInput = driver.FindElement(
            By.XPath("//input[@type='text' and @id='username']"));

        Console.WriteLine($"Found element with two conditions: id={usernameInput.GetAttribute("id")}");
        Assert.That(usernameInput.GetAttribute("id"), Is.EqualTo("username"));

        // Combine conditions with 'or'
        IList<IWebElement> inputs = driver.FindElements(
            By.XPath("//input[@id='username' or @id='password']"));

        Console.WriteLine($"Found {inputs.Count} elements matching either condition");
        Assert.That(inputs.Count, Is.EqualTo(2));
    }

    [Test]
    public void XPath_Axes_ParentAndSibling()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/tables");

        // Find a cell and navigate to its parent row
        // ancestor:: goes up multiple levels
        IWebElement row = driver.FindElement(
            By.XPath("//td[text()='Smith']/ancestor::tr"));

        // Get all cells in that row
        IList<IWebElement> cells = row.FindElements(By.TagName("td"));
        Console.WriteLine($"Row has {cells.Count} cells");

        // following-sibling:: gets elements at the same level, after current
        IWebElement lastNameCell = driver.FindElement(
            By.XPath("//table[@id='table1']//td[text()='Smith']"));
        IWebElement firstNameCell = driver.FindElement(
            By.XPath("//table[@id='table1']//td[text()='Smith']/following-sibling::td[1]"));

        Console.WriteLine($"Last name: {lastNameCell.Text}");
        Console.WriteLine($"First name: {firstNameCell.Text}");
    }

    [Test]
    public void XPath_WildcardAndIndex()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/tables");

        // Wildcard: * matches any tag
        IWebElement anyElement = driver.FindElement(
            By.XPath("//*[@id='table1']"));
        Console.WriteLine($"Wildcard found: {anyElement.TagName}");

        // Index: [n] selects the nth match (1-based)
        IWebElement secondRow = driver.FindElement(
            By.XPath("//table[@id='table1']//tbody/tr[2]"));

        IList<IWebElement> cells = secondRow.FindElements(By.TagName("td"));
        Console.WriteLine($"Second row, first cell: {cells[0].Text}");
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
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


@pytest.fixture
def driver():
    """Create a ChromeDriver, yield it, then quit."""
    options = Options()
    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


def test_basic_xpath_attribute_match(driver):
    """Find elements using attribute-match XPath."""
    driver.get("https://the-internet.herokuapp.com/login")

    # XPath: find input where id attribute equals 'username'
    username = driver.find_element(By.XPATH, "//input[@id='username']")
    username.send_keys("tomsmith")

    # XPath: find input where type attribute equals 'password'
    password = driver.find_element(By.XPATH, "//input[@type='password']")
    password.send_keys("SuperSecretPassword!")

    print("Fields populated using attribute-match XPaths")
    assert username.get_attribute("value") == "tomsmith"


def test_xpath_text_match(driver):
    """Find an element by its visible text content."""
    driver.get("https://the-internet.herokuapp.com/login")

    # text() matches the exact visible text of an element
    heading = driver.find_element(By.XPATH, "//h2[text()='Login Page']")

    print(f"Found heading: {heading.text}")
    assert heading.text == "Login Page"


def test_xpath_contains_function(driver):
    """Use contains() for partial attribute or text matching."""
    driver.get("https://the-internet.herokuapp.com/login")

    # contains(@attribute, 'partial') matches partial attribute values
    button = driver.find_element(
        By.XPATH, "//button[contains(@class, 'radius')]"
    )
    print(f"Button text: {button.text}")
    assert "Login" in button.text

    # contains(text(), 'partial') matches partial text content
    page_text = driver.find_element(
        By.XPATH, "//*[contains(text(), 'Username')]"
    )
    print(f"Found element containing 'Username': {page_text.tag_name}")


def test_xpath_starts_with_function(driver):
    """Use starts-with() to match the beginning of attribute values."""
    driver.get("https://the-internet.herokuapp.com/login")

    # Useful for dynamic IDs like "user_12345"
    username_field = driver.find_element(
        By.XPATH, "//input[starts-with(@id, 'user')]"
    )
    username_field.send_keys("tomsmith")

    actual_id = username_field.get_attribute("id")
    print(f"Found field starting with 'user': {actual_id}")
    assert actual_id == "username"


def test_xpath_multiple_conditions(driver):
    """Combine conditions with 'and' / 'or'."""
    driver.get("https://the-internet.herokuapp.com/login")

    # 'and' — both conditions must be true
    username_input = driver.find_element(
        By.XPATH, "//input[@type='text' and @id='username']"
    )
    print(f"Found with 'and': id={username_input.get_attribute('id')}")
    assert username_input.get_attribute("id") == "username"

    # 'or' — either condition can be true
    inputs = driver.find_elements(
        By.XPATH, "//input[@id='username' or @id='password']"
    )
    print(f"Found {len(inputs)} elements matching 'or' condition")
    assert len(inputs) == 2


def test_xpath_axes_parent_and_sibling(driver):
    """Use XPath axes to navigate between related elements."""
    driver.get("https://the-internet.herokuapp.com/tables")

    # ancestor:: navigates up the tree to find a parent/grandparent
    row = driver.find_element(
        By.XPATH, "//td[text()='Smith']/ancestor::tr"
    )
    cells = row.find_elements(By.TAG_NAME, "td")
    print(f"Row has {len(cells)} cells")

    # following-sibling:: finds elements at the same level, after current
    last_name_cell = driver.find_element(
        By.XPATH, "//table[@id='table1']//td[text()='Smith']"
    )
    first_name_cell = driver.find_element(
        By.XPATH,
        "//table[@id='table1']//td[text()='Smith']/following-sibling::td[1]",
    )

    print(f"Last name: {last_name_cell.text}")
    print(f"First name: {first_name_cell.text}")


def test_xpath_wildcard_and_index(driver):
    """Use wildcard (*) and index ([n]) in XPath."""
    driver.get("https://the-internet.herokuapp.com/tables")

    # * matches any tag name
    any_element = driver.find_element(By.XPATH, "//*[@id='table1']")
    print(f"Wildcard found: {any_element.tag_name}")

    # [n] selects the nth match (1-based indexing)
    second_row = driver.find_element(
        By.XPATH, "//table[@id='table1']//tbody/tr[2]"
    )
    cells = second_row.find_elements(By.TAG_NAME, "td")
    print(f"Second row, first cell: {cells[0].text}")
```

---

## Step-by-Step Walkthrough

### Building an XPath step by step:

**Goal:** Find the "Edit" link in the first row of a table.

1. Start broad: `//a` — matches all links on the page
2. Add text filter: `//a[text()='edit']` — matches links with text "edit"
3. Too many matches? Scope to the table: `//table[@id='table1']//a[text()='edit']`
4. Need a specific row? Add position: `//table[@id='table1']//tbody/tr[1]//a[text()='edit']`
5. Test it: press `Ctrl+F` in Chrome DevTools Elements panel, paste the XPath, confirm it highlights exactly one element

### Testing XPaths in Chrome DevTools:

1. Open DevTools (`F12`)
2. Go to the **Elements** tab
3. Press `Ctrl+F` — a search bar appears at the bottom
4. Type your XPath expression (e.g., `//input[@id='username']`)
5. DevTools shows "1 of 1" if exactly one match is found
6. The matching element is highlighted in the DOM tree
7. Refine your XPath until you get exactly one match

---

## Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using absolute XPath | `/html/body/div[2]/form/input[1]` breaks when any parent changes | Always use relative XPath starting with `//` |
| Forgetting XPath is 1-based | `//li[0]` matches nothing | XPath indexes start at 1: `//li[1]` |
| Using `text()` with whitespace | `//h2[text()='Login Page']` fails if HTML has `\n  Login Page\n` | Use `contains(text(), 'Login')` or `normalize-space()` |
| Confusing `contains(@class, ...)` with multi-class elements | `contains(@class, 'btn')` also matches `class="submit-btn"` | Be specific: `contains(@class, 'btn ')` (with trailing space) or use `contains(concat(' ', @class, ' '), ' btn ')` |
| Missing `//` for descendant search | `/div/input` only works from root | Use `//div//input` to search at any depth |

---

## Best Practices

1. **Prefer relative XPath over absolute** — absolute XPaths break with any structural change. Relative XPaths are resilient.

2. **Use attributes first, axes second** — `//input[@id='email']` is simpler and faster than `//label[text()='Email']/following-sibling::input`. Use axes only when attributes are insufficient.

3. **Prefer `contains()` for dynamic classes** — many frameworks generate classes like `btn-primary-active-12345`. Using `contains(@class, 'btn-primary')` survives the dynamic suffix.

4. **Keep XPaths short** — the longer the XPath, the more fragile it is. If your XPath has more than 3 levels, consider whether a simpler locator exists.

5. **Test in DevTools first** — always verify your XPath in the browser before writing code. This saves debugging time.

6. **Consider CSS selectors for simple cases** — XPath is powerful but CSS selectors are faster and more readable for simple attribute/class matching (covered tomorrow).

---

## Hands-On Exercise

### Task

Navigate to `https://the-internet.herokuapp.com/tables` and write tests that:

1. Find the table with `id="table1"` using XPath
2. Find all rows in the table body using `//table[@id='table1']//tbody/tr`
3. Print the number of rows
4. Find the email address in the row where last name is "Smith" using `following-sibling::`
5. Find all "edit" links in the table using `contains(text(), 'edit')`
6. Find the row where the due amount is "$100.00" using `ancestor::` to get the full row

### Expected Output

```
Table found: table
Number of data rows: 4
Smith's email: jsmith@gmail.com
Number of edit links: 4
Row with $100.00 due: Bach Jason jbach@gmail.com $100.00 http://www.jbach.com edit delete
```

---

## Real-World Scenario

You are testing an enterprise application with a data grid. The grid has no IDs on individual cells, and the class names are auto-generated (like `ag-cell-value-234`). The only stable references are the column headers and the data content.

You need to verify that when you search for employee "Jane Smith", her department shows as "Engineering".

**XPath solution:**

```
//td[text()='Jane Smith']/following-sibling::td[@data-column='department']
```

Or using `ancestor::` to get the whole row first:

```
//td[text()='Jane Smith']/ancestor::tr//td[contains(@class, 'department')]
```

XPath axes make this possible where simple ID/name locators cannot help. This is why XPath is an essential tool in every automation engineer's toolkit.

---

## Resources

- [XPath Tutorial (W3Schools)](https://www.w3schools.com/xml/xpath_intro.asp)
- [XPath Axes (W3Schools)](https://www.w3schools.com/xml/xpath_axes.asp)
- [XPath Functions Reference](https://www.w3schools.com/xml/xsl_functions.asp)
- [Selenium XPath Locators](https://www.selenium.dev/documentation/webdriver/elements/locators/#xpath)
- [Chrome DevTools — Search Elements](https://developer.chrome.com/docs/devtools/dom/#search)
- [XPath Cheat Sheet (Devhints)](https://devhints.io/xpath)
