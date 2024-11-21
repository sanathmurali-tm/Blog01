# Automating Screenshots and Video Recording in Test Automation Frameworks with Selective Disabling

Capturing screenshots and video recordings during test execution is essential for debugging and improving the quality of test reports. However, there may be situations where you don't want to capture these resources for specific test cases. In this blog, we will explore how to selectively disable screenshots and video recordings for specific test cases using a centralized configuration file (`config.json`). This solution will be implemented for **Robot Framework, Pytest, and WebdriverIO**.

---

## **1. Robot Framework (Python)**

In Robot Framework, we can use a custom listener to capture screenshots and video recordings after each test. To make this process flexible, we load the test configuration from a `config.json` file, which allows us to disable screenshots or video recordings for specific test cases.

### **Selective Disabling of Screenshots and Video Recordings in Robot Framework**

We will define a custom listener that reads the configuration file and conditionally takes screenshots and records videos for specific test cases. If a test case is listed in the configuration file, we will skip the screenshot or video recording for that test case.

**Code for Screenshot Automation in Robot Framework :**

```python
import json
import os
from robot.libraries.BuiltIn import BuiltIn
from robot.api import logger
import re
import AllureLibrary

class EndUserKeywordScreenshot:
    ROBOT_LISTENER_API_VERSION = 3

    def __init__(self):
        # Load configuration
        self.config = self.load_config()
        self.disable_screenshots = set(self.config.get("disable_screenshots", []))

    def load_config(self):
        """Load configuration from a JSON file."""
        config_path = os.path.join(os.getcwd(), 'tests/Config/config.json')
        try:
            with open(config_path, 'r') as file:
                return json.load(file)
        except FileNotFoundError:
            logger.warn(f"Could not load config.json: File not found at {config_path}")
        except Exception as e:
            logger.warn(f"Error loading config.json: {e}")
        return {}

    def sanitize_filename(self, name):
        """Sanitize a string to make it safe for filenames."""
        return re.sub(r'[<>:"/\\|?*]', '_', name)

    def end_user_keyword(self, data, implementation, result):
        """Executed after each user keyword ends."""
        try:
            # Get the name of the currently running test case
            test_name = BuiltIn().get_variable_value("${TEST NAME}")

            # Check if screenshots are disabled for this test case
            if test_name in self.disable_screenshots:
                logger.info(f"Skipping screenshot for test case: {test_name}")
                return  # Skip taking screenshot

            # Capture screenshot if not disabled
            output_dir = os.path.join(os.getcwd(), 'tests/Artifacts/screenshots')
            if not os.path.exists(output_dir):
                os.makedirs(output_dir)

            sanitized_data = self.sanitize_filename(str(data))
            screenshot_path = os.path.join(output_dir, f'screenshot_{sanitized_data}.png')
            self.browserLibrary = BuiltIn().get_library_instance('Browser')
            self.browserLibrary.take_screenshot(filename=screenshot_path)
            try:
                AllureLibrary.attach_file(source=screenshot_path, name='screenshot', attachment_type='PNG')
            except Exception as e:
                print(f"Failed to attach screenshot: {e}")
        except Exception as e:
            logger.warn(f"Failed to take screenshot: {e}")
```

### **Selective Disabling of Video Recordings in Robot Framework**

Similarly, we can control video recording using a custom listener or keyword in Robot Framework. By checking against the configuration file, we can decide whether or not to record video for a specific test case.

```python
class EndUserKeywordVideoRecording:
    def __init__(self):
        self.config = self.load_config()
        self.disable_video = set(self.config.get("disable_screen_recording", []))

    def start_video_recording(self, test_name):
        """Check if video recording should be enabled."""
        if test_name in self.disable_video:
            print(f"Skipping video recording for: {test_name}")
            return False  # Skip video recording
        return True  # Enable video recording
```

### **Sample `config.json` for Robot Framework :**

Robot Framework uses keywords to manage video recording. Using the same `config.json`, you can enable or disable video recording for specific test cases.

```json
{
  "disable_screenshots": ["Verify user is able to filter product by brand"],
  "disable_screen_recording": ["Verify pop up window is displayed"]
}
```

---

## **2. Pytest Framework(Python)**

In Pytest, we can use fixtures to capture screenshots and video recordings. The configuration is loaded from the `config.json` file, which allows for disabling these features for specific test cases.

### **Selective Disabling of Screenshots and Video Recordings in Pytest**

1. **Screenshot Automation :** Use a fixture to capture screenshots on failure while respecting the configuration settings to disable screenshots for specific tests.
2. **Video Recording Automation :** Similarly, use a fixture to control video recording based on the test case configuration.

