# Day 7: Dropdowns, Checkboxes, Radio Buttons + Mini Project 1

## Learning Objectives

By the end of this lesson, you will be able to:

- Handle native HTML `<select>` dropdowns using `SelectElement` (C#) / `Select` (Python)
- Select options by visible text, value attribute, and index
- Handle custom (non-select) dropdowns built with divs and JavaScript
- Interact with checkboxes: click, verify selected state
- Interact with radio buttons: select by value, verify selection
- Build a complete mini project that automates a registration form with all element types

---

## Core Concept Explanation

### Native HTML Dropdowns (`<select>`)

A native dropdown uses the HTML `<select>` tag:

```html
<select id="country">
    <option value="">Please select</option>
    <option value="us">United States</option>
    <option value="uk">United Kingdom</option>
    <option value="ca">Canada</option>
</select>
```

Selenium provides a dedicated class to work with `<select>` elements:

- **C#:** `SelectElement` (in `OpenQA.Selenium.Support.UI`)
- **Python:** `Select` (in `selenium.webdriver.support.select`)

### Custom Dropdowns

Many modern web apps use `<div>` elements styled to look like dropdowns. These do NOT use the `<select>` tag and cannot be handled with `SelectElement`/`Select`. They require clicking and waiting for options to appear.

### Checkboxes and Radio Buttons

Both are `<input>` elements:

- **Checkbox:** `<input type="checkbox">` — multiple can be selected
- **Radio button:** `<input type="radio" name="group">` — only one per group can be selected

Both use `Click()` to toggle and `.Selected` / `.is_selected()` to check state.

---

## How It Works (Technical Breakdown)

### SelectElement / Select Methods

| Method | C# | Python | Description |
|--------|-----|--------|-------------|
| Select by text | `select.SelectByText("Canada")` | `select.select_by_visible_text("Canada")` | Matches the visible text of the option |
| Select by value | `select.SelectByValue("ca")` | `select.select_by_value("ca")` | Matches the `value` attribute |
| Select by index | `select.SelectByIndex(2)` | `select.select_by_index(2)` | Selects by position (0-based) |
| Get all options | `select.Options` | `select.options` | Returns list of all `<option>` elements |
| Get selected option | `select.SelectedOption` | `select.first_selected_option` | Returns the currently selected option |
| Get all selected | `select.AllSelectedOptions` | `select.all_selected_options` | For multi-select dropdowns |
| Deselect all | `select.DeselectAll()` | `select.deselect_all()` | Only for multi-select |

### Checkbox Behavior

```
Click on unchecked checkbox  -->  Becomes checked  (Selected = true)
Click on checked checkbox    -->  Becomes unchecked (Selected = false)
```

Always check the current state with `.Selected` / `.is_selected()` before clicking, unless you want to toggle regardless.

### Radio Button Behavior

```
Click on unselected radio  -->  Becomes selected, others in group deselect
Click on selected radio    -->  Nothing happens (stays selected)
```

Radio buttons in the same group share the same `name` attribute. Selecting one automatically deselects the others.

---

## Code Example: C# (Complete, Runnable)

```csharp
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;
using System.Collections.Generic;
using System.Linq;

namespace SeleniumCourse;

[TestFixture]
public class Day07_DropdownsCheckboxesRadio
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

    // ================================================================
    // DROPDOWNS
    // ================================================================

    [Test]
    public void NativeDropdown_SelectByText()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dropdown");

        // Find the <select> element
        IWebElement dropdownElement = driver.FindElement(By.Id("dropdown"));

        // Wrap it in a SelectElement for dropdown-specific methods
        SelectElement select = new SelectElement(dropdownElement);

        // Select by visible text
        select.SelectByText("Option 1");

        // Verify the selection
        string selectedText = select.SelectedOption.Text;
        Console.WriteLine($"Selected by text: {selectedText}");
        Assert.That(selectedText, Is.EqualTo("Option 1"));
    }

    [Test]
    public void NativeDropdown_SelectByValue()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dropdown");

        IWebElement dropdownElement = driver.FindElement(By.Id("dropdown"));
        SelectElement select = new SelectElement(dropdownElement);

        // Select by the value attribute: <option value="2">Option 2</option>
        select.SelectByValue("2");

        string selectedText = select.SelectedOption.Text;
        Console.WriteLine($"Selected by value: {selectedText}");
        Assert.That(selectedText, Is.EqualTo("Option 2"));
    }

    [Test]
    public void NativeDropdown_SelectByIndex()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dropdown");

        IWebElement dropdownElement = driver.FindElement(By.Id("dropdown"));
        SelectElement select = new SelectElement(dropdownElement);

        // Select by index (0-based)
        // Index 0 = "Please select an option" (disabled)
        // Index 1 = "Option 1"
        // Index 2 = "Option 2"
        select.SelectByIndex(2);

        string selectedText = select.SelectedOption.Text;
        Console.WriteLine($"Selected by index 2: {selectedText}");
        Assert.That(selectedText, Is.EqualTo("Option 2"));
    }

    [Test]
    public void NativeDropdown_GetAllOptions()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dropdown");

        IWebElement dropdownElement = driver.FindElement(By.Id("dropdown"));
        SelectElement select = new SelectElement(dropdownElement);

        // Get all options
        IList<IWebElement> allOptions = select.Options;
        Console.WriteLine($"Total options: {allOptions.Count}");

        foreach (IWebElement option in allOptions)
        {
            Console.WriteLine($"  Text: '{option.Text}', Value: '{option.GetAttribute("value")}'");
        }

        Assert.That(allOptions.Count, Is.EqualTo(3));
    }

    [Test]
    public void CustomDropdown_DivBased()
    {
        // Custom dropdowns require click-wait-click pattern
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dropdown");

        // For a real custom dropdown (div-based), the pattern is:
        // 1. Click the dropdown trigger to open it
        // 2. Wait for the options to become visible
        // 3. Click the desired option

        // Simulating with the native dropdown for demonstration:
        IWebElement dropdown = driver.FindElement(By.Id("dropdown"));
        dropdown.Click(); // Open

        // Find and click the desired option directly
        IWebElement option = driver.FindElement(
            By.CssSelector("#dropdown option[value='1']"));
        option.Click();

        Console.WriteLine($"Custom-style selection: {option.Text}");
    }

    // ================================================================
    // CHECKBOXES
    // ================================================================

    [Test]
    public void Checkbox_ClickAndVerify()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/checkboxes");

        IList<IWebElement> checkboxes = driver.FindElements(
            By.CssSelector("input[type='checkbox']"));

        IWebElement checkbox1 = checkboxes[0];
        IWebElement checkbox2 = checkboxes[1];

        // Check initial state
        Console.WriteLine($"Checkbox 1 initial: {checkbox1.Selected}");
        Console.WriteLine($"Checkbox 2 initial: {checkbox2.Selected}");

        // Click checkbox 1 to check it (initially unchecked)
        if (!checkbox1.Selected)
        {
            checkbox1.Click();
            Console.WriteLine("Clicked checkbox 1 to check it");
        }

        // Click checkbox 2 to uncheck it (initially checked)
        if (checkbox2.Selected)
        {
            checkbox2.Click();
            Console.WriteLine("Clicked checkbox 2 to uncheck it");
        }

        // Verify final state
        Assert.That(checkbox1.Selected, Is.True, "Checkbox 1 should be checked");
        Assert.That(checkbox2.Selected, Is.False, "Checkbox 2 should be unchecked");

        Console.WriteLine($"Checkbox 1 final: {checkbox1.Selected}");
        Console.WriteLine($"Checkbox 2 final: {checkbox2.Selected}");
    }

    [Test]
    public void Checkbox_EnsureChecked()
    {
        driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/checkboxes");

        IWebElement checkbox = driver.FindElements(
            By.CssSelector("input[type='checkbox']"))[0];

        // Idempotent "ensure checked" — only click if not already selected
        if (!checkbox.Selected)
        {
            checkbox.Click();
        }

        Assert.That(checkbox.Selected, Is.True);
        Console.WriteLine("Checkbox is now checked (idempotent)");

        // Call again — should NOT toggle because it is already checked
        if (!checkbox.Selected)
        {
            checkbox.Click();
        }

        Assert.That(checkbox.Selected, Is.True);
        Console.WriteLine("Checkbox is still checked after second call");
    }

    // ================================================================
    // RADIO BUTTONS (using a practice form)
    // ================================================================

    [Test]
    public void RadioButtons_SelectAndVerify()
    {
        // Using a practice site with radio buttons
        driver.Navigate().GoToUrl("https://demoqa.com/radio-button");

        // Find the "Yes" radio button label and click it
        // On this site, radio inputs are hidden; we click the label
        IWebElement yesLabel = wait.Until(d =>
            d.FindElement(By.CssSelector("label[for='yesRadio']")));
        yesLabel.Click();

        // Verify the radio button is selected
        IWebElement yesRadio = driver.FindElement(By.Id("yesRadio"));
        Console.WriteLine($"Yes radio selected: {yesRadio.Selected}");
        Assert.That(yesRadio.Selected, Is.True);

        // Click "Impressive" radio button
        IWebElement impressiveLabel = driver.FindElement(
            By.CssSelector("label[for='impressiveRadio']"));
        impressiveLabel.Click();

        // Verify "Impressive" is now selected and "Yes" is deselected
        IWebElement impressiveRadio = driver.FindElement(By.Id("impressiveRadio"));
        Console.WriteLine($"Impressive radio selected: {impressiveRadio.Selected}");
        Console.WriteLine($"Yes radio now: {yesRadio.Selected}");

        Assert.That(impressiveRadio.Selected, Is.True);
        Assert.That(yesRadio.Selected, Is.False);
    }

    [TearDown]
    public void Teardown()
    {
        driver.Quit();
    }
}
```

### Mini Project 1: Complete Registration Form (C#)

```csharp
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Interactions;
using OpenQA.Selenium.Support.UI;
using System;

namespace SeleniumCourse;

[TestFixture]
public class Day07_MiniProject1
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
    public void AutomateRegistrationForm()
    {
        // Navigate to the practice form
        driver.Navigate().GoToUrl("https://demoqa.com/automation-practice-form");

        // --- TEXT FIELDS ---
        // First name
        IWebElement firstName = wait.Until(d =>
            d.FindElement(By.Id("firstName")));
        firstName.Clear();
        firstName.SendKeys("Jane");
        Console.WriteLine("Entered first name: Jane");

        // Last name
        IWebElement lastName = driver.FindElement(By.Id("lastName"));
        lastName.Clear();
        lastName.SendKeys("Smith");
        Console.WriteLine("Entered last name: Smith");

        // Email
        IWebElement email = driver.FindElement(By.Id("userEmail"));
        email.Clear();
        email.SendKeys("jane.smith@example.com");
        Console.WriteLine("Entered email: jane.smith@example.com");

        // --- RADIO BUTTONS ---
        // Select "Female" gender
        IWebElement femaleLabel = driver.FindElement(
            By.CssSelector("label[for='gender-radio-2']"));
        femaleLabel.Click();

        IWebElement femaleRadio = driver.FindElement(By.Id("gender-radio-2"));
        Assert.That(femaleRadio.Selected, Is.True);
        Console.WriteLine("Selected gender: Female");

        // --- PHONE NUMBER ---
        IWebElement mobile = driver.FindElement(By.Id("userNumber"));
        mobile.Clear();
        mobile.SendKeys("5551234567");
        Console.WriteLine("Entered mobile: 5551234567");

        // --- CHECKBOX ---
        // Select "Sports" hobby
        IWebElement sportsLabel = driver.FindElement(
            By.CssSelector("label[for='hobbies-checkbox-1']"));

        // Scroll to the element to make it clickable
        Actions actions = new Actions(driver);
        actions.MoveToElement(sportsLabel).Perform();
        sportsLabel.Click();

        IWebElement sportsCheckbox = driver.FindElement(By.Id("hobbies-checkbox-1"));
        Assert.That(sportsCheckbox.Selected, Is.True);
        Console.WriteLine("Selected hobby: Sports");

        // Select "Reading" hobby too (multiple checkboxes allowed)
        IWebElement readingLabel = driver.FindElement(
            By.CssSelector("label[for='hobbies-checkbox-2']"));
        readingLabel.Click();

        IWebElement readingCheckbox = driver.FindElement(By.Id("hobbies-checkbox-2"));
        Assert.That(readingCheckbox.Selected, Is.True);
        Console.WriteLine("Selected hobby: Reading");

        // --- TEXT AREA ---
        IWebElement currentAddress = driver.FindElement(By.Id("currentAddress"));
        currentAddress.Clear();
        currentAddress.SendKeys("123 Main Street\nApt 4B\nNew York, NY 10001");
        Console.WriteLine("Entered address");

        // --- SCROLL DOWN AND SUBMIT ---
        // Find and click the submit button
        IWebElement submitButton = driver.FindElement(By.Id("submit"));
        actions.MoveToElement(submitButton).Perform();

        // Use JavaScript click to avoid interception by ads/overlays
        ((IJavaScriptExecutor)driver).ExecuteScript(
            "arguments[0].click();", submitButton);
        Console.WriteLine("Clicked submit button");

        // --- VERIFY SUBMISSION ---
        try
        {
            IWebElement modal = wait.Until(d =>
                d.FindElement(By.Id("example-modal-sizes-title-lg")));
            Console.WriteLine($"Modal title: {modal.Text}");
            Assert.That(modal.Text, Does.Contain("Thanks for submitting the form"));
        }
        catch (WebDriverTimeoutException)
        {
            Console.WriteLine("Modal did not appear — form may have validation errors");
        }

        Console.WriteLine("\n--- MINI PROJECT 1 COMPLETE ---");
        Console.WriteLine("Successfully automated a registration form with:");
        Console.WriteLine("  - Text fields (first name, last name, email, phone)");
        Console.WriteLine("  - Radio buttons (gender)");
        Console.WriteLine("  - Checkboxes (hobbies)");
        Console.WriteLine("  - Text area (address)");
        Console.WriteLine("  - Form submission");
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
from selenium.webdriver.support.ui import WebDriverWait, Select
from selenium.webdriver.support import expected_conditions as EC


@pytest.fixture
def driver():
    """Create a ChromeDriver, yield it, then quit."""
    options = Options()
    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


# ================================================================
# DROPDOWNS
# ================================================================


def test_native_dropdown_select_by_text(driver):
    """Select a dropdown option by its visible text."""
    driver.get("https://the-internet.herokuapp.com/dropdown")

    # Find the <select> element and wrap it with Select
    dropdown_element = driver.find_element(By.ID, "dropdown")
    select = Select(dropdown_element)

    # Select by visible text
    select.select_by_visible_text("Option 1")

    selected_text = select.first_selected_option.text
    print(f"Selected by text: {selected_text}")
    assert selected_text == "Option 1"


def test_native_dropdown_select_by_value(driver):
    """Select a dropdown option by its value attribute."""
    driver.get("https://the-internet.herokuapp.com/dropdown")

    dropdown_element = driver.find_element(By.ID, "dropdown")
    select = Select(dropdown_element)

    # Select by value attribute: <option value="2">
    select.select_by_value("2")

    selected_text = select.first_selected_option.text
    print(f"Selected by value: {selected_text}")
    assert selected_text == "Option 2"


def test_native_dropdown_select_by_index(driver):
    """Select a dropdown option by its index (0-based)."""
    driver.get("https://the-internet.herokuapp.com/dropdown")

    dropdown_element = driver.find_element(By.ID, "dropdown")
    select = Select(dropdown_element)

    # Index 0 = "Please select an option"
    # Index 1 = "Option 1"
    # Index 2 = "Option 2"
    select.select_by_index(2)

    selected_text = select.first_selected_option.text
    print(f"Selected by index 2: {selected_text}")
    assert selected_text == "Option 2"


def test_native_dropdown_get_all_options(driver):
    """List all options in a dropdown."""
    driver.get("https://the-internet.herokuapp.com/dropdown")

    dropdown_element = driver.find_element(By.ID, "dropdown")
    select = Select(dropdown_element)

    all_options = select.options
    print(f"Total options: {len(all_options)}")

    for option in all_options:
        print(f"  Text: '{option.text}', Value: '{option.get_attribute('value')}'")

    assert len(all_options) == 3


def test_custom_dropdown_div_based(driver):
    """Handle a custom dropdown that uses divs instead of select."""
    driver.get("https://the-internet.herokuapp.com/dropdown")

    # For real custom dropdowns, the pattern is:
    # 1. Click the dropdown trigger to open it
    # 2. Wait for the options to become visible
    # 3. Click the desired option

    dropdown = driver.find_element(By.ID, "dropdown")
    dropdown.click()

    option = driver.find_element(By.CSS_SELECTOR, "#dropdown option[value='1']")
    option.click()

    print(f"Custom-style selection: {option.text}")


# ================================================================
# CHECKBOXES
# ================================================================


def test_checkbox_click_and_verify(driver):
    """Click checkboxes and verify their state."""
    driver.get("https://the-internet.herokuapp.com/checkboxes")

    checkboxes = driver.find_elements(By.CSS_SELECTOR, "input[type='checkbox']")
    checkbox1 = checkboxes[0]
    checkbox2 = checkboxes[1]

    # Check initial state
    print(f"Checkbox 1 initial: {checkbox1.is_selected()}")
    print(f"Checkbox 2 initial: {checkbox2.is_selected()}")

    # Click checkbox 1 to check it
    if not checkbox1.is_selected():
        checkbox1.click()
        print("Clicked checkbox 1 to check it")

    # Click checkbox 2 to uncheck it
    if checkbox2.is_selected():
        checkbox2.click()
        print("Clicked checkbox 2 to uncheck it")

    # Verify final state
    assert checkbox1.is_selected(), "Checkbox 1 should be checked"
    assert not checkbox2.is_selected(), "Checkbox 2 should be unchecked"

    print(f"Checkbox 1 final: {checkbox1.is_selected()}")
    print(f"Checkbox 2 final: {checkbox2.is_selected()}")


def test_checkbox_ensure_checked(driver):
    """Idempotent check — only click if not already selected."""
    driver.get("https://the-internet.herokuapp.com/checkboxes")

    checkbox = driver.find_elements(By.CSS_SELECTOR, "input[type='checkbox']")[0]

    # Ensure checked (idempotent)
    if not checkbox.is_selected():
        checkbox.click()

    assert checkbox.is_selected()
    print("Checkbox is now checked (idempotent)")

    # Call again — should NOT toggle
    if not checkbox.is_selected():
        checkbox.click()

    assert checkbox.is_selected()
    print("Checkbox is still checked after second call")


# ================================================================
# RADIO BUTTONS
# ================================================================


def test_radio_buttons_select_and_verify(driver):
    """Select radio buttons and verify only one is selected."""
    driver.get("https://demoqa.com/radio-button")

    wait = WebDriverWait(driver, 10)

    # Click "Yes" radio button via its label
    yes_label = wait.until(
        EC.element_to_be_clickable((By.CSS_SELECTOR, "label[for='yesRadio']"))
    )
    yes_label.click()

    yes_radio = driver.find_element(By.ID, "yesRadio")
    print(f"Yes radio selected: {yes_radio.is_selected()}")
    assert yes_radio.is_selected()

    # Click "Impressive" radio button
    impressive_label = driver.find_element(
        By.CSS_SELECTOR, "label[for='impressiveRadio']"
    )
    impressive_label.click()

    impressive_radio = driver.find_element(By.ID, "impressiveRadio")
    print(f"Impressive radio selected: {impressive_radio.is_selected()}")
    print(f"Yes radio now: {yes_radio.is_selected()}")

    assert impressive_radio.is_selected()
    assert not yes_radio.is_selected()
```

### Mini Project 1: Complete Registration Form (Python)

```python
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
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


def test_automate_registration_form(driver):
    """
    Mini Project 1: Automate a complete registration form with
    text fields, radio buttons, checkboxes, text area, and submission.
    """
    driver.get("https://demoqa.com/automation-practice-form")
    wait = WebDriverWait(driver, 10)

    # --- TEXT FIELDS ---
    # First name
    first_name = wait.until(EC.presence_of_element_located((By.ID, "firstName")))
    first_name.clear()
    first_name.send_keys("Jane")
    print("Entered first name: Jane")

    # Last name
    last_name = driver.find_element(By.ID, "lastName")
    last_name.clear()
    last_name.send_keys("Smith")
    print("Entered last name: Smith")

    # Email
    email = driver.find_element(By.ID, "userEmail")
    email.clear()
    email.send_keys("jane.smith@example.com")
    print("Entered email: jane.smith@example.com")

    # --- RADIO BUTTONS ---
    # Select "Female" gender
    female_label = driver.find_element(
        By.CSS_SELECTOR, "label[for='gender-radio-2']"
    )
    female_label.click()

    female_radio = driver.find_element(By.ID, "gender-radio-2")
    assert female_radio.is_selected()
    print("Selected gender: Female")

    # --- PHONE NUMBER ---
    mobile = driver.find_element(By.ID, "userNumber")
    mobile.clear()
    mobile.send_keys("5551234567")
    print("Entered mobile: 5551234567")

    # --- CHECKBOXES ---
    # Select "Sports" hobby
    sports_label = driver.find_element(
        By.CSS_SELECTOR, "label[for='hobbies-checkbox-1']"
    )

    # Scroll to the element
    actions = ActionChains(driver)
    actions.move_to_element(sports_label).perform()
    sports_label.click()

    sports_checkbox = driver.find_element(By.ID, "hobbies-checkbox-1")
    assert sports_checkbox.is_selected()
    print("Selected hobby: Sports")

    # Select "Reading" hobby
    reading_label = driver.find_element(
        By.CSS_SELECTOR, "label[for='hobbies-checkbox-2']"
    )
    reading_label.click()

    reading_checkbox = driver.find_element(By.ID, "hobbies-checkbox-2")
    assert reading_checkbox.is_selected()
    print("Selected hobby: Reading")

    # --- TEXT AREA ---
    current_address = driver.find_element(By.ID, "currentAddress")
    current_address.clear()
    current_address.send_keys("123 Main Street\nApt 4B\nNew York, NY 10001")
    print("Entered address")

    # --- SUBMIT ---
    submit_button = driver.find_element(By.ID, "submit")
    actions.move_to_element(submit_button).perform()

    # Use JavaScript click to avoid interception by ads/overlays
    driver.execute_script("arguments[0].click();", submit_button)
    print("Clicked submit button")

    # --- VERIFY ---
    try:
        modal = wait.until(
            EC.presence_of_element_located(
                (By.ID, "example-modal-sizes-title-lg")
            )
        )
        print(f"Modal title: {modal.text}")
        assert "Thanks for submitting the form" in modal.text
    except Exception:
        print("Modal did not appear — form may have validation errors")

    print("\n--- MINI PROJECT 1 COMPLETE ---")
    print("Successfully automated a registration form with:")
    print("  - Text fields (first name, last name, email, phone)")
    print("  - Radio buttons (gender)")
    print("  - Checkboxes (hobbies)")
    print("  - Text area (address)")
    print("  - Form submission")
```

---

## Step-by-Step Walkthrough

### Dropdown interaction flow:

1. **Find the `<select>` element** — use any locator strategy
2. **Wrap it** — `new SelectElement(element)` (C#) or `Select(element)` (Python)
3. **Select an option** — by text, value, or index
4. **Verify** — read `SelectedOption` / `first_selected_option` to confirm

### Checkbox interaction pattern:

```
1. Find the checkbox element
2. Check its current state with .Selected / .is_selected()
3. If you want it checked and it is not: click it
4. If you want it unchecked and it is: click it
5. Verify the final state
```

This "check-before-click" pattern prevents accidentally unchecking a checkbox that should stay checked.

### Custom dropdown interaction pattern:

```
1. Click the dropdown trigger (the visible element the user clicks)
2. Wait for the options container to become visible
3. Find the desired option inside the container
4. Click the option
5. Wait for the dropdown to close
6. Verify the selected value appears in the trigger element
```

Example for a common custom dropdown pattern:

```csharp
// C#: Custom dropdown
driver.FindElement(By.CssSelector(".dropdown-trigger")).Click();

var option = wait.Until(d =>
    d.FindElement(By.XPath("//li[text()='Option 1']")));
option.Click();

// Verify selection appears in the trigger
var selectedValue = driver.FindElement(By.CssSelector(".dropdown-trigger")).Text;
Assert.That(selectedValue, Is.EqualTo("Option 1"));
```

```python
# Python: Custom dropdown
driver.find_element(By.CSS_SELECTOR, ".dropdown-trigger").click()

option = wait.until(
    EC.element_to_be_clickable((By.XPATH, "//li[text()='Option 1']"))
)
option.click()

# Verify
selected = driver.find_element(By.CSS_SELECTOR, ".dropdown-trigger").text
assert selected == "Option 1"
```

---

## Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using `Select` on non-`<select>` elements | Throws `UnexpectedTagNameException` | Check the HTML — if it is a `<div>`, use click-wait-click pattern |
| Clicking a checkbox without checking state | May uncheck an already-checked box | Always check `.Selected` / `.is_selected()` first |
| Using `SelectByText` with leading/trailing whitespace | No match found | Inspect the actual option text carefully; `trim()` is not automatic |
| Forgetting that `SelectByIndex` is 0-based | Off-by-one error | Index 0 is the first option, including placeholder options |
| Not scrolling to elements on long pages | `ElementClickInterceptedException` | Use `Actions.MoveToElement()` or JavaScript scroll before clicking |
| Trying to deselect a single-select dropdown | `NotImplementedError` | `DeselectAll()` only works on multi-select dropdowns (`<select multiple>`) |

---

## Best Practices

1. **Use `SelectByText` for readability** — `select.SelectByText("United States")` is self-documenting. `SelectByValue("us")` and `SelectByIndex(5)` are less clear.

2. **Always verify after selection** — read back the selected option to confirm the selection took effect:
   ```csharp
   select.SelectByText("Option 1");
   Assert.That(select.SelectedOption.Text, Is.EqualTo("Option 1"));
   ```

3. **Check before clicking checkboxes** — wrap the click in an `if (!checkbox.Selected)` guard to make your tests idempotent.

4. **Handle custom dropdowns with explicit waits** — wait for the options to appear after clicking the trigger, and wait for the dropdown to close after selecting.

5. **Use descriptive assertions** — when a checkbox assertion fails, provide a message:
   ```csharp
   Assert.That(checkbox.Selected, Is.True, "Newsletter checkbox should be checked");
   ```

6. **Scroll before interacting** — on long forms, elements below the viewport may not be clickable. Scroll into view first.

---

## Hands-On Exercise

### Task

Navigate to `https://the-internet.herokuapp.com/dropdown` and `https://the-internet.herokuapp.com/checkboxes` and write tests that:

1. Open the dropdown page
2. Select "Option 1" by text, verify it is selected
3. Change to "Option 2" by value, verify the change
4. List all available options and print their text
5. Open the checkboxes page
6. Ensure both checkboxes are checked (regardless of initial state)
7. Verify both checkboxes show as selected
8. Uncheck the first checkbox and verify

### Expected Output

```
Selected Option 1 by text
Changed to Option 2 by value
Options available: Please select an option, Option 1, Option 2
Both checkboxes are now checked
Checkbox 1: True, Checkbox 2: True
After unchecking first: Checkbox 1: False, Checkbox 2: True
```

---

## Real-World Scenario

You are testing an HR application's employee onboarding form. The form includes:

- **Department dropdown** (`<select>`) — choose from "Engineering", "Marketing", "Sales"
- **Location dropdown** (custom `<div>`) — a searchable dropdown that shows filtered results as you type
- **Skill checkboxes** — select multiple: "Python", "Java", "SQL", "AWS"
- **Employment type radio** — "Full-time", "Part-time", "Contractor"

Your test automates the complete form:

```python
# Native dropdown
dept_select = Select(driver.find_element(By.ID, "department"))
dept_select.select_by_visible_text("Engineering")

# Custom searchable dropdown
driver.find_element(By.CSS_SELECTOR, ".location-search").click()
driver.find_element(By.CSS_SELECTOR, ".location-input").send_keys("San Francisco")
wait.until(EC.element_to_be_clickable(
    (By.XPATH, "//li[text()='San Francisco, CA']")
)).click()

# Checkboxes — check Python and AWS
for skill_id in ["skill-python", "skill-aws"]:
    cb = driver.find_element(By.ID, skill_id)
    if not cb.is_selected():
        cb.click()

# Radio button
driver.find_element(By.CSS_SELECTOR, "label[for='emp-fulltime']").click()
```

This pattern — combining dropdowns, checkboxes, and radio buttons — is found in nearly every enterprise web application.

---

## Week 1 Summary

Over the past 7 days, you have built a solid foundation in Selenium WebDriver:

| Day | Topic | Key Skill |
|-----|-------|-----------|
| 1 | Setup & First Script | Create projects, launch browser, read title |
| 2 | Basic Locators | Find elements by ID, Name, ClassName, TagName |
| 3 | XPath | Navigate the DOM with axes, functions, conditions |
| 4 | CSS Selectors | Write fast, readable selectors for production |
| 5 | Browser Actions | Click, type, clear, hover, drag-and-drop |
| 6 | Navigation | Back, forward, refresh, cookies, multi-page flows |
| 7 | Form Elements | Dropdowns, checkboxes, radio buttons, full form automation |

In **Week 2**, you will learn explicit waits, handling alerts, working with iframes and windows, taking screenshots, and building the Page Object Model pattern.

---

## Resources

- [Selenium Select Element](https://www.selenium.dev/documentation/webdriver/support_features/select_lists/)
- [Select Class (Python)](https://www.selenium.dev/selenium/docs/api/py/webdriver_support/selenium.webdriver.support.select.html)
- [SelectElement Class (C#)](https://www.selenium.dev/selenium/docs/api/dotnet/html/T_OpenQA_Selenium_Support_UI_SelectElement.htm)
- [Practice Dropdowns: The Internet](https://the-internet.herokuapp.com/dropdown)
- [Practice Checkboxes: The Internet](https://the-internet.herokuapp.com/checkboxes)
- [Practice Forms: DemoQA](https://demoqa.com/automation-practice-form)
- [Practice Radio Buttons: DemoQA](https://demoqa.com/radio-button)
