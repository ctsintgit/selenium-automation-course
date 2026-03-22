# Day 23: Shadow DOM — Locating and Interacting with Shadow Elements

## Learning Objectives

By the end of this lesson, you will be able to:

- Understand what Shadow DOM is and why web developers use it
- Distinguish between shadow host, shadow root, and shadow tree
- Use Selenium 4's native `GetShadowRoot()` / `shadow_root` to access shadow DOM elements
- Chain into nested shadow DOMs (shadow root inside shadow root)
- Find and interact with elements inside a shadow root
- Use JavaScriptExecutor as a fallback for complex shadow DOM scenarios
- Automate real-world web components that use shadow DOM

---

## 1. Core Concept Explanation

### What Is Shadow DOM?

Shadow DOM is a web standard that lets developers create **encapsulated** components. Think of it as a "hidden" DOM tree attached to a regular HTML element. The elements inside the shadow tree are isolated from the rest of the page:

- **CSS does not leak in or out.** Styles inside the shadow DOM do not affect the main page, and vice versa.
- **JavaScript queries do not cross the boundary.** `document.querySelector()` cannot see inside a shadow root.
- **Selenium locators also cannot cross the boundary** without special handling.

### Why Does Shadow DOM Exist?

Web components (custom HTML elements like `<my-dropdown>`, `<video-player>`) use shadow DOM so their internal structure does not conflict with the host page. Common examples:

- Browser built-in elements: `<video>`, `<input type="range">`, `<details>`
- UI component libraries: Salesforce Lightning, Ionic, Vaadin
- Chrome browser pages: `chrome://settings`, `chrome://downloads`

### Key Terminology

```
<my-component>          <-- Shadow Host (regular DOM element)
  #shadow-root (open)   <-- Shadow Root (the boundary)
    <div class="inner">  <-- Shadow Tree (hidden elements)
      <button>Click</button>
    </div>
</my-component>
```

| Term | Definition |
|------|-----------|
| **Shadow Host** | The regular DOM element that "hosts" the shadow tree |
| **Shadow Root** | The root node of the shadow tree (the boundary) |
| **Shadow Tree** | The hidden DOM tree inside the shadow root |
| **Open Shadow DOM** | JavaScript and Selenium can access it (mode: 'open') |
| **Closed Shadow DOM** | JavaScript cannot access it via `.shadowRoot` (mode: 'closed') |

Selenium 4 can access **open** shadow DOMs. Closed shadow DOMs require JS workarounds.

---

## 2. How It Works (Technical Breakdown)

### The Problem

Standard Selenium locators search the main DOM tree. They stop at shadow boundaries:

```
document
  |-- <body>
  |     |-- <div id="app">         <-- Selenium CAN find this
  |     |     |-- <my-component>   <-- Selenium CAN find this (shadow host)
  |     |     |     #shadow-root
  |     |     |       |-- <button> <-- Selenium CANNOT find this directly
```

If you try `driver.FindElement(By.CssSelector("my-component button"))`, it fails because `button` is inside the shadow root.

### The Solution: Selenium 4 Shadow Root API

Selenium 4 introduced a clean API to cross the shadow boundary:

1. Find the **shadow host** element (the custom element tag)
2. Call `.GetShadowRoot()` (C#) or `.shadow_root` (Python) on it
3. Use the returned shadow root object to find elements inside

```
// Pseudocode
shadowHost = driver.FindElement(By.CssSelector("my-component"))
shadowRoot = shadowHost.GetShadowRoot()  // cross the boundary
button = shadowRoot.FindElement(By.CssSelector("button"))  // inside shadow
button.Click()
```

### Locator Restrictions Inside Shadow Root

When searching inside a shadow root, only **CSS selectors** work reliably. XPath does not work inside shadow roots because XPath operates on the XML document tree, which does not include shadow trees.

| Locator Strategy | Works Inside Shadow Root? |
|-----------------|--------------------------|
| By.CssSelector | Yes |
| By.ClassName | Yes |
| By.Id | Yes |
| By.TagName | Yes |
| By.XPath | No |
| By.Name | Yes |

---

## 3. Code Example: C# (Complete, Runnable)

```csharp
// File: Day23_ShadowDom.cs
// Prerequisites:
//   dotnet new nunit -n Day23
//   cd Day23
//   dotnet add package Selenium.WebDriver
//   dotnet add package Selenium.Support

using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;

namespace Day23
{
    [TestFixture]
    public class ShadowDomTests
    {
        private IWebDriver driver;
        private WebDriverWait wait;

        [SetUp]
        public void SetUp()
        {
            var options = new ChromeOptions();
            options.AddArgument("--start-maximized");
            driver = new ChromeDriver(options);
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        [Test]
        public void AccessSingleShadowDom()
        {
            // Navigate to a page with shadow DOM elements
            // We'll use a test page that has web components
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            // For demonstration, let's create a shadow DOM element via JS
            // (since few public test sites have shadow DOM)
            var js = (IJavaScriptExecutor)driver;
            js.ExecuteScript(@"
                // Create a custom element with shadow DOM
                var host = document.createElement('div');
                host.id = 'shadow-host';
                document.body.prepend(host);

                var shadowRoot = host.attachShadow({mode: 'open'});
                shadowRoot.innerHTML = `
                    <div class='shadow-content'>
                        <h2 id='shadow-title'>Shadow DOM Title</h2>
                        <input id='shadow-input' type='text' placeholder='Type here' />
                        <button id='shadow-btn'>Shadow Button</button>
                    </div>
                `;
            ");

            // Step 1: Find the shadow host (regular DOM element)
            IWebElement shadowHost = driver.FindElement(By.Id("shadow-host"));

            // Step 2: Get the shadow root using Selenium 4 API
            ISearchContext shadowRoot = shadowHost.GetShadowRoot();

            // Step 3: Find elements INSIDE the shadow root
            // IMPORTANT: Only CSS selectors work here, not XPath
            IWebElement title = shadowRoot.FindElement(
                By.CssSelector("#shadow-title")
            );
            Console.WriteLine($"Shadow title text: {title.Text}");
            Assert.That(title.Text, Is.EqualTo("Shadow DOM Title"));

            // Interact with shadow DOM input
            IWebElement input = shadowRoot.FindElement(
                By.CssSelector("#shadow-input")
            );
            input.SendKeys("Hello Shadow DOM!");
            Console.WriteLine("Typed into shadow DOM input.");

            // Click shadow DOM button
            IWebElement button = shadowRoot.FindElement(
                By.CssSelector("#shadow-btn")
            );
            button.Click();
            Console.WriteLine("Clicked shadow DOM button.");
        }

        [Test]
        public void AccessNestedShadowDom()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

            var js = (IJavaScriptExecutor)driver;

            // Create nested shadow DOM structure:
            // outer-component > #shadow-root > inner-component > #shadow-root > button
            js.ExecuteScript(@"
                // Outer component
                var outer = document.createElement('div');
                outer.id = 'outer-host';
                document.body.prepend(outer);

                var outerShadow = outer.attachShadow({mode: 'open'});
                outerShadow.innerHTML = `
                    <p id='outer-text'>Outer Shadow Content</p>
                    <div id='inner-host'></div>
                `;

                // Inner component (nested inside outer's shadow)
                var innerHost = outerShadow.querySelector('#inner-host');
                var innerShadow = innerHost.attachShadow({mode: 'open'});
                innerShadow.innerHTML = `
                    <p id='inner-text'>Inner Shadow Content</p>
                    <button id='inner-btn'>Deep Button</button>
                `;
            ");

            // Step 1: Get outer shadow root
            IWebElement outerHost = driver.FindElement(By.Id("outer-host"));
            ISearchContext outerShadow = outerHost.GetShadowRoot();

            // Step 2: Find inner host INSIDE the outer shadow
            IWebElement innerHost = outerShadow.FindElement(
                By.CssSelector("#inner-host")
            );

            // Step 3: Get inner shadow root (chaining!)
            ISearchContext innerShadow = innerHost.GetShadowRoot();

            // Step 4: Find elements inside the inner shadow
            IWebElement innerText = innerShadow.FindElement(
                By.CssSelector("#inner-text")
            );
            Console.WriteLine($"Inner shadow text: {innerText.Text}");
            Assert.That(innerText.Text, Is.EqualTo("Inner Shadow Content"));

            IWebElement deepButton = innerShadow.FindElement(
                By.CssSelector("#inner-btn")
            );
            deepButton.Click();
            Console.WriteLine("Clicked button inside nested shadow DOM.");
        }

        [Test]
        public void AccessShadowDomViaJavaScript()
        {
            // JSExecutor fallback for complex or closed shadow DOMs
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

            var js = (IJavaScriptExecutor)driver;

            // Create shadow DOM
            js.ExecuteScript(@"
                var host = document.createElement('div');
                host.id = 'js-shadow-host';
                document.body.prepend(host);
                var sr = host.attachShadow({mode: 'open'});
                sr.innerHTML = '<p id=""js-para"">JS Shadow Text</p>';
            ");

            // Use JS to reach into shadow DOM
            IWebElement paragraph = (IWebElement)js.ExecuteScript(
                "return document.querySelector('#js-shadow-host')" +
                ".shadowRoot.querySelector('#js-para');"
            );

            Console.WriteLine($"Text via JS: {paragraph.Text}");
            Assert.That(paragraph.Text, Is.EqualTo("JS Shadow Text"));
        }

        [Test]
        public void AccessNestedShadowDomViaJavaScript()
        {
            // For deeply nested shadow DOMs, JS chaining is sometimes cleaner
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

            var js = (IJavaScriptExecutor)driver;

            js.ExecuteScript(@"
                var level1 = document.createElement('div');
                level1.id = 'level1';
                document.body.prepend(level1);

                var sr1 = level1.attachShadow({mode: 'open'});
                sr1.innerHTML = '<div id=""level2""></div>';

                var level2 = sr1.querySelector('#level2');
                var sr2 = level2.attachShadow({mode: 'open'});
                sr2.innerHTML = '<div id=""level3""></div>';

                var level3 = sr2.querySelector('#level3');
                var sr3 = level3.attachShadow({mode: 'open'});
                sr3.innerHTML = '<span id=""deep-text"">3 levels deep!</span>';
            ");

            // Chain through all three shadow roots in one JS call
            IWebElement deepElement = (IWebElement)js.ExecuteScript(@"
                return document.querySelector('#level1')
                    .shadowRoot.querySelector('#level2')
                    .shadowRoot.querySelector('#level3')
                    .shadowRoot.querySelector('#deep-text');
            ");

            Console.WriteLine($"Deep element text: {deepElement.Text}");
            Assert.That(deepElement.Text, Is.EqualTo("3 levels deep!"));
        }

        [Test]
        public void GetAllElementsInsideShadowDom()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/");

            var js = (IJavaScriptExecutor)driver;
            js.ExecuteScript(@"
                var host = document.createElement('div');
                host.id = 'list-host';
                document.body.prepend(host);
                var sr = host.attachShadow({mode: 'open'});
                sr.innerHTML = `
                    <ul>
                        <li class='item'>Item 1</li>
                        <li class='item'>Item 2</li>
                        <li class='item'>Item 3</li>
                    </ul>
                `;
            ");

            // Get shadow root
            ISearchContext shadowRoot = driver.FindElement(
                By.Id("list-host")
            ).GetShadowRoot();

            // FindElements (plural) works inside shadow root too
            var items = shadowRoot.FindElements(By.CssSelector(".item"));

            Console.WriteLine($"Found {items.Count} items in shadow DOM:");
            foreach (var item in items)
            {
                Console.WriteLine($"  - {item.Text}");
            }

            Assert.That(items.Count, Is.EqualTo(3));
        }

        [Test]
        public void ChromeSettingsPageShadowDom()
        {
            // Real-world example: Chrome's settings page uses nested shadow DOM
            driver.Navigate().GoToUrl("chrome://settings/");

            // Wait for the page to load
            wait.Until(d =>
            {
                try
                {
                    // The settings page has a deeply nested shadow DOM
                    // settings-ui > #shadow-root > settings-main > ...
                    var settingsUi = d.FindElement(By.TagName("settings-ui"));
                    return settingsUi != null;
                }
                catch
                {
                    return false;
                }
            });

            // Access via Selenium 4 shadow root
            IWebElement settingsUi = driver.FindElement(
                By.TagName("settings-ui")
            );
            ISearchContext settingsShadow = settingsUi.GetShadowRoot();

            // Find the main content area inside shadow root
            IWebElement settingsMain = settingsShadow.FindElement(
                By.CssSelector("settings-main")
            );

            Console.WriteLine("Successfully accessed Chrome settings shadow DOM!");
            Assert.That(settingsMain, Is.Not.Null);
        }
    }
}
```

---

## 4. Code Example: Python (Complete, Runnable)

```python
# File: test_day23_shadow_dom.py
# Prerequisites:
#   pip install selenium pytest

import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


@pytest.fixture
def driver():
    """Set up Chrome browser with Selenium 4 auto driver management."""
    options = Options()
    options.add_argument("--start-maximized")
    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


@pytest.fixture
def wait(driver):
    """Explicit wait instance."""
    return WebDriverWait(driver, 10)


class TestSingleShadowDom:

    def test_access_shadow_dom_elements(self, driver):
        """Access elements inside a single-level shadow DOM."""
        driver.get("https://the-internet.herokuapp.com/")

        # Create a shadow DOM element for testing
        driver.execute_script("""
            var host = document.createElement('div');
            host.id = 'shadow-host';
            document.body.prepend(host);

            var shadowRoot = host.attachShadow({mode: 'open'});
            shadowRoot.innerHTML = `
                <div class="shadow-content">
                    <h2 id="shadow-title">Shadow DOM Title</h2>
                    <input id="shadow-input" type="text"
                           placeholder="Type here" />
                    <button id="shadow-btn">Shadow Button</button>
                </div>
            `;
        """)

        # Step 1: Find the shadow host
        shadow_host = driver.find_element(By.ID, "shadow-host")

        # Step 2: Get shadow root using Selenium 4's shadow_root property
        shadow_root = shadow_host.shadow_root

        # Step 3: Find elements inside shadow root
        # IMPORTANT: Only CSS selectors work, not XPath
        title = shadow_root.find_element(By.CSS_SELECTOR, "#shadow-title")
        print(f"Shadow title text: {title.text}")
        assert title.text == "Shadow DOM Title"

        # Type into shadow DOM input
        input_el = shadow_root.find_element(
            By.CSS_SELECTOR, "#shadow-input"
        )
        input_el.send_keys("Hello Shadow DOM!")
        print("Typed into shadow DOM input.")

        # Click shadow DOM button
        button = shadow_root.find_element(By.CSS_SELECTOR, "#shadow-btn")
        button.click()
        print("Clicked shadow DOM button.")


class TestNestedShadowDom:

    def test_access_nested_shadow_dom(self, driver):
        """Chain through multiple shadow DOM levels."""
        driver.get("https://the-internet.herokuapp.com/")

        # Create nested shadow DOM
        driver.execute_script("""
            var outer = document.createElement('div');
            outer.id = 'outer-host';
            document.body.prepend(outer);

            var outerShadow = outer.attachShadow({mode: 'open'});
            outerShadow.innerHTML = `
                <p id="outer-text">Outer Shadow Content</p>
                <div id="inner-host"></div>
            `;

            var innerHost = outerShadow.querySelector('#inner-host');
            var innerShadow = innerHost.attachShadow({mode: 'open'});
            innerShadow.innerHTML = `
                <p id="inner-text">Inner Shadow Content</p>
                <button id="inner-btn">Deep Button</button>
            `;
        """)

        # Step 1: Get outer shadow root
        outer_host = driver.find_element(By.ID, "outer-host")
        outer_shadow = outer_host.shadow_root

        # Step 2: Find inner host inside outer shadow
        inner_host = outer_shadow.find_element(
            By.CSS_SELECTOR, "#inner-host"
        )

        # Step 3: Get inner shadow root (chaining)
        inner_shadow = inner_host.shadow_root

        # Step 4: Find elements in innermost shadow
        inner_text = inner_shadow.find_element(
            By.CSS_SELECTOR, "#inner-text"
        )
        print(f"Inner shadow text: {inner_text.text}")
        assert inner_text.text == "Inner Shadow Content"

        deep_button = inner_shadow.find_element(
            By.CSS_SELECTOR, "#inner-btn"
        )
        deep_button.click()
        print("Clicked button inside nested shadow DOM.")


class TestJSFallback:

    def test_shadow_dom_via_javascript(self, driver):
        """Use JavaScriptExecutor as fallback for shadow DOM access."""
        driver.get("https://the-internet.herokuapp.com/")

        driver.execute_script("""
            var host = document.createElement('div');
            host.id = 'js-shadow-host';
            document.body.prepend(host);
            var sr = host.attachShadow({mode: 'open'});
            sr.innerHTML = '<p id="js-para">JS Shadow Text</p>';
        """)

        # JS approach — chain .shadowRoot.querySelector() in one call
        paragraph = driver.execute_script(
            "return document.querySelector('#js-shadow-host')"
            ".shadowRoot.querySelector('#js-para');"
        )

        print(f"Text via JS: {paragraph.text}")
        assert paragraph.text == "JS Shadow Text"

    def test_deeply_nested_via_javascript(self, driver):
        """JS chaining through 3 levels of shadow DOM."""
        driver.get("https://the-internet.herokuapp.com/")

        driver.execute_script("""
            var level1 = document.createElement('div');
            level1.id = 'level1';
            document.body.prepend(level1);

            var sr1 = level1.attachShadow({mode: 'open'});
            sr1.innerHTML = '<div id="level2"></div>';

            var level2 = sr1.querySelector('#level2');
            var sr2 = level2.attachShadow({mode: 'open'});
            sr2.innerHTML = '<div id="level3"></div>';

            var level3 = sr2.querySelector('#level3');
            var sr3 = level3.attachShadow({mode: 'open'});
            sr3.innerHTML = '<span id="deep-text">3 levels deep!</span>';
        """)

        deep_element = driver.execute_script("""
            return document.querySelector('#level1')
                .shadowRoot.querySelector('#level2')
                .shadowRoot.querySelector('#level3')
                .shadowRoot.querySelector('#deep-text');
        """)

        print(f"Deep element text: {deep_element.text}")
        assert deep_element.text == "3 levels deep!"


class TestShadowDomCollections:

    def test_find_multiple_elements_in_shadow(self, driver):
        """Use find_elements to get a list from shadow root."""
        driver.get("https://the-internet.herokuapp.com/")

        driver.execute_script("""
            var host = document.createElement('div');
            host.id = 'list-host';
            document.body.prepend(host);
            var sr = host.attachShadow({mode: 'open'});
            sr.innerHTML = `
                <ul>
                    <li class="item">Item 1</li>
                    <li class="item">Item 2</li>
                    <li class="item">Item 3</li>
                </ul>
            `;
        """)

        shadow_root = driver.find_element(By.ID, "list-host").shadow_root

        # find_elements (plural) works inside shadow root
        items = shadow_root.find_elements(By.CSS_SELECTOR, ".item")

        print(f"Found {len(items)} items in shadow DOM:")
        for item in items:
            print(f"  - {item.text}")

        assert len(items) == 3


class TestChromeSettings:

    def test_chrome_settings_shadow_dom(self, driver, wait):
        """Real-world: Chrome's settings page uses deep shadow DOM."""
        driver.get("chrome://settings/")

        # Wait for settings-ui element
        settings_ui = wait.until(
            EC.presence_of_element_located((By.TAG_NAME, "settings-ui"))
        )

        # Access shadow root
        shadow_root = settings_ui.shadow_root

        # Find main content
        settings_main = shadow_root.find_element(
            By.CSS_SELECTOR, "settings-main"
        )

        print("Successfully accessed Chrome settings shadow DOM!")
        assert settings_main is not None


class TestShadowDomHelper:

    def test_reusable_shadow_helper(self, driver):
        """Demonstrate a reusable helper function for shadow DOM access."""
        driver.get("https://the-internet.herokuapp.com/")

        driver.execute_script("""
            var host = document.createElement('div');
            host.id = 'helper-host';
            document.body.prepend(host);
            var sr = host.attachShadow({mode: 'open'});
            sr.innerHTML = `
                <div class="container">
                    <span class="label">Name:</span>
                    <input class="field" type="text" value="John" />
                </div>
            `;
        """)

        def get_shadow_element(drv, host_selector, inner_selector):
            """
            Helper: find an element inside a shadow root.
            Args:
                drv: WebDriver instance
                host_selector: CSS selector for the shadow host
                inner_selector: CSS selector for the element inside shadow
            Returns:
                WebElement inside the shadow root
            """
            host = drv.find_element(By.CSS_SELECTOR, host_selector)
            shadow = host.shadow_root
            return shadow.find_element(By.CSS_SELECTOR, inner_selector)

        # Use the helper
        label = get_shadow_element(driver, "#helper-host", ".label")
        field = get_shadow_element(driver, "#helper-host", ".field")

        print(f"Label: {label.text}")
        print(f"Field value: {field.get_attribute('value')}")

        assert label.text == "Name:"
        assert field.get_attribute("value") == "John"
```

---

## 5. Step-by-Step Walkthrough

### Accessing a Single-Level Shadow DOM

1. **Inspect the page** in DevTools. Shadow DOM elements appear under `#shadow-root (open)` in the Elements panel.
2. **Identify the shadow host** — the custom HTML element tag (e.g., `<my-component>`, `<settings-ui>`).
3. **Find the shadow host** with a standard Selenium locator: `driver.FindElement(By.TagName("my-component"))`.
4. **Cross the boundary** by calling `.GetShadowRoot()` (C#) or `.shadow_root` (Python).
5. **Search inside** the shadow root using CSS selectors only.

### Accessing Nested Shadow DOMs

1. Find the outermost shadow host.
2. Get its shadow root.
3. Inside that shadow root, find the next shadow host.
4. Get that host's shadow root.
5. Repeat until you reach the element you need.

### When to Use JS Fallback

- The shadow DOM is **closed** (mode: 'closed') — Selenium's native API throws an error.
- You need to traverse 3+ levels and want a single line of code.
- You need to read `shadowRoot` properties that Selenium does not expose.

---

## 6. Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using XPath inside shadow root | XPath does not cross shadow boundaries | Use CSS selectors only |
| Searching from `driver` instead of shadow root | Element not found — it is inside the shadow | Always search from the shadow root object |
| Not waiting for shadow DOM to render | Shadow root is null because component has not loaded yet | Use explicit wait for shadow host presence |
| Confusing shadow host with shadow root | Trying to find children of the host directly | Call `.GetShadowRoot()` / `.shadow_root` first |
| Assuming all custom elements have shadow DOM | Not every `<my-component>` uses shadow DOM | Inspect DevTools to verify `#shadow-root` exists |

---

## 7. Best Practices

1. **Always verify shadow DOM exists in DevTools first.** Open the Elements panel and look for `#shadow-root (open)` under the element.

2. **Create helper methods** to reduce repetition:
   ```csharp
   public ISearchContext GetShadowRoot(string hostSelector)
   {
       var host = driver.FindElement(By.CssSelector(hostSelector));
       return host.GetShadowRoot();
   }
   ```

3. **Prefer Selenium 4 native API** over JavaScriptExecutor. The native API is more readable and maintainable.

4. **Wait for shadow host to be present** before accessing its shadow root. Web components may take time to initialize.

5. **Document shadow DOM paths** in your Page Object classes. Shadow DOM paths are fragile — when the component changes, the path changes.

6. **Use CSS selectors with IDs or stable classes.** Avoid complex selectors that depend on DOM structure inside the shadow tree.

---

## 8. Hands-On Exercise

### Task: Interact with Shadow DOM Elements

**Objective:** Create a test that navigates to Chrome's settings page and reads content from inside its nested shadow DOM structure.

**Steps:**
1. Navigate to `chrome://settings/`
2. Access the `settings-ui` shadow root
3. Inside that shadow root, find `settings-main`
4. Access `settings-main`'s shadow root
5. Find the settings page body content
6. Assert that the page loaded successfully

**Expected Output:**
```
Successfully navigated to chrome://settings/
Found settings-ui shadow host
Accessed settings-ui shadow root
Found settings-main inside shadow root
Chrome settings shadow DOM traversal complete!
```

---

## 9. Real-World Scenario

### Scenario: Testing a Salesforce Lightning Web Component

Salesforce Lightning uses shadow DOM extensively. A common test might need to interact with a custom dropdown:

```python
def select_lightning_option(driver, wait, dropdown_label, option_text):
    """
    Select an option from a Salesforce Lightning dropdown.
    Lightning components use nested shadow DOM.
    """
    # Step 1: Find the lightning-combobox by its label
    comboboxes = driver.find_elements(
        By.TAG_NAME, "lightning-combobox"
    )

    target = None
    for combobox in comboboxes:
        shadow = combobox.shadow_root
        label = shadow.find_element(By.CSS_SELECTOR, "label")
        if label.text == dropdown_label:
            target = shadow
            break

    assert target is not None, f"Dropdown '{dropdown_label}' not found"

    # Step 2: Click to open the dropdown
    input_el = target.find_element(By.CSS_SELECTOR, "input")
    input_el.click()

    # Step 3: Wait for options to appear
    # Options may be inside another shadow DOM level
    listbox = wait.until(lambda d: target.find_element(
        By.CSS_SELECTOR, "lightning-base-combobox-item"
    ))

    # Step 4: Find and click the desired option
    options = target.find_elements(
        By.CSS_SELECTOR, "lightning-base-combobox-item"
    )
    for option in options:
        option_shadow = option.shadow_root
        span = option_shadow.find_element(
            By.CSS_SELECTOR, ".slds-truncate"
        )
        if span.text == option_text:
            option.click()
            break

    print(f"Selected '{option_text}' from '{dropdown_label}' dropdown")
```

---

## 10. Resources

- [Selenium 4 Shadow DOM Documentation](https://www.selenium.dev/documentation/webdriver/elements/shadow_dom/)
- [MDN: Using Shadow DOM](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM)
- [MDN: Element.shadowRoot](https://developer.mozilla.org/en-US/docs/Web/API/Element/shadowRoot)
- [Web Components Specification](https://www.w3.org/TR/shadow-dom/)
- [Chrome DevTools - Inspect Shadow DOM](https://developer.chrome.com/docs/devtools/dom/#shadow-dom)
