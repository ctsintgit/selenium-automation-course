# Day 17: Page Object Model (POM) — Architecture and Principles

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain what the Page Object Model is and why it is the industry standard
- Apply POM principles: separate page structure from test logic
- Build a Page Object class with locators as properties and actions as methods
- Write clean, maintainable tests that use Page Objects
- Follow the "one page = one class" rule
- Create a multi-page flow (login to dashboard) using POM

---

## 1. What Is the Page Object Model?

The Page Object Model (POM) is a design pattern where each web page (or significant component) in your application is represented by a class. That class contains:

- **Locators** — how to find elements on the page (By.Id, By.CssSelector, etc.)
- **Methods** — actions a user can perform on the page (login, add to cart, search)

Tests never directly call `driver.FindElement()`. Instead, they call methods on page objects.

### Without POM (fragile, duplicated)

```csharp
// Test 1
driver.FindElement(By.Id("user-name")).SendKeys("user");
driver.FindElement(By.Id("password")).SendKeys("pass");
driver.FindElement(By.Id("login-button")).Click();

// Test 2 — same locators repeated
driver.FindElement(By.Id("user-name")).SendKeys("other_user");
driver.FindElement(By.Id("password")).SendKeys("other_pass");
driver.FindElement(By.Id("login-button")).Click();
```

If the `id="login-button"` changes to `id="submit-btn"`, you must update every test.

### With POM (maintainable, clean)

```csharp
// Test 1
loginPage.Login("user", "pass");

// Test 2
loginPage.Login("other_user", "other_pass");
```

If the locator changes, you update it in one place — the `LoginPage` class.

---

## 2. POM Principles

1. **Encapsulation** — Locators are private to the page class. Tests never see raw `By` selectors.
2. **Single Responsibility** — Each page class handles only one page or component.
3. **Methods return pages** — A `Login()` method returns an `InventoryPage` object, modeling the navigation flow.
4. **No assertions in page objects** — Page objects describe what the page can do. Tests decide what is correct.
5. **Readability** — Tests read like user stories: `loginPage.Login(user, pass)` then `inventoryPage.GetProductCount()`.

---

## 3. Anatomy of a Page Object

```
Page Object Class
├── Constructor (receives IWebDriver)
├── Locators (private By fields or properties)
├── Action Methods (Login, Search, AddToCart)
│   └── Return the next page object or void
├── Query Methods (GetErrorMessage, GetProductCount)
│   └── Return strings, ints, bools
└── Wait Logic (encapsulated inside methods)
```

---

## 4. Code Example: C# — Page Objects and Tests

### Project Structure

```
SeleniumTests/
├── Base/
│   └── BaseTest.cs
├── Pages/
│   ├── LoginPage.cs
│   └── InventoryPage.cs
├── Tests/
│   └── LoginTests.cs
└── SeleniumTests.csproj
```

### LoginPage.cs

```csharp
// File: Pages/LoginPage.cs
using OpenQA.Selenium;
using OpenQA.Selenium.Support.UI;
using System;

namespace SeleniumTests.Pages
{
    /// <summary>
    /// Represents the Sauce Demo login page.
    /// All locators and actions for this page are encapsulated here.
    /// </summary>
    public class LoginPage
    {
        private readonly IWebDriver _driver;
        private readonly WebDriverWait _wait;

        // --- LOCATORS (private — tests never see these) ---
        private readonly By _usernameInput = By.Id("user-name");
        private readonly By _passwordInput = By.Id("password");
        private readonly By _loginButton = By.Id("login-button");
        private readonly By _errorMessage = By.CssSelector("[data-test='error']");

        /// <summary>
        /// Constructor — receives the driver from the test.
        /// </summary>
        public LoginPage(IWebDriver driver)
        {
            _driver = driver;
            _wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
        }

        // --- ACTION METHODS ---

        /// <summary>
        /// Enters username and password, clicks login.
        /// Returns InventoryPage because a successful login navigates there.
        /// </summary>
        public InventoryPage Login(string username, string password)
        {
            EnterUsername(username);
            EnterPassword(password);
            ClickLogin();

            // Return the next page object — models the navigation flow
            return new InventoryPage(_driver);
        }

        /// <summary>
        /// Attempts login with invalid credentials (does not return InventoryPage).
        /// Returns this LoginPage because we stay on the same page.
        /// </summary>
        public LoginPage LoginExpectingError(string username, string password)
        {
            EnterUsername(username);
            EnterPassword(password);
            ClickLogin();
            return this;
        }

        public LoginPage EnterUsername(string username)
        {
            var element = _wait.Until(d => d.FindElement(_usernameInput));
            element.Clear();
            element.SendKeys(username);
            return this; // Fluent pattern — allows method chaining
        }

        public LoginPage EnterPassword(string password)
        {
            var element = _driver.FindElement(_passwordInput);
            element.Clear();
            element.SendKeys(password);
            return this;
        }

        public void ClickLogin()
        {
            _driver.FindElement(_loginButton).Click();
        }

        // --- QUERY METHODS ---

        /// <summary>
        /// Returns the error message text, or empty string if no error is visible.
        /// </summary>
        public string GetErrorMessage()
        {
            try
            {
                var error = _wait.Until(d => d.FindElement(_errorMessage));
                return error.Text;
            }
            catch (WebDriverTimeoutException)
            {
                return string.Empty;
            }
        }

        /// <summary>
        /// Returns true if the error message container is displayed.
        /// </summary>
        public bool IsErrorDisplayed()
        {
            try
            {
                return _driver.FindElement(_errorMessage).Displayed;
            }
            catch (NoSuchElementException)
            {
                return false;
            }
        }
    }
}
```

