# Day 4: CSS Selectors (Preferred in Production)

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain why CSS selectors are preferred over XPath in most production test suites
- Write CSS selectors using ID, class, tag, and attribute syntax
- Use attribute selectors for exact, partial, prefix, and suffix matching
- Chain selectors with combinators (descendant, child, adjacent, general sibling)
- Apply pseudo-classes to select elements by position
- Convert XPath expressions from Day 3 into equivalent CSS selectors

---

## Core Concept Explanation

### Why CSS Selectors Are Preferred

CSS selectors are the native language browsers use to apply styles. Because of this, browsers have highly optimized CSS selector engines. In production Selenium tests, CSS selectors offer:

| Advantage | Explanation |
|-----------|-------------|
| **Faster execution** | Browsers parse CSS selectors natively; XPath requires an additional engine |
| **More readable** | `#login .btn` is easier to read than `//div[@id='login']//button[contains(@class,'btn')]` |
| **Consistent across browsers** | CSS selector behavior is identical in all browsers; XPath has minor cross-browser quirks |
| **Familiar to developers** | Front-end developers already know CSS selectors from writing stylesheets |

### When to Still Use XPath

CSS selectors cannot do everything. Use XPath when you need:

- **Text matching** — CSS has no `text()` equivalent
- **Upward traversal** — CSS cannot select parent elements (no `parent::` or `ancestor::`)
- **Complex sibling logic** — CSS sibling selectors are limited compared to XPath axes

**Rule of thumb:** Use CSS selectors by default. Switch to XPath only when CSS cannot express what you need.

---

## How It Works (Technical Breakdown)

### CSS Selector Syntax

#### Basic Selectors

| Selector | Matches | Example |
|----------|---------|---------|
| `#id` | Element with specific ID | `#username` |
| `.class` | Elements with specific class | `.btn-primary` |
| `tag` | Elements of specific type | `input` |
| `tag#id` | Tag with specific ID | `input#username` |
| `tag.class` | Tag with specific class | `button.submit` |
| `.class1.class2` | Element with both classes | `.btn.primary` |

#### Attribute Selectors

| Selector | Match Type | Example |
|----------|-----------|---------|
| `[attr='value']` | Exact match | `[type='text']` |
| `[attr*='value']` | Contains substring | `[class*='error']` |
| `[attr^='value']` | Starts with | `[id^='user']` |
| `[attr$='value']` | Ends with | `[src$='.png']` |
| `[attr]` | Attribute exists (any value) | `[disabled]` |

#### Combinators

| Combinator | Meaning | Example |
|-----------|---------|---------|
| ` ` (space) | Descendant (any depth) | `form input` — input anywhere inside form |
| `>` | Direct child only | `form > input` — input directly inside form |
| `+` | Adjacent sibling (immediately after) | `label + input` — input right after label |
| `~` | General sibling (anywhere after) | `h2 ~ p` — any p after h2 at same level |

#### Pseudo-classes

| Pseudo-class | Matches | Example |
|-------------|---------|---------|
| `:first-child` | First child element | `li:first-child` |
| `:last-child` | Last child element | `li:last-child` |
| `:nth-child(n)` | Nth child (1-based) | `tr:nth-child(3)` |
| `:nth-child(odd)` | Odd-numbered children | `tr:nth-child(odd)` |
| `:nth-child(even)` | Even-numbered children | `tr:nth-child(even)` |
| `:not(selector)` | Elements NOT matching | `input:not([type='hidden'])` |

---

## Code Example: C# (Complete, Runnable)

