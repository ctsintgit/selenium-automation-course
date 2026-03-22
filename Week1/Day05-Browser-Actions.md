# Day 5: Browser Actions — Click, SendKeys, Clear, Submit

## Learning Objectives

By the end of this lesson, you will be able to:

- Use `Click()`, `SendKeys()`, `Clear()`, and `Submit()` to interact with page elements
- Send special keys (Enter, Tab, Escape) using the `Keys` class
- Use the Actions class (C#) / ActionChains (Python) for complex interactions
- Perform hover, right-click, double-click, and drag-and-drop operations
- Combine multiple actions into fluent action sequences

---

## Core Concept Explanation

### The IWebElement / WebElement Interface

Once you locate an element with `FindElement`, you get back an element object. This object exposes methods to interact with the element:

| Method | What It Does | Typical Use |
|--------|-------------|-------------|
| `Click()` | Simulates a mouse click | Buttons, links, checkboxes, radio buttons |
| `SendKeys(text)` | Types text into the element | Input fields, text areas |
| `Clear()` | Clears existing text | Input fields before retyping |
| `Submit()` | Submits the containing form | Any element inside a `<form>` |

### Properties You Can Read

| Property (C#) | Property (Python) | Returns |
|--------------|-------------------|---------|
| `.Text` | `.text` | Visible text content |
| `.Enabled` | `.is_enabled()` | Whether the element is interactive |
| `.Displayed` | `.is_displayed()` | Whether the element is visible |
| `.Selected` | `.is_selected()` | Whether checkbox/radio is checked |
| `.TagName` | `.tag_name` | The HTML tag (e.g., `"input"`) |
| `.GetAttribute("name")` | `.get_attribute("name")` | Value of an HTML attribute |

---

## How It Works (Technical Breakdown)

### Click Flow

1. Selenium sends a POST request to the browser driver to click the element
2. The driver scrolls the element into view if needed
3. The driver calculates the element's center coordinates
4. The driver dispatches a mouse click event at those coordinates
5. If the element is obscured by another element, a `ElementClickInterceptedException` is thrown

### SendKeys Flow

1. Selenium sends a POST request with the text characters
2. The driver focuses the element (if not already focused)
3. Each character is dispatched as a keydown/keypress/keyup event
4. Special keys (Enter, Tab) are sent as their key codes

### Actions Class / ActionChains

For interactions that go beyond simple click and type, Selenium provides an advanced API:

- **C#:** `Actions` class in `OpenQA.Selenium.Interactions`
- **Python:** `ActionChains` class in `selenium.webdriver.common.action_chains`

These let you build a sequence of low-level actions (move mouse, press button, hold key) and execute them as a batch.

---

## Code Example: C# (Complete, Runnable)

```csharp
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Interactions;
using OpenQA.Selenium.Support.UI;
using System;

namespace SeleniumCourse;

[TestFixture]
public class Day05_BrowserActions
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
    public void ClickAction()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/add_remove_elements/");

        // Find the "Add Element" button and click it
        IWebElement addButton = driver.FindElement(By.CssSelector("button[onclick='addElement()']"));
        addButton.Click();

        // Verify a new "Delete" button appeared
        IWebElement deleteButton = wait.Until(d =>
            d.FindElement(By.CssSelector(".added-manually")));

        Console.WriteLine($"Delete button text: {deleteButton.Text}");
        Assert.That(deleteButton.Displayed, Is.True);

        // Click "Add Element" two more times
        addButton.Click();
        addButton.Click();

        // Verify three delete buttons now exist
        var deleteButtons = driver.FindElements(By.CssSelector(".added-manually"));
        Console.WriteLine($"Number of delete buttons: {deleteButtons.Count}");
        Assert.That(deleteButtons.Count, Is.EqualTo(3));
    }

    [Test]
    public void SendKeysAndClear()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Find the username field
        IWebElement usernameField = driver.FindElement(By.Id("username"));

        // Type text into the field
        usernameField.SendKeys("wronguser");
        Console.WriteLine($"After SendKeys: {usernameField.GetAttribute("value")}");

        // Clear the field — removes all existing text
        usernameField.Clear();
        Console.WriteLine($"After Clear: '{usernameField.GetAttribute("value")}'");

        // Type the correct username
        usernameField.SendKeys("tomsmith");
        Console.WriteLine($"After re-type: {usernameField.GetAttribute("value")}");

        Assert.That(usernameField.GetAttribute("value"), Is.EqualTo("tomsmith"));
    }

    [Test]
    public void SendSpecialKeys()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        IWebElement usernameField = driver.FindElement(By.Id("username"));
        IWebElement passwordField = driver.FindElement(By.Id("password"));

        // Type username
        usernameField.SendKeys("tomsmith");

        // Press TAB to move to the next field
        usernameField.SendKeys(Keys.Tab);

        // The password field should now be focused — type into it
        passwordField.SendKeys("SuperSecretPassword!");

        // Press ENTER to submit the form (instead of clicking the button)
        passwordField.SendKeys(Keys.Enter);

        // Wait for the success message
        IWebElement flash = wait.Until(d =>
            d.FindElement(By.Id("flash")));

        Console.WriteLine($"Flash message: {flash.Text}");
        Assert.That(flash.Text, Does.Contain("You logged into a secure area!"));
    }

    [Test]
    public void SubmitForm()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        // Fill in the form
        driver.FindElement(By.Id("username")).SendKeys("tomsmith");
        driver.FindElement(By.Id("password")).SendKeys("SuperSecretPassword!");

        // Submit() can be called on any element inside a <form>
        // It submits the form that contains the element
        driver.FindElement(By.Id("username")).Submit();

        // Wait for result
        IWebElement flash = wait.Until(d =>
            d.FindElement(By.Id("flash")));

        Assert.That(flash.Text, Does.Contain("You logged into a secure area!"));
    }

    [Test]
    public void HoverAction()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/hovers");

        // Find the first avatar image
        IWebElement firstAvatar = driver.FindElements(
            By.CssSelector(".figure"))[0];

        // Create an Actions object to perform the hover
        Actions actions = new Actions(driver);

        // MoveToElement performs a mouse hover
        actions.MoveToElement(firstAvatar).Perform();

        // After hovering, a caption should appear
        IWebElement caption = wait.Until(d =>
        {
            var el = d.FindElement(By.CssSelector(".figure:first-child .figcaption h5"));
            return el.Displayed ? el : null;
        });

        Console.WriteLine($"Hover revealed: {caption.Text}");
        Assert.That(caption.Text, Does.Contain("user1"));
    }

    [Test]
    public void RightClickAction()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/context_menu");

        // Find the area where right-click triggers a context menu
        IWebElement hotSpot = driver.FindElement(By.Id("hot-spot"));

        // Perform a right-click (context click)
        Actions actions = new Actions(driver);
        actions.ContextClick(hotSpot).Perform();

        // An alert dialog should appear after right-clicking
        IAlert alert = wait.Until(d =>
        {
            try { return d.SwitchTo().Alert(); }
            catch { return null; }
        });

        Console.WriteLine($"Alert text: {alert.Text}");
        Assert.That(alert.Text, Does.Contain("You selected a context menu"));

        // Dismiss the alert
        alert.Accept();
    }

    [Test]
    public void DoubleClickAction()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

        // Navigate to a page (using the-internet as base)
        // For demonstration, double-click an element
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/add_remove_elements/");

        IWebElement addButton = driver.FindElement(
            By.CssSelector("button[onclick='addElement()']"));

        // Double-click adds the element twice in rapid succession
        Actions actions = new Actions(driver);
        actions.DoubleClick(addButton).Perform();

        // Verify elements were added
        var deleteButtons = driver.FindElements(By.CssSelector(".added-manually"));
        Console.WriteLine($"Elements after double-click: {deleteButtons.Count}");
    }

    [Test]
    public void DragAndDropAction()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/drag_and_drop");

        // Find source and target columns
        IWebElement columnA = driver.FindElement(By.Id("column-a"));
        IWebElement columnB = driver.FindElement(By.Id("column-b"));

        string beforeDrag = columnA.Text;
        Console.WriteLine($"Column A before drag: {beforeDrag}");

        // Perform drag and drop
        Actions actions = new Actions(driver);
        actions.DragAndDrop(columnA, columnB).Perform();

        // Note: DragAndDrop may not work on all sites due to HTML5 drag/drop
        // implementation differences. JavaScript-based drag/drop may be needed.
        string afterDrag = driver.FindElement(By.Id("column-a")).Text;
        Console.WriteLine($"Column A after drag: {afterDrag}");
    }

    [Test]
    public void ActionChain_CombinedSequence()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/login");

        IWebElement usernameField = driver.FindElement(By.Id("username"));
        IWebElement passwordField = driver.FindElement(By.Id("password"));
        IWebElement loginButton = driver.FindElement(By.CssSelector("button.radius"));

        // Build a sequence of actions and execute them all at once
        Actions actions = new Actions(driver);
        actions
            .Click(usernameField)               // Click into username field
            .SendKeys("tomsmith")               // Type username
            .Click(passwordField)               // Click into password field
            .SendKeys("SuperSecretPassword!")    // Type password
            .Click(loginButton)                 // Click login button
            .Perform();                         // Execute the entire sequence

        // Wait for result
        IWebElement flash = wait.Until(d =>
            d.FindElement(By.Id("flash")));

        Console.WriteLine($"Login result: {flash.Text}");
        Assert.That(flash.Text, Does.Contain("You logged into a secure area!"));
    }

    [Test]
    public void KeyCombinations()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/inputs");

        IWebElement inputField = driver.FindElement(By.CssSelector("input[type='number']"));
        inputField.SendKeys("12345");

        // Select all text with Ctrl+A
        Actions actions = new Actions(driver);
        actions
            .Click(inputField)
            .KeyDown(Keys.Control)
            .SendKeys("a")
            .KeyUp(Keys.Control)
            .Perform();

        // Type new text (replaces the selected text)
        inputField.SendKeys("67890");

        Console.WriteLine($"Field value after replace: {inputField.GetAttribute("value")}");
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
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


@pytest.fixture
def driver():
    """Create a ChromeDriver, yield it, then quit."""
    options = Options()
    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


def test_click_action(driver):
    """Click buttons and verify results."""
    driver.get("https://the-internet.herokuapp.com/add_remove_elements/")

    # Find and click the "Add Element" button
    add_button = driver.find_element(
        By.CSS_SELECTOR, "button[onclick='addElement()']"
    )
    add_button.click()

    # Verify a new "Delete" button appeared
    wait = WebDriverWait(driver, 10)
    delete_button = wait.until(
        EC.presence_of_element_located((By.CSS_SELECTOR, ".added-manually"))
    )

    print(f"Delete button text: {delete_button.text}")
    assert delete_button.is_displayed()

    # Click two more times
    add_button.click()
    add_button.click()

    delete_buttons = driver.find_elements(By.CSS_SELECTOR, ".added-manually")
    print(f"Number of delete buttons: {len(delete_buttons)}")
    assert len(delete_buttons) == 3


def test_send_keys_and_clear(driver):
    """Type text, clear, and retype."""
    driver.get("https://the-internet.herokuapp.com/login")

    username_field = driver.find_element(By.ID, "username")

    # Type text
    username_field.send_keys("wronguser")
    print(f"After send_keys: {username_field.get_attribute('value')}")

    # Clear the field
    username_field.clear()
    print(f"After clear: '{username_field.get_attribute('value')}'")

    # Retype
    username_field.send_keys("tomsmith")
    print(f"After re-type: {username_field.get_attribute('value')}")

    assert username_field.get_attribute("value") == "tomsmith"


def test_send_special_keys(driver):
    """Use Keys.TAB, Keys.ENTER for keyboard navigation."""
    driver.get("https://the-internet.herokuapp.com/login")

    username_field = driver.find_element(By.ID, "username")
    password_field = driver.find_element(By.ID, "password")

    # Type username
    username_field.send_keys("tomsmith")

    # Press TAB to move to password field
    username_field.send_keys(Keys.TAB)

    # Type password
    password_field.send_keys("SuperSecretPassword!")

    # Press ENTER to submit (instead of clicking the button)
    password_field.send_keys(Keys.ENTER)

    # Wait for the success message
    wait = WebDriverWait(driver, 10)
    flash = wait.until(EC.presence_of_element_located((By.ID, "flash")))

    print(f"Flash message: {flash.text}")
    assert "You logged into a secure area!" in flash.text


def test_submit_form(driver):
    """Use submit() to submit a form."""
    driver.get("https://the-internet.herokuapp.com/login")

    driver.find_element(By.ID, "username").send_keys("tomsmith")
    driver.find_element(By.ID, "password").send_keys("SuperSecretPassword!")

    # submit() submits the form containing the element
    driver.find_element(By.ID, "username").submit()

    wait = WebDriverWait(driver, 10)
    flash = wait.until(EC.presence_of_element_located((By.ID, "flash")))
    assert "You logged into a secure area!" in flash.text


def test_hover_action(driver):
    """Hover over an element to reveal hidden content."""
    driver.get("https://the-internet.herokuapp.com/hovers")

    # Find the first avatar
    first_avatar = driver.find_elements(By.CSS_SELECTOR, ".figure")[0]

    # Perform hover using ActionChains
    actions = ActionChains(driver)
    actions.move_to_element(first_avatar).perform()

    # After hovering, a caption should appear
    wait = WebDriverWait(driver, 10)
    caption = wait.until(
        EC.visibility_of_element_located(
            (By.CSS_SELECTOR, ".figure:first-child .figcaption h5")
        )
    )

    print(f"Hover revealed: {caption.text}")
    assert "user1" in caption.text


def test_right_click_action(driver):
    """Perform a right-click (context click) on an element."""
    driver.get("https://the-internet.herokuapp.com/context_menu")

    hot_spot = driver.find_element(By.ID, "hot-spot")

    # Right-click using ActionChains
    actions = ActionChains(driver)
    actions.context_click(hot_spot).perform()

    # Handle the alert dialog
    wait = WebDriverWait(driver, 10)
    alert = wait.until(EC.alert_is_present())

    print(f"Alert text: {alert.text}")
    assert "You selected a context menu" in alert.text

    alert.accept()


def test_double_click_action(driver):
    """Perform a double-click on an element."""
    driver.get("https://the-internet.herokuapp.com/add_remove_elements/")

    add_button = driver.find_element(
        By.CSS_SELECTOR, "button[onclick='addElement()']"
    )

    # Double-click
    actions = ActionChains(driver)
    actions.double_click(add_button).perform()

    delete_buttons = driver.find_elements(By.CSS_SELECTOR, ".added-manually")
    print(f"Elements after double-click: {len(delete_buttons)}")


def test_drag_and_drop_action(driver):
    """Drag an element and drop it onto another."""
    driver.get("https://the-internet.herokuapp.com/drag_and_drop")

    column_a = driver.find_element(By.ID, "column-a")
    column_b = driver.find_element(By.ID, "column-b")

    before_drag = column_a.text
    print(f"Column A before drag: {before_drag}")

    # Perform drag and drop
    actions = ActionChains(driver)
    actions.drag_and_drop(column_a, column_b).perform()

    after_drag = driver.find_element(By.ID, "column-a").text
    print(f"Column A after drag: {after_drag}")


def test_action_chain_combined_sequence(driver):
    """Build a sequence of actions and execute them together."""
    driver.get("https://the-internet.herokuapp.com/login")

    username_field = driver.find_element(By.ID, "username")
    password_field = driver.find_element(By.ID, "password")
    login_button = driver.find_element(By.CSS_SELECTOR, "button.radius")

    # Chain multiple actions into one sequence
    actions = ActionChains(driver)
    actions \
        .click(username_field) \
        .send_keys("tomsmith") \
        .click(password_field) \
        .send_keys("SuperSecretPassword!") \
        .click(login_button) \
        .perform()

    wait = WebDriverWait(driver, 10)
    flash = wait.until(EC.presence_of_element_located((By.ID, "flash")))

    print(f"Login result: {flash.text}")
    assert "You logged into a secure area!" in flash.text


def test_key_combinations(driver):
    """Use keyboard shortcuts like Ctrl+A."""
    driver.get("https://the-internet.herokuapp.com/inputs")

    input_field = driver.find_element(By.CSS_SELECTOR, "input[type='number']")
    input_field.send_keys("12345")

    # Select all text with Ctrl+A
    actions = ActionChains(driver)
    actions \
        .click(input_field) \
        .key_down(Keys.CONTROL) \
        .send_keys("a") \
        .key_up(Keys.CONTROL) \
        .perform()

    # Type new text to replace selected text
    input_field.send_keys("67890")

    print(f"Field value after replace: {input_field.get_attribute('value')}")
```

---

## Step-by-Step Walkthrough

### The login flow broken down:

1. **`Clear()`** — always clear a field before typing, in case it has a default value or previous input
2. **`SendKeys("text")`** — type the text character by character; the browser processes each keystroke
3. **`Click()`** — Selenium scrolls the button into view, calculates its center, and dispatches a click event
4. After click, the page navigates — you must wait for the new page to load before asserting

### Action chains explained:

Action chains work like a recording:

```
actions.click(field)        # Record: click the field
       .send_keys("text")  # Record: type "text"
       .click(button)      # Record: click the button
       .perform()          # Play: execute all recorded actions in order
```

Nothing happens until `.Perform()` / `.perform()` is called. This ensures all actions execute as one atomic sequence.

---

## Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Not clearing before typing | Previous text remains: `"oldtextnewtext"` | Always call `Clear()` before `SendKeys()` |
| Clicking a hidden element | `ElementNotInteractableException` | Use explicit wait for visibility: `wait.Until(EC.ElementToBeClickable(...))` |
| Using `Thread.Sleep` before clicking | Unreliable timing, slows tests | Use explicit waits with `WebDriverWait` |
| Forgetting `.Perform()` on Actions | Nothing happens — no error, no action | Always end action chains with `.Perform()` / `.perform()` |
| Sending Keys.ENTER on non-form elements | May not submit anything | Use `Click()` on the submit button, or call `Submit()` on a form element |
| Calling `Submit()` on elements outside a form | `NoSuchElementException` or unexpected behavior | Ensure the element is inside a `<form>` tag |

---

## Best Practices

1. **Always clear before typing** — `Clear()` then `SendKeys()` is the safe pattern. Never assume a field is empty.

2. **Use explicit waits before interacting** — wait for `ElementToBeClickable` before clicking, not just `PresenceOfElement`.

3. **Prefer `Click()` over `Submit()`** — `Click()` on the submit button is more explicit and matches user behavior. `Submit()` is a shortcut that may behave differently.

4. **Use Actions for complex interactions only** — simple click and type operations do not need `Actions`/`ActionChains`. Reserve them for hover, drag-and-drop, and keyboard shortcuts.

5. **Handle alerts immediately** — if a click triggers a JavaScript alert, switch to it with `driver.SwitchTo().Alert()` before interacting with any other element.

---

## Hands-On Exercise

### Task

Automate the login form at `https://the-internet.herokuapp.com/login`:

1. Clear the username field, then type `"tomsmith"`
2. Clear the password field, then type `"SuperSecretPassword!"`
3. Use `Keys.TAB` to navigate between fields (do not click the password field directly)
4. Use `Keys.ENTER` to submit the form (do not click the login button)
5. Verify the success message contains `"You logged into a secure area!"`
6. Find and click the **Logout** button
7. Verify you are back on the login page

### Expected Output

```
Username entered via Tab navigation
Password entered
Form submitted via Enter key
Login success message: You logged into a secure area!
Logout clicked
Back on login page: True
```

---

## Real-World Scenario

You are testing a rich text editor (like TinyMCE or CKEditor) embedded in a web application. Users format text using keyboard shortcuts:

- `Ctrl+B` for bold
- `Ctrl+I` for italic
- `Ctrl+A` to select all

The Actions/ActionChains API lets you automate these combinations:

```csharp
// C#: Bold a word in an editor
Actions actions = new Actions(driver);
actions
    .Click(editorBody)
    .SendKeys("Important text")
    .KeyDown(Keys.Control).SendKeys("a").KeyUp(Keys.Control)  // Select all
    .KeyDown(Keys.Control).SendKeys("b").KeyUp(Keys.Control)  // Bold
    .Perform();
```

```python
# Python: Bold a word in an editor
actions = ActionChains(driver)
actions \
    .click(editor_body) \
    .send_keys("Important text") \
    .key_down(Keys.CONTROL).send_keys("a").key_up(Keys.CONTROL) \
    .key_down(Keys.CONTROL).send_keys("b").key_up(Keys.CONTROL) \
    .perform()
```

Without the Actions API, automating keyboard shortcuts would be impossible.

---

## Resources

- [Selenium WebElement Interactions](https://www.selenium.dev/documentation/webdriver/elements/interactions/)
- [Selenium Actions API](https://www.selenium.dev/documentation/webdriver/actions_api/)
- [Keyboard Actions](https://www.selenium.dev/documentation/webdriver/actions_api/keyboard/)
- [Mouse Actions](https://www.selenium.dev/documentation/webdriver/actions_api/mouse/)
- [Keys Enum Reference (C#)](https://www.selenium.dev/selenium/docs/api/dotnet/html/T_OpenQA_Selenium_Keys.htm)
- [Keys Class Reference (Python)](https://www.selenium.dev/selenium/docs/api/py/webdriver/selenium.webdriver.common.keys.html)
- [Practice Site: The Internet](https://the-internet.herokuapp.com/)
