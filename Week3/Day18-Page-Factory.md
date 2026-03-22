# Day 18: Page Factory Pattern (C#) and Equivalent in Python

## Learning Objectives

By the end of this lesson, you will be able to:

- Use the C# Page Factory pattern with `[FindsBy]` attributes and `PageFactory.InitElements()`
- Install and configure the SeleniumExtras.PageObjects NuGet package
- Understand lazy initialization of web elements
- Implement the Python equivalent using `@property` decorators
- Compare Page Factory vs manual `By` locators and choose the right approach
- Refactor a plain POM to use the Page Factory pattern

---

## 1. Core Concept: What Is Page Factory?

Page Factory is an extension of the Page Object Model that uses annotations/attributes to declare element locators directly on fields. The framework automatically initializes these elements when the page object is created.

**Plain POM (Day 17):**

```csharp
private readonly By _usernameInput = By.Id("user-name");
// ...
driver.FindElement(_usernameInput).SendKeys("user");
```

**Page Factory:**

```csharp
[FindsBy(How = How.Id, Using = "user-name")]
private IWebElement UsernameInput;
// ...
UsernameInput.SendKeys("user");  // Element is found automatically
```

The key difference: with Page Factory, you work with `IWebElement` directly instead of `By` locators. The framework handles the `FindElement` call for you.

---

## 2. How It Works

### C# Page Factory

1. You annotate fields with `[FindsBy]` specifying the locator strategy and value
2. In the constructor, you call `PageFactory.InitElements(driver, this)`
3. The framework creates proxy objects for each annotated field
4. When you first access a field (e.g., `UsernameInput.SendKeys()`), it calls `FindElement` behind the scenes
5. This is called **lazy initialization** — the element is not located until you use it

### Python Equivalent

Python does not have a built-in Page Factory in Selenium 4. There are two common approaches:

- **`@property` decorators** — Define element access as properties that call `find_element` each time (recommended)
- **Third-party `page-factory` package** — Mimics Java/C# Page Factory with decorators

We will focus on the `@property` approach because it requires no extra dependencies and is the most Pythonic.

---

## 3. C# Page Factory Setup

### Install the NuGet Package

Page Factory was removed from the core Selenium 4 package. Install the community-maintained package:

```bash
dotnet add package DotNetSeleniumExtras.PageObjects
```

> **Note:** The package name is `DotNetSeleniumExtras.PageObjects`. The namespace is `SeleniumExtras.PageObjects`.

---

## 4. Code Example: C# — Page Factory

```csharp
// File: Pages/LoginPageFactory.cs
using OpenQA.Selenium;
using OpenQA.Selenium.Support.UI;
using SeleniumExtras.PageObjects;
using System;

namespace SeleniumTests.Pages
{
    /// <summary>
    /// Login page using the Page Factory pattern.
    /// Elements are declared as fields with [FindsBy] attributes.
    /// </summary>
    public class LoginPageFactory
    {
        private readonly IWebDriver _driver;
        private readonly WebDriverWait _wait;

        // --- PAGE FACTORY ELEMENTS ---
        // These fields are automatically initialized by PageFactory.InitElements()

        [FindsBy(How = How.Id, Using = "user-name")]
        private IWebElement UsernameInput;

        [FindsBy(How = How.Id, Using = "password")]
        private IWebElement PasswordInput;

        [FindsBy(How = How.Id, Using = "login-button")]
        private IWebElement LoginButton;

        [FindsBy(How = How.CssSelector, Using = "[data-test='error']")]
        private IWebElement ErrorMessage;

        // You can also use FindsByAll for elements that match multiple criteria
        // [FindsByAll]
        // [FindsBy(How = How.TagName, Using = "input")]
        // [FindsBy(How = How.ClassName, Using = "form_input")]
        // private IWebElement FormInput;

        /// <summary>
        /// Constructor — calls PageFactory.InitElements to wire up all [FindsBy] fields.
        /// </summary>
        public LoginPageFactory(IWebDriver driver)
        {
            _driver = driver;
            _wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));

            // THIS IS THE KEY LINE — initializes all [FindsBy] fields
            PageFactory.InitElements(driver, this);
        }

        // --- ACTION METHODS ---

        public InventoryPageFactory Login(string username, string password)
        {
            // No need to call FindElement — just use the fields directly
            UsernameInput.Clear();
            UsernameInput.SendKeys(username);

            PasswordInput.Clear();
            PasswordInput.SendKeys(password);

            LoginButton.Click();

            return new InventoryPageFactory(_driver);
        }

        public LoginPageFactory LoginExpectingError(string username, string password)
        {
            UsernameInput.Clear();
            UsernameInput.SendKeys(username);

            PasswordInput.Clear();
            PasswordInput.SendKeys(password);

            LoginButton.Click();

            return this;
        }

        // --- QUERY METHODS ---

        public string GetErrorMessage()
        {
            try
            {
                // Wait for the error to appear, then read its text
                _wait.Until(d => ErrorMessage.Displayed);
                return ErrorMessage.Text;
            }
            catch (WebDriverTimeoutException)
            {
                return string.Empty;
            }
        }

        public bool IsErrorDisplayed()
        {
            try
            {
                return ErrorMessage.Displayed;
            }
            catch (NoSuchElementException)
            {
                return false;
            }
        }
    }
}
```

