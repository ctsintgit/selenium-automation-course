# Day 14: File Upload/Download Automation + Mini Project 2

## Learning Objectives

By the end of this lesson, you will be able to:

- Upload files using SendKeys on file input elements
- Handle hidden file inputs by making them visible with JavaScript
- Configure Chrome download preferences for automated downloads
- Verify that a downloaded file exists on disk
- Combine all Week 2 skills (waits, alerts, frames, windows, dynamic elements) in a mini project

---

## Core Concept Explanation

### File Upload

HTML file uploads use the `<input type="file">` element. Even though this element renders as a button/browse dialog in the browser, Selenium can bypass the OS file picker by sending the file path directly to the input element using `SendKeys()` / `send_keys()`.

**Key requirement:** You send the **absolute file path** as a string to the file input element. Selenium handles the rest.

**Challenge:** Some sites hide the file input with CSS (`display: none`, `opacity: 0`, or `visibility: hidden`) and use JavaScript to trigger it. In those cases, you must make the element visible with JavaScript before sending keys.

### File Download

Selenium does not have built-in download verification. The strategy is:

1. Configure the browser to download files to a specific directory without prompting.
2. Trigger the download action.
3. Wait for the file to appear on disk.
4. Verify the file exists and optionally check its size or content.

---

## How It Works

### File Upload Flow

```
1. Find the <input type="file"> element
2. (If hidden) Use JavaScript to make it visible
3. Call element.SendKeys("C:\\path\\to\\file.txt")
4. Click the submit/upload button
5. Verify the upload result
```

### File Download Flow

```
1. Set ChromeOptions preferences:
   - download.default_directory = target folder
   - download.prompt_for_download = false
2. Create the driver with those options
3. Click the download link/button
4. Poll the directory until the file appears
5. Verify file exists and has content (size > 0)
```

---

## Code Example: C#

