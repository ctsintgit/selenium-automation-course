# Day 26: Selenium Grid — Hub/Node Architecture, Setup

## Learning Objectives

By the end of this lesson, you will be able to:

- Understand what Selenium Grid is and why distributed testing matters
- Describe Grid 4's architecture: Router, Distributor, Session Map, Node
- Set up Grid in Standalone mode for quick local testing
- Set up Grid with Hub/Node mode for distributed execution
- Deploy Grid using Docker and docker-compose
- Connect your tests to a remote Grid using RemoteWebDriver
- Configure browser capabilities and options for remote execution
- Monitor Grid status via the Grid UI dashboard

---

## 1. Core Concept Explanation

### What Is Selenium Grid?

Selenium Grid lets you run tests on browsers hosted on **remote machines**. Instead of launching Chrome on your local computer, your test sends commands to a Grid server, which routes them to an available browser on any connected machine.

Think of it as a taxi dispatcher: your test requests "I need a Chrome browser," the Grid finds an available Chrome instance on one of its nodes, and routes all your commands there.

### Why Use Selenium Grid?

| Benefit | Explanation |
|---------|-------------|
| **Parallel execution** | Run 50 tests simultaneously on 50 browsers across 10 machines |
| **Cross-browser testing** | Test on Chrome, Firefox, and Edge without installing all on one machine |
| **Cross-OS testing** | Windows, Linux, macOS nodes — all from one test runner |
| **Scale** | Add more nodes when you need more capacity |
| **Resource isolation** | Tests run on dedicated machines, not your dev laptop |
| **CI/CD integration** | Grid runs as infrastructure; tests just point to a URL |

### Grid 4 Architecture

Selenium Grid 4 (released with Selenium 4) is a complete rewrite. It has several components:

```
                    ┌─────────────────────┐
  Your Test ──────> │      Router         │ (Entry point - receives requests)
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │    Distributor      │ (Finds available node/slot)
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │    Session Map      │ (Tracks active sessions)
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │  Node 1  │    │  Node 2  │    │  Node 3  │
        │ Chrome   │    │ Firefox  │    │ Chrome   │
        │ Firefox  │    │ Edge     │    │ Edge     │
        └──────────┘    └──────────┘    └──────────┘
```

| Component | Role |
|-----------|------|
| **Router** | Receives WebDriver requests and forwards them |
| **Distributor** | Matches a new session request to an available node slot |
| **Session Map** | Stores which node is handling which session |
| **Node** | Hosts actual browsers, executes commands |

### Deployment Modes

| Mode | What It Does | When to Use |
|------|-------------|-------------|
| **Standalone** | All components in one process | Local testing, quick experiments |
| **Hub + Node** | Hub (Router+Distributor+SessionMap) + separate Nodes | Small teams, simple distributed setup |
| **Fully Distributed** | Each component runs independently | Large-scale enterprise deployments |

---

## 2. How It Works (Technical Breakdown)

### Session Lifecycle

1. Your test creates a `RemoteWebDriver` pointing to the Grid URL
2. The **Router** receives the "new session" request
3. The **Distributor** checks all registered Nodes for a matching browser slot
4. A **Node** is selected, and it launches the browser
5. The **Session Map** records: Session ID -> Node address
6. All subsequent commands from your test go through the Router, which looks up the Session Map and forwards to the correct Node
7. When `driver.Quit()` is called, the session is removed from the map

### The Grid URL

Your tests connect to Grid via a single URL:
```
http://<grid-host>:<port>
```

Default ports:
- Standalone: `http://localhost:4444`
- Hub: `http://localhost:4444`
- Node: registers automatically with the Hub

---

## 3. Code Example: C# (Complete, Runnable)