```csharp
// File: Pages/InventoryPageFactory.cs
using OpenQA.Selenium;
using OpenQA.Selenium.Support.UI;
using SeleniumExtras.PageObjects;
using System;
using System.Collections.Generic;
using System.Linq;

namespace SeleniumTests.Pages
{
    public class InventoryPageFactory
    {
        private readonly IWebDriver _driver;
        private readonly WebDriverWait _wait;

        [FindsBy(How = How.ClassName, Using = "title")]
        private IWebElement PageTitle;

        [FindsBy(How = How.ClassName, Using = "inventory_item")]
        private IList<IWebElement> InventoryItems;

        [FindsBy(How = How.ClassName, Using = "inventory_item_name")]
        private IList<IWebElement> ProductNames;

        [FindsBy(How = How.CssSelector, Using = "[data-test^='add-to-cart']")]
        private IList<IWebElement> AddToCartButtons;

        [FindsBy(How = How.ClassName, Using = "shopping_cart_badge")]
        private IWebElement CartBadge;

        public InventoryPageFactory(IWebDriver driver)
        {
            _driver = driver;
            _wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
            PageFactory.InitElements(driver, this);

            // Wait for the page to be ready
            _wait.Until(d => PageTitle.Displayed);
        }

        public string GetPageTitle() => PageTitle.Text;

        public int GetProductCount() => InventoryItems.Count;

        public List<string> GetProductNames() =>
            ProductNames.Select(e => e.Text).ToList();

        public InventoryPageFactory AddFirstProductToCart()
        {
            if (AddToCartButtons.Count > 0)
            {
                AddToCartButtons[0].Click();
            }
            return this;
        }

        public int GetCartItemCount()
        {
            try
            {
                return int.Parse(CartBadge.Text);
            }
            catch
            {
                return 0;
            }
        }
    }
}
```

```csharp
// File: Tests/LoginFactoryTests.cs
using NUnit.Framework;
using SeleniumTests.Base;
using SeleniumTests.Pages;

namespace SeleniumTests.Tests
{
    [TestFixture]
    public class LoginFactoryTests : BaseTest
    {
        private LoginPageFactory loginPage;

        [SetUp]
        public void NavigateToLogin()
        {
            loginPage = new LoginPageFactory(Driver);
        }

        [Test]
        public void ValidLogin_ShouldShowProducts()
        {
            var inventoryPage = loginPage.Login("standard_user", "secret_sauce");

            Assert.That(inventoryPage.GetPageTitle(), Is.EqualTo("Products"));
            Assert.That(inventoryPage.GetProductCount(), Is.EqualTo(6));
        }

        [Test]
        public void InvalidLogin_ShouldShowError()
        {
            loginPage.LoginExpectingError("bad_user", "bad_pass");

            Assert.That(loginPage.GetErrorMessage(),
                Does.Contain("Username and password do not match"));
        }

        [Test]
        public void AddProductToCart_ShouldShowBadge()
        {
            var inventoryPage = loginPage.Login("standard_user", "secret_sauce");
            inventoryPage.AddFirstProductToCart();

            Assert.That(inventoryPage.GetCartItemCount(), Is.EqualTo(1));
        }
    }
}
```

---

## 5. Code Example: Python — @property Equivalent

Python does not have `[FindsBy]` attributes. The recommended Pythonic approach uses `@property` decorators that find elements on each access, providing the same lazy behavior.