```csharp
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using System;
using System.Collections.Generic;

namespace SeleniumCourse;

[TestFixture]
public class Day04_CssSelectors
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
    public void BasicSelectors_IdClassTag()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // CSS by ID: #id
        IWebElement usernameField = driver.FindElement(
            By.CssSelector("#username"));
        usernameField.SendKeys("tomsmith");

        // CSS by tag + attribute: input[name='password']
        IWebElement passwordField = driver.FindElement(
            By.CssSelector("input[name='password']"));
        passwordField.SendKeys("SuperSecretPassword!");

        // CSS by class: .classname
        IWebElement loginButton = driver.FindElement(
            By.CssSelector(".radius"));

        Console.WriteLine($"Username value: {usernameField.GetAttribute("value")}");
        Console.WriteLine($"Button text: {loginButton.Text}");

        Assert.That(usernameField.GetAttribute("value"), Is.EqualTo("tomsmith"));
    }

    [Test]
    public void AttributeSelectors()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Exact attribute match: [attr='value']
        IWebElement textInput = driver.FindElement(
            By.CssSelector("input[type='text']"));
        Console.WriteLine($"Exact match found: {textInput.GetAttribute("id")}");

        // Starts with: [attr^='value']
        IWebElement startsWithUser = driver.FindElement(
            By.CssSelector("input[id^='user']"));
        Console.WriteLine($"Starts with 'user': {startsWithUser.GetAttribute("id")}");

        // Contains substring: [attr*='value']
        IWebElement containsName = driver.FindElement(
            By.CssSelector("input[id*='erna']"));
        Console.WriteLine($"Contains 'erna': {containsName.GetAttribute("id")}");

        // Ends with: [attr$='value']
        IWebElement endsWithWord = driver.FindElement(
            By.CssSelector("input[id$='word']"));
        Console.WriteLine($"Ends with 'word': {endsWithWord.GetAttribute("id")}");

        // Attribute exists (any value): [attr]
        IList<IWebElement> withType = driver.FindElements(
            By.CssSelector("input[type]"));
        Console.WriteLine($"Inputs with 'type' attribute: {withType.Count}");
    }

    [Test]
    public void CombinatorSelectors()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/tables");

        // Descendant (space): find any input inside the table at any depth
        IList<IWebElement> allCells = driver.FindElements(
            By.CssSelector("#table1 td"));
        Console.WriteLine($"Total cells in table1 (descendant): {allCells.Count}");

        // Direct child (>): tbody directly inside table
        IWebElement tbody = driver.FindElement(
            By.CssSelector("#table1 > tbody"));
        Console.WriteLine($"Direct child tbody found: {tbody.TagName}");

        // Adjacent sibling (+): element immediately after another
        // Find the table that comes right after an h3
        IWebElement tableAfterH3 = driver.FindElement(
            By.CssSelector("h3 + table"));
        Console.WriteLine($"Table after h3: id={tableAfterH3.GetAttribute("id")}");
    }

    [Test]
    public void PseudoClassSelectors()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/tables");

        // :first-child — select the first row in tbody
        IWebElement firstRow = driver.FindElement(
            By.CssSelector("#table1 tbody tr:first-child"));
        string firstRowText = firstRow.Text;
        Console.WriteLine($"First row: {firstRowText}");

        // :last-child — select the last row in tbody
        IWebElement lastRow = driver.FindElement(
            By.CssSelector("#table1 tbody tr:last-child"));
        Console.WriteLine($"Last row: {lastRow.Text}");

        // :nth-child(n) — select a specific row (1-based)
        IWebElement secondRow = driver.FindElement(
            By.CssSelector("#table1 tbody tr:nth-child(2)"));
        Console.WriteLine($"Second row: {secondRow.Text}");

        // :not() — find inputs that are NOT of type hidden
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");
        IList<IWebElement> visibleInputs = driver.FindElements(
            By.CssSelector("input:not([type='hidden'])"));
        Console.WriteLine($"Visible inputs: {visibleInputs.Count}");
    }

    [Test]
    public void MultipleClassSelector()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Combining tag + multiple attributes
        // input that is type text AND has id username
        IWebElement specificInput = driver.FindElement(
            By.CssSelector("input[type='text'][id='username']"));

        Console.WriteLine($"Found: {specificInput.GetAttribute("id")}");
        Assert.That(specificInput.GetAttribute("id"), Is.EqualTo("username"));
    }

    [Test]
    public void ComplexSelector_RealWorldExample()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/tables");

        // Complex: find all edit links inside table1 rows
        IList<IWebElement> editLinks = driver.FindElements(
            By.CssSelector("#table1 tbody tr td a[href*='edit']"));
        Console.WriteLine($"Edit links in table1: {editLinks.Count}");

        // Get the third row's last cell
        IWebElement thirdRowAction = driver.FindElement(
            By.CssSelector("#table1 tbody tr:nth-child(3) td:last-child"));
        Console.WriteLine($"Third row actions: {thirdRowAction.Text}");
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
    """Create a ChromeDriver, yield it, then quit."""
    options = Options()
    drv = webdriver.Chrome(options=options)
    drv.implicitly_wait(5)
    yield drv
    drv.quit()


def test_basic_selectors_id_class_tag(driver):
    """Use #id, .class, and tag selectors."""
    driver.get("https://the-internet.herokuapp.com/login")

    # CSS by ID: #id
    username_field = driver.find_element(By.CSS_SELECTOR, "#username")
    username_field.send_keys("tomsmith")

    # CSS by tag + attribute: input[name='password']
    password_field = driver.find_element(
        By.CSS_SELECTOR, "input[name='password']"
    )
    password_field.send_keys("SuperSecretPassword!")

    # CSS by class: .classname
    login_button = driver.find_element(By.CSS_SELECTOR, ".radius")

    print(f"Username value: {username_field.get_attribute('value')}")
    print(f"Button text: {login_button.text}")
    assert username_field.get_attribute("value") == "tomsmith"


def test_attribute_selectors(driver):
    """Use attribute selectors: exact, contains, starts-with, ends-with."""
    driver.get("https://the-internet.herokuapp.com/login")

    # Exact match: [attr='value']
    text_input = driver.find_element(By.CSS_SELECTOR, "input[type='text']")
    print(f"Exact match found: {text_input.get_attribute('id')}")

    # Starts with: [attr^='value']
    starts_with = driver.find_element(By.CSS_SELECTOR, "input[id^='user']")
    print(f"Starts with 'user': {starts_with.get_attribute('id')}")

    # Contains: [attr*='value']
    contains = driver.find_element(By.CSS_SELECTOR, "input[id*='erna']")
    print(f"Contains 'erna': {contains.get_attribute('id')}")

    # Ends with: [attr$='value']
    ends_with = driver.find_element(By.CSS_SELECTOR, "input[id$='word']")
    print(f"Ends with 'word': {ends_with.get_attribute('id')}")

    # Attribute exists (any value): [attr]
    with_type = driver.find_elements(By.CSS_SELECTOR, "input[type]")
    print(f"Inputs with 'type' attribute: {len(with_type)}")


def test_combinator_selectors(driver):
    """Use descendant, child, and sibling combinators."""
    driver.get("https://the-internet.herokuapp.com/tables")

    # Descendant (space): any td inside #table1
    all_cells = driver.find_elements(By.CSS_SELECTOR, "#table1 td")
    print(f"Total cells in table1: {len(all_cells)}")

    # Direct child (>): tbody directly inside #table1
    tbody = driver.find_element(By.CSS_SELECTOR, "#table1 > tbody")
    print(f"Direct child tbody found: {tbody.tag_name}")

    # Adjacent sibling (+): table immediately after h3
    table_after_h3 = driver.find_element(By.CSS_SELECTOR, "h3 + table")
    print(f"Table after h3: id={table_after_h3.get_attribute('id')}")


def test_pseudo_class_selectors(driver):
    """Use :first-child, :last-child, :nth-child, :not pseudo-classes."""
    driver.get("https://the-internet.herokuapp.com/tables")

    # :first-child
    first_row = driver.find_element(
        By.CSS_SELECTOR, "#table1 tbody tr:first-child"
    )
    print(f"First row: {first_row.text}")

    # :last-child
    last_row = driver.find_element(
        By.CSS_SELECTOR, "#table1 tbody tr:last-child"
    )
    print(f"Last row: {last_row.text}")

    # :nth-child(n) — 1-based index
    second_row = driver.find_element(
        By.CSS_SELECTOR, "#table1 tbody tr:nth-child(2)"
    )
    print(f"Second row: {second_row.text}")

    # :not() — exclude elements matching a condition
    driver.get("https://the-internet.herokuapp.com/login")
    visible_inputs = driver.find_elements(
        By.CSS_SELECTOR, "input:not([type='hidden'])"
    )
    print(f"Visible inputs: {len(visible_inputs)}")


def test_multiple_attributes(driver):
    """Combine multiple attribute selectors on one element."""
    driver.get("https://the-internet.herokuapp.com/login")

    # Chain attribute selectors (AND logic)
    specific_input = driver.find_element(
        By.CSS_SELECTOR, "input[type='text'][id='username']"
    )
    print(f"Found: {specific_input.get_attribute('id')}")
    assert specific_input.get_attribute("id") == "username"


def test_complex_real_world_selector(driver):
    """Build complex selectors for table data."""
    driver.get("https://the-internet.herokuapp.com/tables")

    # All edit links inside table1 rows
    edit_links = driver.find_elements(
        By.CSS_SELECTOR, "#table1 tbody tr td a[href*='edit']"
    )
    print(f"Edit links in table1: {len(edit_links)}")

    # Third row's last cell
    third_row_action = driver.find_element(
        By.CSS_SELECTOR, "#table1 tbody tr:nth-child(3) td:last-child"
    )
    print(f"Third row actions: {third_row_action.text}")
```

