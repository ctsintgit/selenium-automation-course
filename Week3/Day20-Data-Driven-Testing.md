# Day 20: Data-Driven Testing — Parameterized Tests, CSV/JSON Input

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain what data-driven testing is and why it reduces test duplication
- Use NUnit's `[TestCase]` and `[TestCaseSource]` for parameterized tests in C#
- Use pytest's `@pytest.mark.parametrize` for parameterized tests in Python
- Read test data from CSV and JSON files
- Separate test data from test logic cleanly
- Build a parameterized login test with multiple credential combinations

---

## 1. What Is Data-Driven Testing?

Data-driven testing is a technique where the same test logic runs multiple times with different input data. Instead of writing five separate test methods that do the same thing with different values, you write one test and feed it five sets of data.

### Without Data-Driven Testing (duplicated)

```python
def test_login_standard_user():
    login("standard_user", "secret_sauce")
    assert "inventory" in driver.current_url

def test_login_problem_user():
    login("problem_user", "secret_sauce")
    assert "inventory" in driver.current_url

def test_login_performance_user():
    login("performance_glitch_user", "secret_sauce")
    assert "inventory" in driver.current_url
```

### With Data-Driven Testing (clean)

```python
@pytest.mark.parametrize("username", [
    "standard_user",
    "problem_user",
    "performance_glitch_user"
])
def test_login_valid_users(username):
    login(username, "secret_sauce")
    assert "inventory" in driver.current_url
```

One test method, three executions, three results. If the login flow changes, you update one method instead of three.

---

## 2. How It Works

### C# (NUnit)

- **`[TestCase]`** — Inline data directly in the attribute. Best for small, simple data sets.
- **`[TestCaseSource]`** — Points to a method, property, or class that provides data. Best for complex or external data.

### Python (pytest)

- **`@pytest.mark.parametrize`** — Decorator that takes parameter names and a list of values. Supports tuples for multiple parameters.

Both frameworks generate a separate test result for each data set, so you can see exactly which combination passed or failed.

---

## 3. Code Example: C# — Inline Parameters with [TestCase]

```csharp
// File: Tests/LoginDataDrivenTests.cs
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;

namespace SeleniumTests.Tests
{
    [TestFixture]
    public class LoginDataDrivenTests
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
            driver.Navigate().GoToUrl("https://www.saucedemo.com");
        }

        // --- INLINE DATA WITH [TestCase] ---
        // Each [TestCase] generates a separate test run.
        // Parameters map to the method arguments in order.

        [TestCase("standard_user", "secret_sauce", true)]
        [TestCase("problem_user", "secret_sauce", true)]
        [TestCase("performance_glitch_user", "secret_sauce", true)]
        [TestCase("locked_out_user", "secret_sauce", false)]
        [TestCase("invalid_user", "wrong_password", false)]
        [TestCase("", "", false)]
        public void Login_ShouldMatchExpectedOutcome(
            string username, string password, bool shouldSucceed)
        {
            // Arrange: enter credentials
            driver.FindElement(By.Id("user-name")).SendKeys(username);
            driver.FindElement(By.Id("password")).SendKeys(password);

            // Act: click login
            driver.FindElement(By.Id("login-button")).Click();

            if (shouldSucceed)
            {
                // Assert: should reach inventory page
                wait.Until(d => d.Url.Contains("inventory"));
                Assert.That(driver.Url, Does.Contain("inventory"),
                    $"Expected successful login for '{username}'");
            }
            else
            {
                // Assert: should show error message
                var error = wait.Until(d =>
                    d.FindElement(By.CssSelector("[data-test='error']")));
                Assert.That(error.Displayed, Is.True,
                    $"Expected error message for '{username}'");
            }
        }

        [TearDown]
        public void TearDown()
        {
            driver?.Quit();
        }
    }
}
```

---

## 4. Code Example: C# — External Data with [TestCaseSource]

### Test Data File

