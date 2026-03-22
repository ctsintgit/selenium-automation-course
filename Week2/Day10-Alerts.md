# Day 10: Alerts -- Accept, Dismiss, GetText, SendKeys

## Learning Objectives

By the end of this lesson, you will be able to:

- Distinguish between JavaScript alerts, confirms, and prompts
- Switch to an alert and interact with it (accept, dismiss, read text, send input)
- Wait for an alert before switching to it
- Handle unexpected alerts gracefully
- Identify and interact with non-native alerts (SweetAlert, modal dialogs)

---

## Core Concept Explanation

JavaScript provides three native dialog types that pause script execution and require user interaction:

| Dialog Type | Function | Buttons | Has Input? |
|-------------|----------|---------|------------|
| **Alert** | `window.alert("message")` | OK | No |
| **Confirm** | `window.confirm("message")` | OK, Cancel | No |
| **Prompt** | `window.prompt("message", "default")` | OK, Cancel | Yes (text field) |

These native dialogs are **not part of the DOM**. You cannot find them with `FindElement`. Instead, you must switch the driver's focus to the alert using `driver.SwitchTo().Alert()` (C#) or `driver.switch_to.alert` (Python), which returns an `IAlert` / `Alert` object.

If you try to interact with the page while an alert is open, Selenium throws an `UnhandledAlertException`.

---

## How It Works

1. **Trigger the alert** -- click a button or execute JavaScript that opens a dialog.
2. **Wait for the alert** -- use `ExpectedConditions.AlertIsPresent()` to ensure the alert exists before switching.
3. **Switch to the alert** -- `driver.SwitchTo().Alert()` returns an alert object.
4. **Interact with the alert:**
   - `.Text` / `.text` -- read the alert message
   - `.Accept()` / `.accept()` -- click OK
   - `.Dismiss()` / `.dismiss()` -- click Cancel
   - `.SendKeys("text")` / `.send_keys("text")` -- type into the prompt input field
5. **Focus returns to the page** automatically after accepting or dismissing.

---

## Code Example: C#