```python
# File: pages/login_page_property.py
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class LoginPage:
    """
    Login page using @property decorators as a Python equivalent
    of C#'s Page Factory. Each property calls find_element when accessed,
    ensuring fresh element references every time.
    """

    # Locator constants — define them once, use in properties
    _USERNAME_INPUT = (By.ID, "user-name")
    _PASSWORD_INPUT = (By.ID, "password")
    _LOGIN_BUTTON = (By.ID, "login-button")
    _ERROR_MESSAGE = (By.CSS_SELECTOR, "[data-test='error']")

    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)

    # --- ELEMENT PROPERTIES (Python's Page Factory equivalent) ---
    # Each property finds the element fresh every time it is accessed.
    # This avoids StaleElementReferenceException.

    @property
    def username_input(self):
        """Returns the username input element."""
        return self.wait.until(
            EC.presence_of_element_located(self._USERNAME_INPUT)
        )

    @property
    def password_input(self):
        """Returns the password input element."""
        return self.driver.find_element(*self._PASSWORD_INPUT)

    @property
    def login_button(self):
        """Returns the login button element."""
        return self.driver.find_element(*self._LOGIN_BUTTON)

    @property
    def error_message(self):
        """Returns the error message element."""
        return self.wait.until(
            EC.presence_of_element_located(self._ERROR_MESSAGE)
        )

    # --- ACTION METHODS ---

    def login(self, username: str, password: str):
        """Logs in and returns the InventoryPage."""
        self.username_input.clear()
        self.username_input.send_keys(username)
        self.password_input.clear()
        self.password_input.send_keys(password)
        self.login_button.click()

        from pages.inventory_page_property import InventoryPage
        return InventoryPage(self.driver)

    def login_expecting_error(self, username: str, password: str):
        """Attempts login expecting failure. Returns self."""
        self.username_input.clear()
        self.username_input.send_keys(username)
        self.password_input.clear()
        self.password_input.send_keys(password)
        self.login_button.click()
        return self

    def get_error_message(self) -> str:
        """Returns error text or empty string."""
        try:
            return self.error_message.text
        except Exception:
            return ""
```

```python
# File: pages/inventory_page_property.py
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class InventoryPage:
    """Inventory page using @property decorators."""

    _INVENTORY_ITEMS = (By.CLASS_NAME, "inventory_item")
    _PAGE_TITLE = (By.CLASS_NAME, "title")
    _ADD_TO_CART_BUTTONS = (By.CSS_SELECTOR, "[data-test^='add-to-cart']")
    _CART_BADGE = (By.CLASS_NAME, "shopping_cart_badge")

    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)
        # Validate page loaded
        self.wait.until(EC.presence_of_element_located(self._PAGE_TITLE))

    @property
    def page_title(self):
        return self.driver.find_element(*self._PAGE_TITLE)

    @property
    def inventory_items(self):
        return self.driver.find_elements(*self._INVENTORY_ITEMS)

    @property
    def add_to_cart_buttons(self):
        return self.driver.find_elements(*self._ADD_TO_CART_BUTTONS)

    @property
    def cart_badge(self):
        return self.driver.find_element(*self._CART_BADGE)

    def get_page_title(self) -> str:
        return self.page_title.text

    def get_product_count(self) -> int:
        return len(self.inventory_items)

    def add_first_product_to_cart(self):
        buttons = self.add_to_cart_buttons
        if buttons:
            buttons[0].click()
        return self

    def get_cart_item_count(self) -> int:
        try:
            return int(self.cart_badge.text)
        except Exception:
            return 0
```

```python
# File: tests/test_login_property.py
import pytest
from pages.login_page_property import LoginPage


class TestLoginWithProperties:
    """Tests using the @property-based Page Factory approach."""

    def test_valid_login(self, driver):
        login_page = LoginPage(driver)
        inventory_page = login_page.login("standard_user", "secret_sauce")

        assert inventory_page.get_page_title() == "Products"
        assert inventory_page.get_product_count() == 6

    def test_invalid_login(self, driver):
        login_page = LoginPage(driver)
        login_page.login_expecting_error("invalid", "wrong")

        assert "Username and password do not match" in login_page.get_error_message()

    def test_add_product_to_cart(self, driver):
        login_page = LoginPage(driver)
        inventory_page = login_page.login("standard_user", "secret_sauce")
        inventory_page.add_first_product_to_cart()

        assert inventory_page.get_cart_item_count() == 1
```

---

## 6. Step-by-Step Walkthrough

### C# Page Factory Initialization Flow