```json
// File: TestData/login_data.json
[
    {
        "username": "standard_user",
        "password": "secret_sauce",
        "shouldSucceed": true,
        "description": "Valid standard user"
    },
    {
        "username": "locked_out_user",
        "password": "secret_sauce",
        "shouldSucceed": false,
        "description": "Locked out user"
    },
    {
        "username": "invalid_user",
        "password": "wrong_pass",
        "shouldSucceed": false,
        "description": "Invalid credentials"
    },
    {
        "username": "",
        "password": "",
        "shouldSucceed": false,
        "description": "Empty credentials"
    }
]
```

### Test Data Provider and Test

```csharp
// File: Tests/LoginJsonDrivenTests.cs
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using System;
using System.Collections.Generic;
using System.IO;
using System.Text.Json;

namespace SeleniumTests.Tests
{
    [TestFixture]
    public class LoginJsonDrivenTests
    {
        private IWebDriver driver;
        private WebDriverWait wait;

        [SetUp]
        public void SetUp()
        {
            var options = new ChromeOptions();
            driver = new ChromeDriver(options);
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));
            driver.Navigate().GoToUrl("https://www.saucedemo.com");
        }

        /// <summary>
        /// Static method that reads test data from a JSON file.
        /// NUnit calls this to get the parameter sets.
        /// </summary>
        private static IEnumerable<TestCaseData> LoginTestData()
        {
            string jsonPath = Path.Combine(
                TestContext.CurrentContext.TestDirectory, "TestData", "login_data.json");
            string json = File.ReadAllText(jsonPath);

            // Deserialize JSON array into a list of dictionaries
            var dataList = JsonSerializer.Deserialize<List<Dictionary<string, JsonElement>>>(json);

            foreach (var data in dataList)
            {
                string username = data["username"].GetString();
                string password = data["password"].GetString();
                bool shouldSucceed = data["shouldSucceed"].GetBoolean();
                string description = data["description"].GetString();

                // TestCaseData allows custom test names
                yield return new TestCaseData(username, password, shouldSucceed)
                    .SetName($"Login_{description.Replace(" ", "_")}");
            }
        }

        [Test]
        [TestCaseSource(nameof(LoginTestData))]
        public void Login_WithJsonData(string username, string password, bool shouldSucceed)
        {
            // Enter credentials
            var userField = driver.FindElement(By.Id("user-name"));
            var passField = driver.FindElement(By.Id("password"));

            userField.SendKeys(username);
            passField.SendKeys(password);
            driver.FindElement(By.Id("login-button")).Click();

            if (shouldSucceed)
            {
                wait.Until(d => d.Url.Contains("inventory"));
                Assert.That(driver.Url, Does.Contain("inventory"));
            }
            else
            {
                var error = wait.Until(d =>
                    d.FindElement(By.CssSelector("[data-test='error']")));
                Assert.That(error.Displayed, Is.True);
            }
        }

        [TearDown]
        public void TearDown()
        {
            driver?.Quit();
        }
    }
}
```

### Reading from CSV (C#)

```csharp
// File: Tests/LoginCsvDrivenTests.cs (data provider method only)
private static IEnumerable<TestCaseData> LoginCsvData()
{
    string csvPath = Path.Combine(
        TestContext.CurrentContext.TestDirectory, "TestData", "login_data.csv");

    // Skip the header row
    var lines = File.ReadAllLines(csvPath).Skip(1);

    foreach (var line in lines)
    {
        var fields = line.Split(',');
        string username = fields[0].Trim();
        string password = fields[1].Trim();
        bool shouldSucceed = bool.Parse(fields[2].Trim());
        string description = fields[3].Trim();

        yield return new TestCaseData(username, password, shouldSucceed)
            .SetName($"CsvLogin_{description.Replace(" ", "_")}");
    }
}
```