```csharp
// File: Day26_SeleniumGrid.cs
// Prerequisites:
//   dotnet new nunit -n Day26
//   cd Day26
//   dotnet add package Selenium.WebDriver
//   dotnet add package Selenium.Support
//
// Grid Setup (run in terminal before tests):
//   Option A - Standalone:
//     java -jar selenium-server-4.x.x.jar standalone
//
//   Option B - Docker (recommended):
//     docker-compose up -d (see docker-compose.yml below)

using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Firefox;
using OpenQA.Selenium.Edge;
using OpenQA.Selenium.Remote;
using OpenQA.Selenium.Support.UI;
using System;

namespace Day26
{
    [TestFixture]
    public class GridBasicTests
    {
        // Grid URL — change to match your setup
        private const string GridUrl = "http://localhost:4444";

        private IWebDriver driver;
        private WebDriverWait wait;

        [TearDown]
        public void TearDown()
        {
            // Always quit to release the Grid slot
            driver?.Quit();
        }

        [Test]
        public void ConnectToGridWithChrome()
        {
            // Create Chrome options
            var options = new ChromeOptions();
            options.AddArgument("--window-size=1920,1080");

            // In CI/headless environments:
            // options.AddArgument("--headless=new");

            // Connect to Grid using RemoteWebDriver
            driver = new RemoteWebDriver(
                new Uri(GridUrl),  // Grid URL
                options            // Browser options
            );

            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));

            // From here, everything is identical to local execution
            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            Console.WriteLine($"Title: {driver.Title}");
            Console.WriteLine(
                $"Session ID: {((RemoteWebDriver)driver).SessionId}"
            );

            Assert.That(driver.Title, Does.Contain("Internet"));
        }

        [Test]
        public void ConnectToGridWithFirefox()
        {
            var options = new FirefoxOptions();

            driver = new RemoteWebDriver(
                new Uri(GridUrl),
                options
            );

            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            Console.WriteLine($"Firefox on Grid - Title: {driver.Title}");
            Assert.That(driver.Title, Does.Contain("Internet"));
        }

        [Test]
        public void ConnectToGridWithEdge()
        {
            var options = new EdgeOptions();

            driver = new RemoteWebDriver(
                new Uri(GridUrl),
                options
            );

            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            Console.WriteLine($"Edge on Grid - Title: {driver.Title}");
            Assert.That(driver.Title, Does.Contain("Internet"));
        }

        [Test]
        public void FullTestOnGrid()
        {
            // Complete login test running on Grid
            var options = new ChromeOptions();
            options.AddArgument("--window-size=1920,1080");

            driver = new RemoteWebDriver(new Uri(GridUrl), options);
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(10));

            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/login"
            );

            // Interact with elements — same API as local
            driver.FindElement(By.Id("username")).SendKeys("tomsmith");
            driver.FindElement(By.Id("password"))
                .SendKeys("SuperSecretPassword!");
            driver.FindElement(
                By.CssSelector("button[type='submit']")
            ).Click();

            // Wait for result
            IWebElement flash = wait.Until(
                d => d.FindElement(By.Id("flash"))
            );

            Console.WriteLine($"Result: {flash.Text}");
            Assert.That(flash.Text, Does.Contain("You logged into"));
        }
    }

    // =============================================
    // CONFIGURABLE GRID TESTS
    // =============================================
    [TestFixture]
    public class ConfigurableGridTests
    {
        private IWebDriver driver;

        [TearDown]
        public void TearDown()
        {
            driver?.Quit();
        }

        /// <summary>
        /// Factory method: creates either local or remote driver
        /// based on configuration.
        /// </summary>
        private IWebDriver CreateDriver(string browser = "chrome")
        {
            string gridUrl = Environment.GetEnvironmentVariable(
                "SELENIUM_GRID_URL"
            );
            bool useGrid = !string.IsNullOrEmpty(gridUrl);

            if (browser.ToLower() == "chrome")
            {
                var options = new ChromeOptions();
                options.AddArgument("--window-size=1920,1080");

                if (useGrid)
                {
                    options.AddArgument("--headless=new");
                    Console.WriteLine(
                        $"Connecting to Grid at: {gridUrl}"
                    );
                    return new RemoteWebDriver(
                        new Uri(gridUrl), options
                    );
                }
                else
                {
                    Console.WriteLine("Running locally (no Grid)");
                    return new ChromeDriver(options);
                }
            }
            else if (browser.ToLower() == "firefox")
            {
                var options = new FirefoxOptions();

                if (useGrid)
                {
                    options.AddArgument("--headless");
                    return new RemoteWebDriver(
                        new Uri(gridUrl), options
                    );
                }
                else
                {
                    return new FirefoxDriver(options);
                }
            }

            throw new ArgumentException($"Unknown browser: {browser}");
        }

        [Test]
        public void TestWithConfigurableDriver()
        {
            // Set SELENIUM_GRID_URL=http://localhost:4444
            // to run on Grid, or leave unset for local
            driver = CreateDriver("chrome");

            driver.Navigate().GoToUrl(
                "https://the-internet.herokuapp.com/"
            );

            Assert.That(driver.Title, Does.Contain("Internet"));
            Console.WriteLine(
                $"Driver type: {driver.GetType().Name}"
            );
        }
    }

    // =============================================
    // GRID WITH PLATFORM / BROWSER VERSION
    // =============================================
    [TestFixture]
    public class GridCapabilitiesTests
    {
        private IWebDriver driver;

        [TearDown]
        public void TearDown()
        {
            driver?.Quit();
        }

        [Test]
        public void RequestSpecificBrowserVersion()
        {
            var options = new ChromeOptions();
            // Request a specific browser version
            // Grid will match this to an available node
            options.BrowserVersion = "stable";

            // Request a specific platform
            options.PlatformName = "linux";

            options.AddArgument("--headless=new");
            options.AddArgument("--window-size=1920,1080");

            try
            {
                driver = new RemoteWebDriver(
                    new Uri("http://localhost:4444"),
                    options
                );

                driver.Navigate().GoToUrl(
                    "https://the-internet.herokuapp.com/"
                );

                // Get actual capabilities
                var caps = ((RemoteWebDriver)driver).Capabilities;
                Console.WriteLine(
                    $"Browser: {caps.GetCapability("browserName")}"
                );
                Console.WriteLine(
                    $"Version: {caps.GetCapability("browserVersion")}"
                );
                Console.WriteLine(
                    $"Platform: {caps.GetCapability("platformName")}"
                );
            }
            catch (WebDriverException ex)
            {
                Console.WriteLine(
                    $"Grid could not match request: {ex.Message}"
                );
                Assert.Inconclusive(
                    "Grid does not have matching node"
                );
            }
        }
    }
}
```