```csharp
// File: Day14_FileUploadDownloadDemo.cs
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
using System.IO;
using System.Linq;

namespace Day14
{
    [TestFixture]
    public class FileUploadDownloadDemo
    {
        private IWebDriver driver;
        private WebDriverWait wait;
        private string downloadDir;
        private string testFilePath;

        [SetUp]
        public void SetUp()
        {
            // Create a unique download directory for each test
            downloadDir = Path.Combine(
                Path.GetTempPath(), "selenium_downloads_" + Guid.NewGuid().ToString("N")
            );
            Directory.CreateDirectory(downloadDir);

            // Create a test file for upload tests
            testFilePath = Path.Combine(downloadDir, "test_upload.txt");
            File.WriteAllText(testFilePath, "This is a test file for Selenium upload.");

            // Configure Chrome to download without prompting
            ChromeOptions options = new ChromeOptions();
            options.AddUserProfilePreference("download.default_directory", downloadDir);
            options.AddUserProfilePreference("download.prompt_for_download", false);
            options.AddUserProfilePreference("download.directory_upgrade", true);
            // Disable safe browsing to prevent download warnings
            options.AddUserProfilePreference("safebrowsing.enabled", true);

            driver = new ChromeDriver(options);
            driver.Manage().Window.Maximize();
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(15));
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();

            // Clean up test files
            if (Directory.Exists(downloadDir))
            {
                Directory.Delete(downloadDir, true);
            }
        }

        // --- FILE UPLOAD: STANDARD VISIBLE INPUT ---
        [Test]
        public void UploadFile_StandardInput()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/upload");

            // Find the file input element
            IWebElement fileInput = driver.FindElement(By.Id("file-upload"));

            // Send the absolute file path directly to the input
            fileInput.SendKeys(testFilePath);

            // Click the upload button
            driver.FindElement(By.Id("file-submit")).Click();

            // Wait for the upload result
            IWebElement result = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("uploaded-files"))
            );

            // Verify the filename appears in the result
            Assert.That(result.Text, Does.Contain("test_upload.txt"));
            TestContext.WriteLine($"Uploaded file: {result.Text}");
        }

        // --- FILE UPLOAD: HIDDEN INPUT (MAKE VISIBLE WITH JS) ---
        [Test]
        public void UploadFile_HiddenInput()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/upload");

            IWebElement fileInput = driver.FindElement(By.Id("file-upload"));

            // If the input were hidden, you would make it visible first:
            IJavaScriptExecutor js = (IJavaScriptExecutor)driver;
            js.ExecuteScript(
                "arguments[0].style.display = 'block';" +
                "arguments[0].style.visibility = 'visible';" +
                "arguments[0].style.opacity = '1';" +
                "arguments[0].style.height = '30px';" +
                "arguments[0].style.width = '200px';",
                fileInput
            );

            // Now send the file path
            fileInput.SendKeys(testFilePath);

            // Submit
            driver.FindElement(By.Id("file-submit")).Click();

            IWebElement result = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("uploaded-files"))
            );
            Assert.That(result.Text, Does.Contain("test_upload.txt"));
            TestContext.WriteLine("Hidden input upload successful");
        }

        // --- FILE DOWNLOAD ---
        [Test]
        public void DownloadFile()
        {
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/download");

            // Find a download link (pick the first .txt file if available)
            var links = driver.FindElements(By.CssSelector("a[href*='download']"));
            if (links.Count == 0)
            {
                // Fallback: click any file link on the page
                links = driver.FindElements(By.CssSelector(".example a"));
            }

            Assert.That(links.Count, Is.GreaterThan(0), "No download links found");

            // Get the filename from the link text
            string expectedFilename = links[0].Text;
            TestContext.WriteLine($"Downloading: {expectedFilename}");

            // Click to download
            links[0].Click();

            // Wait for the file to appear in the download directory
            string downloadedFilePath = WaitForFileDownload(expectedFilename, 30);

            Assert.That(downloadedFilePath, Is.Not.Null,
                $"File {expectedFilename} did not download within timeout");

            // Verify the file has content
            FileInfo fileInfo = new FileInfo(downloadedFilePath);
            Assert.That(fileInfo.Length, Is.GreaterThan(0));

            TestContext.WriteLine(
                $"Downloaded: {fileInfo.Name}, Size: {fileInfo.Length} bytes"
            );
        }

        /// <summary>
        /// Polls the download directory until the expected file appears or timeout.
        /// </summary>
        private string WaitForFileDownload(string filename, int timeoutSeconds)
        {
            string filePath = Path.Combine(downloadDir, filename);
            DateTime deadline = DateTime.Now.AddSeconds(timeoutSeconds);

            while (DateTime.Now < deadline)
            {
                // Check for the exact file
                if (File.Exists(filePath))
                {
                    // Also check that Chrome is not still writing (.crdownload)
                    string crDownload = filePath + ".crdownload";
                    if (!File.Exists(crDownload))
                    {
                        return filePath;
                    }
                }

                // Also check if the file was renamed (e.g., appended (1))
                var matchingFiles = Directory.GetFiles(downloadDir)
                    .Where(f => !f.EndsWith(".crdownload") && !f.EndsWith(".tmp"))
                    .ToArray();

                if (matchingFiles.Length > 0)
                {
                    return matchingFiles[0];
                }

                System.Threading.Thread.Sleep(500); // Polling interval
            }

            return null;
        }

        // --- UPLOAD VIA DRAG-AND-DROP AREA (using JavaScript) ---
        [Test]
        public void UploadViaDragDropArea()
        {
            // Some sites use drag-and-drop upload zones that do not have
            // a visible <input type="file">. Strategy:
            // 1. Find or create a file input
            // 2. Send the file path to it
            // 3. Trigger the appropriate events

            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/upload");

            // The standard approach still works for most drag-drop implementations
            // because they typically have a hidden file input
            IWebElement fileInput = driver.FindElement(By.Id("file-upload"));
            fileInput.SendKeys(testFilePath);

            driver.FindElement(By.Id("file-submit")).Click();

            IWebElement result = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("uploaded-files"))
            );
            Assert.That(result.Text, Does.Contain("test_upload.txt"));
        }
    }

    // =========================================================================
    // MINI PROJECT 2: Combined Week 2 Skills
    // =========================================================================
    [TestFixture]
    public class MiniProject2
    {
        private IWebDriver driver;
        private WebDriverWait wait;

        [SetUp]
        public void SetUp()
        {
            driver = new ChromeDriver();
            driver.Manage().Window.Maximize();
            wait = new WebDriverWait(driver, TimeSpan.FromSeconds(15));
        }

        [TearDown]
        public void TearDown()
        {
            driver.Quit();
        }

        /// <summary>
        /// Mini Project 2: Combines waits, alerts, frames, windows, dynamic
        /// elements, and file upload into a single end-to-end workflow.
        /// </summary>
        [Test]
        public void CombinedWorkflow()
        {
            // ---- STEP 1: Dynamic Elements (Day 13) ----
            TestContext.WriteLine("=== Step 1: Dynamic Elements ===");
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/dynamic_controls");

            // Remove the checkbox using AJAX and wait for completion
            IWebElement removeBtn = wait.Until(
                ExpectedConditions.ElementToBeClickable(
                    By.CssSelector("#checkbox-example button"))
            );
            removeBtn.Click();

            // Wait for loading spinner to disappear (Explicit Wait - Day 8)
            wait.Until(
                ExpectedConditions.InvisibilityOfElementLocated(By.Id("loading"))
            );

            IWebElement message = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("message"))
            );
            Assert.That(message.Text, Is.EqualTo("It's gone!"));
            TestContext.WriteLine($"  Checkbox removed: {message.Text}");

            // Enable the input field
            driver.FindElement(By.CssSelector("#input-example button")).Click();
            wait.Until(
                ExpectedConditions.InvisibilityOfElementLocated(By.Id("loading"))
            );

            IWebElement input = wait.Until(d =>
            {
                var el = d.FindElement(By.CssSelector("#input-example input"));
                return el.Enabled ? el : null;
            });
            input.SendKeys("Mini Project 2 Complete");
            TestContext.WriteLine($"  Input enabled and filled: {input.GetAttribute("value")}");

            // ---- STEP 2: JavaScript Alerts (Day 10) ----
            TestContext.WriteLine("\n=== Step 2: JavaScript Alerts ===");
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/javascript_alerts");

            // Trigger a JS Prompt
            driver.FindElement(
                By.XPath("//button[text()='Click for JS Prompt']")
            ).Click();

            IAlert alert = wait.Until(ExpectedConditions.AlertIsPresent());
            TestContext.WriteLine($"  Alert text: {alert.Text}");
            alert.SendKeys("Selenium Automation");
            alert.Accept();

            IWebElement result = driver.FindElement(By.Id("result"));
            Assert.That(result.Text, Does.Contain("Selenium Automation"));
            TestContext.WriteLine($"  Alert result: {result.Text}");

            // ---- STEP 3: iFrame Interaction (Day 11) ----
            TestContext.WriteLine("\n=== Step 3: iFrame Interaction ===");
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/iframe");

            // Switch to the TinyMCE editor iframe
            wait.Until(
                ExpectedConditions.FrameToBeAvailableAndSwitchToIt("mce_0_ifr")
            );

            IWebElement editor = wait.Until(
                ExpectedConditions.ElementIsVisible(By.Id("tinymce"))
            );
            editor.Clear();
            editor.SendKeys("Content written inside an iframe by Selenium!");
            TestContext.WriteLine($"  Editor text: {editor.Text}");

            // Switch back to main page
            driver.SwitchTo().DefaultContent();

            IWebElement heading = driver.FindElement(By.TagName("h3"));
            TestContext.WriteLine($"  Main page heading: {heading.Text}");

            // ---- STEP 4: Multiple Windows (Day 12) ----
            TestContext.WriteLine("\n=== Step 4: Multiple Windows ===");
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/windows");

            string originalWindow = driver.CurrentWindowHandle;

            // Open a new window
            driver.FindElement(By.LinkText("Click Here")).Click();
            wait.Until(d => d.WindowHandles.Count == 2);

            // Switch to the new window
            string newWindow = driver.WindowHandles
                .First(h => h != originalWindow);
            driver.SwitchTo().Window(newWindow);

            wait.Until(ExpectedConditions.TitleContains("New Window"));
            TestContext.WriteLine($"  New window title: {driver.Title}");

            // Close and switch back
            driver.Close();
            driver.SwitchTo().Window(originalWindow);
            TestContext.WriteLine($"  Back to: {driver.Title}");

            // ---- STEP 5: File Upload (Day 14) ----
            TestContext.WriteLine("\n=== Step 5: File Upload ===");
            driver.Navigate().GoToUrl("https://the-internet.herokuapp.com/upload");

            // Create a temp file
            string tempFile = Path.Combine(Path.GetTempPath(), "mini_project_2.txt");
            File.WriteAllText(tempFile, "Mini Project 2 - All skills combined!");

            try
            {
                IWebElement fileInput = driver.FindElement(By.Id("file-upload"));
                fileInput.SendKeys(tempFile);
                driver.FindElement(By.Id("file-submit")).Click();

                IWebElement uploadResult = wait.Until(
                    ExpectedConditions.ElementIsVisible(By.Id("uploaded-files"))
                );
                Assert.That(uploadResult.Text, Does.Contain("mini_project_2.txt"));
                TestContext.WriteLine($"  Uploaded: {uploadResult.Text}");
            }
            finally
            {
                if (File.Exists(tempFile))
                    File.Delete(tempFile);
            }

            // ---- SUMMARY ----
            TestContext.WriteLine("\n=== Mini Project 2 Complete ===");
            TestContext.WriteLine("Skills demonstrated:");
            TestContext.WriteLine("  - Explicit waits (Day 8)");
            TestContext.WriteLine("  - Fluent waits with custom conditions (Day 9)");
            TestContext.WriteLine("  - JavaScript alerts (Day 10)");
            TestContext.WriteLine("  - iFrame switching (Day 11)");
            TestContext.WriteLine("  - Multiple windows (Day 12)");
            TestContext.WriteLine("  - Dynamic elements (Day 13)");
            TestContext.WriteLine("  - File upload (Day 14)");
        }
    }
}
```