```csv
// File: TestData/login_data.csv
username,password,shouldSucceed,description
standard_user,secret_sauce,true,Valid standard user
locked_out_user,secret_sauce,false,Locked out user
invalid_user,wrong_pass,false,Invalid credentials
,,false,Empty credentials
```

---

## 5. Code Example: Python — @pytest.mark.parametrize

### Inline Parameters

```python
# File: tests/test_login_parametrize.py
import pytest
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


@pytest.fixture
def driver():
    options = webdriver.ChromeOptions()
    options.add_argument("--start-maximized")
    browser = webdriver.Chrome(options=options)
    browser.get("https://www.saucedemo.com")
    yield browser
    browser.quit()


class TestLoginParametrized:
    """Data-driven login tests using inline parameterization."""

    # Each tuple is (username, password, should_succeed)
    # pytest creates a separate test for each tuple

    @pytest.mark.parametrize("username, password, should_succeed", [
        ("standard_user", "secret_sauce", True),
        ("problem_user", "secret_sauce", True),
        ("performance_glitch_user", "secret_sauce", True),
        ("locked_out_user", "secret_sauce", False),
        ("invalid_user", "wrong_password", False),
        ("", "", False),
    ])
    def test_login(self, driver, username, password, should_succeed):
        """Tests login with various credential combinations."""
        # Enter credentials
        driver.find_element(By.ID, "user-name").send_keys(username)
        driver.find_element(By.ID, "password").send_keys(password)
        driver.find_element(By.ID, "login-button").click()

        wait = WebDriverWait(driver, 10)

        if should_succeed:
            wait.until(EC.url_contains("inventory"))
            assert "inventory" in driver.current_url, \
                f"Expected successful login for '{username}'"
        else:
            error = wait.until(
                EC.presence_of_element_located(
                    (By.CSS_SELECTOR, "[data-test='error']")
                )
            )
            assert error.is_displayed(), \
                f"Expected error message for '{username}'"
```

### Using IDs for Readable Test Names

```python
@pytest.mark.parametrize("username, password, should_succeed", [
    ("standard_user", "secret_sauce", True),
    ("locked_out_user", "secret_sauce", False),
    ("invalid_user", "wrong_pass", False),
], ids=[
    "valid_standard_user",
    "locked_out_user",
    "invalid_credentials",
])
def test_login_with_ids(self, driver, username, password, should_succeed):
    # ... same test logic
    pass
```

---

## 6. Code Example: Python — Reading from JSON and CSV Files

### JSON Data File

```json
// File: test_data/login_data.json
[
    {
        "username": "standard_user",
        "password": "secret_sauce",
        "should_succeed": true,
        "description": "Valid standard user"
    },
    {
        "username": "locked_out_user",
        "password": "secret_sauce",
        "should_succeed": false,
        "description": "Locked out user"
    },
    {
        "username": "invalid_user",
        "password": "wrong_pass",
        "should_succeed": false,
        "description": "Invalid credentials"
    },
    {
        "username": "",
        "password": "",
        "should_succeed": false,
        "description": "Empty credentials"
    }
]
```

### CSV Data File

```csv
# File: test_data/login_data.csv
username,password,should_succeed,description
standard_user,secret_sauce,True,Valid standard user
locked_out_user,secret_sauce,False,Locked out user
invalid_user,wrong_pass,False,Invalid credentials
,,False,Empty credentials
```

### Data Loader Utility