---

## Step-by-Step Walkthrough

### Converting XPath to CSS Selector

Here is a side-by-side conversion table for common patterns:

| Goal | XPath | CSS Selector |
|------|-------|-------------|
| By ID | `//input[@id='username']` | `#username` or `input#username` |
| By class | `//div[@class='alert']` | `.alert` or `div.alert` |
| By attribute | `//input[@type='text']` | `input[type='text']` |
| Contains in attr | `//div[contains(@class, 'error')]` | `div[class*='error']` |
| Starts with | `//input[starts-with(@id, 'user')]` | `input[id^='user']` |
| Descendant | `//form//input` | `form input` |
| Direct child | `//form/input` | `form > input` |
| Multiple conditions | `//input[@type='text' and @name='q']` | `input[type='text'][name='q']` |
| Nth element | `//tr[3]` | `tr:nth-child(3)` |
| First element | `//li[1]` | `li:first-child` |
| Last element | `//li[last()]` | `li:last-child` |

### What CSS Selectors Cannot Do (Use XPath Instead)

| Goal | XPath | CSS Equivalent |
|------|-------|---------------|
| Match by text | `//a[text()='Login']` | Not possible |
| Navigate to parent | `//input/parent::div` | Not possible |
| Navigate to ancestor | `//td/ancestor::table` | Not possible |
| Preceding sibling | `//input/preceding-sibling::label` | Not possible |