```csharp
// File: Day10_AlertsDemo.cs
// Prerequisites:
//   dotnet add package Selenium.WebDriver
//   dotnet add package Selenium.Support
//   dotnet add package DotNetSeleniumExtras.WaitHelpers
//   dotnet add package NUnit
//   dotnet add package NUnit3TestAdapter

using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using SeleniumExtras.WaitHelpers;
using System;

namespace Day10
{
    [TestFixture]
    public class AlertsDemo
    {
        private IWebDriver driver;
        private WebDriverWait wait;

        [SetUp]
        public void SetUp()
        {
            driver = new ChromeDriver();
            driver.Manage().Window.Maximize();
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        // --- JAVASCRIPT ALERT (OK button only) ---
        [Test]
        public void HandleJavaScriptAlert()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/javascript_alerts");

            // Click the button that triggers a simple alert
            driver.FindElement(By.XPath("//button[text()='Click for JS Alert']")).Click();

            // Wait for the alert to appear
            IAlert alert = wait.Until(ExpectedConditions.AlertIsPresent());

            // Read the alert text
            string alertText = alert.Text;
            TestContext.WriteLine($"Alert text: {alertText}");
            Assert.That(alertText, Is.EqualTo("I am a JS Alert"));

            // Accept the alert (click OK)
            alert.Accept();

            // Verify the result on the page
            IWebElement result = driver.FindElement(By.Id("result"));
            Assert.That(result.Text, Is.EqualTo("You successfully clicked an alert"));
        }

        // --- JAVASCRIPT CONFIRM (OK and Cancel buttons) ---
        [Test]
        public void HandleJavaScriptConfirm_Accept()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/javascript_alerts");

            driver.FindElement(By.XPath("//button[text()='Click for JS Confirm']")).Click();

            IAlert alert = wait.Until(ExpectedConditions.AlertIsPresent());

            string alertText = alert.Text;
            TestContext.WriteLine($"Confirm text: {alertText}");
            Assert.That(alertText, Is.EqualTo("I am a JS Confirm"));

            // Accept the confirm (click OK)
            alert.Accept();

            IWebElement result = driver.FindElement(By.Id("result"));
            Assert.That(result.Text, Is.EqualTo("You clicked: Ok"));
        }

        [Test]
        public void HandleJavaScriptConfirm_Dismiss()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/javascript_alerts");

            driver.FindElement(By.XPath("//button[text()='Click for JS Confirm']")).Click();

            IAlert alert = wait.Until(ExpectedConditions.AlertIsPresent());

            // Dismiss the confirm (click Cancel)
            alert.Dismiss();

            IWebElement result = driver.FindElement(By.Id("result"));
            Assert.That(result.Text, Is.EqualTo("You clicked: Cancel"));
        }

        // --- JAVASCRIPT PROMPT (OK, Cancel, and text input) ---
        [Test]
        public void HandleJavaScriptPrompt()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/javascript_alerts");

            driver.FindElement(By.XPath("//button[text()='Click for JS Prompt']")).Click();

            IAlert alert = wait.Until(ExpectedConditions.AlertIsPresent());

            string alertText = alert.Text;
            TestContext.WriteLine($"Prompt text: {alertText}");
            Assert.That(alertText, Is.EqualTo("I am a JS prompt"));

            // Type text into the prompt's input field
            alert.SendKeys("Hello Selenium!");

            // Accept the prompt (click OK)
            alert.Accept();

            IWebElement result = driver.FindElement(By.Id("result"));
            Assert.That(result.Text, Is.EqualTo("You entered: Hello Selenium!"));
        }

        [Test]
        public void HandleJavaScriptPrompt_Dismiss()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/javascript_alerts");

            driver.FindElement(By.XPath("//button[text()='Click for JS Prompt']")).Click();

            IAlert alert = wait.Until(ExpectedConditions.AlertIsPresent());

            // Dismiss the prompt (click Cancel) -- typed text is discarded
            alert.Dismiss();

            IWebElement result = driver.FindElement(By.Id("result"));
            Assert.That(result.Text, Is.EqualTo("You entered: null"));
        }

        // --- WAITING FOR ALERT SAFELY ---
        [Test]
        public void CheckIfAlertPresent()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/javascript_alerts");

            // Before triggering -- no alert should be present
            bool alertPresent = IsAlertPresent();
            Assert.That(alertPresent, Is.False);

            // Trigger an alert
            driver.FindElement(By.XPath("//button[text()='Click for JS Alert']")).Click();

            // Now an alert should be present
            alertPresent = IsAlertPresent();
            Assert.That(alertPresent, Is.True);

            // Accept it to clean up
            driver.SwitchTo().Alert().Accept();
        }

        /// <summary>
        /// Helper method to check if an alert is present without throwing.
        /// </summary>
        private bool IsAlertPresent()
        {
            try
            {
                WebDriverWait shortWait = new WebDriverWait(driver, TimeSpan.FromSeconds(2));
                shortWait.Until(ExpectedConditions.AlertIsPresent());
                return true;
            }
            catch (WebDriverTimeoutException)
            {
                return false;
            }
        }

        // --- NON-NATIVE ALERT (SweetAlert / Modal) ---
        [Test]
        public void HandleNonNativeAlert()
        {
            // Non-native alerts (SweetAlert, Bootstrap modals, etc.) are regular
            // DOM elements. You interact with them using standard FindElement.
            // This example uses a page with a SweetAlert-style modal.

            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/entry_ad");

            // Wait for the modal to be visible
            IWebElement modal = wait.Until(
                ExpectedConditions.ElementIsVisible(By.CssSelector(".modal"))
            );

            // Read the modal title
            string title = modal.FindElement(By.CssSelector(".modal-title h3")).Text;
            TestContext.WriteLine($"Modal title: {title}");

            // Close the modal by clicking the close link
            IWebElement closeBtn = wait.Until(
                ExpectedConditions.ElementToBeClickable(
                    By.CssSelector(".modal-footer p"))
            );
            closeBtn.Click();

            // Wait for modal to disappear
            wait.Until(
                ExpectedConditions.InvisibilityOfElementLocated(By.CssSelector(".modal"))
            );

            TestContext.WriteLine("Non-native modal dismissed successfully");
        }
    }
}
```

---

## Code Example: Python