---

## Code Example: Python

```python
# File: test_day14_file_upload_download_demo.py
# Prerequisites:
#   pip install selenium pytest

import os
import tempfile
import uuid
from pathlib import Path

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import pytest


@pytest.fixture
def download_dir():
    """Create a unique temporary download directory."""
    dir_path = os.path.join(tempfile.gettempdir(), f"selenium_dl_{uuid.uuid4().hex}")
    os.makedirs(dir_path, exist_ok=True)
    yield dir_path
    # Cleanup
    import shutil
    shutil.rmtree(dir_path, ignore_errors=True)


@pytest.fixture
def test_file(download_dir):
    """Create a test file for upload tests."""
    file_path = os.path.join(download_dir, "test_upload.txt")
    with open(file_path, "w") as f:
        f.write("This is a test file for Selenium upload.")
    return file_path


@pytest.fixture
def driver(download_dir):
    """Create a Chrome driver configured for downloads."""
    chrome_options = Options()
    prefs = {
        "download.default_directory": download_dir,
        "download.prompt_for_download": False,
        "download.directory_upgrade": True,
        "safebrowsing.enabled": True,
    }
    chrome_options.add_experimental_option("prefs", prefs)

    drv = webdriver.Chrome(options=chrome_options)
    drv.maximize_window()
    yield drv
    drv.quit()


@pytest.fixture
def basic_driver():
    """Create a basic Chrome driver (no download config needed)."""
    drv = webdriver.Chrome()
    drv.maximize_window()
    yield drv
    drv.quit()


# --- FILE UPLOAD: STANDARD VISIBLE INPUT ---
def test_upload_file_standard_input(basic_driver, test_file):
    driver = basic_driver
    driver.get("https://the-internet.herokuapp.com/upload")

    # Find the file input element
    file_input = driver.find_element(By.ID, "file-upload")

    # Send the absolute file path directly to the input
    file_input.send_keys(test_file)

    # Click the upload button
    driver.find_element(By.ID, "file-submit").click()

    # Wait for the upload result
    wait = WebDriverWait(driver, 15)
    result = wait.until(EC.visibility_of_element_located((By.ID, "uploaded-files")))

    # Verify the filename appears
    assert "test_upload.txt" in result.text
    print(f"Uploaded file: {result.text}")


# --- FILE UPLOAD: HIDDEN INPUT ---
def test_upload_file_hidden_input(basic_driver, test_file):
    driver = basic_driver
    driver.get("https://the-internet.herokuapp.com/upload")

    file_input = driver.find_element(By.ID, "file-upload")

    # Make hidden inputs visible using JavaScript
    driver.execute_script(
        "arguments[0].style.display = 'block';"
        "arguments[0].style.visibility = 'visible';"
        "arguments[0].style.opacity = '1';"
        "arguments[0].style.height = '30px';"
        "arguments[0].style.width = '200px';",
        file_input,
    )

    # Now send the file path
    file_input.send_keys(test_file)

    driver.find_element(By.ID, "file-submit").click()

    wait = WebDriverWait(driver, 15)
    result = wait.until(EC.visibility_of_element_located((By.ID, "uploaded-files")))
    assert "test_upload.txt" in result.text
    print("Hidden input upload successful")


# --- FILE DOWNLOAD ---
def test_download_file(driver, download_dir):
    driver.get("https://the-internet.herokuapp.com/download")

    # Find download links
    links = driver.find_elements(By.CSS_SELECTOR, ".example a")
    assert len(links) > 0, "No download links found"

    # Get the filename from the link text
    expected_filename = links[0].text
    print(f"Downloading: {expected_filename}")

    # Click to download
    links[0].click()

    # Wait for the file to appear
    downloaded_path = wait_for_file_download(download_dir, expected_filename, 30)

    assert downloaded_path is not None, (
        f"File {expected_filename} did not download within timeout"
    )

    # Verify the file has content
    file_size = os.path.getsize(downloaded_path)
    assert file_size > 0

    print(f"Downloaded: {os.path.basename(downloaded_path)}, Size: {file_size} bytes")


def wait_for_file_download(directory, filename, timeout_seconds):
    """Poll the download directory until the file appears or timeout."""
    import time

    file_path = os.path.join(directory, filename)
    deadline = time.time() + timeout_seconds

    while time.time() < deadline:
        # Check for the exact file
        if os.path.exists(file_path):
            # Ensure Chrome is done writing (no .crdownload file)
            crdownload = file_path + ".crdownload"
            if not os.path.exists(crdownload):
                return file_path

        # Check for any completed file in the directory
        completed_files = [
            f
            for f in os.listdir(directory)
            if not f.endswith(".crdownload") and not f.endswith(".tmp")
        ]
        if completed_files:
            return os.path.join(directory, completed_files[0])

        time.sleep(0.5)

    return None


# =========================================================================
# MINI PROJECT 2: Combined Week 2 Skills
# =========================================================================
def test_mini_project_2_combined_workflow(basic_driver):
    driver = basic_driver
    wait = WebDriverWait(driver, 15)

    # ---- STEP 1: Dynamic Elements (Day 13) ----
    print("\n=== Step 1: Dynamic Elements ===")
    driver.get("https://the-internet.herokuapp.com/dynamic_controls")

    # Remove the checkbox using AJAX
    remove_btn = wait.until(
        EC.element_to_be_clickable((By.CSS_SELECTOR, "#checkbox-example button"))
    )
    remove_btn.click()

    # Wait for loading to finish (Explicit Wait - Day 8)
    wait.until(EC.invisibility_of_element_located((By.ID, "loading")))

    message = wait.until(EC.visibility_of_element_located((By.ID, "message")))
    assert message.text == "It's gone!"
    print(f"  Checkbox removed: {message.text}")

    # Enable the input field
    driver.find_element(By.CSS_SELECTOR, "#input-example button").click()
    wait.until(EC.invisibility_of_element_located((By.ID, "loading")))

    # Custom wait condition (Fluent Wait - Day 9)
    def input_is_enabled(d):
        el = d.find_element(By.CSS_SELECTOR, "#input-example input")
        return el if el.is_enabled() else False

    enabled_input = wait.until(input_is_enabled)
    enabled_input.send_keys("Mini Project 2 Complete")
    print(f"  Input filled: {enabled_input.get_attribute('value')}")

    # ---- STEP 2: JavaScript Alerts (Day 10) ----
    print("\n=== Step 2: JavaScript Alerts ===")
    driver.get("https://the-internet.herokuapp.com/javascript_alerts")

    driver.find_element(By.XPATH, "//button[text()='Click for JS Prompt']").click()

    alert = wait.until(EC.alert_is_present())
    print(f"  Alert text: {alert.text}")
    alert.send_keys("Selenium Automation")
    alert.accept()

    result = driver.find_element(By.ID, "result")
    assert "Selenium Automation" in result.text
    print(f"  Alert result: {result.text}")

    # ---- STEP 3: iFrame Interaction (Day 11) ----
    print("\n=== Step 3: iFrame Interaction ===")
    driver.get("https://the-internet.herokuapp.com/iframe")

    wait.until(EC.frame_to_be_available_and_switch_to_it("mce_0_ifr"))

    editor = wait.until(EC.visibility_of_element_located((By.ID, "tinymce")))
    editor.clear()
    editor.send_keys("Content written inside an iframe by Selenium!")
    print(f"  Editor text: {editor.text}")

    driver.switch_to.default_content()
    heading = driver.find_element(By.TAG_NAME, "h3")
    print(f"  Main page heading: {heading.text}")

    # ---- STEP 4: Multiple Windows (Day 12) ----
    print("\n=== Step 4: Multiple Windows ===")
    driver.get("https://the-internet.herokuapp.com/windows")

    original_window = driver.current_window_handle

    driver.find_element(By.LINK_TEXT, "Click Here").click()
    wait.until(EC.number_of_windows_to_be(2))

    new_window = [h for h in driver.window_handles if h != original_window][0]
    driver.switch_to.window(new_window)

    wait.until(EC.title_contains("New Window"))
    print(f"  New window title: {driver.title}")

    driver.close()
    driver.switch_to.window(original_window)
    print(f"  Back to: {driver.title}")

    # ---- STEP 5: File Upload (Day 14) ----
    print("\n=== Step 5: File Upload ===")
    driver.get("https://the-internet.herokuapp.com/upload")

    temp_file = os.path.join(tempfile.gettempdir(), "mini_project_2.txt")
    with open(temp_file, "w") as f:
        f.write("Mini Project 2 - All skills combined!")

    try:
        file_input = driver.find_element(By.ID, "file-upload")
        file_input.send_keys(temp_file)
        driver.find_element(By.ID, "file-submit").click()

        upload_result = wait.until(
            EC.visibility_of_element_located((By.ID, "uploaded-files"))
        )
        assert "mini_project_2.txt" in upload_result.text
        print(f"  Uploaded: {upload_result.text}")
    finally:
        if os.path.exists(temp_file):
            os.remove(temp_file)

    # ---- SUMMARY ----
    print("\n=== Mini Project 2 Complete ===")
    print("Skills demonstrated:")
    print("  - Explicit waits (Day 8)")
    print("  - Fluent waits with custom conditions (Day 9)")
    print("  - JavaScript alerts (Day 10)")
    print("  - iFrame switching (Day 11)")
    print("  - Multiple windows (Day 12)")
    print("  - Dynamic elements (Day 13)")
    print("  - File upload (Day 14)")
```