---

## Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Space in class name | `.btn primary` means "element with class `primary` inside element with class `btn`" | Use `.btn.primary` (no space) for an element with both classes |
| Forgetting `#` for ID | `CssSelector("username")` searches for a `<username>` tag | Use `#username` for ID selectors |
| Using `:nth-child(0)` | CSS pseudo-classes are 1-based | Start counting from 1: `:nth-child(1)` |
| Over-specifying the selector | `div.container > div.row > div.col-md-6 > form > div > input#email` | Simplify to `#email` — shorter selectors are more resilient |
| Confusing `:nth-child` with `:nth-of-type` | `:nth-child(2)` selects the 2nd child regardless of type; `:nth-of-type(2)` selects the 2nd child of that specific type | Use `:nth-of-type` when mixed element types exist at the same level |

---

## Best Practices

1. **Use CSS selectors as your default locator** — they are faster, more readable, and natively supported by browsers.

2. **Prefer short selectors** — `#username` is better than `div.form-group > input#username`. The shorter the selector, the less likely it breaks when the page changes.

3. **Avoid chaining more than 3 levels** — `#content .table tbody tr:nth-child(2) td:first-child` is about the maximum readable length.

4. **Use attribute selectors for dynamic content** — when IDs contain dynamic parts (`user-12345`), use `[id^='user-']` to match the stable prefix.