```python
# File: test_day10_alerts_demo.py
# Prerequisites:
#   pip install selenium pytest

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException
import pytest


@pytest.fixture
def driver():
    """Create a Chrome browser instance for each test."""
    drv = webdriver.Chrome()
    drv.maximize_window()
    yield drv
    drv.quit()


# --- JAVASCRIPT ALERT (OK button only) ---
def test_handle_javascript_alert(driver):
    driver.get("https://the-internet.herokuapp.com/javascript_alerts")

    # Click the button that triggers a simple alert
    driver.find_element(By.XPATH, "//button[text()='Click for JS Alert']").click()

    # Wait for the alert to appear
    wait = WebDriverWait(driver, 10)
    alert = wait.until(EC.alert_is_present())

    # Read the alert text
    alert_text = alert.text
    print(f"Alert text: {alert_text}")
    assert alert_text == "I am a JS Alert"

    # Accept the alert (click OK)
    alert.accept()

    # Verify the result on the page
    result = driver.find_element(By.ID, "result")
    assert result.text == "You successfully clicked an alert"


# --- JAVASCRIPT CONFIRM (OK and Cancel buttons) ---
def test_handle_confirm_accept(driver):
    driver.get("https://the-internet.herokuapp.com/javascript_alerts")

    driver.find_element(By.XPATH, "//button[text()='Click for JS Confirm']").click()

    wait = WebDriverWait(driver, 10)
    alert = wait.until(EC.alert_is_present())

    print(f"Confirm text: {alert.text}")
    assert alert.text == "I am a JS Confirm"

    # Accept (click OK)
    alert.accept()

    result = driver.find_element(By.ID, "result")
    assert result.text == "You clicked: Ok"


def test_handle_confirm_dismiss(driver):
    driver.get("https://the-internet.herokuapp.com/javascript_alerts")

    driver.find_element(By.XPATH, "//button[text()='Click for JS Confirm']").click()

    wait = WebDriverWait(driver, 10)
    alert = wait.until(EC.alert_is_present())

    # Dismiss (click Cancel)
    alert.dismiss()

    result = driver.find_element(By.ID, "result")
    assert result.text == "You clicked: Cancel"


# --- JAVASCRIPT PROMPT (OK, Cancel, and text input) ---
def test_handle_prompt_with_text(driver):
    driver.get("https://the-internet.herokuapp.com/javascript_alerts")

    driver.find_element(By.XPATH, "//button[text()='Click for JS Prompt']").click()

    wait = WebDriverWait(driver, 10)
    alert = wait.until(EC.alert_is_present())

    print(f"Prompt text: {alert.text}")
    assert alert.text == "I am a JS prompt"

    # Type text into the prompt
    alert.send_keys("Hello Selenium!")

    # Accept (click OK)
    alert.accept()

    result = driver.find_element(By.ID, "result")
    assert result.text == "You entered: Hello Selenium!"


def test_handle_prompt_dismiss(driver):
    driver.get("https://the-internet.herokuapp.com/javascript_alerts")

    driver.find_element(By.XPATH, "//button[text()='Click for JS Prompt']").click()

    wait = WebDriverWait(driver, 10)
    alert = wait.until(EC.alert_is_present())

    # Dismiss (click Cancel) -- text is discarded
    alert.dismiss()

    result = driver.find_element(By.ID, "result")
    assert result.text == "You entered: null"


# --- CHECKING IF ALERT IS PRESENT ---
def test_check_if_alert_present(driver):
    driver.get("https://the-internet.herokuapp.com/javascript_alerts")

    # No alert should be present initially
    assert not is_alert_present(driver)

    # Trigger an alert
    driver.find_element(By.XPATH, "//button[text()='Click for JS Alert']").click()

    # Now an alert should be present
    assert is_alert_present(driver)

    # Clean up
    driver.switch_to.alert.accept()


def is_alert_present(driver, timeout=2):
    """Helper function to check if a JavaScript alert is present."""
    try:
        WebDriverWait(driver, timeout).until(EC.alert_is_present())
        return True
    except TimeoutException:
        return False


# --- TRIGGERING ALERT VIA JAVASCRIPT ---
def test_trigger_alert_via_javascript(driver):
    driver.get("https://the-internet.herokuapp.com/javascript_alerts")

    # You can trigger alerts via JavaScript execution
    driver.execute_script("alert('Custom alert from Selenium!');")

    wait = WebDriverWait(driver, 10)
    alert = wait.until(EC.alert_is_present())

    assert alert.text == "Custom alert from Selenium!"
    alert.accept()


# --- NON-NATIVE ALERT (SweetAlert / Modal) ---
def test_handle_non_native_alert(driver):
    # Non-native alerts are regular DOM elements -- use FindElement
    driver.get("https://the-internet.herokuapp.com/entry_ad")

    wait = WebDriverWait(driver, 10)

    # Wait for the modal to be visible
    modal = wait.until(
        EC.visibility_of_element_located((By.CSS_SELECTOR, ".modal"))
    )

    # Read the modal title
    title = modal.find_element(By.CSS_SELECTOR, ".modal-title h3").text
    print(f"Modal title: {title}")

    # Close the modal
    close_btn = wait.until(
        EC.element_to_be_clickable((By.CSS_SELECTOR, ".modal-footer p"))
    )
    close_btn.click()

    # Wait for modal to disappear
    wait.until(
        EC.invisibility_of_element_located((By.CSS_SELECTOR, ".modal"))
    )

    print("Non-native modal dismissed successfully")
```

---

## Step-by-Step Walkthrough