```python
# File: utils/data_loader.py
import json
import csv
from pathlib import Path


def load_json_test_data(file_name: str) -> list:
    """
    Loads test data from a JSON file in the test_data directory.
    Returns a list of dictionaries.
    """
    data_dir = Path(__file__).parent.parent / "test_data"
    file_path = data_dir / file_name

    with open(file_path, "r") as f:
        return json.load(f)


def load_csv_test_data(file_name: str) -> list:
    """
    Loads test data from a CSV file.
    Returns a list of dictionaries (one per row).
    """
    data_dir = Path(__file__).parent.parent / "test_data"
    file_path = data_dir / file_name

    data = []
    with open(file_path, "r") as f:
        reader = csv.DictReader(f)
        for row in reader:
            # Convert string 'True'/'False' to actual booleans
            if "should_succeed" in row:
                row["should_succeed"] = row["should_succeed"].strip().lower() == "true"
            data.append(row)
    return data


def json_to_params(file_name: str, fields: list) -> list:
    """
    Converts JSON test data to a list of tuples suitable for
    @pytest.mark.parametrize.

    Example:
        json_to_params("login_data.json", ["username", "password", "should_succeed"])
        Returns: [("standard_user", "secret_sauce", True), ...]
    """
    data = load_json_test_data(file_name)
    return [tuple(item[field] for field in fields) for item in data]


def json_to_params_with_ids(file_name: str, fields: list, id_field: str) -> tuple:
    """
    Returns (params_list, ids_list) for parametrize with readable test names.
    """
    data = load_json_test_data(file_name)
    params = [tuple(item[field] for field in fields) for item in data]
    ids = [item[id_field] for item in data]
    return params, ids
```

### Tests Using External Data

```python
# File: tests/test_login_data_driven.py
import pytest
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from utils.data_loader import json_to_params, json_to_params_with_ids, load_csv_test_data


@pytest.fixture
def driver():
    options = webdriver.ChromeOptions()
    browser = webdriver.Chrome(options=options)
    browser.get("https://www.saucedemo.com")
    yield browser
    browser.quit()


# --- Load data ONCE at module level (not inside each test) ---
LOGIN_PARAMS, LOGIN_IDS = json_to_params_with_ids(
    "login_data.json",
    ["username", "password", "should_succeed"],
    "description"
)

CSV_DATA = load_csv_test_data("login_data.csv")
CSV_PARAMS = [(row["username"], row["password"], row["should_succeed"]) for row in CSV_DATA]
CSV_IDS = [row["description"] for row in CSV_DATA]


class TestLoginFromJson:
    """Data-driven tests reading from JSON file."""

    @pytest.mark.parametrize(
        "username, password, should_succeed",
        LOGIN_PARAMS,
        ids=LOGIN_IDS
    )
    def test_login(self, driver, username, password, should_succeed):
        driver.find_element(By.ID, "user-name").send_keys(username)
        driver.find_element(By.ID, "password").send_keys(password)
        driver.find_element(By.ID, "login-button").click()

        wait = WebDriverWait(driver, 10)

        if should_succeed:
            wait.until(EC.url_contains("inventory"))
            assert "inventory" in driver.current_url
        else:
            error = wait.until(
                EC.presence_of_element_located(
                    (By.CSS_SELECTOR, "[data-test='error']")
                )
            )
            assert error.is_displayed()


class TestLoginFromCsv:
    """Data-driven tests reading from CSV file."""

    @pytest.mark.parametrize(
        "username, password, should_succeed",
        CSV_PARAMS,
        ids=CSV_IDS
    )
    def test_login(self, driver, username, password, should_succeed):
        driver.find_element(By.ID, "user-name").send_keys(username)
        driver.find_element(By.ID, "password").send_keys(password)
        driver.find_element(By.ID, "login-button").click()

        wait = WebDriverWait(driver, 10)

        if should_succeed:
            wait.until(EC.url_contains("inventory"))
            assert "inventory" in driver.current_url
        else:
            error = wait.until(
                EC.presence_of_element_located(
                    (By.CSS_SELECTOR, "[data-test='error']")
                )
            )
            assert error.is_displayed()
```

---

## 7. Step-by-Step Walkthrough

### How NUnit [TestCaseSource] Works

1. NUnit discovers the `[TestCaseSource(nameof(LoginTestData))]` attribute
2. It calls the `LoginTestData()` method, which reads the JSON file
3. Each `TestCaseData` object becomes a separate test execution
4. The test method receives the parameters as arguments
5. NUnit reports each execution independently: `Login_Valid_standard_user PASSED`