5. **Combine with `By.Id` for simple cases** — there is no need to write `By.CssSelector("#username")` when `By.Id("username")` is available. Use CSS selectors when you need combinators or attribute matching.

6. **Test selectors in the browser console** — run `document.querySelectorAll("your-selector")` in the console to see what matches before writing test code.

---

## Hands-On Exercise

### Task

Navigate to `https://the-internet.herokuapp.com/tables` and write tests using only CSS selectors:

1. Select the first table by its ID
2. Count all rows in the first table's body
3. Select the third row using `:nth-child(3)`
4. Select all cells in the "Email" column (3rd column) using `:nth-child(3)`
5. Select all "delete" links using an attribute selector on the href
6. Select the last row's first cell using `:last-child` and `:first-child`

### Expected Output

```
Table found: table1
Number of body rows: 4
Third row text: Conway Tim tconway@earthlink.net $50.00 http://www.timconway.com edit delete
Email addresses: ['jsmith@gmail.com', 'fbach@yahoo.com', 'tconway@earthlink.net', 'jdoe@hotmail.com']
Number of delete links: 4
Last row, first cell: Doe
```

---

## Real-World Scenario

Your company's web application uses a React framework. The HTML looks like this:

```html
<div data-testid="user-table">
  <div class="table-row" data-row-id="usr_001">
    <span class="cell name">Alice Johnson</span>
    <span class="cell email">alice@company.com</span>
    <span class="cell role">Admin</span>
    <button class="btn btn-danger" data-action="delete">Delete</button>
  </div>
  <div class="table-row" data-row-id="usr_002">
    ...
  </div>
</div>
```

CSS selectors handle this elegantly:

```
// Find the user table
[data-testid='user-table']

// Find all rows
[data-testid='user-table'] .table-row

// Find the delete button in the first row
[data-testid='user-table'] .table-row:first-child .btn-danger

// Find a specific row by data attribute
.table-row[data-row-id='usr_001']

// Find the email cell in that row
.table-row[data-row-id='usr_001'] .cell.email
```

React applications commonly use `data-testid` attributes specifically for test automation. CSS selectors with `[data-testid='...']` are the cleanest way to use them.

---

## CSS vs XPath Comparison Summary

| Feature | CSS Selector | XPath |
|---------|-------------|-------|
| Speed | Faster (native browser engine) | Slightly slower |
| Readability | More concise | More verbose |
| Text matching | Not supported | `text()`, `contains(text(), ...)` |
| Upward traversal | Not supported | `parent::`, `ancestor::` |
| Attribute matching | Full support | Full support |
| Position/index | `:nth-child(n)` | `[n]` |
| Negation | `:not(selector)` | `not()` |
| Browser consistency | Excellent | Good (minor quirks in IE) |
| **Recommendation** | **Use by default** | **Use when CSS is insufficient** |

---

## Resources

- [CSS Selectors Reference (MDN)](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_selectors)
- [CSS Selector Tester (w3schools)](https://www.w3schools.com/cssref/trysel.php)
- [Selenium CSS Selector Locators](https://www.selenium.dev/documentation/webdriver/elements/locators/#css-selector)
- [CSS Selectors Cheat Sheet](https://devhints.io/css)
- [document.querySelectorAll (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelectorAll)