---

## 4. Code Example: Python (Complete, Runnable)

```python
# File: test_day26_selenium_grid.py
# Prerequisites:
#   pip install selenium pytest
#
# Grid Setup (run before tests):
#   Option A: java -jar selenium-server-4.x.x.jar standalone
#   Option B: docker-compose up -d (see docker-compose.yml below)

import os
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options as ChromeOptions
from selenium.webdriver.firefox.options import Options as FirefoxOptions
from selenium.webdriver.edge.options import Options as EdgeOptions
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


# Grid URL — change to match your setup
GRID_URL = os.environ.get("SELENIUM_GRID_URL", "http://localhost:4444")


# =============================================
# FIXTURES
# =============================================

@pytest.fixture
def chrome_on_grid():
    """Chrome browser running on Selenium Grid."""
    options = ChromeOptions()
    options.add_argument("--window-size=1920,1080")
    # options.add_argument("--headless=new")  # for CI

    driver = webdriver.Remote(
        command_executor=GRID_URL,
        options=options
    )
    yield driver
    driver.quit()


@pytest.fixture
def firefox_on_grid():
    """Firefox browser running on Selenium Grid."""
    options = FirefoxOptions()

    driver = webdriver.Remote(
        command_executor=GRID_URL,
        options=options
    )
    yield driver
    driver.quit()


@pytest.fixture
def wait(chrome_on_grid):
    """Explicit wait bound to chrome_on_grid fixture."""
    return WebDriverWait(chrome_on_grid, 10)


# =============================================
# BASIC GRID CONNECTION TESTS
# =============================================

class TestGridConnection:

    def test_connect_chrome_to_grid(self, chrome_on_grid):
        """Connect to Grid with Chrome and verify it works."""
        chrome_on_grid.get("https://the-internet.herokuapp.com/")

        title = chrome_on_grid.title
        print(f"Chrome on Grid - Title: {title}")
        print(f"Session ID: {chrome_on_grid.session_id}")

        assert "Internet" in title

    def test_connect_firefox_to_grid(self, firefox_on_grid):
        """Connect to Grid with Firefox."""
        firefox_on_grid.get("https://the-internet.herokuapp.com/")

        title = firefox_on_grid.title
        print(f"Firefox on Grid - Title: {title}")

        assert "Internet" in title

    def test_connect_edge_to_grid(self):
        """Connect to Grid with Edge."""
        options = EdgeOptions()

        driver = webdriver.Remote(
            command_executor=GRID_URL,
            options=options
        )

        try:
            driver.get("https://the-internet.herokuapp.com/")
            print(f"Edge on Grid - Title: {driver.title}")
            assert "Internet" in driver.title
        finally:
            driver.quit()


# =============================================
# FULL TEST ON GRID
# =============================================

class TestFullWorkflow:

    def test_login_on_grid(self, chrome_on_grid, wait):
        """Complete login test running on Selenium Grid."""
        chrome_on_grid.get("https://the-internet.herokuapp.com/login")

        # All interactions are identical to local execution
        chrome_on_grid.find_element(By.ID, "username").send_keys(
            "tomsmith"
        )
        chrome_on_grid.find_element(By.ID, "password").send_keys(
            "SuperSecretPassword!"
        )
        chrome_on_grid.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        ).click()

        flash = wait.until(
            EC.visibility_of_element_located((By.ID, "flash"))
        )

        print(f"Result: {flash.text}")
        assert "You logged into" in flash.text


# =============================================
# CONFIGURABLE LOCAL/GRID DRIVER
# =============================================

class TestConfigurableDriver:

    @staticmethod
    def create_driver(browser="chrome"):
        """
        Factory: creates local or remote driver based on env var.
        Set SELENIUM_GRID_URL to use Grid, or leave unset for local.
        """
        grid_url = os.environ.get("SELENIUM_GRID_URL")
        use_grid = bool(grid_url)

        if browser == "chrome":
            options = ChromeOptions()
            options.add_argument("--window-size=1920,1080")

            if use_grid:
                options.add_argument("--headless=new")
                print(f"Connecting to Grid: {grid_url}")
                return webdriver.Remote(
                    command_executor=grid_url,
                    options=options
                )
            else:
                print("Running locally")
                return webdriver.Chrome(options=options)

        elif browser == "firefox":
            options = FirefoxOptions()

            if use_grid:
                options.add_argument("--headless")
                return webdriver.Remote(
                    command_executor=grid_url,
                    options=options
                )
            else:
                return webdriver.Firefox(options=options)

        raise ValueError(f"Unknown browser: {browser}")

    def test_configurable_chrome(self):
        """Test that works both locally and on Grid."""
        driver = self.create_driver("chrome")
        try:
            driver.get("https://the-internet.herokuapp.com/")
            assert "Internet" in driver.title
            print(f"Driver type: {type(driver).__name__}")
        finally:
            driver.quit()


# =============================================
# GRID CAPABILITIES
# =============================================

class TestGridCapabilities:

    def test_request_specific_platform(self):
        """Request a specific platform from Grid."""
        options = ChromeOptions()
        options.add_argument("--headless=new")
        options.add_argument("--window-size=1920,1080")

        # Request specific platform
        options.set_capability("platformName", "linux")

        try:
            driver = webdriver.Remote(
                command_executor=GRID_URL,
                options=options
            )

            driver.get("https://the-internet.herokuapp.com/")

            # Read actual capabilities
            caps = driver.capabilities
            print(f"Browser: {caps.get('browserName')}")
            print(f"Version: {caps.get('browserVersion')}")
            print(f"Platform: {caps.get('platformName')}")

            assert "Internet" in driver.title
            driver.quit()

        except Exception as e:
            print(f"Grid could not match request: {e}")
            pytest.skip("Grid does not have matching node")

    def test_screenshots_on_grid(self):
        """Screenshots work on remote Grid nodes too."""
        options = ChromeOptions()
        options.add_argument("--window-size=1920,1080")

        driver = webdriver.Remote(
            command_executor=GRID_URL,
            options=options
        )

        try:
            driver.get("https://the-internet.herokuapp.com/login")

            screenshot_dir = os.path.join(
                os.path.dirname(__file__), "TestResults"
            )
            os.makedirs(screenshot_dir, exist_ok=True)

            filepath = os.path.join(
                screenshot_dir, "grid_screenshot.png"
            )
            driver.save_screenshot(filepath)

            assert os.path.exists(filepath)
            print(f"Grid screenshot saved: {filepath}")
        finally:
            driver.quit()


# =============================================
# GRID STATUS CHECK
# =============================================

class TestGridStatus:

    def test_check_grid_status(self):
        """
        Check Grid status before running tests.
        Grid UI is at http://localhost:4444/ui
        Grid status API at http://localhost:4444/status
        """
        import urllib.request
        import json

        try:
            url = f"{GRID_URL}/status"
            with urllib.request.urlopen(url, timeout=5) as response:
                data = json.loads(response.read().decode())

            ready = data["value"]["ready"]
            nodes = data["value"].get("nodes", [])

            print(f"Grid ready: {ready}")
            print(f"Nodes connected: {len(nodes)}")

            for node in nodes:
                slots = node.get("slots", [])
                print(
                    f"  Node {node.get('id', 'unknown')[:8]}... "
                    f"- {len(slots)} slot(s)"
                )
                for slot in slots:
                    stereotype = slot.get("stereotype", {})
                    browser = stereotype.get("browserName", "unknown")
                    session = slot.get("session")
                    status = "BUSY" if session else "FREE"
                    print(f"    {browser}: {status}")

            assert ready, "Grid should be ready"

        except Exception as e:
            pytest.skip(f"Grid not available: {e}")
```