1. **Navigate to the alerts page** -- The test site provides buttons that trigger each dialog type.
2. **Click a trigger button** -- This opens the JavaScript dialog.
3. **Wait for the alert** -- `ExpectedConditions.AlertIsPresent()` / `EC.alert_is_present()` polls until the alert exists and returns the `IAlert` / `Alert` object.
4. **Read the text** -- `.Text` / `.text` returns the message displayed in the dialog.
5. **Send keys (prompts only)** -- `.SendKeys("text")` / `.send_keys("text")` types into the prompt input. This has no effect on alerts or confirms.
6. **Accept or dismiss** -- `.Accept()` clicks OK; `.Dismiss()` clicks Cancel. For a simple alert, both do the same thing.
7. **Verify the result** -- After dismissing the dialog, focus returns to the page and you can assert outcomes.

---

## Common Mistakes & How to Avoid Them

### 1. Not waiting for the alert before switching
```csharp
// BAD: alert may not be present yet
IAlert alert = driver.SwitchTo().Alert(); // NoAlertPresentException!

// GOOD: wait first
IAlert alert = wait.Until(ExpectedConditions.AlertIsPresent());
```

### 2. Trying to find alert elements with FindElement
```python
# BAD: alerts are not DOM elements
driver.find_element(By.ID, "alert-ok-button")  # NoSuchElementException

# GOOD: use switch_to.alert
alert = driver.switch_to.alert
alert.accept()
```

### 3. Sending keys to an alert or confirm (not a prompt)
Calling `SendKeys` on a regular alert or confirm will throw an error in some browsers or be silently ignored. Only prompts have an input field.

### 4. Confusing native alerts with modal dialogs
Libraries like SweetAlert, Bootstrap, and Material UI create modals that look like alerts but are regular HTML elements. If `driver.switch_to.alert` throws `NoAlertPresentException`, the dialog is probably a DOM element -- use `FindElement` instead.

### 5. Forgetting to accept/dismiss the alert
If you leave an alert open and try to interact with the page, Selenium throws `UnhandledAlertException`. Always accept or dismiss before continuing.

---

## Best Practices

1. **Always wait for alerts** using `ExpectedConditions.AlertIsPresent()` before switching.
2. **Read the alert text first**, then accept or dismiss. Once accepted, the text is gone.
3. **Use a helper method** like `IsAlertPresent()` for scenarios where an alert may or may not appear.
4. **Identify the dialog type** before writing automation. Check if it is a native browser dialog or a DOM-based modal.
5. **Handle unexpected alerts** in your test teardown to prevent cascading failures.
6. **Use `execute_script`** to trigger alerts in tests when the page does not provide a trigger button.

---

## Hands-On Exercise

**Goal:** Automate all three alert types on the JavaScript Alerts page.

1. Navigate to `https://the-internet.herokuapp.com/javascript_alerts`.
2. Handle the JS Alert: read the text, accept it, verify the result message.
3. Handle the JS Confirm: dismiss it, verify the result says "Cancel".
4. Handle the JS Prompt: type "Test Automation", accept it, verify the result shows the typed text.
5. Handle the JS Prompt again: dismiss without typing, verify the result says "null".

**Expected output:**
```
Alert text: I am a JS Alert
Result: You successfully clicked an alert

Confirm text: I am a JS Confirm
Result: You clicked: Cancel

Prompt text: I am a JS prompt
Result: You entered: Test Automation

Prompt dismissed result: You entered: null

All alert tests passed!
```

---

## Real-World Scenario

Many web applications show a confirmation dialog when the user attempts to delete a record:

```python
# User clicks "Delete Account" button
driver.find_element(By.ID, "delete-account-btn").click()

# A native confirm dialog appears: "Are you sure you want to delete your account?"
wait = WebDriverWait(driver, 5)
alert = wait.until(EC.alert_is_present())

# Log the confirmation message
print(f"Confirmation: {alert.text}")

# In a "happy path" test, accept the deletion
alert.accept()

# In a "cancel" test, dismiss to cancel
# alert.dismiss()

# Verify the account deletion was processed
wait.until(EC.visibility_of_element_located((By.ID, "deletion-confirmation")))
```

Some applications use a non-native confirmation modal instead. In that case:

```python
# A Bootstrap modal appears with a "Confirm Delete" button
modal = wait.until(EC.visibility_of_element_located((By.ID, "delete-modal")))
confirm_btn = modal.find_element(By.CSS_SELECTOR, ".btn-danger")
confirm_btn.click()
```

The key skill is identifying which type of dialog you are dealing with and using the correct approach.

---

## Resources

- [Selenium Alerts Documentation](https://www.selenium.dev/documentation/webdriver/interactions/alerts/)
- [The Internet Herokuapp - JavaScript Alerts](https://the-internet.herokuapp.com/javascript_alerts)
- [MDN: Window.alert()](https://developer.mozilla.org/en-US/docs/Web/API/Window/alert)
- [MDN: Window.confirm()](https://developer.mozilla.org/en-US/docs/Web/API/Window/confirm)
- [MDN: Window.prompt()](https://developer.mozilla.org/en-US/docs/Web/API/Window/prompt)