### InventoryPage.cs

```csharp
// File: Pages/InventoryPage.cs
using OpenQA.Selenium;
using OpenQA.Selenium.Support.UI;
using System;
using System.Collections.Generic;
using System.Linq;

namespace SeleniumTests.Pages
{
    /// <summary>
    /// Represents the Sauce Demo inventory/products page.
    /// </summary>
    public class InventoryPage
    {
        private readonly IWebDriver _driver;
        private readonly WebDriverWait _wait;

        // --- LOCATORS ---
        private readonly By _inventoryItems = By.ClassName("inventory_item");
        private readonly By _pageTitle = By.ClassName("title");
        private readonly By _addToCartButtons = By.CssSelector("[data-test^='add-to-cart']");
        private readonly By _cartBadge = By.ClassName("shopping_cart_badge");
        private readonly By _sortDropdown = By.ClassName("product_sort_container");

        public InventoryPage(IWebDriver driver)
        {
            _driver = driver;
            _wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));

            // Verify we actually landed on the inventory page
            _wait.Until(d => d.FindElement(_pageTitle));
        }

        // --- QUERY METHODS ---

        public string GetPageTitle()
        {
            return _driver.FindElement(_pageTitle).Text;
        }

        public int GetProductCount()
        {
            var items = _driver.FindElements(_inventoryItems);
            return items.Count;
        }

        public List<string> GetProductNames()
        {
            var nameElements = _driver.FindElements(
                By.ClassName("inventory_item_name"));
            return nameElements.Select(e => e.Text).ToList();
        }

        // --- ACTION METHODS ---

        public InventoryPage AddFirstProductToCart()
        {
            var buttons = _driver.FindElements(_addToCartButtons);
            if (buttons.Count > 0)
            {
                buttons[0].Click();
            }
            return this;
        }

        public int GetCartItemCount()
        {
            try
            {
                var badge = _driver.FindElement(_cartBadge);
                return int.Parse(badge.Text);
            }
            catch (NoSuchElementException)
            {
                return 0;
            }
        }
    }
}
```

### LoginTests.cs (using Page Objects)

```csharp
// File: Tests/LoginTests.cs
using NUnit.Framework;
using SeleniumTests.Base;
using SeleniumTests.Pages;

namespace SeleniumTests.Tests
{
    [TestFixture]
    public class LoginTests : BaseTest
    {
        private LoginPage loginPage;

        [SetUp]
        public void NavigateToLogin()
        {
            // Create a fresh LoginPage for each test
            loginPage = new LoginPage(Driver);
        }

        [Test]
        public void ValidLogin_ShouldShowInventoryPage()
        {
            // Act — Login returns an InventoryPage
            var inventoryPage = loginPage.Login("standard_user", "secret_sauce");

            // Assert — verify we are on the inventory page
            Assert.That(inventoryPage.GetPageTitle(), Is.EqualTo("Products"));
            Assert.That(inventoryPage.GetProductCount(), Is.GreaterThan(0));
        }

        [Test]
        public void InvalidLogin_ShouldShowErrorMessage()
        {
            // Act
            loginPage.LoginExpectingError("invalid_user", "wrong_pass");

            // Assert
            string error = loginPage.GetErrorMessage();
            Assert.That(error, Does.Contain("Username and password do not match"));
        }

        [Test]
        public void EmptyCredentials_ShouldRequireUsername()
        {
            // Act
            loginPage.LoginExpectingError("", "");

            // Assert
            Assert.That(loginPage.GetErrorMessage(),
                Does.Contain("Username is required"));
        }

        [Test]
        public void ValidLogin_ShouldShowSixProducts()
        {
            // Act
            var inventoryPage = loginPage.Login("standard_user", "secret_sauce");

            // Assert
            Assert.That(inventoryPage.GetProductCount(), Is.EqualTo(6),
                "Sauce Demo should display exactly 6 products");
        }
    }
}
```