---

## 5. Docker Compose Setup

Save this as `docker-compose.yml` in your project root:

```yaml
# docker-compose.yml
# Start: docker-compose up -d
# Stop:  docker-compose down
# UI:    http://localhost:4444/ui

version: "3.8"

services:
  # Selenium Hub (Router + Distributor + Session Map)
  selenium-hub:
    image: selenium/hub:4.18.0
    container_name: selenium-hub
    ports:
      - "4442:4442"   # Event bus publish
      - "4443:4443"   # Event bus subscribe
      - "4444:4444"   # Grid URL (your tests connect here)
    environment:
      - SE_NODE_MAX_SESSIONS=4
      - SE_SESSION_REQUEST_TIMEOUT=300

  # Chrome Node
  chrome-node:
    image: selenium/node-chrome:4.18.0
    container_name: chrome-node
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=4
      - SE_NODE_OVERRIDE_MAX_SESSIONS=true
    shm_size: "2gb"   # Prevent Chrome crashes

  # Firefox Node
  firefox-node:
    image: selenium/node-firefox:4.18.0
    container_name: firefox-node
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=4
      - SE_NODE_OVERRIDE_MAX_SESSIONS=true
    shm_size: "2gb"

  # Edge Node
  edge-node:
    image: selenium/node-edge:4.18.0
    container_name: edge-node
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=4
      - SE_NODE_OVERRIDE_MAX_SESSIONS=true
    shm_size: "2gb"
```