**Code for Screenshot Automation in Pytest :**

```python
import pytest
import json
from allure_commons.types import AttachmentType
import allure

# Load configuration from JSON file
def load_config():
    with open('config.json', 'r') as f:
        return json.load(f)

config = load_config()

@pytest.fixture(autouse=True)
def capture_screenshot_on_failure(page, request):
    # Get the test name from the request object
    test_name = request.node.name
    disable_screenshots = config.get('disable_screenshots', [])

    # Check if screenshots are disabled for this test case
    if test_name in disable_screenshots:
        yield
        return  # Skip screenshot capture

    # Capture screenshot if the test fails
    yield
    if request.node.rep_call.failed:
        screenshot_path = f'screenshots/{test_name}.png'
        page.screenshot(path=screenshot_path)
        allure.attach.file(screenshot_path, name="Screenshot", attachment_type=AttachmentType.PNG)
```

### **Video Recording Automation in Pytest**

```python
@pytest.fixture(scope="function")
def context_with_video(browser):
    # Load config to check if video recording is enabled or disabled
    disable_video = config.get("disable_screen_recording", [])
    test_name = pytest.current_test.name

    if test_name in disable_video:
        # Skip video recording for this test case
        yield browser.new_context()
        return

    # Enable video recording for the test
    context = browser.new_context(record_video_dir="videos/")
    yield context
    context.close()  # Ensure the video is saved when the context is closed
```

---

## **3. WebdriverIO (JavaScript)**

In WebdriverIO, we use hooks to manage the screenshot and video recording process. By reading the `config.json` file, we can control whether screenshots or video recordings should be captured for specific test cases.

### **Selective Disabling of Screenshots and Video Recordings in WebdriverIO**

1. **Screenshot Automation :** Use the afterTest hook to conditionally skip screenshots based on the test name.
2. **Video Recording Automation :** Control video recording in the afterSuite hook, enabling or disabling based on the suite name.

**Code Example for Screenshot Automation in WebdriverIO :**

```javascript
const fs = require("fs");
const allure = require("allure-commandline");

exports.config = {
  afterTest: async function (
    test,
    context,
    { error, result, duration, passed }
  ) {
    const config = require("./config.json");
    const disabledScreenshots = config.disable_screenshots || [];

    // Skip screenshot if the test name is in the disable_screenshots list
    if (disabledScreenshots.includes(test.title)) {
      console.log(`Skipping screenshot for: ${test.title}`);
      return;
    }

    // Take screenshot if the test fails
    if (!passed) {
      const screenshotPath = `./screenshots/${test.title}.png`;
      await browser.saveScreenshot(screenshotPath);
      allure.addAttachment(
        "Screenshot",
        Buffer.from(fs.readFileSync(screenshotPath)),
        "image/png"
      );
    }
  },

  afterSuite: function (suite) {
    const config = require("./config.json");
    const disabledVideoRecording = config.disable_screen_recording || [];

    // Skip video recording for this suite
    if (disabledVideoRecording.includes(suite.name)) {
      console.log(`Skipping video recording for suite: ${suite.name}`);
      // Add logic to skip video recording if needed
    }
  },
};
```

### **Video Recording Automation in WebdriverIO**

```javascript
const Video = require("wdio-video-reporter");

exports.config = {
  reporters: [
    [
      Video,
      {
        saveAllVideos: false, // Only save videos for failed tests
        videoSlowdownMultiplier: 3, // Slows video for better visibility
      },
    ],
  ],

  afterSuite: function (suite) {
    const config = require("./config.json");
    const disabledVideoRecording = config.disable_screen_recording || [];

    // Skip video recording for this suite
    if (disabledVideoRecording.includes(suite.name)) {
      console.log(`Skipping video recording for suite: ${suite.name}`);
    }
  },
};
```

---

## **4. Configuration File (`config.json`)**

The config.json file is the central configuration file that allows controlling the behavior of screenshots and video recordings for specific test cases.

**Sample `config.json` :**

```json
{
  "recordVideo": true,
  "disable_screenshots": [
    "Verify user is able to filter product by brand",
    "Verify user can submit a bug report"
  ],
  "disable_screen_recording": ["Verify pop up window is displayed"]
}
```

In this configuration file:

- `disable_screenshots` : A list of test case names where screenshots should not be captured.
- `disable_screen_recording` : A list of test suite names where video recordings should not be captured.

---

## **Conclusion**

By using a centralized configuration file (`config.json`), we can selectively disable screenshots and video recordings for specific test cases across **Robot Framework, Pytest, and WebdriverIO**. This approach enhances flexibility, reduces unnecessary resource usage, and helps streamline test reporting based on specific needs or circumstances.