---

## 5. Code Example: Python — Page Objects and Tests

### Project Structure

```
selenium_tests/
├── conftest.py
├── pages/
│   ├── __init__.py
│   ├── login_page.py
│   └── inventory_page.py
├── tests/
│   ├── __init__.py
│   └── test_login.py
```

### login_page.py

```python
# File: pages/login_page.py
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class LoginPage:
    """
    Page Object for the Sauce Demo login page.
    Locators are stored as tuples: (By strategy, value).
    """

    # --- LOCATORS (class-level constants) ---
    USERNAME_INPUT = (By.ID, "user-name")
    PASSWORD_INPUT = (By.ID, "password")
    LOGIN_BUTTON = (By.ID, "login-button")
    ERROR_MESSAGE = (By.CSS_SELECTOR, "[data-test='error']")

    def __init__(self, driver):
        """Receives the WebDriver instance from the test."""
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)

    # --- ACTION METHODS ---

    def login(self, username: str, password: str):
        """
        Enters credentials and clicks login.
        Returns an InventoryPage object (import here to avoid circular imports).
        """
        self.enter_username(username)
        self.enter_password(password)
        self.click_login()

        # Import here to avoid circular dependency
        from pages.inventory_page import InventoryPage
        return InventoryPage(self.driver)

    def login_expecting_error(self, username: str, password: str):
        """
        Attempts login with credentials expected to fail.
        Returns self (LoginPage) because we stay on this page.
        """
        self.enter_username(username)
        self.enter_password(password)
        self.click_login()
        return self

    def enter_username(self, username: str):
        element = self.wait.until(
            EC.presence_of_element_located(self.USERNAME_INPUT)
        )
        element.clear()
        element.send_keys(username)
        return self  # Fluent pattern

    def enter_password(self, password: str):
        element = self.driver.find_element(*self.PASSWORD_INPUT)
        element.clear()
        element.send_keys(password)
        return self

    def click_login(self):
        self.driver.find_element(*self.LOGIN_BUTTON).click()

    # --- QUERY METHODS ---

    def get_error_message(self) -> str:
        """Returns the error message text, or empty string if none."""
        try:
            error = self.wait.until(
                EC.presence_of_element_located(self.ERROR_MESSAGE)
            )
            return error.text
        except Exception:
            return ""

    def is_error_displayed(self) -> bool:
        """Returns True if the error container is visible."""
        try:
            return self.driver.find_element(*self.ERROR_MESSAGE).is_displayed()
        except Exception:
            return False
```

### inventory_page.py

```python
# File: pages/inventory_page.py
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class InventoryPage:
    """Page Object for the Sauce Demo inventory/products page."""

    # --- LOCATORS ---
    INVENTORY_ITEMS = (By.CLASS_NAME, "inventory_item")
    PAGE_TITLE = (By.CLASS_NAME, "title")
    ADD_TO_CART_BUTTONS = (By.CSS_SELECTOR, "[data-test^='add-to-cart']")
    CART_BADGE = (By.CLASS_NAME, "shopping_cart_badge")

    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)
        # Verify we landed on the correct page
        self.wait.until(EC.presence_of_element_located(self.PAGE_TITLE))

    # --- QUERY METHODS ---

    def get_page_title(self) -> str:
        return self.driver.find_element(*self.PAGE_TITLE).text

    def get_product_count(self) -> int:
        items = self.driver.find_elements(*self.INVENTORY_ITEMS)
        return len(items)

    def get_product_names(self) -> list:
        elements = self.driver.find_elements(By.CLASS_NAME, "inventory_item_name")
        return [el.text for el in elements]

    # --- ACTION METHODS ---

    def add_first_product_to_cart(self):
        buttons = self.driver.find_elements(*self.ADD_TO_CART_BUTTONS)
        if buttons:
            buttons[0].click()
        return self

    def get_cart_item_count(self) -> int:
        try:
            badge = self.driver.find_element(*self.CART_BADGE)
            return int(badge.text)
        except Exception:
            return 0
```

### test_login.py