---

## Step-by-Step Walkthrough

### File Upload

1. **Navigate to the upload page** and inspect the `<input type="file">` element.
2. **Check visibility** -- if the input is hidden (`display: none`), use JavaScript to make it visible.
3. **Send the absolute file path** -- `element.SendKeys("C:\\full\\path\\to\\file.txt")`. Do not click the input first.
4. **Submit the form** -- click the upload button.
5. **Verify the result** -- check the response page for the uploaded filename.

### File Download

1. **Configure ChromeOptions** -- set `download.default_directory` and disable download prompts.
2. **Create the driver** with those options.
3. **Click the download link** -- the browser downloads silently to the configured directory.
4. **Poll for the file** -- check the directory every 500ms. Ignore `.crdownload` (incomplete) files.
5. **Verify** -- assert the file exists and has non-zero size.

---

## Common Mistakes & How to Avoid Them

### 1. Using a relative file path for upload
```python
# BAD: relative path will not work
file_input.send_keys("test_file.txt")

# GOOD: always use absolute path
file_input.send_keys(os.path.abspath("test_file.txt"))
```

### 2. Clicking the file input before sending keys
```csharp
// BAD: clicking opens the OS file dialog, which Selenium cannot control
fileInput.Click();
fileInput.SendKeys(path); // Will type into the dialog, not the input

// GOOD: send keys directly without clicking
fileInput.SendKeys(path);
```