1. `new LoginPageFactory(driver)` is called
2. Constructor calls `PageFactory.InitElements(driver, this)`
3. `InitElements` scans the class for fields with `[FindsBy]` attributes
4. For each field, it creates a **proxy** object (not a real element yet)
5. When you call `UsernameInput.SendKeys("user")`, the proxy calls `driver.FindElement(By.Id("user-name"))` at that moment
6. This lazy approach means elements are found only when needed

### Python @property Flow

1. `login_page = LoginPage(driver)` stores the driver
2. When you call `login_page.username_input.send_keys("user")`:
   - Python calls the `@property` getter method
   - The getter calls `wait.until(EC.presence_of_element_located(...))`
   - The element is found and returned
   - `.send_keys("user")` is called on the returned element
3. Next time you access `username_input`, it finds the element again (fresh reference)

---

## 7. Comparing Page Factory vs Plain POM

| Feature | Plain POM (Day 17) | Page Factory (Day 18) |
|---------|--------------------|-----------------------|
| Element declaration | `By` locators as fields | `IWebElement` fields with `[FindsBy]` |
| Element lookup | Explicit `FindElement()` calls | Automatic via proxy/property |
| Staleness handling | You control when to re-find | Proxies re-find on each access (C#) |
| Code verbosity | Slightly more boilerplate | Less code in methods |
| Extra dependency | None | `DotNetSeleniumExtras.PageObjects` (C#) |
| Python support | Native | Requires `@property` workaround |
| Debugging | Easy — you see every FindElement call | Harder — element resolution is hidden |
| Community preference | More common in modern projects | Declining in popularity |

### When to Use Which

- **Use Plain POM** when you want full control, easier debugging, and no extra dependencies
- **Use Page Factory** when you prefer less boilerplate and your team is familiar with the pattern
- **Most modern teams prefer Plain POM** because Page Factory was removed from core Selenium 4 and the community package is not actively maintained

---

## 8. Common Mistakes and How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `PageFactory.InitElements()` | All element fields are null | Always call it in the constructor |
| Using Page Factory without the NuGet package | Compilation error — `FindsBy` not found | Install `DotNetSeleniumExtras.PageObjects` |
| Caching stale elements in Python | `StaleElementReferenceException` | Use `@property` so elements are found fresh each time |
| Using `[FindsBy]` with dynamic locators | Cannot pass runtime values to attributes | Use plain `By` locators for dynamic elements |
| Mixing Page Factory and plain locators in one class | Inconsistent, confusing | Pick one approach per page object |

---

## 9. Best Practices

1. **Pick one pattern per project** — Either use Page Factory everywhere or use plain POM everywhere. Mixing creates confusion.
2. **Python: prefer `@property`** — It is native, requires no extra package, and avoids stale elements.
3. **C#: consider plain POM for new projects** — The `DotNetSeleniumExtras` package is community-maintained and may lag behind Selenium updates.
4. **Always wait before interacting** — Whether you use Page Factory or plain POM, wrap element access in explicit waits.
5. **Use `IList<IWebElement>` for collections** — Page Factory supports lists of elements for things like product grids.

---

## 10. Hands-On Exercise

**Task:** Refactor Day 17's plain POM into the Page Factory pattern.

1. Create `LoginPageFactory.cs` using `[FindsBy]` attributes
2. Create `InventoryPageFactory.cs` using `[FindsBy]` attributes
3. Create the Python equivalent using `@property` decorators
4. Write three tests proving both approaches produce identical results

**Expected Output:**

```
Passed!  - Failed:     0, Passed:     3, Skipped:     0, Total:     3
```

---

## 11. Real-World Scenario

Many legacy Selenium projects written in Java or C# before 2020 use Page Factory heavily. When upgrading to Selenium 4, teams often discover that Page Factory has been removed from the core library. They face a choice:

- **Install the community package** and keep using Page Factory (quick fix)
- **Refactor to plain POM** (more work upfront, better long-term maintainability)

Most teams that refactor report cleaner code and easier debugging. The `@property` approach in Python has always been the standard there because Python never had a built-in Page Factory.

---

## 12. Resources

- [DotNetSeleniumExtras.PageObjects NuGet](https://www.nuget.org/packages/DotNetSeleniumExtras.PageObjects)
- [Selenium Page Object Models](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/)
- [Python Property Decorators](https://docs.python.org/3/library/functions.html#property)
- [Why Page Factory Was Removed from Selenium 4](https://www.selenium.dev/blog/2021/selenium-4-is-released/)