```python
# File: tests/test_login.py
import pytest
from pages.login_page import LoginPage


class TestLogin:
    """Tests for the Sauce Demo login functionality using Page Objects."""

    def test_valid_login_shows_inventory(self, driver):
        """Successful login should navigate to the Products page."""
        login_page = LoginPage(driver)
        inventory_page = login_page.login("standard_user", "secret_sauce")

        assert inventory_page.get_page_title() == "Products"
        assert inventory_page.get_product_count() > 0

    def test_invalid_login_shows_error(self, driver):
        """Invalid credentials should display an error message."""
        login_page = LoginPage(driver)
        login_page.login_expecting_error("invalid_user", "wrong_pass")

        error = login_page.get_error_message()
        assert "Username and password do not match" in error

    def test_empty_credentials_shows_error(self, driver):
        """Empty credentials should show 'Username is required'."""
        login_page = LoginPage(driver)
        login_page.login_expecting_error("", "")

        assert "Username is required" in login_page.get_error_message()

    def test_valid_login_shows_six_products(self, driver):
        """The inventory page should display exactly 6 products."""
        login_page = LoginPage(driver)
        inventory_page = login_page.login("standard_user", "secret_sauce")

        assert inventory_page.get_product_count() == 6
```

---

## 6. Step-by-Step Walkthrough

### How `loginPage.Login("user", "pass")` works internally:

1. Test calls `loginPage.Login("standard_user", "secret_sauce")`
2. `Login()` calls `EnterUsername()` which finds `By.Id("user-name")` and types the username
3. `Login()` calls `EnterPassword()` which finds `By.Id("password")` and types the password
4. `Login()` calls `ClickLogin()` which finds `By.Id("login-button")` and clicks it
5. `Login()` creates and returns a new `InventoryPage(driver)`
6. `InventoryPage` constructor waits for the page title to appear (validation)
7. Test calls `inventoryPage.GetPageTitle()` and asserts the result

The test never touches a `By` locator or calls `FindElement` directly.

---

## 7. Common Mistakes and How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Putting assertions in page objects | Violates single responsibility; page objects become brittle | Page objects return data; tests make assertions |
| Exposing `By` locators as public | Tests couple to locator details | Keep locators private; expose only action/query methods |
| One giant page object for the entire site | Unmaintainable, hundreds of methods | One class per page or significant component |
| Not returning page objects from navigation methods | Caller doesn't know which page they are on | `Login()` returns `InventoryPage`; `Logout()` returns `LoginPage` |
| Hardcoding waits in every method | Duplicated timeout values | Set the wait once in the constructor |

---

## 8. Best Practices

1. **One page = one class** — `LoginPage`, `InventoryPage`, `CartPage`, `CheckoutPage`
2. **Methods return the next page** — `Login()` returns `InventoryPage`, modeling user navigation
3. **No assertions in page objects** — Keep them reusable across positive and negative tests
4. **Use explicit waits inside page objects** — The page object knows when its elements are ready
5. **Fluent interface** — Return `this`/`self` from setter methods to allow chaining: `loginPage.EnterUsername("user").EnterPassword("pass").ClickLogin()`
6. **Constructor validates the page** — Check that a key element exists when the page object is created

---

## 9. Hands-On Exercise

**Task:** Create a complete POM for the Sauce Demo login-to-cart flow.

1. `LoginPage` — `Login()`, `LoginExpectingError()`, `GetErrorMessage()`
2. `InventoryPage` — `GetProductCount()`, `AddProductToCart(index)`, `GetCartCount()`
3. Write 4 tests:
   - Valid login shows Products title
   - Adding a product increments the cart badge to 1
   - Adding two products shows cart badge of 2
   - Invalid login shows error message

**Expected Output:**

```
PASSED test_valid_login_shows_products_title
PASSED test_add_one_product_to_cart
PASSED test_add_two_products_to_cart
PASSED test_invalid_login_shows_error
========================= 4 passed =========================
```

---

## 10. Real-World Scenario

At a company with 500 Selenium tests, the development team redesigns the login page. The HTML IDs change from `user-name` to `email-input` and `login-button` to `submit-form`.

**Without POM:** You search-and-replace across 50 test files, hoping you catch every occurrence. One miss causes a cascade of failures.

**With POM:** You open `LoginPage.cs`, change two locator strings, and all 50 tests pass again. The change takes 30 seconds.

This is why every professional Selenium framework uses POM.

---

## 11. Resources

- [Selenium Page Object Models](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/)
- [Martin Fowler on Page Objects](https://martinfowler.com/bliki/PageObject.html)
- [Sauce Demo Practice Site](https://www.saucedemo.com)
- [NUnit Framework](https://docs.nunit.org/)
- [pytest Documentation](https://docs.pytest.org/)