### How pytest @pytest.mark.parametrize Works

1. pytest discovers the `@pytest.mark.parametrize` decorator
2. It reads the list of tuples
3. For each tuple, it creates a separate test instance
4. The test function receives the values as arguments
5. Results show each combination: `test_login[Valid standard user] PASSED`

---

## 8. Common Mistakes and How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Hardcoding data in test methods | Adding new data requires changing test code | Move data to external files (JSON, CSV) |
| Loading data inside the test method | File is read on every test invocation | Load at module level or in a fixture with `session` scope |
| Not providing test IDs | Output shows `test_login[0]`, `test_login[1]` — meaningless | Use `ids` parameter or `SetName()` |
| CSV with inconsistent formatting | Parsing breaks on commas in data | Use JSON for complex data or quote CSV fields |
| Forgetting to copy data files to output (C#) | `FileNotFoundException` at runtime | Set `CopyToOutputDirectory` in `.csproj` |
| Mixing test logic with data validation | Test does too many things | Keep the test focused — validate data separately |

---

## 9. Best Practices

1. **One test, many data sets** — If the test logic is the same, parameterize it
2. **Meaningful test IDs** — Use descriptions so failures are easy to identify
3. **External data for large sets** — Inline `[TestCase]` for up to 5 items, JSON/CSV for more
4. **Keep data files simple** — Flat structures. Avoid deeply nested JSON for test data.
5. **Separate valid and invalid data** — Consider having `valid_logins.json` and `invalid_logins.json` for clarity
6. **Version control your data files** — They are part of your test suite

---

## 10. Hands-On Exercise

**Task:** Build a fully data-driven test suite for the Sauce Demo login page.

1. Create `test_data/login_data.json` with at least 6 credential combinations (valid, locked, invalid, empty)
2. Create a data loader utility
3. Write a parameterized test that:
   - Reads credentials from JSON
   - Attempts login
   - Asserts success or failure based on the expected outcome
   - Uses descriptive test IDs

**Expected Output:**

```
tests/test_login_data_driven.py::TestLoginFromJson::test_login[Valid standard user] PASSED
tests/test_login_data_driven.py::TestLoginFromJson::test_login[Valid problem user] PASSED
tests/test_login_data_driven.py::TestLoginFromJson::test_login[Locked out user] PASSED
tests/test_login_data_driven.py::TestLoginFromJson::test_login[Invalid credentials] PASSED
tests/test_login_data_driven.py::TestLoginFromJson::test_login[Empty credentials] PASSED
tests/test_login_data_driven.py::TestLoginFromJson::test_login[Empty password only] PASSED
========================= 6 passed in 25.14s =========================
```

---

## 11. Real-World Scenario

An e-commerce site has a search feature. The QA team needs to verify that searching for various terms returns the correct number of results. Instead of writing 50 test methods, they create a CSV file:

```csv
search_term,expected_min_results,category
laptop,10,electronics
"running shoes",5,sports
"",0,none
"!@#$%",0,none
"a",50,all
```

One parameterized test runs all 50 scenarios. When the product catalog changes, they update the CSV file — no code changes. When a new search edge case is discovered, they add one row to the CSV.

This approach is especially powerful for form validation testing, where you need to verify dozens of input combinations (too long, too short, special characters, SQL injection strings, XSS payloads).

---

## 12. Resources

- [NUnit TestCase Attribute](https://docs.nunit.org/articles/nunit/writing-tests/attributes/testcase.html)
- [NUnit TestCaseSource](https://docs.nunit.org/articles/nunit/writing-tests/attributes/testcasesource.html)
- [pytest parametrize](https://docs.pytest.org/en/stable/how-to/parametrize.html)
- [Python csv module](https://docs.python.org/3/library/csv.html)
- [Python json module](https://docs.python.org/3/library/json.html)