### Standalone mode (simpler, single container):

```yaml
# docker-compose-standalone.yml
version: "3.8"

services:
  selenium-standalone:
    image: selenium/standalone-chrome:4.18.0
    container_name: selenium-standalone
    ports:
      - "4444:4444"    # Grid URL
      - "7900:7900"    # VNC viewer (password: secret)
    shm_size: "2gb"
    environment:
      - SE_NODE_MAX_SESSIONS=4
      - SE_NODE_OVERRIDE_MAX_SESSIONS=true
```

---

## 6. Step-by-Step Walkthrough

### Setting Up Grid with Docker

1. **Install Docker Desktop** on your machine
2. **Save `docker-compose.yml`** in your project folder
3. **Start Grid:** `docker-compose up -d`
4. **Verify:** Open `http://localhost:4444/ui` in your browser — you should see the Grid dashboard
5. **Run tests:** Point your tests to `http://localhost:4444`
6. **Stop Grid:** `docker-compose down`

### Connecting Tests to Grid

1. Replace `new ChromeDriver(options)` with `new RemoteWebDriver(new Uri("http://localhost:4444"), options)`
2. Replace `webdriver.Chrome(options=options)` with `webdriver.Remote(command_executor="http://localhost:4444", options=options)`
3. Everything else stays the same — same locators, same waits, same assertions

---