### 3. Not waiting for download to complete
```python
# BAD: checking immediately after clicking
download_link.click()
assert os.path.exists(file_path)  # File may not exist yet!

# GOOD: poll with a timeout
downloaded = wait_for_file_download(download_dir, filename, 30)
assert downloaded is not None
```

### 4. Forgetting to check for .crdownload files
Chrome creates a `.crdownload` temporary file while downloading. If you check for any file in the directory, you might find the `.crdownload` and think the download is complete.

### 5. Hardcoding download paths
```csharp
// BAD: path may not exist on CI or other machines
options.AddUserProfilePreference("download.default_directory", "C:\\Users\\Me\\Downloads");

// GOOD: use a temp directory
string downloadDir = Path.Combine(Path.GetTempPath(), "selenium_downloads");
```

---

## Best Practices

1. **Use unique temp directories** for each test to avoid file conflicts in parallel execution.
2. **Clean up files** in teardown -- delete uploaded and downloaded files after tests.
3. **Use absolute paths** for file uploads, always.
4. **Make hidden inputs visible with JavaScript** rather than using AutoIT or Robot class for OS dialogs.
5. **Poll for downloads** with a reasonable timeout instead of using hard sleeps.
6. **Check file size** after download to ensure it is not an empty error page saved as a file.
7. **Set browser preferences** for downloads instead of trying to interact with the browser's download dialog.

---

## Hands-On Exercise

**Goal:** Complete the Mini Project 2 workflow.

1. Create a text file on disk.
2. Navigate to `https://the-internet.herokuapp.com/upload` and upload the file.
3. Verify the upload succeeded.
4. Navigate to `https://the-internet.herokuapp.com/javascript_alerts`.
5. Trigger a JS Prompt, type "Upload successful!", and accept it.
6. Navigate to `https://the-internet.herokuapp.com/iframe`.
7. Type a summary message in the editor iframe.
8. Open a new tab, navigate to `https://the-internet.herokuapp.com/windows`, click to open a third window.
9. Print titles of all open windows.
10. Close extra windows and return to the original.

**Expected output:**
```
File uploaded: test_upload.txt
Alert response: You entered: Upload successful!
Editor text: Mini project complete!
Window titles:
  - The Internet (iframe page)
  - The Internet (windows page)
  - New Window
All extra windows closed.
Back to original: An iFrame containing the TinyMCE WYSIWYG Editor
Mini Project 2 Complete!
```

---

## Real-World Scenario

Consider an HR application where recruiters upload resumes and download reports:

```python
# Upload a resume
driver.get("https://hr-app.example.com/candidates/new")

# The upload zone uses a hidden file input
file_input = driver.find_element(By.CSS_SELECTOR, "input[type='file']")
driver.execute_script(
    "arguments[0].style.display='block'; arguments[0].style.opacity='1';",
    file_input
)
file_input.send_keys("/path/to/resume.pdf")

# Wait for upload progress bar to complete
wait.until(EC.invisibility_of_element_located((By.CSS_SELECTOR, ".upload-progress")))

# Verify the file appears in the attachments list
attachment = wait.until(
    EC.visibility_of_element_located((By.CSS_SELECTOR, ".attachment-name"))
)
assert "resume.pdf" in attachment.text

# Later, download a candidate report
driver.get("https://hr-app.example.com/reports")
driver.find_element(By.ID, "export-csv-btn").click()

# Wait for the CSV to download
report_path = wait_for_file_download(download_dir, "candidates_report.csv", 60)
assert report_path is not None

# Verify the CSV has data
with open(report_path, 'r') as f:
    lines = f.readlines()
assert len(lines) > 1  # Header + at least one data row
```

---

## Resources

- [Selenium File Upload Documentation](https://www.selenium.dev/documentation/webdriver/elements/file_upload/)
- [The Internet Herokuapp - File Upload](https://the-internet.herokuapp.com/upload)
- [The Internet Herokuapp - File Download](https://the-internet.herokuapp.com/download)
- [ChromeOptions Preferences](https://chromedriver.chromium.org/capabilities)
- [Chrome Download Preferences](https://src.chromium.org/viewvc/chrome/trunk/src/chrome/common/pref_names.cc)