## 7. Common Mistakes & How to Avoid Them

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `driver.Quit()` | Grid slot stays occupied forever | Always quit in teardown/finally |
| Not setting `shm_size` in Docker | Chrome crashes with "session deleted" | Add `shm_size: "2gb"` to docker-compose |
| Hardcoding Grid URL | Tests break when Grid moves | Use environment variable |
| Starting too many sessions | Grid rejects requests, tests timeout | Match `SE_NODE_MAX_SESSIONS` to your parallel count |
| Not waiting for Grid to start | Tests fail because Grid is not ready yet | Check `/status` endpoint before running |
| Using file paths on Grid | Files are on the node, not your machine | Use Grid's file transfer or remote URLs |

---

## 8. Best Practices

1. **Always use Docker for Grid setup.** Manual Java JAR setup is fragile and hard to reproduce.

2. **Use `docker-compose down` when done.** Leftover containers consume resources.

3. **Set reasonable session limits.** Each browser session uses ~300-500MB RAM.

4. **Monitor Grid health.** The `/ui` dashboard shows active sessions, node status, and queue depth.

5. **Use the factory pattern** for local/remote driver creation. One code change should switch between local and Grid.

6. **Set session timeouts.** Grid has built-in timeouts to clean up abandoned sessions.

7. **Pin Docker image versions.** Use `selenium/node-chrome:4.18.0`, not `selenium/node-chrome:latest`.

---

## 8. Hands-On Exercise

### Task: Set Up Grid with Docker and Run Tests Remotely

**Objective:** Deploy a Selenium Grid with Chrome and Firefox nodes, then run the same test on both browsers.

**Steps:**
1. Save the `docker-compose.yml` file
2. Run `docker-compose up -d`
3. Open `http://localhost:4444/ui` and verify nodes are registered
4. Write a test that navigates to `https://the-internet.herokuapp.com/login`
5. Run the test against Chrome node
6. Run the same test against Firefox node
7. Check the Grid UI to see session activity
8. Run `docker-compose down`

**Expected Output:**
```
Chrome on Grid - Title: The Internet
Session ID: abc123...
Firefox on Grid - Title: The Internet
Session ID: def456...
Both browsers passed!
```

---

## 9. Real-World Scenario

### Scenario: Cross-Browser Test Suite on Grid

```python
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options as ChromeOptions
from selenium.webdriver.firefox.options import Options as FirefoxOptions


GRID_URL = "http://selenium-grid.internal:4444"


@pytest.fixture(params=["chrome", "firefox"])
def browser(request):
    """Parametrized fixture — runs each test on both browsers."""
    browser_name = request.param

    if browser_name == "chrome":
        options = ChromeOptions()
        options.add_argument("--headless=new")
        options.add_argument("--window-size=1920,1080")
    elif browser_name == "firefox":
        options = FirefoxOptions()
        options.add_argument("--headless")
        options.add_argument("--width=1920")
        options.add_argument("--height=1080")
    else:
        raise ValueError(f"Unknown browser: {browser_name}")

    driver = webdriver.Remote(
        command_executor=GRID_URL,
        options=options
    )
    driver._browser_name = browser_name
    yield driver
    driver.quit()


class TestCrossBrowser:
    """Every test in this class runs on both Chrome and Firefox."""

    def test_homepage_loads(self, browser):
        browser.get("https://the-internet.herokuapp.com/")
        assert "Internet" in browser.title
        print(f"[{browser._browser_name}] Homepage loaded")

    def test_login_works(self, browser):
        browser.get("https://the-internet.herokuapp.com/login")
        browser.find_element(By.ID, "username").send_keys("tomsmith")
        browser.find_element(By.ID, "password").send_keys(
            "SuperSecretPassword!"
        )
        browser.find_element(
            By.CSS_SELECTOR, "button[type='submit']"
        ).click()
        # ... assertions ...
        print(f"[{browser._browser_name}] Login test passed")
```

---

## 10. Resources

- [Selenium Grid Documentation](https://www.selenium.dev/documentation/grid/)
- [Docker Selenium Images](https://github.com/SeleniumHQ/docker-selenium)
- [Grid 4 Architecture](https://www.selenium.dev/documentation/grid/architecture/)
- [Grid Configuration](https://www.selenium.dev/documentation/grid/configuration/)
- [RemoteWebDriver Documentation](https://www.selenium.dev/documentation/webdriver/remote_webdriver/)
